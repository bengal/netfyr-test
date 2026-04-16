# SPEC-001: Workspace Setup

## What
Set up the Rust workspace with a root `Cargo.toml` and seven crates under `crates/`:
- `netfyr-state` (library) -- core state types and set operations
- `netfyr-reconcile` (library) -- reconciliation engine
- `netfyr-backend` (library) -- backend trait and rtnetlink implementation
- `netfyr-policy` (library) -- policy types and factories
- `netfyr-varlink` (library) -- Varlink protocol definitions and client/server
- `netfyr-cli` (binary) -- user-facing CLI
- `netfyr-daemon` (binary) -- long-running daemon for dynamic factories

Each library crate contains a `Cargo.toml` and `src/lib.rs`. The binary crates contain a `Cargo.toml` and `src/main.rs` with a minimal `fn main()`.

Integration tests live in `tests/` as shell scripts (see "Integration test infrastructure" below).

The root `Cargo.toml` defines workspace-level Cargo features: `dhcp`, `systemd`, `varlink`.

### Dependency policy

External crate dependencies must be kept to a strict minimum. Only add an external dependency when it is genuinely necessary — i.e., implementing the functionality from scratch would be unreasonable or error-prone (e.g., `rtnetlink` for netlink protocol, `clap` for CLI argument parsing, `tokio` for async runtime). If a task can be accomplished with the standard library or a small amount of code, do not pull in a crate.

Each spec lists its external dependencies explicitly. No additional external crates may be introduced without justification. When choosing between crates that provide similar functionality, prefer the one with fewer transitive dependencies.

Rationale: a small dependency chain means faster builds, smaller binaries, fewer supply-chain risks, and easier auditing. netfyr is a system tool — it should be lean.

## Why
The workspace structure is the foundation for all subsequent development. It establishes the crate boundaries that reflect the layered architecture: State (pure data), Policy (factories), Reconciliation (merge logic), Backend (system I/O), Varlink (IPC protocol), CLI (user-facing binary), and Daemon (long-running process for dynamic factories like DHCP).

## User interaction
Developers clone the repository and run `cargo build` to compile all crates. They can also build individual crates with `cargo build -p netfyr-state`, etc. Feature flags are passed at the workspace level: `cargo build --features dhcp,systemd`.

## Implementation details
- Crate: (root workspace)
- Files:
  - `Cargo.toml` (workspace root)
  - `crates/netfyr-state/Cargo.toml`
  - `crates/netfyr-state/src/lib.rs`
  - `crates/netfyr-reconcile/Cargo.toml`
  - `crates/netfyr-reconcile/src/lib.rs`
  - `crates/netfyr-backend/Cargo.toml`
  - `crates/netfyr-backend/src/lib.rs`
  - `crates/netfyr-policy/Cargo.toml`
  - `crates/netfyr-policy/src/lib.rs`
  - `crates/netfyr-varlink/Cargo.toml`
  - `crates/netfyr-varlink/src/lib.rs`
  - `crates/netfyr-cli/Cargo.toml`
  - `crates/netfyr-cli/src/main.rs`
  - `crates/netfyr-daemon/Cargo.toml`
  - `crates/netfyr-daemon/src/main.rs`
  - `tests/helpers.sh` (shared shell functions for integration tests)
- Dependencies (external crates): none at this stage (stubs only)

### Root Cargo.toml structure

```toml
[workspace]
resolver = "2"
members = [
    "crates/netfyr-state",
    "crates/netfyr-reconcile",
    "crates/netfyr-backend",
    "crates/netfyr-policy",
    "crates/netfyr-varlink",
    "crates/netfyr-cli",
    "crates/netfyr-daemon",
]

[workspace.features]
dhcp = []
systemd = []
varlink = []
```

### Stub lib.rs content
Each library `src/lib.rs` contains only:
```rust
//! netfyr-<name> crate
```

### Stub main.rs content
Each binary crate's `src/main.rs` contains only:
```rust
fn main() {
    println!("netfyr");
}
```

### Stub Cargo.toml per crate
Each crate's `Cargo.toml` contains the crate name, version `0.1.0`, edition `2021`, and an empty `[dependencies]` section. Example:
```toml
[package]
name = "netfyr-state"
version = "0.1.0"
edition = "2021"

[dependencies]
```

### Integration test infrastructure

Integration tests are shell scripts in `tests/`, run via `make integration-test` (or a CI step). Each script uses `unshare --user --net` to create an unprivileged network namespace, sets up veth pairs and dnsmasq as needed, runs `netfyr` CLI commands, and verifies the result with standard tools (`ip`, `grep`, etc.).

Shell scripts avoid the problem of calling `unshare(2)` from a multi-threaded Rust test binary (which fails with `EINVAL`). Each test script runs as a separate single-threaded process.

**`tests/helpers.sh`** -- Shared shell functions sourced by all test scripts:

- **`netns_setup`** -- Runs the rest of the script inside a new user + network namespace via `unshare --user --net`. Fails the test if namespaces are unavailable.
- **`create_veth VETH0 VETH1`** -- Creates a veth pair, brings both ends up.
- **`add_address IFACE CIDR`** -- Adds an IP address to an interface.
- **`start_dnsmasq IFACE SERVER_IP RANGE_START RANGE_END LEASE_TIME`** -- Starts a dnsmasq DHCP server on the given interface. Stores the PID for cleanup.
- **`cleanup`** -- Kills dnsmasq and cleans up. Registered as a `trap EXIT` handler.
- **`assert_eq`, `assert_match`, `assert_has_address`, `assert_link_up`** -- Test assertion helpers.

**Example test script** (`tests/test_dhcp_lease.sh`):

```bash
#!/bin/bash
set -euo pipefail
source "$(dirname "$0")/helpers.sh"

netns_setup

create_veth veth-dhcp0 veth-dhcp1
add_address veth-dhcp1 10.99.0.1/24
start_dnsmasq veth-dhcp1 10.99.0.1 10.99.0.100 10.99.0.200 120

cat > /tmp/dhcp-policy.yaml <<EOF
kind: policy
name: dhcp-test
factory: dhcpv4
selector:
  name: veth-dhcp0
EOF

netfyr apply /tmp/dhcp-policy.yaml

# Wait for DHCP lease
sleep 5

assert_has_address veth-dhcp0 "10.99.0."
assert_link_up veth-dhcp0
echo "PASS: DHCP lease acquired"
```

**Running integration tests:**

```bash
# Run all integration tests
make integration-test

# Run a single test
bash tests/test_dhcp_lease.sh
```

Tests fail (exit non-zero) if prerequisites (namespaces, dnsmasq) are unavailable — this ensures missing dependencies are caught immediately rather than silently ignored.

## Depends on
(none)

## Acceptance criteria
```gherkin
Feature: Rust workspace setup
  Scenario: Root workspace compiles successfully
    Given a fresh clone of the repository
    When the developer runs "cargo build" from the project root
    Then the build succeeds with exit code 0
    And all 7 crates are compiled

  Scenario: Individual crate compiles in isolation
    Given the workspace is set up
    When the developer runs "cargo build -p netfyr-state"
    Then only netfyr-state and its dependencies are compiled
    And the build succeeds

  Scenario: Workspace members are correctly listed
    Given the root Cargo.toml exists
    When the workspace members list is inspected
    Then it contains exactly 7 entries: netfyr-state, netfyr-reconcile, netfyr-backend, netfyr-policy, netfyr-varlink, netfyr-cli, netfyr-daemon

  Scenario: CLI crate produces a binary
    Given the workspace is set up
    When the developer runs "cargo build -p netfyr-cli"
    Then a binary named "netfyr-cli" is produced in the target directory
    And running the binary prints "netfyr" to stdout

  Scenario: Daemon crate produces a binary
    Given the workspace is set up
    When the developer runs "cargo build -p netfyr-daemon"
    Then a binary named "netfyr-daemon" is produced in the target directory
    And running the binary prints "netfyr" to stdout

  Scenario: Workspace features are defined
    Given the root Cargo.toml exists
    When the workspace features section is inspected
    Then it defines features: dhcp, systemd, varlink
    And each feature defaults to an empty dependency list

  Scenario: Library crates have correct structure
    Given the workspace is set up
    When the directory structure is inspected
    Then each library crate has a Cargo.toml and src/lib.rs
    And netfyr-cli and netfyr-daemon each have a Cargo.toml and src/main.rs
    And no crate has extraneous files

  Scenario: Integration test helpers exist
    Given the workspace is set up
    When the file tests/helpers.sh is inspected
    Then it defines functions: netns_setup, create_veth, add_address, start_dnsmasq, cleanup
    And it is sourced by all test scripts in tests/
```
