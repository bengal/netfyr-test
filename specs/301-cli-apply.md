# SPEC-301: CLI Apply Command

## What
Implement the `netfyr apply <path>...` CLI command. This command loads policy definitions from YAML files or directories, and either applies them directly (daemon-free mode) or submits them to the daemon (daemon mode). It supports a `--dry-run` flag to preview changes without applying.

### Two runtime modes

The CLI automatically detects which mode to use:

1. **Daemon-free mode** (daemon not running): The CLI loads policies, runs the reconciliation engine, queries the current system state via the backend, generates a diff, and applies the changes directly. Only static policies are supported -- if any DHCP policy is present, the CLI fails with an error instructing the user to start the daemon.

2. **Daemon mode** (daemon running): The CLI loads policies from YAML files and submits them to the daemon via Varlink (SPEC-404). The daemon reconciles all policies (static + DHCP), applies the effective state, and returns the result. Replace-all semantics: each `netfyr apply` replaces the daemon's entire policy set.

Mode detection: the CLI attempts to connect to the Varlink socket at `/run/netfyr/netfyr.sock`. If the connection succeeds, daemon mode is used. If it fails (socket not found or connection refused), daemon-free mode is used.

## Why
`netfyr apply` is the primary user-facing command and the core workflow. It implements a simple mental model: "make the system look like this." The two-mode design keeps simple cases simple (no daemon needed for static configs on servers) while supporting dynamic factories (DHCP) when the daemon is running. The auto-detection means users always run the same command -- `netfyr apply` does the right thing.

## User interaction

### Basic usage
```bash
# Apply a single policy file
netfyr apply /etc/netfyr/policies/eth0.yaml

# Apply all policies in a directory
netfyr apply /etc/netfyr/policies/

# Apply multiple paths
netfyr apply /etc/netfyr/policies/eth0.yaml /etc/netfyr/policies/dns.yaml

# Preview changes without applying
netfyr apply --dry-run /etc/netfyr/policies/
```

### Output format

```
# Successful apply (daemon-free mode):
Applied 3 changes (2 entities modified, 1 entity added).
  ~ ethernet eth0: mtu 1500 -> 9000
  ~ ethernet eth0: added address 10.0.1.51/24
  + ethernet eth1: created

# No changes needed:
No changes needed. System is already in desired state.

# DHCP policy without daemon:
Error: policy "eth0-dhcp" uses factory "dhcpv4" which requires the netfyr daemon.
Start the daemon with: systemctl start netfyr

# Daemon mode:
Submitted 3 policies to daemon. Applied 2 changes.
  ~ ethernet eth0: mtu 1500 -> 9000
  + ethernet eth1: created (DHCP lease acquired: 10.0.1.50/24)

# Partial failure:
Applied 2 of 3 changes. 1 failed.
  ~ ethernet eth0: mtu 1500 -> 9000
  ~ ethernet eth0: added address 10.0.1.51/24
  x ethernet eth99: failed -- interface not found

# Conflicts detected:
Warning: 1 field conflict detected. Conflicting fields were not applied.
  ethernet eth0 mtu: policy "team-a" sets 9000, policy "team-b" sets 1500 (both priority 100)
Applied 2 changes (1 entity modified).
  ~ ethernet eth0: added address 10.0.1.51/24
  ~ dns global: added server 10.0.1.2

# Dry-run:
Dry run: 2 changes would be applied.
  ~ ethernet eth0: mtu 1500 -> 9000
  + ethernet eth1: mtu 1500, addresses [10.0.1.50/24]
```

### Exit codes
- `0`: All operations succeeded (or no changes needed).
- `1`: Partial failure (some operations succeeded, some failed) or conflicts detected.
- `2`: Total failure or fatal error (e.g., cannot parse YAML, cannot connect to netlink, DHCP policy without daemon).

## Implementation details
- Crate: `netfyr-cli`
- Files: `src/main.rs`, `src/apply.rs`
- Dependencies (external crates): `clap` (CLI argument parsing), `tokio` (async runtime), `anyhow` (error handling), `colored` (terminal colors)

### CLI argument parsing (`src/main.rs`)

```rust
#[derive(Parser)]
#[command(name = "netfyr", about = "Declarative Linux network configuration")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Apply network policies to the system
    Apply(ApplyArgs),
    /// Query current system network state
    Query(QueryArgs),
}

#[derive(Args)]
struct ApplyArgs {
    /// Paths to YAML files or directories containing policies
    #[arg(required = true)]
    paths: Vec<PathBuf>,

    /// Show what would change without applying
    #[arg(long)]
    dry_run: bool,
}
```

### Apply flow (`src/apply.rs`)

```rust
pub async fn run_apply(args: ApplyArgs) -> Result<ExitCode> {
    // 1. Load policies from paths
    let policy_set = load_policies(&args.paths)?;

    // 2. Detect mode: try connecting to daemon via Varlink
    if let Ok(client) = varlink_connect("/run/netfyr/netfyr.sock") {
        // Daemon mode
        return run_apply_daemon(client, policy_set, args.dry_run).await;
    }

    // Daemon-free mode
    // 3. Check for non-static policies
    if policy_set.iter().any(|p| p.factory_type != FactoryType::Static) {
        let dhcp_policies: Vec<_> = policy_set.iter()
            .filter(|p| p.factory_type != FactoryType::Static)
            .map(|p| &p.name)
            .collect();
        return Err(anyhow!(
            "policies {:?} use non-static factories which require the netfyr daemon.\n\
             Start the daemon with: systemctl start netfyr",
            dhcp_policies
        ));
    }

    // 4. Run reconciliation (static only)
    let reconciliation_result = reconcile(policy_set)?;

    // 5. Report conflicts (if any)
    if !reconciliation_result.conflicts.is_empty() {
        report_conflicts(&reconciliation_result.conflicts);
    }

    // 6. Query current system state
    let registry = create_backend_registry();
    let actual_state = registry.query_all().await?;

    // 7. Generate diff
    let diff = generate_diff(&reconciliation_result.effective_state, &actual_state);

    // 8. If dry-run, display and exit
    if args.dry_run {
        display_diff_report(&diff);
        let code = if diff.is_empty() { 0 } else { 1 };
        return Ok(ExitCode::from(code));
    }

    // 9. If no changes, exit
    if diff.is_empty() {
        println!("No changes needed. System is already in desired state.");
        return Ok(ExitCode::from(0));
    }

    // 10. Apply
    let report = registry.apply(&diff).await?;

    // 11. Display results
    display_apply_report(&report);

    // 12. Determine exit code
    Ok(exit_code_from_report(&report, &reconciliation_result.conflicts))
}

async fn run_apply_daemon(
    client: VarlinkClient,
    policy_set: PolicySet,
    dry_run: bool,
) -> Result<ExitCode> {
    if dry_run {
        let diff = client.dry_run(policy_set).await?;
        display_diff_report(&diff);
        let code = if diff.is_empty() { 0 } else { 1 };
        return Ok(ExitCode::from(code));
    }

    let result = client.submit_policies(policy_set).await?;
    display_apply_report(&result);
    Ok(exit_code_from_report(&result, &[]))
}
```

### Policy loading

1. For each path in `args.paths`:
   - If it's a file: parse the YAML file.
   - If it's a directory: recursively find all `.yaml` and `.yml` files and parse each.
2. For each YAML document:
   - If it has `kind: policy`: parse as a Policy definition.
   - If it has no `kind` field (bare state): auto-wrap into a static policy with default priority (100). The policy name is derived from the filename (without extension).
3. Multi-document YAML files (separated by `---`) produce multiple policies.

### Error handling

- YAML parse errors: print the file path, line number, and error message. Exit code 2.
- Reconciliation errors: print the error. Exit code 2.
- Backend connection failure: print the error. Exit code 2.
- DHCP policy without daemon: print the error with instructions to start daemon. Exit code 2.
- Per-operation failures: continue and report. Exit code 1 (partial) or 2 (total failure).

## Depends on
- SPEC-007 (Policy types and parsing)
- SPEC-008 (Bare state shorthand auto-wrapping)
- SPEC-103 (Backend apply implementation)
- SPEC-201 (Reconciliation engine)
- SPEC-203 (Diff generation)
- SPEC-404 (Varlink API for daemon mode)

## Integration test infrastructure
Integration tests are shell scripts in `tests/`. Each script uses `unshare --user --net` to create an unprivileged network namespace, sets up veth pairs, writes YAML policy files, runs `netfyr apply`, and verifies the kernel state with `ip` commands. No root required.

Shell test script rules (from SPEC-001):
- **Naming**: `301-description.sh` — the Makefile discovers tests via `tests/[0-9]*.sh`.
- **No skip**: if a prerequisite is missing (binary, `unshare`, `dnsmasq`), the script must `echo "FAIL: ..." >&2; exit 1`. Never `exit 0` on failure. Never wrap `netns_setup` or `start_dnsmasq` with `|| exit 0`.
- **Binary path**: locate binaries via `SCRIPT_DIR/../target/debug/netfyr` (overridable with `NETFYR_BIN` env var). Check with `[[ ! -x "$NETFYR_BIN" ]]` and `exit 1` if missing.
- **Helpers**: source `tests/helpers.sh` which provides `netns_setup`, `create_veth`, `add_address`, `start_dnsmasq`, `cleanup`, and assertion functions.

## Verification
In addition to `cargo test` and `cargo clippy`, run `make integration-test` to execute all shell integration test scripts. All tests must pass. This is a required verification step — the story is not complete until shell tests pass.

## Acceptance criteria
```gherkin
Feature: netfyr apply CLI command (daemon-free mode)
  Scenario: Apply a single YAML policy file
    Given a valid YAML file "/tmp/eth0.yaml" defining ethernet/eth0 with mtu=1500
    And the system interface eth0 exists with mtu=1500
    When the user runs "netfyr apply /tmp/eth0.yaml"
    Then the output says "No changes needed. System is already in desired state."
    And the exit code is 0

  Scenario: Apply a YAML file with changes needed
    Given a valid YAML file defining ethernet/eth0 with mtu=9000
    And the system interface eth0 has mtu=1500
    When the user runs "netfyr apply /tmp/eth0.yaml"
    Then the output shows "Applied 1 change"
    And the output shows "mtu: 1500 -> 9000"
    And the system interface eth0 now has mtu=9000
    And the exit code is 0

  Scenario: Apply all files in a directory
    Given a directory "/tmp/policies/" containing "eth0.yaml" and "dns.yaml"
    When the user runs "netfyr apply /tmp/policies/"
    Then both policy files are loaded and reconciled
    And changes for both entities are applied

  Scenario: Bare state YAML is auto-wrapped into static policy
    Given a YAML file with no "kind" field, containing type=ethernet, name=eth0, mtu=1500
    When the user runs "netfyr apply /tmp/eth0.yaml"
    Then the file is auto-wrapped into a static policy with default priority 100
    And the policy name is derived from the filename ("eth0")

  Scenario: DHCP policy without daemon fails with clear error
    Given a YAML file defining a dhcpv4 policy for eth0
    And the netfyr daemon is not running
    When the user runs "netfyr apply /tmp/eth0-dhcp.yaml"
    Then the output shows an error mentioning "requires the netfyr daemon"
    And the output includes "systemctl start netfyr"
    And the exit code is 2

  Scenario: Dry-run shows diff without applying
    Given a YAML file defining ethernet/eth0 with mtu=9000
    And the system has eth0 with mtu=1500
    When the user runs "netfyr apply --dry-run /tmp/eth0.yaml"
    Then the output shows the diff (mtu: 1500 -> 9000)
    And the system mtu is still 1500

  Scenario: Dry-run with no changes needed
    Given a YAML file matching the current system state
    When the user runs "netfyr apply --dry-run /tmp/eth0.yaml"
    Then the output says "No changes needed"
    And the exit code is 0

  Scenario: Partial failure reports mixed results
    Given a YAML file defining changes to eth0 (exists) and eth99 (does not exist)
    When the user runs "netfyr apply /tmp/policies/"
    Then the output shows the successful change for eth0
    And the output shows the failure for eth99
    And the exit code is 1

  Scenario: Total failure returns exit code 2
    Given a YAML file defining changes only to eth99 (does not exist)
    When the user runs "netfyr apply /tmp/eth99.yaml"
    Then the output shows the failure
    And the exit code is 2

  Scenario: YAML parse error returns exit code 2
    Given an invalid YAML file with syntax errors
    When the user runs "netfyr apply /tmp/bad.yaml"
    Then the output shows the parse error with file path and line number
    And the exit code is 2

  Scenario: Conflicts are reported as warnings
    Given two YAML files where both set eth0 mtu at the same priority with different values
    When the user runs "netfyr apply /tmp/policies/"
    Then the output shows a conflict warning for eth0 mtu
    And the conflicting field is not applied
    And other non-conflicting changes are applied
    And the exit code is 1

  Scenario: No path arguments shows error
    When the user runs "netfyr apply" with no arguments
    Then clap shows a usage error requiring at least one path
    And the exit code is 2

  Scenario: Path does not exist shows error
    When the user runs "netfyr apply /nonexistent/path"
    Then the output shows "path not found: /nonexistent/path"
    And the exit code is 2

Feature: netfyr apply CLI command (daemon mode)
  Scenario: Apply submits policies to daemon when running
    Given the netfyr daemon is running and listening on /run/netfyr/netfyr.sock
    And a YAML file defining ethernet/eth0 with mtu=9000
    When the user runs "netfyr apply /tmp/eth0.yaml"
    Then the CLI connects to the daemon via Varlink
    And submits all policies with replace-all semantics
    And the daemon reconciles and applies the changes
    And the output shows "Submitted 1 policy to daemon"

  Scenario: Daemon mode supports DHCP policies
    Given the netfyr daemon is running
    And a YAML file defining a dhcpv4 policy for eth0
    When the user runs "netfyr apply /tmp/eth0-dhcp.yaml"
    Then the CLI submits the DHCP policy to the daemon
    And the daemon starts a DHCP client on eth0
    And the exit code is 0

  Scenario: Dry-run in daemon mode asks daemon for diff
    Given the netfyr daemon is running
    And a YAML file defining ethernet/eth0 with mtu=9000
    When the user runs "netfyr apply --dry-run /tmp/eth0.yaml"
    Then the CLI calls DryRun on the daemon via Varlink
    And the output shows the diff without applying

  Scenario: Replace-all semantics in daemon mode
    Given the netfyr daemon is running with previously submitted policies for eth0 and eth1
    And a YAML file defining only a policy for eth0
    When the user runs "netfyr apply /tmp/eth0.yaml"
    Then the daemon replaces its entire policy set with just the eth0 policy
    And the eth1 policy is removed
    And reconciliation re-runs (eth1 state may be cleaned up)

Feature: Integration tests for CLI apply (shell scripts)
  Scenario: Apply static policy in namespace
    Given a shell script running inside `unshare --user --net`
    And a veth pair "veth-test0"/"veth-test1" with default mtu 1500
    And a YAML file defining ethernet "veth-test0" with mtu=1400
    When `netfyr apply policy.yaml` is run
    Then the exit code is 0
    And `ip link show veth-test0` shows mtu 1400

  Scenario: Apply with address in namespace
    Given a namespace with a veth pair "veth-test0"/"veth-test1"
    And a YAML file defining ethernet "veth-test0" with mtu=1400 and addresses=["10.99.0.1/24"]
    When `netfyr apply policy.yaml` is run
    Then the exit code is 0
    And `ip addr show veth-test0` includes "10.99.0.1/24"

  Scenario: Dry-run does not change state in namespace
    Given a namespace with a veth pair "veth-test0" at mtu 1500
    And a YAML file defining mtu=1400
    When `netfyr apply --dry-run policy.yaml` is run
    Then `ip link show veth-test0` still shows mtu 1500
```
