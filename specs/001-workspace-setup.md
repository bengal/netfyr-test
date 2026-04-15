# SPEC-001: Workspace Setup

## What
Set up the Rust workspace with a root `Cargo.toml` and eight crates under `crates/`:
- `netfyr-state` (library) -- core state types and set operations
- `netfyr-reconcile` (library) -- reconciliation engine
- `netfyr-backend` (library) -- backend trait and rtnetlink implementation
- `netfyr-policy` (library) -- policy types and factories
- `netfyr-varlink` (library) -- Varlink protocol definitions and client/server
- `netfyr-cli` (binary) -- user-facing CLI
- `netfyr-daemon` (binary) -- long-running daemon for dynamic factories
- `netfyr-test-utils` (library, dev-only) -- integration test infrastructure

Each library crate contains a `Cargo.toml` and `src/lib.rs`. The binary crates contain a `Cargo.toml` and `src/main.rs` with a minimal `fn main()`.

The root `Cargo.toml` defines workspace-level Cargo features: `dhcp`, `systemd`, `varlink`.

### Dependency policy

External crate dependencies must be kept to a strict minimum. Only add an external dependency when it is genuinely necessary — i.e., implementing the functionality from scratch would be unreasonable or error-prone (e.g., `rtnetlink` for netlink protocol, `clap` for CLI argument parsing, `tokio` for async runtime). If a task can be accomplished with the standard library or a small amount of code, do not pull in a crate.

Each spec lists its external dependencies explicitly. No additional external crates may be introduced without justification. When choosing between crates that provide similar functionality, prefer the one with fewer transitive dependencies.

Rationale: a small dependency chain means faster builds, smaller binaries, fewer supply-chain risks, and easier auditing. netfyr is a system tool — it should be lean.

## Why
The workspace structure is the foundation for all subsequent development. It establishes the crate boundaries that reflect the layered architecture: State (pure data), Policy (factories), Reconciliation (merge logic), Backend (system I/O), Varlink (IPC protocol), CLI (user-facing binary), and Daemon (long-running process for dynamic factories like DHCP). The `netfyr-test-utils` crate provides shared integration test infrastructure using unprivileged network namespaces so that all crates can write kernel-level integration tests without requiring root.

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
  - `crates/netfyr-test-utils/Cargo.toml`
  - `crates/netfyr-test-utils/src/lib.rs`
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
    "crates/netfyr-test-utils",
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

### netfyr-test-utils crate

The `netfyr-test-utils` crate provides shared integration test infrastructure. It is a dev-dependency for other crates (not shipped in release builds). It provides:

- **`NetNs`** -- Creates an unprivileged user + network namespace using `unshare(CLONE_NEWUSER | CLONE_NEWNET)`. Maps the current UID/GID to root inside the namespace. The current process (or a forked child) runs with full network capabilities inside the namespace without requiring actual root. Cleans up on `Drop`.

- **`NetNs::create_veth(name_a, name_b)`** -- Creates a veth pair inside the namespace for testing.

- **`NetNs::run(closure)`** -- Runs a closure inside the namespace.

- **`DhcpTestServer`** -- Spawns a `dnsmasq` instance inside the namespace configured as a minimal DHCP server with a given interface and address range. Used by DHCP integration tests. Kills dnsmasq on `Drop`.

This infrastructure allows all integration tests to run as unprivileged `cargo test` without root, `#[ignore]`, or feature gating.

## Depends on
(none)

## Acceptance criteria
```gherkin
Feature: Rust workspace setup
  Scenario: Root workspace compiles successfully
    Given a fresh clone of the repository
    When the developer runs "cargo build" from the project root
    Then the build succeeds with exit code 0
    And all 8 crates are compiled

  Scenario: Individual crate compiles in isolation
    Given the workspace is set up
    When the developer runs "cargo build -p netfyr-state"
    Then only netfyr-state and its dependencies are compiled
    And the build succeeds

  Scenario: Workspace members are correctly listed
    Given the root Cargo.toml exists
    When the workspace members list is inspected
    Then it contains exactly 8 entries: netfyr-state, netfyr-reconcile, netfyr-backend, netfyr-policy, netfyr-varlink, netfyr-cli, netfyr-daemon, netfyr-test-utils

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

  Scenario: Test utils crate compiles
    Given the workspace is set up
    When the developer runs "cargo build -p netfyr-test-utils"
    Then the build succeeds
```
