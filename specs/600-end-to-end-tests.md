# SPEC-600: End-to-End Integration Tests

## What
Add a suite of end-to-end shell integration tests that simulate real user workflows from start to finish. Unlike per-story tests (which verify a single feature in isolation), these tests exercise the full pipeline: writing YAML policies, starting the daemon, running `netfyr apply`, and verifying the resulting system state with both `netfyr query` and `ip` commands.

These tests catch integration issues that per-story tests miss — for example, DHCP and static policies coexisting on different interfaces, replace-all semantics across daemon interactions, policy persistence across daemon restarts, and conflict detection with real reconciliation.

## Why
Per-story shell tests verify that individual features work. But netfyr's value is in the interaction between features: reconciliation merges policies from multiple sources, the daemon manages factory lifecycles alongside static policies, replace-all semantics must correctly tear down old state, and conflict detection must surface real field collisions. These cross-cutting behaviors are only exercised when multiple features are used together in realistic scenarios.

End-to-end tests also serve as executable documentation of how a user is expected to interact with netfyr.

## User interaction
Developers run end-to-end tests the same way as all other shell integration tests:

```bash
# Run all integration tests (including end-to-end)
make integration-test

# Run a single end-to-end test
bash tests/600-e2e-static-apply.sh
```

## Implementation details
- Crate: (none — test-only story, no Rust code changes)
- Files: `tests/600-e2e-*.sh` (shell test scripts only)
- Dependencies (external crates): none

### Isolation model

End-to-end tests use the same isolation as per-story tests: `unshare --user --net` for network namespace isolation, plus temp directories for all daemon state (socket, policy store). No containers are needed because:
- Network isolation is provided by the user namespace.
- Filesystem isolation is achieved by configuring the daemon via environment variables (`NETFYR_SOCKET_PATH`, `NETFYR_POLICY_DIR`) to use temp directories instead of system paths (`/run/netfyr/`, `/var/lib/netfyr/`).
- The tests never touch real system paths or require root.

### Common test structure

Each test script follows this pattern:

```bash
#!/bin/bash
# 600-e2e-description.sh -- End-to-end: <what this tests>.
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
TMPDIR_TEST=$(mktemp -d)
trap 'kill "${DAEMON_PID:-}" 2>/dev/null; cleanup; rm -rf "$TMPDIR_TEST"' EXIT

SOCKET_PATH="$TMPDIR_TEST/netfyr.sock"
POLICY_DIR="$TMPDIR_TEST/policies"
mkdir -p "$POLICY_DIR"

# Set up network topology (veth pairs, addresses, dnsmasq as needed)
# ...

# Start daemon
NETFYR_SOCKET_PATH="$SOCKET_PATH" \
NETFYR_POLICY_DIR="$POLICY_DIR" \
    "$NETFYR_DAEMON_BIN" &
DAEMON_PID=$!

# Wait for daemon socket
for i in $(seq 1 50); do
    [[ -S "$SOCKET_PATH" ]] && break
    sleep 0.1
done
if [[ ! -S "$SOCKET_PATH" ]]; then
    echo "FAIL: daemon socket did not appear" >&2; exit 1
fi

# Write policies, apply, verify
# ...

echo "PASS: 600-e2e-description"
```

### Test scenarios

#### 1. Static policy apply and query (`600-e2e-static-apply.sh`)
Simulates a user configuring a static interface:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Write a static policy YAML setting `veth-e2e0` to mtu=1400 and address `10.99.0.1/24`.
3. Start the daemon.
4. Run `netfyr apply` with the policy.
5. Verify with `ip link show veth-e2e0` that mtu is 1400.
6. Verify with `ip addr show veth-e2e0` that the address is present.
7. Run `netfyr query -s name=veth-e2e0 -o json` and verify the JSON output contains mtu=1400 and the address.

#### 2. DHCP and static coexistence (`600-e2e-dhcp-and-static.sh`)
Simulates a system with a DHCP-configured interface alongside a statically configured one:
1. Create two veth pairs: `veth-static0`/`veth-static1` and `veth-dhcp0`/`veth-dhcp1`.
2. Configure `veth-dhcp1` with address `10.99.1.1/24` and start dnsmasq serving `10.99.1.100-10.99.1.200`.
3. Write two policy files: a static policy for `veth-static0` (mtu=1400, address `10.99.0.1/24`) and a DHCP policy for `veth-dhcp0`.
4. Start the daemon and apply both policies.
5. Wait for the DHCP lease (poll up to 10 s).
6. Verify `veth-static0` has mtu=1400 and address `10.99.0.1/24`.
7. Verify `veth-dhcp0` has an address in the `10.99.1.0/24` range.
8. Verify neither interface's configuration interferes with the other.

#### 3. Replace-all semantics (`600-e2e-replace-all.sh`)
Simulates a user changing their policy set:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Start the daemon.
3. Apply policy A: `veth-e2e0` mtu=1400, address `10.99.0.1/24`.
4. Verify mtu=1400 and address present.
5. Apply policy B: `veth-e2e0` mtu=1300 (no address field).
6. Verify mtu is now 1300.
7. Verify the address `10.99.0.1/24` has been removed (replace-all: policy A is gone, so its address is no longer desired).

#### 4. Policy persistence across daemon restart (`600-e2e-daemon-restart.sh`)
Simulates a daemon restart (e.g., `systemctl restart netfyr`):
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Start the daemon with a persistent policy directory.
3. Apply a static policy setting mtu=1400.
4. Verify mtu=1400.
5. Kill the daemon (`kill $DAEMON_PID`), wait for it to exit.
6. Start a new daemon instance using the same policy directory and socket path.
7. Wait for the new daemon to come up.
8. Verify mtu is still 1400 (daemon reloaded persisted policy and re-applied).

#### 5. Conflict detection (`600-e2e-conflict.sh`)
Simulates two policies competing for the same field:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Write two policies for `veth-e2e0` at the same priority (100):
   - Policy A: mtu=1400.
   - Policy B: mtu=1300.
3. Start the daemon and apply both policies.
4. Capture the output of `netfyr apply`.
5. Verify the output contains a conflict warning mentioning `mtu`.
6. Verify the exit code is 1 (partial success due to conflict).

#### 6. Dry-run does not modify state (`600-e2e-dry-run.sh`)
Simulates a user previewing changes:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Start the daemon.
3. Run `netfyr apply --dry-run policy.yaml` with a policy setting mtu=1400.
4. Verify `ip link show veth-e2e0` still shows the default mtu (not 1400).
5. Verify the dry-run output mentions the mtu change that would be applied.

#### 7. Apply directory of policies (`600-e2e-apply-directory.sh`)
Simulates a user with a policy directory (like `/etc/netfyr/policies/`):
1. Create two veth pairs: `veth-a0`/`veth-a1` and `veth-b0`/`veth-b1`.
2. Write two YAML files in a directory: one for `veth-a0` (mtu=1400) and one for `veth-b0` (mtu=1300).
3. Start the daemon.
4. Run `netfyr apply $POLICY_DIR/` (passing the directory).
5. Verify `veth-a0` has mtu=1400 and `veth-b0` has mtu=1300.

#### 8. Single address applied and preserved (`600-e2e-addr-single.sh`)
Verifies basic address application:
1. Create veth pair `veth-addr0`/`veth-addr1`.
2. Start the daemon.
3. Write a policy with one address: `10.99.0.1/24`.
4. Run `netfyr apply`.
5. Verify with `ip addr show veth-addr0` that `10.99.0.1/24` is present.
6. Verify with `netfyr query -s name=veth-addr0 -o json` that the addresses list contains exactly `["10.99.0.1/24"]`.

#### 9. Five addresses applied in order (`600-e2e-addr-five.sh`)
Verifies that multiple addresses are applied and their order is preserved:
1. Create veth pair `veth-addr0`/`veth-addr1`.
2. Start the daemon.
3. Write a policy with 5 addresses: `10.99.0.1/24`, `10.99.0.2/24`, `10.99.0.3/24`, `10.99.0.4/24`, `10.99.0.5/24`.
4. Run `netfyr apply`.
5. Verify with `ip addr show veth-addr0` that all 5 addresses are present.
6. Verify with `netfyr query -s name=veth-addr0 -o json` that the addresses list contains all 5 addresses in the same order as the YAML.

#### 10. Twenty addresses applied in order (`600-e2e-addr-twenty.sh`)
Stress test with many addresses:
1. Create veth pair `veth-addr0`/`veth-addr1`.
2. Start the daemon.
3. Write a policy with 20 addresses: `10.99.0.1/24` through `10.99.0.20/24`.
4. Run `netfyr apply`.
5. Verify with `ip addr show veth-addr0` that all 20 addresses are present.
6. Verify with `netfyr query -s name=veth-addr0 -o json` that the addresses list contains all 20 addresses in the same order as the YAML.

#### 11. Address replacement (`600-e2e-addr-replace.sh`)
Verifies that replacing the address set works correctly:
1. Create veth pair `veth-addr0`/`veth-addr1`.
2. Start the daemon.
3. Apply policy A with 5 addresses: `10.99.0.1/24` through `10.99.0.5/24`.
4. Verify all 5 are present.
5. Apply policy B with 5 different addresses: `10.99.1.1/24` through `10.99.1.5/24`.
6. Verify with `ip addr show` that none of the old addresses (`10.99.0.*`) are present.
7. Verify with `netfyr query` that the new 5 addresses are present in the correct order.

#### 12. Idempotent address re-apply (`600-e2e-addr-idempotent.sh`)
Verifies that applying the same addresses twice is clean:
1. Create veth pair `veth-addr0`/`veth-addr1`.
2. Start the daemon.
3. Apply a policy with 5 addresses: `10.99.0.1/24` through `10.99.0.5/24`.
4. Verify all 5 are present in order.
5. Apply the exact same policy again.
6. Verify all 5 are still present in the same order.
7. Verify no duplicates (each address appears exactly once in `ip addr show`).

#### 13. Duplicate address in YAML is rejected (`600-e2e-addr-duplicate-reject.sh`)
Verifies that duplicate addresses in YAML produce a validation error:
1. Create veth pair `veth-addr0`/`veth-addr1`.
2. Start the daemon.
3. Write a policy with duplicate addresses: `10.99.0.1/24`, `10.99.0.2/24`, `10.99.0.1/24`.
4. Run `netfyr apply`.
5. Verify the exit code is non-zero (2 for validation error).
6. Verify the error output mentions duplicate address.
7. Verify `ip addr show veth-addr0` has no addresses (nothing was applied).

#### 14. Overlapping subnets (`600-e2e-addr-overlapping-subnets.sh`)
Verifies that multiple addresses on different subnets work:
1. Create veth pair `veth-addr0`/`veth-addr1`.
2. Start the daemon.
3. Write a policy with addresses on different subnets: `10.99.0.1/24`, `10.99.1.1/24`, `10.99.2.1/24`.
4. Run `netfyr apply`.
5. Verify all 3 addresses are present with `ip addr show`.
6. Verify order is preserved via `netfyr query`.

#### 15. Address removal via replace-all (`600-e2e-addr-removal.sh`)
Verifies that addresses are cleaned up when a policy with no addresses replaces one that had them:
1. Create veth pair `veth-addr0`/`veth-addr1`.
2. Start the daemon.
3. Apply a policy with mtu=1400 and 3 addresses: `10.99.0.1/24`, `10.99.0.2/24`, `10.99.0.3/24`.
4. Verify all 3 addresses are present.
5. Apply a new policy with only mtu=1400 (no addresses field).
6. Verify all 3 addresses have been removed from the interface.
7. Verify mtu is still 1400 (only addresses were removed).

#### 16. Unmanaged interfaces are untouched (`600-e2e-unmanaged.sh`)
Verifies that interfaces not targeted by any policy are left alone:
1. Create three veth pairs: `veth-managed0`/`veth-managed1`, `veth-other0`/`veth-other1`, `veth-dhcp0`/`veth-dhcp1`.
2. Manually configure `veth-other0` with mtu=1400 and address `10.99.2.1/24` using `ip` commands.
3. Start dnsmasq on `veth-dhcp1`.
4. Apply policies only for `veth-managed0` (static) and `veth-dhcp0` (DHCP).
5. Wait for DHCP lease.
6. Verify `veth-other0` still has mtu=1400 and address `10.99.2.1/24` — completely untouched.

## Depends on
- SPEC-001 (test infrastructure, helpers.sh, Makefile)
- SPEC-006 (schema validation — needed for duplicate address rejection test)
- SPEC-103 (backend apply — needed for static policies to take effect)
- SPEC-201 (reconciliation — merges multiple policies)
- SPEC-202 (conflict detection — needed for conflict test)
- SPEC-203 (diff generation — computes changes)
- SPEC-301 (CLI apply — the command being tested)
- SPEC-302 (CLI query — used for verification)
- SPEC-401 (DHCP factory — needed for DHCP tests)
- SPEC-402 (policy store — needed for persistence test)
- SPEC-403 (daemon — needed for all daemon-mode tests)

## Integration test infrastructure
This story produces only shell test scripts — no Rust code. All tests are shell scripts in `tests/`.

Shell test script rules (from SPEC-001):
- **Naming**: `600-e2e-description.sh` — the Makefile discovers tests via `tests/[0-9]*.sh`.
- **No skip**: if a prerequisite is missing (binary, `unshare`, `dnsmasq`), the script must `echo "FAIL: ..." >&2; exit 1`. Never `exit 0` on failure. Never wrap `netns_setup` or `start_dnsmasq` with `|| exit 0`.
- **Binary path**: locate binaries via `SCRIPT_DIR/../target/debug/netfyr` and `SCRIPT_DIR/../target/debug/netfyr-daemon` (overridable with `NETFYR_BIN` and `NETFYR_DAEMON_BIN` env vars). Check with `[[ ! -x "$NETFYR_BIN" ]]` and `exit 1` if missing.
- **Helpers**: source `tests/helpers.sh` which provides `netns_setup`, `create_veth`, `add_address`, `start_dnsmasq`, `cleanup`, and assertion functions.

## Verification
Run `make integration-test` to execute all shell integration test scripts. All tests must pass. This is a required verification step — the story is not complete until shell tests pass. Since this story produces no Rust code, `cargo test` is still run (to verify no regressions) but the primary verification is `make integration-test`.

## Acceptance criteria
```gherkin
Feature: End-to-end static policy workflow
  Scenario: Apply static policy and verify with ip and netfyr query
    Given a namespace with a veth pair "veth-e2e0"/"veth-e2e1"
    And the daemon is running
    And a YAML policy setting veth-e2e0 mtu=1400 and address 10.99.0.1/24
    When `netfyr apply policy.yaml` is run
    Then `ip link show veth-e2e0` shows mtu 1400
    And `ip addr show veth-e2e0` shows 10.99.0.1/24
    And `netfyr query -s name=veth-e2e0 -o json` includes mtu=1400 and the address

Feature: End-to-end DHCP and static coexistence
  Scenario: DHCP and static policies on different interfaces do not interfere
    Given a namespace with veth pairs for static and DHCP
    And dnsmasq serving 10.99.1.100-10.99.1.200 on the DHCP server end
    And the daemon is running
    And a static policy for veth-static0 and a DHCP policy for veth-dhcp0
    When both policies are applied
    Then veth-static0 has mtu=1400 and address 10.99.0.1/24
    And veth-dhcp0 has an address in the 10.99.1.0/24 range
    And neither interface's configuration is affected by the other

Feature: End-to-end replace-all semantics
  Scenario: Applying a new policy set removes the old one
    Given a namespace with a veth pair and the daemon running
    And policy A is applied (mtu=1400, address 10.99.0.1/24)
    And mtu=1400 and the address are confirmed present
    When policy B is applied (mtu=1300, no address)
    Then mtu is 1300
    And address 10.99.0.1/24 is no longer present on the interface

Feature: End-to-end policy persistence across daemon restart
  Scenario: Daemon restart preserves applied configuration
    Given a namespace with a veth pair and the daemon running
    And a static policy setting mtu=1400 is applied and verified
    When the daemon is killed and a new instance is started with the same policy directory
    Then mtu is still 1400 after the new daemon starts

Feature: End-to-end conflict detection
  Scenario: Conflicting policies produce a warning
    Given a namespace with a veth pair and the daemon running
    And two policies for the same interface at the same priority with different mtu values
    When both policies are applied via `netfyr apply`
    Then the output contains a conflict warning mentioning mtu
    And the exit code is 1

Feature: End-to-end dry-run
  Scenario: Dry-run shows changes without applying them
    Given a namespace with a veth pair and the daemon running
    When `netfyr apply --dry-run policy.yaml` is run with a policy setting mtu=1400
    Then the output mentions the mtu change
    And `ip link show` still shows the original mtu (not 1400)

Feature: End-to-end apply directory
  Scenario: Applying a directory of policies configures multiple interfaces
    Given a namespace with two veth pairs and the daemon running
    And a directory containing policies for veth-a0 (mtu=1400) and veth-b0 (mtu=1300)
    When `netfyr apply $DIR/` is run
    Then veth-a0 has mtu=1400 and veth-b0 has mtu=1300

Feature: End-to-end single address
  Scenario: Apply a single address and verify it is present
    Given a namespace with a veth pair "veth-addr0"/"veth-addr1" and the daemon running
    And a YAML policy setting veth-addr0 addresses=["10.99.0.1/24"]
    When `netfyr apply policy.yaml` is run
    Then `ip addr show veth-addr0` shows 10.99.0.1/24
    And `netfyr query -s name=veth-addr0 -o json` shows addresses=["10.99.0.1/24"]

Feature: End-to-end five addresses in order
  Scenario: Apply five addresses and verify order is preserved
    Given a namespace with a veth pair "veth-addr0"/"veth-addr1" and the daemon running
    And a YAML policy setting veth-addr0 addresses=["10.99.0.1/24", "10.99.0.2/24", "10.99.0.3/24", "10.99.0.4/24", "10.99.0.5/24"]
    When `netfyr apply policy.yaml` is run
    Then `ip addr show veth-addr0` shows all 5 addresses
    And `netfyr query -s name=veth-addr0 -o json` returns addresses in the same order as the YAML

Feature: End-to-end twenty addresses in order
  Scenario: Apply twenty addresses and verify order is preserved
    Given a namespace with a veth pair "veth-addr0"/"veth-addr1" and the daemon running
    And a YAML policy setting veth-addr0 addresses=["10.99.0.1/24" through "10.99.0.20/24"]
    When `netfyr apply policy.yaml` is run
    Then `ip addr show veth-addr0` shows all 20 addresses
    And `netfyr query -s name=veth-addr0 -o json` returns all 20 addresses in the same order as the YAML

Feature: End-to-end address replacement
  Scenario: Replacing all addresses removes old ones and adds new ones in order
    Given a namespace with a veth pair and the daemon running
    And policy A is applied with addresses=["10.99.0.1/24" through "10.99.0.5/24"]
    And all 5 addresses are confirmed present
    When policy B is applied with addresses=["10.99.1.1/24" through "10.99.1.5/24"]
    Then none of the 10.99.0.* addresses are present
    And all 5 new 10.99.1.* addresses are present in the correct order

Feature: End-to-end idempotent address re-apply
  Scenario: Applying the same addresses twice produces no duplicates
    Given a namespace with a veth pair and the daemon running
    And a policy with addresses=["10.99.0.1/24" through "10.99.0.5/24"] is applied
    And all 5 are confirmed present in order
    When the exact same policy is applied again
    Then all 5 addresses are still present in the same order
    And no address appears more than once in `ip addr show`

Feature: End-to-end duplicate address rejection
  Scenario: Duplicate addresses in YAML produce a validation error
    Given a namespace with a veth pair and the daemon running
    And a YAML policy with addresses=["10.99.0.1/24", "10.99.0.2/24", "10.99.0.1/24"]
    When `netfyr apply policy.yaml` is run
    Then the exit code is 2 (validation error)
    And the error output mentions duplicate address "10.99.0.1/24"
    And `ip addr show veth-addr0` has no addresses (nothing was applied)

Feature: End-to-end overlapping subnets
  Scenario: Addresses on different subnets coexist on the same interface
    Given a namespace with a veth pair and the daemon running
    And a YAML policy with addresses=["10.99.0.1/24", "10.99.1.1/24", "10.99.2.1/24"]
    When `netfyr apply policy.yaml` is run
    Then all 3 addresses are present on the interface
    And `netfyr query` returns them in the same order as the YAML

Feature: End-to-end address removal via replace-all
  Scenario: Replacing a policy that has addresses with one that has none removes all addresses
    Given a namespace with a veth pair and the daemon running
    And a policy with mtu=1400 and addresses=["10.99.0.1/24", "10.99.0.2/24", "10.99.0.3/24"] is applied
    And all 3 addresses are confirmed present
    When a new policy with only mtu=1400 (no addresses) is applied
    Then all 3 addresses are removed from the interface
    And mtu is still 1400

Feature: End-to-end unmanaged interfaces
  Scenario: Interfaces without policies are not modified
    Given a namespace with three veth pairs and the daemon running
    And veth-other0 is manually configured with mtu=1400 and address 10.99.2.1/24
    And policies exist only for veth-managed0 (static) and veth-dhcp0 (DHCP)
    When policies are applied and the DHCP lease is acquired
    Then veth-other0 still has mtu=1400 and address 10.99.2.1/24
```
