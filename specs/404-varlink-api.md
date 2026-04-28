# SPEC-404: Varlink API

## What
Define the Varlink interface for communication between the `netfyr` CLI and the `netfyr-daemon`. The interface provides methods to submit policies (with replace-all semantics), query current state, perform dry-run previews, and retrieve system overview information. The interface definition lives in the `netfyr-varlink` crate as both a `.varlink` interface file and generated Rust types.

## Why
The CLI and daemon need a well-defined IPC protocol. Varlink is a simple, JSON-based interface description language designed for system services on Linux. It uses Unix domain sockets, supports service discovery, and has a straightforward request/response model. Varlink is a natural fit for netfyr because it is the standard IPC protocol for modern Linux system services, integrates with systemd socket activation, and has Rust client/server libraries.

## User interaction
Users do not interact with the Varlink API directly. The CLI automatically detects whether the daemon is running (by attempting to connect to `/run/netfyr/netfyr.sock`) and routes `apply` and `query` commands through the API when the daemon is available.

## Implementation details
- Crate: `netfyr-varlink` (library)
- Files: `src/lib.rs`, `src/io.netfyr.varlink`, `src/client.rs`, `src/types.rs`
- Dependencies (external crates): `varlink` (Varlink protocol implementation), `serde` (serialization), `serde_json` (JSON encoding)

### Varlink interface definition (`src/io.netfyr.varlink`)

```varlink
interface io.netfyr

method SubmitPolicies(policies: []Policy) -> (report: ApplyReport)

method Query(selector: ?Selector) -> (entities: []State)

method DryRun(policies: []Policy) -> (diff: StateDiff)

method GetShowInfo() -> (info: ShowInfo)

type Policy (
    name: string,
    factory: string,
    priority: ?int,
    selector: ?Selector,
    state: ?StateDef,
    states: ?[]StateDef
)

type Selector (
    type: ?string,
    name: ?string,
    driver: ?string,
    mac: ?string,
    pci_path: ?string
)

type StateDef (
    entity_type: string,
    selector: Selector,
    fields: object
)

type State (
    entity_type: string,
    selector: Selector,
    fields: object
)

type ApplyReport (
    succeeded: int,
    failed: int,
    skipped: int,
    changes: []ChangeEntry,
    conflicts: []ConflictEntry
)

type ChangeEntry (
    kind: string,
    entity_type: string,
    entity_name: string,
    description: string,
    status: string
)

type ConflictEntry (
    entity_type: string,
    entity_name: string,
    field_name: string,
    policies: []string,
    values: []string
)

type StateDiff (
    operations: []DiffOperation
)

type DiffOperation (
    kind: string,
    entity_type: string,
    entity_name: string,
    field_changes: []FieldChange
)

type FieldChange (
    field_name: string,
    change_kind: string,
    current: ?object,
    desired: ?object
)

type ShowInfo (
    daemon: DaemonInfo,
    interfaces: []InterfaceInfo
)

type DaemonInfo (
    status: string,
    uptime_seconds: ?int
)

type InterfaceInfo (
    name: string,
    policies: ?[]PolicyInfo,
    dhcp: ?DhcpInfo
)

type PolicyInfo (
    name: string,
    type: string
)

type DhcpInfo (
    state: string,
    lease_address: ?string,
    lease_time_secs: ?int,
    lease_remaining_secs: ?int
)

error InvalidPolicy (reason: string)
error BackendError (reason: string)
error InternalError (reason: string)
```

### Client (`src/client.rs`)

The client is used by the CLI to communicate with the daemon.

```rust
pub struct VarlinkClient {
    connection: varlink::Connection,
}

impl VarlinkClient {
    /// Connect to the daemon's Varlink socket.
    /// Returns Err if the socket does not exist or connection is refused.
    pub fn connect(socket_path: &str) -> Result<Self>;

    /// Submit policies with replace-all semantics.
    /// The daemon replaces its entire policy set, reconciles, and applies.
    pub async fn submit_policies(&self, policies: Vec<Policy>) -> Result<ApplyReport>;

    /// Query current system state via the daemon.
    /// The selector may include a `type` field to filter by entity type.
    pub async fn query(
        &self,
        selector: Option<&Selector>,
    ) -> Result<StateSet>;

    /// Dry-run: compute what would change if these policies were submitted.
    pub async fn dry_run(&self, policies: Vec<Policy>) -> Result<StateDiff>;

    /// Get system overview (daemon status, interfaces, policies, DHCP state).
    pub async fn get_show_info(&self) -> Result<ShowInfo>;
}
```

### Socket path

The default Varlink socket path is `/run/netfyr/netfyr.sock`. Both the daemon and CLI read the `NETFYR_SOCKET_PATH` environment variable to override this path (see SPEC-403 for daemon, SPEC-301/302 for CLI). The override is essential for integration tests, which run in user namespaces where `/run/netfyr/` is not writable.

The socket is:
- Created by the daemon on startup, or by systemd socket activation.
- Used by the CLI for auto-detection: if `connect(socket_path)` succeeds, daemon mode is used.
- Cleaned up when the daemon shuts down (or by systemd).

### Method semantics

**SubmitPolicies**:
- Input: array of Policy objects (the complete desired policy set).
- Semantics: replace-all. The daemon discards its current policy set and adopts the submitted set.
- The daemon persists the new policies, syncs factories (stop removed DHCP, start new DHCP), reconciles, and applies.
- Returns: ApplyReport with the results of the apply operation.
- Errors: `InvalidPolicy` if any policy fails validation, `BackendError` if apply fails entirely.

**Query**:
- Input: optional selector filter. The selector may include a `type` field to filter by entity type (e.g., `type: "ethernet"`), alongside the usual name/driver/mac/pci_path filters.
- Semantics: queries the current system state via the backend. If a `type` is given, queries only that backend. Otherwise queries all backends.
- Returns: array of State objects matching the filters.
- Errors: `BackendError` if the backend query fails.

**DryRun**:
- Input: array of Policy objects (the hypothetical policy set).
- Semantics: computes what would change if these policies were submitted, without actually applying.
- The daemon reconciles the submitted policies (including any running factory states) against the current system state.
- Returns: StateDiff with the operations that would be performed.
- Errors: `InvalidPolicy`, `BackendError`.

**GetShowInfo**:
- Input: none.
- The daemon enumerates all system interfaces via the backend (`query_all()`), cross-references each with the policy store (to find policies targeting that interface) and the factory manager (to find DHCP state), and builds the `ShowInfo` response.
- Returns: `ShowInfo` with daemon status and per-interface details.
- `DaemonInfo.status` is always `"running"` when called via Varlink (the method is only reachable when the daemon is running). `uptime_seconds` is present when status is `"running"`.
- `InterfaceInfo.policies` is present (possibly empty array) for all interfaces when the daemon has policy data. The CLI sets it to absent when fabricating the response in daemon-free mode.
- `InterfaceInfo.dhcp` is present only for interfaces with a DHCP factory. `DhcpInfo.lease_address`, `lease_time_secs`, and `lease_remaining_secs` are present only when `state` is `"running"` (not `"waiting"`).
- Errors: `InternalError` on unexpected failures.

### Type mapping

Varlink types map to Rust types in the `netfyr-varlink` crate. The `types.rs` module provides conversion functions between Varlink JSON types and the core `netfyr-state` / `netfyr-policy` types:

```rust
impl From<Policy> for varlink::Policy { ... }
impl TryFrom<varlink::Policy> for Policy { ... }
impl From<StateSet> for Vec<varlink::State> { ... }
impl From<ApplyReport> for varlink::ApplyReport { ... }
impl From<StateDiff> for varlink::StateDiff { ... }
impl From<ShowInfo> for varlink::ShowInfo { ... }
```

### Error handling

- Connection refused / socket not found: The CLI treats this as "daemon not running" and falls back to daemon-free mode. This is not an error in the Varlink layer.
- `InvalidPolicy`: Returned when submitted policies fail validation (e.g., unknown factory type, missing required fields). The CLI prints the error and exits with code 2.
- `BackendError`: Returned when the backend (netlink) operation fails. The CLI prints the error details.
- `InternalError`: Unexpected daemon errors. The CLI prints a generic error message.

## Depends on
- SPEC-004 (StateSet type)
- SPEC-007 (Policy types)
- SPEC-203 (StateDiff type)

## Integration test infrastructure
Integration tests are shell scripts in `tests/`. Each script uses `unshare --user --net` to create an unprivileged network namespace, starts the daemon, submits policies via `netfyr apply`, and verifies state via `netfyr query`. No root required.

Shell test script rules (from SPEC-001):
- **Naming**: `404-description.sh` — the Makefile discovers tests via `tests/[0-9]*.sh`.
- **No skip**: if a prerequisite is missing (binary, `unshare`, `dnsmasq`), the script must `echo "FAIL: ..." >&2; exit 1`. Never `exit 0` on failure. Never wrap `netns_setup` or `start_dnsmasq` with `|| exit 0`.
- **Binary path**: locate binaries via `SCRIPT_DIR/../target/debug/netfyr` and `SCRIPT_DIR/../target/debug/netfyr-daemon` (overridable with `NETFYR_BIN` and `NETFYR_DAEMON_BIN` env vars). Check with `[[ ! -x "$NETFYR_BIN" ]]` and `exit 1` if missing.
- **Helpers**: source `tests/helpers.sh` which provides `netns_setup`, `create_veth`, `add_address`, `start_dnsmasq`, `cleanup`, and assertion functions.

## Verification
In addition to `cargo test` and `cargo clippy`, run `make integration-test SPEC=404` to execute this story's shell integration tests. All tests must pass. This is a required verification step — the story is not complete until shell tests pass.

## Acceptance criteria
```gherkin
Feature: Varlink API definition
  Scenario: Interface definition file is valid
    Given the file src/io.netfyr.varlink exists
    When it is parsed by a Varlink interface parser
    Then it parses without errors
    And it defines 4 methods: SubmitPolicies, Query, DryRun, GetShowInfo

  Scenario: Client connects to daemon socket
    Given the daemon is running and listening on /run/netfyr/netfyr.sock
    When VarlinkClient::connect("/run/netfyr/netfyr.sock") is called
    Then the connection succeeds

  Scenario: Client fails gracefully when daemon not running
    Given no daemon is running (socket does not exist)
    When VarlinkClient::connect("/run/netfyr/netfyr.sock") is called
    Then the connection returns Err

  Scenario: SubmitPolicies sends policies and receives report
    Given a connected VarlinkClient
    And 2 valid static policies
    When submit_policies is called
    Then the daemon receives both policies
    And an ApplyReport is returned with succeeded/failed/skipped counts

  Scenario: SubmitPolicies with invalid policy returns error
    Given a connected VarlinkClient
    And a policy with factory type "unknown_factory"
    When submit_policies is called
    Then an InvalidPolicy error is returned with a reason string

  Scenario: Query returns entity states
    Given a connected VarlinkClient
    When query(Some(Selector { type: "ethernet" })) is called
    Then a list of State objects is returned
    And each has entity_type, selector, and fields

  Scenario: Query with selector filters results
    Given a connected VarlinkClient
    And the system has interfaces eth0 and eth1
    When query(Some(Selector { name: "eth0" })) is called
    Then only eth0 is returned

  Scenario: DryRun returns diff without applying
    Given a connected VarlinkClient
    And policies that would change eth0 mtu from 1500 to 9000
    When dry_run is called
    Then a StateDiff is returned with a Modify operation for eth0
    And the system mtu is still 1500

  Scenario: GetShowInfo returns system overview
    Given a connected VarlinkClient
    And the daemon has been running for 60 seconds with 3 policies
    And the system has interfaces eth0, eth1, and lo
    And eth0 has a DHCP factory in running state with a lease
    When get_show_info is called
    Then the ShowInfo has daemon.status = "running"
    And daemon.uptime_seconds >= 60
    And interfaces has 3 elements
    And eth0's entry has a policies array and a dhcp object
    And lo's entry has no policies or dhcp

  Scenario: Type conversion roundtrip
    Given a netfyr-policy Policy object
    When converted to varlink::Policy and back
    Then the resulting Policy is identical to the original

  Scenario: ApplyReport conversion preserves all fields
    Given an ApplyReport with 2 succeeded, 1 failed, 0 skipped, and 1 conflict
    When converted to varlink::ApplyReport
    Then succeeded=2, failed=1, skipped=0
    And the conflict entry preserves entity, field, policies, and values

Feature: Integration tests for Varlink API (shell scripts)
  Scenario: Full round-trip in namespace
    Given a shell script running inside `unshare --user --net`
    And a veth pair and the daemon started inside the namespace
    When `netfyr apply policy.yaml` is run with a static policy setting mtu=1400
    Then `netfyr query -s name=veth-test0` shows mtu=1400
    And `ip link show veth-test0` confirms mtu 1400

  Scenario: Replace-all semantics via Varlink
    Given the daemon is running in a namespace with policy A (mtu=1400)
    When `netfyr apply policy-b.yaml` is run with policy B (mtu=1300)
    Then `ip link show veth-test0` shows mtu 1300
```
