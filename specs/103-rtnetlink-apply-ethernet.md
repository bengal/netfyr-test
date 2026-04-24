# SPEC-103: rtnetlink Apply for Ethernet Interfaces

## What
Implement the `NetworkBackend::apply` method for ethernet interfaces using the `rtnetlink` crate. This translates `StateDiff` operations (Add, Modify, Remove) into netlink requests that modify the running kernel configuration. Each operation is executed individually, and the results are collected into an `ApplyReport` indicating which operations succeeded, which failed (with error details), and which were skipped.

## Why
Applying configuration changes is the core purpose of netfyr. After the reconciliation engine computes the desired state and the diff engine determines what needs to change, the backend must execute those changes against the Linux kernel. The rtnetlink crate provides the async Rust interface to do this safely. Detailed per-operation reporting enables partial failure handling (continue-and-report mode) and gives users clear feedback about what happened.

## User interaction
Users trigger apply indirectly through:
- `netfyr apply <path>...` -- applies policies, which internally generates a diff and calls backend apply
- `netfyr apply --dry-run <path>...` -- calls dry_run instead of apply

The output shows the `ApplyReport` summary: how many operations succeeded, failed, or were skipped, with per-operation details on failure.

## Implementation details
- Crate: `netfyr-backend`
- Files: `src/netlink/apply.rs`
- Dependencies (external crates): `rtnetlink`, `netlink-packet-route`, `tokio`

### Operation mapping

The `StateDiff` contains a list of `DiffOperation` entries. Each has an operation kind, entity type, selector, and field changes. For ethernet interfaces:

**Add operation** (rare for physical ethernet -- mainly for setting up an unconfigured interface):
1. Look up the interface by name using `handle.link().get().match_name(name).execute()`.
2. If the interface does not exist and it is a physical device, return an error (cannot create physical ethernet interfaces).
3. If the interface exists but is unconfigured, set it up:
   - Set link up: `handle.link().set(index).up().execute()`
   - Apply field values (mtu, addresses, routes) as in Modify.

**Modify operation** (the common case):
1. Look up the interface index by name.
2. For each field change in the operation:
   - `mtu`: `handle.link().set(index).mtu(value).execute()`
   - `addresses` (add): For each new address, in the order they appear in the desired state list, `handle.address().add(index, address, prefix_len).execute()`. Order is significant: the kernel assigns addresses in the order they are added, and the first address becomes the primary (source) address. Only IPv4 addresses are supported.
   - `addresses` (remove): For each removed address, `handle.address().del(index, address, prefix_len).execute()`
   - `routes` (add): For each new route, `handle.route().add().output_interface(index).destination_prefix(dst, len).gateway(gw).execute()`
   - `routes` (remove): For each removed route, look up the route and delete it via `handle.route().del(route_message).execute()`
   - `operstate` (up/down): `handle.link().set(index).up().execute()` or `.down()`
3. Read-only fields (`carrier`, `speed`, `mac`, `driver`) in the diff are skipped with a reason "read-only field". This is a defensive check — normally SPEC-203 excludes read-only fields from the diff, so this path should not trigger in practice.

**Remove operation** (deconfigure an interface):
1. Look up the interface index by name.
2. Remove all addresses: iterate current addresses, delete each.
3. Remove all routes associated with the interface.
4. Set link down: `handle.link().set(index).down().execute()`
5. Note: physical ethernet interfaces are not deleted from the system -- only deconfigured. The "remove" operation removes the netfyr-managed configuration, not the hardware.

### dry_run implementation

The `dry_run` method performs the same logic as `apply` but without executing any netlink requests. Instead, it:
1. Validates that the target interface exists (by querying).
2. For each operation, builds the list of `PlannedChange` entries describing what would happen.
3. Returns a `DryRunReport`.

### Error handling per operation

Each operation is attempted independently. If one fails:
- The error is captured in `FailedOperation` with the specific `BackendError`.
- Subsequent operations continue (continue-and-report mode).
- Common errors: `ENODEV` (interface not found), `EPERM` (permission denied), `EEXIST` (address already exists), `ESRCH` (route not found for deletion).

### Idempotency

Operations should be idempotent where possible:
- Adding an address that already exists: skip with reason "already present" (do not fail).
- Removing an address that does not exist: skip with reason "not present" (do not fail).
- Removing a route that does not exist: treat as a successful operation — the desired state (route absent) is already achieved. This appears in `ApplyReport.succeeded`, not `skipped`. This commonly occurs when the kernel automatically removes implied routes (e.g., prefix route, local and broadcast host routes) as a side effect of address removal earlier in the same apply operation.
- Setting MTU to its current value: skip with reason "already at desired value".
- Skipped operations appear in `ApplyReport.skipped`, not `failed`.

### Ordering within a single entity

When modifying an entity, field changes are applied in a specific order:
1. Link-level changes first (mtu, link up/down).
2. Address changes second. When replacing the address set: remove old addresses first, then add new addresses in the order they appear in the desired state list. This ensures the kernel's address order matches the YAML order, and the first address in the list becomes the primary (source) address.
3. Route changes last (since routes depend on addresses for next-hop resolution).

## Depends on
- SPEC-101 (NetworkBackend trait, ApplyReport, DryRunReport, StateDiff types)
- SPEC-102 (NetlinkBackend struct and query infrastructure that apply builds upon)

## Integration test infrastructure
Integration tests are shell scripts in `tests/`. Each script uses `unshare --user --net` to create an unprivileged network namespace, sets up veth pairs, runs `netfyr apply` with YAML policies, and verifies the result with `ip` commands. No root required.

Shell test script rules (from SPEC-001):
- **Naming**: `103-description.sh` — the Makefile discovers tests via `tests/[0-9]*.sh`.
- **No skip**: if a prerequisite is missing (binary, `unshare`, `dnsmasq`), the script must `echo "FAIL: ..." >&2; exit 1`. Never `exit 0` on failure. Never wrap `netns_setup` or `start_dnsmasq` with `|| exit 0`.
- **Binary path**: locate binaries via `SCRIPT_DIR/../target/debug/netfyr` (overridable with `NETFYR_BIN` env var). Check with `[[ ! -x "$NETFYR_BIN" ]]` and `exit 1` if missing.
- **Helpers**: source `tests/helpers.sh` which provides `netns_setup`, `create_veth`, `add_address`, `start_dnsmasq`, `cleanup`, and assertion functions.

## Verification
In addition to `cargo test` and `cargo clippy`, run `make integration-test SPEC=103` to execute this story's shell integration tests. All tests must pass. This is a required verification step — the story is not complete until shell tests pass.

## Acceptance criteria
```gherkin
Feature: Apply ethernet configuration via rtnetlink
  Scenario: Modify MTU on an existing ethernet interface
    Given an ethernet interface "eth0" with current mtu 1500
    And a StateDiff with a Modify operation setting mtu to 9000
    When apply is called with the diff
    Then the ApplyReport has 1 succeeded operation
    And the system interface "eth0" now has mtu 9000

  Scenario: Add an IP address to an ethernet interface
    Given an ethernet interface "eth0" with no addresses
    And a StateDiff with a Modify operation adding address "10.0.1.50/24"
    When apply is called with the diff
    Then the ApplyReport has 1 succeeded operation
    And "eth0" now has address "10.0.1.50/24"

  Scenario: Remove an IP address from an ethernet interface
    Given an ethernet interface "eth0" with address "10.0.1.50/24"
    And a StateDiff with a Modify operation removing address "10.0.1.50/24"
    When apply is called with the diff
    Then the ApplyReport has 1 succeeded operation
    And "eth0" no longer has address "10.0.1.50/24"

  Scenario: Add a route via an ethernet interface
    Given an ethernet interface "eth0" with address "10.0.1.50/24"
    And a StateDiff with a Modify operation adding route destination="0.0.0.0/0" gateway="10.0.1.1"
    When apply is called
    Then the ApplyReport has 1 succeeded operation
    And a default route via "10.0.1.1" exists on "eth0"

  Scenario: Remove a route from an ethernet interface
    Given a default route via "10.0.1.1" on "eth0"
    And a StateDiff with a Modify operation removing that route
    When apply is called
    Then the ApplyReport has 1 succeeded operation
    And the default route via "10.0.1.1" no longer exists

  Scenario: Modify operation skips read-only fields (defensive)
    Given a StateDiff with a Modify operation on "eth0" that includes field changes for "carrier" and "speed"
    When apply is called
    Then those field changes are in the skipped list with reason "read-only field"
    And the ApplyReport does not report them as failures
    # Note: this is a defensive check. SPEC-203 requires read-only fields
    # to be excluded from the diff, so the backend should not normally
    # receive them.

  Scenario: Adding an already-existing address is idempotent
    Given an ethernet interface "eth0" with address "10.0.1.50/24"
    And a StateDiff with a Modify operation adding address "10.0.1.50/24"
    When apply is called
    Then the operation is in the skipped list with reason "already present"
    And is_success() returns true (no failures)

  Scenario: Removing a non-existent address is idempotent
    Given an ethernet interface "eth0" with no addresses
    And a StateDiff with a Modify operation removing address "10.0.1.50/24"
    When apply is called
    Then the operation is in the skipped list with reason "not present"
    And is_success() returns true (no failures)

  Scenario: Removing a non-existent route counts as success
    Given an ethernet interface "eth0" with address "10.0.1.50/24" and no routes
    And a StateDiff with a Modify operation removing route destination="10.0.0.0/24" gateway="10.0.1.1"
    When apply is called
    Then the operation is in the succeeded list (not skipped or failed)
    And is_success() returns true

  Scenario: Apply to a non-existent interface reports failure
    Given no interface named "eth99" exists
    And a StateDiff with a Modify operation on "eth99"
    When apply is called
    Then the ApplyReport has 1 failed operation
    And the error is BackendError::NotFound for "eth99"

  Scenario: Multiple operations with partial failure
    Given ethernet interfaces "eth0" (exists) and "eth99" (does not exist)
    And a StateDiff with Modify operations on both interfaces
    When apply is called
    Then the ApplyReport has 1 succeeded (eth0) and 1 failed (eth99)
    And is_partial() returns true

  Scenario: Remove operation deconfigures but does not delete physical interface
    Given an ethernet interface "eth0" with address "10.0.1.50/24" and default route
    And a StateDiff with a Remove operation for "eth0"
    When apply is called
    Then all addresses are removed from "eth0"
    And all routes through "eth0" are removed
    And "eth0" is set to link down
    And the interface "eth0" still exists in the system (not deleted)

  Scenario: Field changes within an entity are applied in correct order
    Given an ethernet interface "eth0" with mtu 1500 and no addresses
    And a StateDiff with a Modify operation setting mtu=9000, adding address "10.0.1.50/24", and adding route "0.0.0.0/0 via 10.0.1.1"
    When apply is called
    Then mtu is set before addresses are added
    And addresses are added before routes are added
    And all operations succeed

  Scenario: Dry-run reports planned changes without modifying the system
    Given an ethernet interface "eth0" with mtu 1500
    And a StateDiff with a Modify operation setting mtu to 9000
    When dry_run is called with the diff
    Then the DryRunReport contains a PlannedChange for "eth0"
    And the PlannedChange shows mtu changing from 1500 to 9000
    And the actual system mtu of "eth0" is still 1500

  Scenario: Dry-run validates that target interface exists
    Given no interface named "eth99" exists
    And a StateDiff with a Modify operation on "eth99"
    When dry_run is called
    Then the DryRunReport indicates the operation would fail with NotFound

  Scenario: Apply with permission denied
    Given the process is running as a non-root user outside any namespace
    And a StateDiff that requires modifying interface "eth0"
    When apply is called
    Then the ApplyReport has 1 failed operation
    And the error is BackendError::PermissionDenied

Feature: Integration tests for ethernet apply (shell scripts)
  Scenario: Set MTU on a veth interface in namespace
    Given a shell script running inside `unshare --user --net`
    And a veth pair "veth-test0"/"veth-test1" with default mtu 1500
    And a YAML policy setting veth-test0 mtu to 1400
    When `netfyr apply policy.yaml` is run
    Then `ip link show veth-test0` shows mtu 1400

  Scenario: Add and remove IP addresses in namespace
    Given a namespace with a veth pair "veth-test0"/"veth-test1"
    And a YAML policy adding address "10.99.0.1/24" to veth-test0
    When `netfyr apply policy.yaml` is run
    Then `ip addr show veth-test0` includes "10.99.0.1/24"
    When a second policy without the address is applied
    Then "10.99.0.1/24" is no longer present on veth-test0

  Scenario: Add a route in namespace
    Given a namespace with a veth pair and veth-test0 at "10.99.0.1/24" link up
    And a YAML policy adding route "10.100.0.0/24 via 10.99.0.2" to veth-test0
    When `netfyr apply policy.yaml` is run
    Then `ip route` shows the route in the namespace routing table

  Scenario: Full round-trip: apply then query
    Given a namespace with a veth pair "veth-test0"/"veth-test1"
    And a YAML policy setting mtu=1400 and address "10.99.0.1/24"
    When `netfyr apply policy.yaml` is run
    And `netfyr query ethernet --selector name=veth-test0` is run
    Then the query output shows mtu=1400 and address "10.99.0.1/24"
```
