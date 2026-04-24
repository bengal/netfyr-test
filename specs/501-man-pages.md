# SPEC-501: Man Pages

## What

Generate man pages for the `netfyr` CLI and all its subcommands using `clap_mangen`. A build-time generator (xtask or build script) produces troff-formatted man pages from the clap argument definitions. The generated pages are:

- `netfyr(1)` — top-level overview and subcommand listing
- `netfyr-apply(1)` — apply policies to the system
- `netfyr-query(1)` — query current network state
- `netfyr-history(1)` — inspect journal of state changes
- `netfyr-revert(1)` — revert to a previous network state
- `netfyr-daemon(8)` — daemon operation, external change detection, journal
- `netfyr-examples(7)` — annotated examples of common configuration scenarios

Each section 1 man page includes NAME, SYNOPSIS, DESCRIPTION, OPTIONS, EXIT STATUS, FILES, EXAMPLES, and SEE ALSO sections. The EXAMPLES section contains at least two real-world usage examples per subcommand. The `netfyr-daemon(8)` page is a hand-written troff file (section 8 per Unix convention for system daemons) that documents daemon behavior including external change detection, its limitations, the journal, and the reconciliation lifecycle. The `netfyr-examples(7)` page is a hand-written troff file (not auto-generated) that provides comprehensive, copy-pasteable configuration examples. The generated and hand-written man pages are placed in a `man/` directory at the workspace root for packaging.

## Why

Man pages are the standard documentation mechanism on Linux systems. Fedora packaging guidelines require man pages for all shipped binaries. Generating them from the clap definitions ensures that the documentation is always in sync with the actual CLI interface — flags, arguments, and descriptions cannot drift out of date. This is a prerequisite for the RPM packaging story (SPEC-502).

## User interaction

```bash
# Generate man pages during development
cargo xtask man

# View a generated man page locally
man ./man/netfyr.1

# After RPM installation, the standard man command works
man netfyr
man netfyr-apply
man netfyr-query
man netfyr-history
man netfyr-revert
man netfyr-daemon
man netfyr-examples
```

### Example man page output (netfyr-apply)

```
NETFYR-APPLY(1)             General Commands Manual            NETFYR-APPLY(1)

NAME
       netfyr-apply — apply network policies to the system

SYNOPSIS
       netfyr apply [--dry-run] <path>...

DESCRIPTION
       Load policy definitions from YAML files or directories, reconcile them
       into an effective desired state, query the current system state, generate
       a diff, and apply the changes.

       If the netfyr daemon is running, policies are submitted to the daemon
       via Varlink. Otherwise, static policies are applied directly.

       If --dry-run is given, show what would change without applying.

OPTIONS
       <path>...
              One or more paths to YAML policy files or directories containing
              them. Directories are searched recursively for .yaml and .yml
              files.

       --dry-run
              Show what would change without applying.

EXIT STATUS
       0      All operations succeeded or no changes needed.

       1      Partial failure or conflicts detected.

       2      Total failure or fatal error.

FILES
       /etc/netfyr/policies/
              Default directory for policy files.

EXAMPLES
       Apply all policies in the default directory:

              netfyr apply /etc/netfyr/policies/

       Preview changes before applying:

              netfyr apply --dry-run /etc/netfyr/policies/server.yaml

SEE ALSO
       netfyr(1), netfyr-query(1), netfyr-history(1), netfyr-revert(1),
       netfyr-daemon(8), netfyr.yaml(5)
```

### Example man page output (netfyr-daemon)

```
NETFYR-DAEMON(8)           System Manager's Manual           NETFYR-DAEMON(8)

NAME
       netfyr-daemon — netfyr network configuration daemon

SYNOPSIS
       netfyr-daemon

DESCRIPTION
       The netfyr daemon is a long-running process that manages network
       configuration. It listens on a Varlink socket for policy submissions
       from the netfyr CLI, manages DHCPv4 client lifecycles, monitors for
       external network changes, and records all state transitions in a
       journal.

       The daemon is typically started via systemd:

              systemctl start netfyr

EXTERNAL CHANGE DETECTION
       The daemon monitors the kernel for network state changes made by
       other tools (e.g., ip(8), NetworkManager) or by the kernel itself
       (e.g., carrier loss). When an external change is detected on a
       managed interface, the daemon records a journal entry with trigger
       type "external_change" but does not re-apply the current desired
       state.

       Important limitations:

       Only managed interfaces are monitored.
              An interface is "managed" when at least one active policy
              targets it. If no policies are active, no external changes
              are detected. The daemon does not monitor interfaces that
              have never been the target of a policy.

       Monitored properties.
              The daemon subscribes to netlink notifications for link
              attribute changes (mtu, state, flags) and IPv4 address
              additions and removals. Only properties that netfyr manages
              on ethernet interfaces are tracked. Changes to properties
              outside netfyr's scope (e.g., IPv6 addresses, routing
              tables, qdiscs) are not detected.

       Debounce window.
              Network changes often arrive in bursts. The daemon coalesces
              notifications within a 500ms window into a single journal
              entry. This means there may be a short delay before a change
              appears in the journal.

       No automatic re-reconciliation.
              The daemon records external changes but does not fight back.
              To restore the desired state, run netfyr apply or submit
              policies again. The next reconciliation cycle (e.g., a DHCP
              event or policy submission) will re-apply the current desired
              state, which may differ from the externally modified state.

JOURNAL
       The daemon writes all state transitions to an append-only journal
       in NDJSON format. This includes policy applications, DHCP events,
       daemon startup, external changes, and reverts.

       The journal is stored at /var/lib/netfyr/journal/ by default
       (configurable via NETFYR_JOURNAL_DIR). Use netfyr history(1) to
       inspect journal entries and netfyr revert(1) to restore a previous
       state.

       Journal files are rotated at 10,000 entries or 50 MB (configurable
       via NETFYR_JOURNAL_MAX_ENTRIES and NETFYR_JOURNAL_MAX_SIZE). Rotated
       files are gzip-compressed and retained for 90 days by default
       (configurable via NETFYR_JOURNAL_RETENTION_DAYS).

ENVIRONMENT
       NETFYR_SOCKET_PATH
              Path to the Varlink socket (default: /run/netfyr/netfyr.sock).

       NETFYR_POLICY_DIR
              Directory for persisted policies (default: /var/lib/netfyr/policies/).

       NETFYR_JOURNAL_DIR
              Directory for journal files (default: /var/lib/netfyr/journal/).

       NETFYR_JOURNAL_MAX_ENTRIES
              Maximum entries per journal file before rotation (default: 10000).

       NETFYR_JOURNAL_MAX_SIZE
              Maximum journal file size in bytes before rotation (default: 52428800).

       NETFYR_JOURNAL_RETENTION_DAYS
              Days to retain rotated journal archives (default: 90).

FILES
       /run/netfyr/netfyr.sock
              Varlink socket for CLI-to-daemon communication.

       /var/lib/netfyr/policies/
              Persisted policies that survive daemon restarts.

       /var/lib/netfyr/journal/
              Journal directory containing current.ndjson and archived
              entries.

       /etc/netfyr/policies/
              User-provided policy files loaded at startup.

SEE ALSO
       netfyr(1), netfyr-apply(1), netfyr-query(1), netfyr-history(1),
       netfyr-revert(1), netfyr.yaml(5), netfyr-examples(7), systemd(1)
```

### Example man page output (netfyr-examples)

```
NETFYR-EXAMPLES(7)         Miscellaneous Information Manual        NETFYR-EXAMPLES(7)

NAME
       netfyr-examples — configuration examples for netfyr

DESCRIPTION
       This page shows annotated YAML configuration examples for common
       netfyr scenarios. Each example can be saved to a file in
       /etc/netfyr/policies/ and applied with netfyr apply.

       For field and format reference, see netfyr.yaml(5).

STATIC IP ON A SINGLE INTERFACE
       Assign a static IPv4 address, default gateway, and custom MTU to
       an interface identified by name.

           type: ethernet
           name: eth0
           mtu: 1500
           addresses:
             - 10.0.1.50/24
           routes:
             - destination: 0.0.0.0/0
               gateway: 10.0.1.1

MULTIPLE INTERFACES IN ONE FILE
       Configure two interfaces in a single file using the multi-document
       separator "---".

           type: ethernet
           name: eth0
           mtu: 9000
           addresses:
             - 10.0.1.50/24
           ---
           type: ethernet
           name: eth1
           mtu: 1500
           addresses:
             - 192.168.1.10/24

DHCP ON AN INTERFACE
       Acquire an address via DHCPv4. Requires the netfyr daemon.

           kind: policy
           name: eth0-dhcp
           factory: dhcpv4
           selector:
             name: eth0

MIXED STATIC AND DHCP
       Two files in /etc/netfyr/policies/: one for a static management
       interface, one for a DHCP uplink.

       File: /etc/netfyr/policies/mgmt.yaml

           type: ethernet
           name: eth0
           addresses:
             - 192.168.100.1/24

       File: /etc/netfyr/policies/uplink.yaml

           kind: policy
           name: uplink-dhcp
           factory: dhcpv4
           selector:
             name: eth1

PRIORITY OVERRIDE
       When two policies set the same field on the same interface, the
       higher priority wins. Here the admin override (priority 200) takes
       precedence over the baseline (default priority 100).

       File: /etc/netfyr/policies/baseline.yaml

           type: ethernet
           name: eth0
           mtu: 1500

       File: /etc/netfyr/policies/override.yaml

           kind: policy
           name: jumbo-frames
           factory: static
           priority: 200
           state:
             type: ethernet
             name: eth0
             mtu: 9000

       Result: eth0 gets mtu 9000 because priority 200 > 100.

SELECTING BY DRIVER
       Apply jumbo frames to all interfaces using the ixgbe driver,
       regardless of interface name.

           type: ethernet
           driver: ixgbe
           mtu: 9000

DRY-RUN WORKFLOW
       Preview changes before applying them:

           $ netfyr apply --dry-run /etc/netfyr/policies/
           ethernet/eth0:
             mtu: 1500 -> 9000

       If the diff looks correct, apply for real:

           $ netfyr apply /etc/netfyr/policies/

INVESTIGATING CHANGES WITH HISTORY
       List recent state changes from the journal:

           $ netfyr history
           SEQ  TIMESTAMP              TRIGGER       ENTITIES  CHANGES
           145  2026-04-20 15:10:05    external      eth0      mtu 1400→1500
           144  2026-04-20 15:00:00    policy-apply  eth0      mtu 1500→1400

       Show full details for a specific entry:

           $ netfyr history --show 145
           Entry #145 at 2026-04-20 15:10:05 UTC
           Trigger: external-change
             Changed entities: eth0
           Diff:
             ~ ethernet eth0
                 mtu: 9000 -> 1500
           Outcome: observed (no action taken)

       Filter by entity and output as JSON for scripting:

           $ netfyr history -s name=eth0 --since 1h -o json | jq '.[].trigger'

EXTERNAL CHANGE DETECTION
       The daemon automatically monitors managed interfaces for changes
       made by other tools. To use this feature:

       1. Create a policy for the interface you want to monitor:

           File: /etc/netfyr/policies/eth0.yaml

              type: ethernet
              name: eth0
              mtu: 9000

       2. Apply the policy and start the daemon:

              $ netfyr apply /etc/netfyr/policies/
              $ sudo systemctl start netfyr

       3. When an external tool changes the interface, the daemon records
          it in the journal:

              $ ip link set eth0 mtu 1500
              $ netfyr history -n 1
              SEQ  TIMESTAMP              TRIGGER   ENTITIES  CHANGES
              146  2026-04-20 15:15:00    external  eth0      mtu 1400→1500

       Important: only interfaces targeted by at least one active policy
       are monitored. If you want to track changes to an interface, you
       must have a policy for it — even a minimal one that only declares
       the type and name. See netfyr-daemon(8) for full details on
       limitations.

REVERTING TO A PREVIOUS STATE
       After investigating changes with netfyr history, revert to a known
       good state:

           $ netfyr history -n 3
           SEQ  TIMESTAMP              TRIGGER       ENTITIES  CHANGES
           146  2026-04-20 15:15:00    external      eth0      mtu 1400→1500
           145  2026-04-20 15:10:05    policy-apply  eth0      mtu 1500→1400
           144  2026-04-20 15:00:00    policy-apply  eth0      mtu 1500→1400, +10.0.1.50/24

           $ netfyr revert 145 --dry-run
           Reverting to state from entry #145 (2026-04-20 15:10:05 UTC)
           Changes that would be applied:
             ~ ethernet eth0
                 mtu: 1500 -> 9000

           $ netfyr revert 145
           Reverting to state from entry #145 (2026-04-20 15:10:05 UTC)
           Applied 1 change (1 entity modified).

       In daemon mode, a warning is printed because the active policies
       are not changed. The next reconciliation cycle may re-apply the
       current desired state.

SEE ALSO
       netfyr(1), netfyr-apply(1), netfyr-query(1), netfyr-history(1),
       netfyr-revert(1), netfyr-daemon(8), netfyr.yaml(5)
```

## Implementation details

- Crate: new `xtask` crate at workspace root (binary)
- Files:
  - `xtask/Cargo.toml`
  - `xtask/src/main.rs` — xtask entry point with `man` subcommand
  - `.cargo/config.toml` — alias `cargo xtask` to `cargo run --package xtask --`
- Hand-written files:
  - `man/netfyr-daemon.8` — hand-written troff man page for the daemon (section 8)
  - `man/netfyr-examples.7` — hand-written troff man page (not auto-generated)
- Dependencies (external crates): `clap_mangen` (man page generation from clap), `clap` (re-use CLI definitions from `netfyr-cli`)

### Xtask structure

The `xtask` crate is a standard Rust xtask pattern — a workspace-local binary used for development automation. It is not shipped in the RPM.

```toml
# xtask/Cargo.toml
[package]
name = "xtask"
version = "0.1.0"
edition = "2021"
publish = false

[dependencies]
clap_mangen = "0.2"
netfyr-cli = { path = "../crates/netfyr-cli" }
clap = { version = "4", features = ["derive"] }
```

```toml
# .cargo/config.toml
[alias]
xtask = "run --package xtask --"
```

### Man page generation (`xtask/src/main.rs`)

```rust
use clap::CommandFactory;
use clap_mangen::Man;
use std::fs;
use std::path::Path;

fn generate_man_pages() -> Result<(), Box<dyn std::error::Error>> {
    let out_dir = Path::new("man");
    fs::create_dir_all(out_dir)?;

    let cmd = netfyr_cli::Cli::command();

    // Generate top-level man page
    let man = Man::new(cmd.clone());
    let mut buf = Vec::new();
    man.render(&mut buf)?;
    fs::write(out_dir.join("netfyr.1"), buf)?;

    // Generate subcommand man pages
    for subcmd in cmd.get_subcommands() {
        let name = format!("netfyr-{}", subcmd.get_name());
        let subcmd = subcmd.clone().name(&name);
        let man = Man::new(subcmd);
        let mut buf = Vec::new();
        man.render(&mut buf)?;
        fs::write(out_dir.join(format!("{name}.1")), buf)?;
    }

    Ok(())
}
```

### Supplementing auto-generated content

`clap_mangen` generates NAME, SYNOPSIS, DESCRIPTION, and OPTIONS automatically from the clap definitions. The following sections must be added manually by the xtask after generation, using `Man::render_*` methods or by appending raw troff:

- **EXIT STATUS**: from the exit code tables defined in each CLI spec (SPEC-301, SPEC-302).
- **FILES**: `/etc/netfyr/policies/`, `/var/lib/netfyr/`.
- **EXAMPLES**: two examples per subcommand, drawn from the User Interaction sections of the respective specs.
- **SEE ALSO**: cross-references between all netfyr man pages.
- **ENVIRONMENT**: document `NO_COLOR` (disables colored output) in `netfyr(1)` and all subcommand pages.

Note: The xtask does not generate or overwrite `man/netfyr-daemon.8` or `man/netfyr-examples.7` — those files are maintained by hand, similar to `man/netfyr.yaml.5` (SPEC-503). A comment at the top of each troff source notes that the file is maintained by hand.

### Files in man/ directory

```
man/
├── netfyr.1              (auto-generated by xtask)
├── netfyr-apply.1        (auto-generated by xtask)
├── netfyr-query.1        (auto-generated by xtask)
├── netfyr-history.1      (auto-generated by xtask)
├── netfyr-revert.1       (auto-generated by xtask)
├── netfyr-daemon.8       (hand-written)
└── netfyr-examples.7     (hand-written)
```

## Depends on

- SPEC-301 (CLI apply — provides clap definitions and exit codes)
- SPEC-302 (CLI query — provides clap definitions and exit codes)
- SPEC-351 (Journal infrastructure — journal behavior documented in netfyr-daemon(8))
- SPEC-352 (History CLI — provides clap definitions for history subcommand)
- SPEC-353 (External change detection — behavior documented in netfyr-daemon(8))
- SPEC-354 (State revert — provides clap definitions for revert subcommand)
- SPEC-403 (Daemon — daemon behavior documented in netfyr-daemon(8))

## Acceptance criteria

```gherkin
Feature: Man page generation

  Scenario: Generate all man pages
    Given the workspace compiles successfully
    When the developer runs "cargo xtask man"
    Then the man/ directory is created
    And it contains netfyr.1, netfyr-apply.1, netfyr-query.1, netfyr-history.1, netfyr-revert.1
    And it does not overwrite the hand-written netfyr-daemon.8 or netfyr-examples.7

  Scenario: Top-level man page lists all subcommands
    Given man pages have been generated
    When the developer views man/netfyr.1
    Then the DESCRIPTION section mentions apply and query subcommands
    And the SEE ALSO section references all subcommand man pages

  Scenario: Subcommand man pages document all flags
    Given man pages have been generated
    When the developer views man/netfyr-apply.1
    Then the OPTIONS section lists --dry-run
    And the OPTIONS section documents the <path> positional argument

  Scenario: Man pages include EXIT STATUS section
    Given man pages have been generated
    When the developer views man/netfyr-apply.1
    Then the EXIT STATUS section documents codes 0, 1, and 2

  Scenario: Man pages include EXAMPLES section
    Given man pages have been generated
    When the developer views man/netfyr-apply.1
    Then the EXAMPLES section contains at least two usage examples

  Scenario: Man pages include FILES section
    Given man pages have been generated
    When the developer views man/netfyr-apply.1
    Then the FILES section lists /etc/netfyr/policies/

  Scenario: Man pages include SEE ALSO cross-references
    Given man pages have been generated
    When the developer views man/netfyr-apply.1
    Then the SEE ALSO section references netfyr(1) and netfyr-query(1) and netfyr.yaml(5)

  Scenario: Man pages render correctly with man command
    Given man pages have been generated
    When the developer runs "man ./man/netfyr.1"
    Then the page renders without troff warnings or errors

  Scenario: Examples man page exists and renders
    Given the file man/netfyr-examples.7 exists
    When the developer runs "man ./man/netfyr-examples.7"
    Then the page renders without troff warnings
    And the NAME section contains "netfyr-examples"

  Scenario: Examples man page covers common scenarios
    Given the examples man page is rendered
    Then it includes sections for:
      | Static IP on a single interface          |
      | Multiple interfaces in one file          |
      | DHCP on an interface                     |
      | Mixed static and DHCP                    |
      | Priority override                        |
      | Selecting by driver                      |
      | Dry-run workflow                         |
      | Investigating changes with history       |
      | External change detection                |
      | Reverting to a previous state            |
    And each section contains a copy-pasteable example

  Scenario: Daemon man page exists and renders
    Given the file man/netfyr-daemon.8 exists
    When the developer runs "man ./man/netfyr-daemon.8"
    Then the page renders without troff warnings
    And the NAME section contains "netfyr-daemon"

  Scenario: Daemon man page documents external change detection
    Given the daemon man page is rendered
    Then the EXTERNAL CHANGE DETECTION section explains:
      | Only managed interfaces are monitored                    |
      | Monitored properties (mtu, state, flags, IPv4 addresses) |
      | Debounce window (500ms)                                  |
      | No automatic re-reconciliation                           |

  Scenario: Daemon man page documents the journal
    Given the daemon man page is rendered
    Then the JOURNAL section describes the NDJSON format
    And it documents journal rotation and retention
    And it references netfyr-history(1) and netfyr-revert(1)

  Scenario: Daemon man page documents environment variables
    Given the daemon man page is rendered
    Then the ENVIRONMENT section lists NETFYR_SOCKET_PATH, NETFYR_POLICY_DIR,
         NETFYR_JOURNAL_DIR, NETFYR_JOURNAL_MAX_ENTRIES, NETFYR_JOURNAL_MAX_SIZE,
         and NETFYR_JOURNAL_RETENTION_DAYS

  Scenario: External change detection example is accurate
    Given the examples man page is rendered
    Then the EXTERNAL CHANGE DETECTION section explains that a policy
         must exist for the interface before changes are tracked
    And the example shows the complete workflow: policy, apply, external
         change, history

  Scenario: Man pages stay in sync with CLI
    Given a new --verbose flag is added to the apply command in clap
    When the developer runs "cargo xtask man"
    Then man/netfyr-apply.1 includes the --verbose flag in OPTIONS

  Scenario: Regeneration is idempotent
    Given man pages have been generated
    When the developer runs "cargo xtask man" a second time
    Then the output files are identical to the first run
```
