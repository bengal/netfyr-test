# SPEC-302: CLI Query Command

## What
Implement the `netfyr query [--selector key=value]... [--output yaml|json]` CLI command. This command queries the current system network state and displays the results. All filtering is done via `--selector` flags — including filtering by entity type (`--selector type=ethernet`). Outputs in YAML (default) or JSON format. Like `netfyr apply`, the query command auto-detects whether the daemon is running and routes accordingly.

### Two runtime modes

1. **Daemon-free mode** (daemon not running): The CLI queries the kernel directly via the backend (rtnetlink).

2. **Daemon mode** (daemon running): The CLI queries the daemon via Varlink (SPEC-404). The daemon returns the current state, which may include DHCP-acquired configuration.

Mode detection: the CLI reads the socket path from the `NETFYR_SOCKET_PATH` environment variable, defaulting to `/run/netfyr/netfyr.sock` if unset. It attempts to connect to that socket. If the connection succeeds, daemon mode is used. If it fails, daemon-free mode is used.

## Why
Querying the current system state is essential for operators to understand what is currently configured, debug issues, and verify that apply operations had the expected effect. The query command is the primary inspection tool. YAML and JSON output formats support both human reading and programmatic consumption (piping to `jq`, `yq`, etc.). Using `--selector` for all filtering (including entity type) keeps the CLI simple and uniform — `type` is just another filter dimension alongside `name`, `driver`, and `mac`.

## User interaction

### Basic usage
```bash
# Show all network entities
netfyr query

# Show a specific interface
netfyr query --selector name=eth0

# Show all ethernet interfaces
netfyr query --selector type=ethernet

# Filter by driver
netfyr query --selector driver=ixgbe

# Filter by MAC address
netfyr query --selector mac=aa:bb:cc:dd:ee:ff

# Multiple selector fields (AND logic)
netfyr query --selector type=ethernet --selector driver=ixgbe

# Short flags
netfyr query -s name=eth0
netfyr query -s name=eth0 -o json

# Output format options
netfyr query --output yaml    # default
netfyr query --output json
netfyr query -o json          # short flag
```

### Output examples

**YAML output** (default):
```yaml
- type: ethernet
  name: eth0
  mtu: 1500
  mac: aa:bb:cc:dd:ee:ff
  carrier: true
  speed: 1000
  addresses:
    - 10.0.1.50/24
    - fe80::1/64
  routes:
    - destination: 0.0.0.0/0
      gateway: 10.0.1.1
```

**JSON output**:
```json
[
  {
    "type": "ethernet",
    "name": "eth0",
    "mtu": 1500,
    "mac": "aa:bb:cc:dd:ee:ff",
    "carrier": true,
    "speed": 1000,
    "addresses": ["10.0.1.50/24", "fe80::1/64"],
    "routes": [{"destination": "0.0.0.0/0", "gateway": "10.0.1.1"}]
  }
]
```

### Exit codes
- `0`: Query succeeded (including when no entities match).
- `2`: Fatal error (e.g., backend failure, invalid arguments).

## Implementation details
- Crate: `netfyr-cli`
- Files: `src/query.rs`
- Dependencies (external crates): `clap` (CLI parsing), `serde_yaml` (YAML output), `serde_json` (JSON output)

### CLI argument definition

```rust
#[derive(Args)]
struct QueryArgs {
    /// Selector filters (can be specified multiple times, AND logic).
    /// Format: key=value (e.g., name=eth0, type=ethernet, driver=ixgbe)
    #[arg(long, short = 's', value_parser = parse_selector)]
    selector: Vec<(String, String)>,

    /// Output format: yaml (default), json
    #[arg(long, short = 'o', default_value = "yaml")]
    output: OutputFormat,
}

#[derive(Clone, ValueEnum)]
enum OutputFormat {
    Yaml,
    Json,
}
```

### Query flow

```rust
pub async fn run_query(args: QueryArgs) -> Result<ExitCode> {
    // 1. Extract type= from selectors (if present), keep the rest
    let (entity_type, selector) = extract_type_and_selector(&args.selector)?;

    // 2. Detect mode: try connecting to daemon via Varlink
    let socket_path = std::env::var("NETFYR_SOCKET_PATH")
        .unwrap_or_else(|_| "/run/netfyr/netfyr.sock".to_string());
    if let Ok(client) = varlink_connect(&socket_path) {
        return run_query_daemon(client, entity_type.as_ref(), selector.as_ref(), &args.output).await;
    }

    // Daemon-free mode: query kernel directly
    let registry = create_backend_registry();

    let state_set = if let Some(ref et) = entity_type {
        registry.query(et, selector.as_ref()).await?
    } else {
        let all = registry.query_all().await?;
        // Post-filter by remaining selectors if any
        filter_by_selector(all, selector.as_ref())
    };

    // Format and print
    match args.output {
        OutputFormat::Yaml => print_yaml(&state_set)?,
        OutputFormat::Json => print_json(&state_set)?,
    }

    Ok(ExitCode::from(0))
}

/// Separates `type=X` from the selector list.
/// Returns (Option<EntityType>, Option<Selector>) where the Selector
/// contains only the non-type fields (name, driver, mac, pci_path).
fn extract_type_and_selector(
    selectors: &[(String, String)]
) -> Result<(Option<EntityType>, Option<Selector>)> {
    let mut entity_type = None;
    let mut remaining = Vec::new();
    for (key, value) in selectors {
        if key == "type" {
            entity_type = Some(EntityType::from_str(value)?);
        } else {
            remaining.push((key.clone(), value.clone()));
        }
    }
    let selector = if remaining.is_empty() { None } else { Some(build_selector(&remaining)?) };
    Ok((entity_type, selector))
}
```

### Selector parsing

The `--selector` flag is parsed as `key=value` pairs. Valid keys:
- `type=ethernet` -> filters by entity type (extracted before backend query)
- `name=eth0` -> Selector { name: Some("eth0") }
- `driver=ixgbe` -> Selector { driver: Some("ixgbe") }
- `mac=aa:bb:cc:dd:ee:ff` -> Selector { mac: Some("aa:bb:cc:dd:ee:ff") }
- `pci_path=0000:03:00.0` -> Selector { pci_path: Some("0000:03:00.0") }

Multiple `--selector` flags are combined with AND logic. Invalid keys are rejected with an error listing valid selector keys.

### Error handling

- Invalid selector key: print error with list of valid selector keys (type, name, driver, mac, pci_path). Exit code 2.
- Invalid type value: print error with list of supported entity types. Exit code 2.
- No matching entities: print empty list (`[]` in JSON, empty YAML sequence). Exit code 0 (not an error).
- Backend query failure: print error. Exit code 2.

## Depends on
- SPEC-102 (Backend query implementation for ethernet)
- SPEC-301 (CLI framework and main binary structure shared with apply command)
- SPEC-404 (Varlink API for daemon mode)

## Integration test infrastructure
Integration tests are shell scripts in `tests/`. Each script uses `unshare --user --net` to create an unprivileged network namespace, sets up veth pairs with known configuration via `ip` commands, runs `netfyr query`, and verifies the output. No root required.

Shell test script rules (from SPEC-001):
- **Naming**: `302-description.sh` — the Makefile discovers tests via `tests/[0-9]*.sh`.
- **No skip**: if a prerequisite is missing (binary, `unshare`, `dnsmasq`), the script must `echo "FAIL: ..." >&2; exit 1`. Never `exit 0` on failure. Never wrap `netns_setup` or `start_dnsmasq` with `|| exit 0`.
- **Binary path**: locate binaries via `SCRIPT_DIR/../target/debug/netfyr` (overridable with `NETFYR_BIN` env var). Check with `[[ ! -x "$NETFYR_BIN" ]]` and `exit 1` if missing.
- **Helpers**: source `tests/helpers.sh` which provides `netns_setup`, `create_veth`, `add_address`, `start_dnsmasq`, `cleanup`, and assertion functions.

## Verification
In addition to `cargo test` and `cargo clippy`, run `make integration-test` to execute all shell integration test scripts. All tests must pass. This is a required verification step — the story is not complete until shell tests pass.

## Acceptance criteria
```gherkin
Feature: netfyr query CLI command (daemon-free mode)
  Scenario: Query all entities without arguments
    Given the system has ethernet interfaces eth0 and eth1
    When the user runs "netfyr query"
    Then the output lists all entities in YAML format
    And each entry shows type, selector properties, and config fields at the top level

  Scenario: Query by entity type via selector
    Given the system has ethernet/eth0 and ethernet/eth1
    When the user runs "netfyr query --selector type=ethernet"
    Then the output contains only ethernet entities

  Scenario: Query with name selector
    Given the system has ethernet/eth0 and ethernet/eth1
    When the user runs "netfyr query --selector name=eth0"
    Then the output contains only eth0
    And eth1 is not shown

  Scenario: Query with driver selector
    Given ethernet/eth0 uses driver ixgbe and ethernet/eth1 uses driver e1000
    When the user runs "netfyr query --selector driver=ixgbe"
    Then the output contains only eth0

  Scenario: Query with multiple selectors uses AND logic
    Given multiple ethernet interfaces with varying drivers and MACs
    When the user runs "netfyr query --selector type=ethernet --selector driver=ixgbe"
    Then only ethernet interfaces using driver ixgbe are shown

  Scenario: YAML output format (default)
    Given ethernet/eth0 exists with mtu=1500 and addresses
    When the user runs "netfyr query --selector name=eth0"
    Then the output is valid YAML
    And it contains the entity type, selector properties, and config fields at the top level

  Scenario: JSON output format
    Given ethernet/eth0 exists
    When the user runs "netfyr query --selector name=eth0 --output json"
    Then the output is valid JSON
    And it can be parsed by standard JSON tools (jq, etc.)

  Scenario: No matching entities returns empty result
    Given no interface named "eth99" exists
    When the user runs "netfyr query --selector name=eth99"
    Then the output is an empty list
    And the exit code is 0

  Scenario: Invalid selector key shows error
    When the user runs "netfyr query --selector invalid_key=value"
    Then the output shows an error about the invalid selector key
    And lists valid selector keys: type, name, driver, mac, pci_path
    And the exit code is 2

  Scenario: Invalid type value shows error
    When the user runs "netfyr query --selector type=foobar"
    Then the output shows an error: "Unknown entity type: foobar"
    And lists valid entity types
    And the exit code is 2

  Scenario: Output can be piped to other tools
    Given ethernet/eth0 exists
    When the user runs "netfyr query -s name=eth0 -o json | jq '.[].mtu'"
    Then the output is the mtu value (e.g., 1500)

  Scenario: Short flag -o works for output format
    When the user runs "netfyr query -s type=ethernet -o json"
    Then the output is valid JSON (same as --output json)

  Scenario: Short flag -s works for selector
    When the user runs "netfyr query -s name=eth0"
    Then the results are filtered to eth0 (same as --selector name=eth0)

Feature: netfyr query CLI command (daemon mode)
  Scenario: Query routes through daemon when running
    Given the netfyr daemon is running and listening on /run/netfyr/netfyr.sock
    When the user runs "netfyr query --selector name=eth0"
    Then the CLI connects to the daemon via Varlink
    And the daemon returns the current state
    And the output shows the entity in YAML format

  Scenario: Daemon mode can return DHCP-acquired state
    Given the netfyr daemon is running with a DHCP lease on eth0
    When the user runs "netfyr query --selector name=eth0"
    Then the output includes DHCP-acquired addresses and routes

Feature: Integration tests for CLI query (shell scripts)
  Scenario: Query veth interface in namespace
    Given a shell script running inside `unshare --user --net`
    And a veth pair "veth-test0"/"veth-test1" with "veth-test0" at mtu 1400 and address "10.99.0.1/24"
    When `netfyr query -s name=veth-test0 -o json` is run
    Then the exit code is 0
    And the JSON output contains one entity with name "veth-test0"
    And the entity has mtu=1400 and addresses containing "10.99.0.1/24"

  Scenario: Query all interfaces in namespace returns both veth ends
    Given a namespace with a veth pair "veth-test0"/"veth-test1"
    When `netfyr query -o json` is run
    Then the JSON output contains at least 2 entities

  Scenario: Query with YAML output in namespace
    Given a namespace with a veth pair "veth-test0"/"veth-test1" with "veth-test0" at mtu 1400
    When `netfyr query -s name=veth-test0` is run
    Then the output is valid YAML
    And it contains "mtu: 1400"
```
