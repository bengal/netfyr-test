# SPEC-102: rtnetlink Query for Ethernet Interfaces

## What
Implement the `NetworkBackend::query` method for ethernet interfaces using the `rtnetlink` crate. This implementation queries the Linux kernel via netlink sockets to discover ethernet interfaces and their current state, mapping kernel attributes to `State` fields. Supports querying all ethernet interfaces or filtering by selector criteria (name, driver, MAC address, etc.). All returned field values are marked with provenance `KernelDefault` since they represent the running system state as reported by the kernel.

## Why
The backend abstraction requires at least one concrete implementation to be useful. Ethernet interfaces are the most fundamental network entity type and the starting point for all networking configuration. The rtnetlink crate provides safe, async Rust bindings to the Linux netlink protocol, which is the standard kernel interface for network configuration on Linux. Querying the current system state is prerequisite to computing diffs and applying changes.

## User interaction
Users interact with this indirectly through CLI commands:
- `netfyr query ethernet` -- lists all ethernet interfaces and their state
- `netfyr query ethernet --selector name=eth0` -- queries a specific interface
- `netfyr apply` / `netfyr apply --dry-run` -- internally queries current state to compute diffs

The query result is a `StateSet` containing `State` entries that the CLI formats for display or the reconciliation engine uses for diff computation.

## Implementation details
- Crate: `netfyr-backend`
- Files: `src/netlink/mod.rs`, `src/netlink/ethernet.rs`, `src/netlink/query.rs`
- Dependencies (external crates): `rtnetlink` (async netlink interface), `netlink-packet-route` (netlink message types), `tokio` (async runtime), `futures` (stream processing)

### Module structure

**`src/netlink/mod.rs`**:
- Declares the `netlink` module with submodules `ethernet`, `query`.
- Exports `NetlinkBackend` struct.
- `NetlinkBackend` holds an `rtnetlink::Handle` (or creates one on demand from a new connection).
- Implements `NetworkBackend` for `NetlinkBackend`, initially supporting only `EntityType::Ethernet`.

**`src/netlink/query.rs`**:
- Contains shared query utilities used across entity types.
- `establish_connection() -> Result<(Connection, Handle, ...)>` -- creates a new netlink connection.
- `matches_selector(link_info: &LinkInfo, selector: &Selector) -> bool` -- checks if a discovered link matches the given selector criteria.
- Helper functions for extracting netlink message attributes.

**`src/netlink/ethernet.rs`**:
- `query_ethernet(handle: &Handle, selector: Option<&Selector>) -> Result<StateSet>` -- main query function.
- Flow:
  1. Use `handle.link().get().execute()` to list all links.
  2. Filter to only ethernet interfaces (check `ARPHRD_ETHER` link type, exclude virtual types like vlan/bridge/bond by checking `link_info` kind attribute).
  3. If selector is provided, further filter by matching criteria.
  4. For each matching link, extract attributes into `State` fields.
  5. Use `handle.address().get().execute()` to dump addresses, match by interface index.
  6. Use `handle.route().get(IpVersion::V4).execute()` and `handle.route().get(IpVersion::V6).execute()` to dump routes, match by output interface index.
  7. Build and return `StateSet`.

### Field mapping (netlink attributes to State fields)

| State field | Netlink source | Notes |
|---|---|---|
| `name` | `IFLA_IFNAME` from link message | String, always present |
| `mtu` | `IFLA_MTU` from link message | u32 |
| `mac` | `IFLA_ADDRESS` from link message | 6-byte hardware address, formatted as `aa:bb:cc:dd:ee:ff` |
| `carrier` | `IFLA_CARRIER` from link message | bool (1 = up, 0 = down) |
| `speed` | Read from `/sys/class/net/<name>/speed` | u32 in Mbps; may be unavailable if link is down, set to None |
| `driver` | `IFLA_INFO_KIND` or `/sys/class/net/<name>/device/driver` symlink | String, e.g., `e1000`, `ixgbe` |
| `operstate` | `IFLA_OPERSTATE` from link message | Enum: Up, Down, Dormant, etc. |
| `addresses` | From address dump (`RTM_GETADDR`), filtered by `ifa_index` | List of CIDR strings, e.g., `["10.0.1.50/24", "fe80::1/64"]` |
| `routes` | From route dump (`RTM_GETROUTE`), filtered by `oif` | List of route objects with destination, gateway, metric, scope |

### Selector matching

When a selector is provided, filter links based on:
- `name`: exact string match against `IFLA_IFNAME`
- `mac`: exact match against `IFLA_ADDRESS` (case-insensitive hex comparison)
- `driver`: exact match against driver name
- `pci-path`: match against `/sys/class/net/<name>/device` PCI path
- `labels`: not available from kernel; skip (labels come from policy metadata)

Multiple selector fields use AND logic (all specified criteria must match).

### Provenance

All fields returned from a kernel query are tagged with `Provenance::KernelDefault`. This distinguishes them from fields that were explicitly configured by a user policy (`Provenance::UserConfigured`) or detected from external tools (`Provenance::ExternalTool`).

### Error handling

- If the netlink connection fails, return `BackendError::QueryFailed`.
- If a specific interface requested by name does not exist, return `BackendError::NotFound`.
- If permission is denied (non-root querying restricted info), return `BackendError::PermissionDenied`.
- Individual attribute parse failures are logged as warnings; the field is omitted from the State rather than failing the entire query.

## Depends on
- SPEC-002 (State, field definitions, and provenance types in netfyr-state)
- SPEC-101 (NetworkBackend trait that this implements)

## Integration test infrastructure
Integration tests are shell scripts in `tests/`. Each script uses `unshare --user --net` to create an unprivileged network namespace, sets up veth pairs with known configuration via `ip` commands, runs `netfyr query` as a subprocess, and verifies the output. No root required.

Shell test script rules (from SPEC-001):
- **Naming**: `102-description.sh` — the Makefile discovers tests via `tests/[0-9]*.sh`.
- **No skip**: if a prerequisite is missing (binary, `unshare`, `dnsmasq`), the script must `echo "FAIL: ..." >&2; exit 1`. Never `exit 0` on failure. Never wrap `netns_setup` or `start_dnsmasq` with `|| exit 0`.
- **Binary path**: locate binaries via `SCRIPT_DIR/../target/debug/netfyr` (overridable with `NETFYR_BIN` env var). Check with `[[ ! -x "$NETFYR_BIN" ]]` and `exit 1` if missing.
- **Helpers**: source `tests/helpers.sh` which provides `netns_setup`, `create_veth`, `add_address`, `start_dnsmasq`, `cleanup`, and assertion functions.

## Verification
In addition to `cargo test` and `cargo clippy`, run `make integration-test` to execute all shell integration test scripts. All tests must pass. This is a required verification step — the story is not complete until shell tests pass.

## Acceptance criteria
```gherkin
Feature: Query ethernet interfaces via rtnetlink
  Scenario: Query all ethernet interfaces on a system with two NICs
    Given a Linux system with ethernet interfaces "eth0" and "eth1"
    And both interfaces are of type ARPHRD_ETHER
    When query is called with entity_type "ethernet" and no selector
    Then the result is Ok containing a StateSet with 2 entities
    And each entity has type "ethernet"
    And each entity has fields: name, mtu, mac

  Scenario: Query a specific ethernet interface by name
    Given a Linux system with ethernet interfaces "eth0" and "eth1"
    When query is called with entity_type "ethernet" and selector name="eth0"
    Then the result is Ok containing a StateSet with 1 entity
    And the entity has selector name "eth0"
    And the entity has the correct mtu, mac, and carrier values for eth0

  Scenario: Query ethernet interface includes IP addresses
    Given an ethernet interface "eth0" with addresses "10.0.1.50/24" and "fe80::1/64"
    When query is called for "eth0"
    Then the entity's "addresses" field contains ["10.0.1.50/24", "fe80::1/64"]
    And the addresses field has provenance KernelDefault

  Scenario: Query ethernet interface includes routes
    Given an ethernet interface "eth0" with a default route via "10.0.1.1"
    And a subnet route "10.0.1.0/24" directly connected
    When query is called for "eth0"
    Then the entity's "routes" field contains the default route and the connected route
    And each route has destination, gateway (if applicable), and metric fields

  Scenario: All queried fields have KernelDefault provenance
    Given an ethernet interface "eth0" with mtu 1500
    When query is called for "eth0"
    Then every field in the returned State has provenance KernelDefault

  Scenario: Query by MAC address selector
    Given ethernet interfaces "eth0" (mac aa:bb:cc:dd:ee:01) and "eth1" (mac aa:bb:cc:dd:ee:02)
    When query is called with selector mac="aa:bb:cc:dd:ee:02"
    Then the result contains exactly one entity with name "eth1"

  Scenario: Query by driver selector
    Given ethernet interface "eth0" using driver "ixgbe" and "eth1" using driver "e1000"
    When query is called with selector driver="ixgbe"
    Then the result contains exactly one entity with name "eth0"

  Scenario: Query with multiple selector fields uses AND logic
    Given ethernet interfaces "eth0" (driver: ixgbe, mac: aa:bb:cc:dd:ee:01) and "eth1" (driver: ixgbe, mac: aa:bb:cc:dd:ee:02)
    When query is called with selector driver="ixgbe" AND mac="aa:bb:cc:dd:ee:01"
    Then the result contains exactly one entity with name "eth0"

  Scenario: Query excludes non-ethernet interfaces
    Given interfaces "eth0" (ethernet), "br0" (bridge), "bond0" (bond), "vlan100" (vlan)
    When query is called with entity_type "ethernet" and no selector
    Then the result contains only "eth0"
    And bridge, bond, and vlan interfaces are excluded

  Scenario: Query for non-existent interface returns NotFound
    Given no interface named "eth99" exists on the system
    When query is called with selector name="eth99"
    Then the result is Err with BackendError::NotFound

  Scenario: Query handles interface with link down gracefully
    Given an ethernet interface "eth0" with carrier down and no link speed available
    When query is called for "eth0"
    Then the entity has carrier=false
    And the speed field is None (omitted)
    And other fields (name, mtu, mac) are still present

  Scenario: query_all includes all ethernet interfaces
    Given a NetlinkBackend that supports entity type "ethernet"
    When query_all is called
    Then the result includes all ethernet interfaces on the system
    And each has the same fields as an individual query would return

Feature: Integration tests for ethernet query (shell scripts)
  Scenario: Query veth interface in namespace
    Given a shell script running inside `unshare --user --net`
    And a veth pair "veth-test0"/"veth-test1" with "veth-test0" at mtu 1400 and address "10.99.0.1/24"
    When `netfyr query ethernet --selector name=veth-test0` is run
    Then the output shows veth-test0 with mtu=1400 and address 10.99.0.1/24

  Scenario: Query returns both ends of a veth pair
    Given a namespace with a veth pair "veth-a"/"veth-b"
    When `netfyr query ethernet` is run
    Then the output includes both veth-a and veth-b

  Scenario: Query by MAC address in namespace
    Given a namespace with a veth pair
    And the MAC address of "veth-test0" is captured via `ip link show`
    When `netfyr query ethernet --selector mac=$MAC` is run
    Then the output contains exactly veth-test0
```
