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

Integration tests are shell scripts in `tests/`, run via `make integration-test`. Each script uses `unshare --user --net` to create an unprivileged network namespace, sets up veth pairs and dnsmasq as needed, runs `netfyr` CLI commands, and verifies the result with standard tools (`ip`, `grep`, etc.).

Shell scripts avoid the problem of calling `unshare(2)` from a multi-threaded Rust test binary (which fails with `EINVAL`). Each test script runs as a separate single-threaded process.

#### Naming convention

Test scripts are named `NNN-description.sh`, where `NNN` is the spec number the test covers and `description` is a hyphen-separated summary of the scenario. Examples: `102-query-veth-by-name.sh`, `401-dhcpv4-acquire-lease.sh`. The file `helpers.sh` is not a test script and is excluded from discovery by the naming convention.

#### No-skip policy

There is no skip mode. If a test cannot run because a prerequisite is missing (binary not built, `unshare` not available, `dnsmasq` not installed), the test **must print a FAIL message to stderr and `exit 1`**. Never `exit 0` on a missing prerequisite. This ensures that problems are caught immediately rather than silently ignored.

This rule applies to every prerequisite check in every test script:
- Binary not found → `echo "FAIL: ..." >&2; exit 1`
- `unshare` not available → `echo "FAIL: ..." >&2; exit 1`
- `dnsmasq` not installed → `echo "FAIL: ..." >&2; exit 1`

The `netns_setup` and `start_dnsmasq` helpers in `helpers.sh` already enforce this. Test scripts must not wrap these calls with `|| exit 0` or other fallbacks that swallow failures.

#### Binary path resolution

Test scripts locate the `netfyr` and `netfyr-daemon` binaries relative to the script's own directory via `SCRIPT_DIR/../target/debug/`. The path can be overridden via environment variables (`NETFYR_BIN`, `NETFYR_DAEMON_BIN`). The `Makefile`'s `integration-test` target runs `cargo build` before running test scripts, ensuring the binaries exist.

#### Verification requirement

**Every story that adds or modifies shell integration test scripts must run `make integration-test SPEC=NNN` as a verification step**, where `NNN` is the story's spec number. This runs only the tests belonging to that story, avoiding failures from tests that belong to stories not yet implemented. `cargo test` alone is not sufficient — it only runs Rust tests, not shell scripts. The story's implementation is not complete until its shell integration tests pass. This is an additional verification step beyond `cargo test` (see the verify prompt's step 5).

**`tests/helpers.sh`** -- Shared shell functions sourced by all test scripts:

- **`netns_setup`** -- Runs the rest of the script inside a new user + network namespace via `unshare --user --net`. Exits with code 1 if `unshare` is not available.
- **`create_veth VETH0 VETH1`** -- Creates a veth pair, brings both ends up.
- **`add_address IFACE CIDR`** -- Adds an IP address to an interface.
- **`start_dnsmasq IFACE SERVER_IP RANGE_START RANGE_END LEASE_TIME`** -- Starts a dnsmasq DHCP server on the given interface. Uses `--bind-dynamic` (not `--bind-interfaces`) so that dnsmasq can receive broadcast DHCP packets on the interface. Writes leases to a temp file via `--dhcp-leasefile`. Exits with code 1 if `dnsmasq` is not installed. Stores the PID for cleanup.
- **`cleanup`** -- Kills dnsmasq processes tracked by `start_dnsmasq` and cleans up. Registered as a `trap EXIT` handler by `netns_setup`. **Important:** test scripts that set their own `trap '...' EXIT` must call `cleanup` within the trap body, because setting a new trap overwrites the previous one.
- **`assert_eq`, `assert_match`, `assert_has_address`, `assert_link_up`** -- Test assertion helpers.

**Example test script** (`tests/401-dhcpv4-lease.sh`):

```bash
#!/bin/bash
# 401-dhcpv4-lease.sh -- Verify DHCPv4 factory acquires a lease.
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/helpers.sh"

NETFYR_BIN="${NETFYR_BIN:-$SCRIPT_DIR/../target/debug/netfyr}"
NETFYR_DAEMON_BIN="${NETFYR_DAEMON_BIN:-$SCRIPT_DIR/../target/debug/netfyr-daemon}"

if [[ ! -x "$NETFYR_BIN" ]]; then
    echo "FAIL: netfyr binary not found at $NETFYR_BIN" >&2
    exit 1
fi
if [[ ! -x "$NETFYR_DAEMON_BIN" ]]; then
    echo "FAIL: netfyr-daemon binary not found at $NETFYR_DAEMON_BIN" >&2
    exit 1
fi

netns_setup "$@"

# -- Inside the namespace --
create_veth veth-dhcp0 veth-dhcp1
add_address veth-dhcp1 10.99.0.1/24
start_dnsmasq veth-dhcp1 10.99.0.1 10.99.0.100 10.99.0.200 120

TMPDIR_TEST=$(mktemp -d)
trap 'kill "${DAEMON_PID:-}" 2>/dev/null; cleanup; rm -rf "$TMPDIR_TEST"' EXIT

SOCKET_PATH="$TMPDIR_TEST/netfyr.sock"
POLICY_DIR="$TMPDIR_TEST/policies"
mkdir -p "$POLICY_DIR"

cat > "$POLICY_DIR/dhcp.yaml" <<'EOF'
kind: policy
name: dhcp-test
factory: dhcpv4
selector:
  name: veth-dhcp0
EOF

NETFYR_SOCKET_PATH="$SOCKET_PATH" \
NETFYR_POLICY_DIR="$POLICY_DIR" \
    "$NETFYR_DAEMON_BIN" &
DAEMON_PID=$!

# Wait for daemon socket (poll up to 5 s).
for i in $(seq 1 50); do
    [[ -S "$SOCKET_PATH" ]] && break
    sleep 0.1
done
if [[ ! -S "$SOCKET_PATH" ]]; then
    echo "FAIL: daemon socket did not appear" >&2; exit 1
fi

NETFYR_SOCKET_PATH="$SOCKET_PATH" "$NETFYR_BIN" apply "$POLICY_DIR/dhcp.yaml"

# Wait for DHCP lease (poll up to 10 s).
for i in $(seq 1 100); do
    ip addr show dev veth-dhcp0 2>/dev/null | grep -q "10.99.0." && break
    sleep 0.1
done

assert_has_address veth-dhcp0 "10.99.0."
assert_link_up veth-dhcp0
echo "PASS: 401-dhcpv4-lease"
```

**Makefile (`integration-test` target):**

The `Makefile` provides a single `integration-test` target. It builds the project first, then discovers and runs test scripts. An optional `SPEC` variable filters to a single story's tests:

```makefile
.PHONY: integration-test

integration-test:
	cargo build
	@if [ -n "$(SPEC)" ]; then \
		scripts=$$(ls tests/$(SPEC)-*.sh 2>/dev/null); \
	else \
		scripts=$$(ls tests/[0-9]*.sh 2>/dev/null); \
	fi; \
	if [ -z "$$scripts" ]; then \
		echo "No integration test scripts found"; \
		exit 0; \
	fi; \
	failed=0; \
	for script in $$scripts; do \
		echo "Running $$script ..."; \
		if bash "$$script"; then \
			echo "PASS: $$script"; \
		else \
			echo "FAIL: $$script"; \
			failed=1; \
		fi; \
	done; \
	if [ "$$failed" -eq 1 ]; then \
		echo "One or more integration tests failed."; \
		exit 1; \
	else \
		echo "All integration tests passed."; \
	fi
```

Key points:
- The glob `tests/[0-9]*.sh` matches numbered test scripts and excludes `helpers.sh`.
- `make integration-test SPEC=401` runs only `tests/401-*.sh` — use this in per-story verification to avoid running tests from other stories.
- `make integration-test` (no filter) runs all tests — use this for end-to-end verification (SPEC-600).
- The `cargo build` step ensures binaries exist before any test runs.

**Running integration tests:**

```bash
# Run all integration tests (builds first)
make integration-test

# Run only tests for a specific story
make integration-test SPEC=401

# Run a single test (binary must already be built)
bash tests/401-dhcpv4-lease.sh
```

## Depends on
(none)

## Verification
In addition to `cargo test` and `cargo clippy`, run `make integration-test SPEC=001` to execute this story's shell integration tests. All tests must pass. This is a required verification step — the story is not complete until shell tests pass.

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

  Scenario: Makefile integration-test target builds and runs tests
    Given the Makefile exists
    When the developer runs "make integration-test"
    Then cargo build runs first
    And all tests/[0-9]*.sh scripts are discovered and executed
    And each test either passes (exit 0) or fails (exit non-zero)
    And the overall exit code is non-zero if any test failed

  Scenario: Test scripts never skip on missing prerequisites
    Given a test script that requires unshare, dnsmasq, or a built binary
    When the prerequisite is missing
    Then the script prints "FAIL: ..." to stderr and exits with code 1
    And never exits with code 0

  Scenario: Test script naming follows convention
    Given the workspace has integration test scripts
    When the test scripts in tests/ are listed
    Then each test script is named NNN-description.sh where NNN is a spec number
    And helpers.sh is the only non-numbered .sh file in tests/
```
