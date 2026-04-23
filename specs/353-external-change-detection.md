# SPEC-353: External Change Detection

## What

Add a netlink monitor to the daemon that detects network state changes made by other tools (e.g., `ip link set`, NetworkManager) or by the kernel itself (e.g., carrier loss, link state changes). When an external change is detected, the daemon records a journal entry with `Trigger::ExternalChange` but does not re-reconcile or revert the change. The operator decides what to do.

## Why

In a managed environment, changes can come from multiple sources: netfyr policies, DHCP leases, manual operator commands, other network management tools, or the kernel itself (carrier detection, hotplug events). Without external change detection, the journal (SPEC-351) only records changes initiated by netfyr. This creates blind spots -- an operator investigating "why is eth0's MTU wrong?" would find no journal entry if the change was made by another tool.

Recording external changes in the journal provides a complete audit trail regardless of the change source. This is especially valuable for:
- Incident investigation: "something changed the MTU at 3am, what was it?"
- Drift detection: "the state no longer matches what netfyr applied"
- Debugging: understanding the interplay between netfyr and other tools

The daemon does not re-reconcile on external changes because that would silently fight other tools. If an operator deliberately sets a temporary MTU for debugging, they don't want netfyr to immediately revert it. The journal records what happened; the operator can run `netfyr apply` to re-enforce desired state if needed.

## User interaction

External change detection is transparent to the user. When enabled (default in daemon mode), external changes appear in `netfyr history` output:

```
SEQ  TIMESTAMP             TRIGGER         ENTITIES   OUTCOME        CHANGES
145  2026-04-20 15:10:05   external        eth0       observed       ~mtu
144  2026-04-20 15:00:00   policy-apply    eth0       applied (1 ok) ~mtu
```

The detail view shows which entities changed:

```
Entry #145 at 2026-04-20 15:10:05 UTC
Trigger: external-change
  Changed entities: eth0
Diff:
  ~ ethernet eth0
      mtu: 9000 -> 1500
Outcome: observed (no action taken)
```

## Implementation details

- Crate: `netfyr-daemon`
- Files:
  - `crates/netfyr-daemon/src/netlink_monitor.rs` -- (new) netlink subscription, debounce, change detection
  - `crates/netfyr-daemon/src/server.rs` -- add netlink monitor branch to the `tokio::select!` event loop
  - `crates/netfyr-daemon/src/reconciler.rs` -- add `set_applying` flag for self-change exclusion
- Dependencies (external crates): none new -- `rtnetlink` and `netlink-sys` are already dependencies via SPEC-102

### Netlink monitoring

The daemon opens a second netlink socket (separate from the one used for queries and applies) subscribed to multicast groups for link, address, and route notifications:

```rust
// crates/netfyr-daemon/src/netlink_monitor.rs

use netlink_sys::{Socket, SocketAddr, protocols::NETLINK_ROUTE};

pub struct NetlinkMonitor {
    /// Receives parsed change notifications.
    change_rx: mpsc::Receiver<NetlinkChange>,
    /// Handle to the monitoring task.
    task: JoinHandle<()>,
}

pub struct NetlinkChange {
    /// The interface index that changed.
    pub ifindex: u32,
    /// The interface name (if available from the notification).
    pub ifname: Option<String>,
    /// What kind of change was observed.
    pub kind: ChangeKind,
}

pub enum ChangeKind {
    LinkChanged,      // IFLA attributes changed (mtu, state, etc.)
    AddressAdded,     // IPv4 address added
    AddressRemoved,   // IPv4 address removed
    RouteAdded,       // IPv4 route added
    RouteRemoved,     // IPv4 route removed
}

impl NetlinkMonitor {
    /// Start the netlink monitor.
    /// Subscribes to RTNLGRP_LINK, RTNLGRP_IPV4_IFADDR, and RTNLGRP_IPV4_ROUTE.
    pub async fn start() -> Result<Self>;

    /// Receive the next coalesced change notification.
    /// Returns None if the monitor is stopped.
    pub async fn next_change(&mut self) -> Option<Vec<NetlinkChange>>;

    /// Stop the monitor.
    pub async fn stop(self);
}
```

### Startup cache pre-population

`RTM_NEWADDR` messages carry only `ifindex`, not `ifname`. The monitor resolves names from an internal `name_cache: HashMap<u32, String>`, populated only when `RTM_NEWLINK` messages arrive. After a daemon restart, this cache is empty. If no link attribute change occurs before an address change, the cache has no mapping and the address event is silently dropped (the `ifname` field is `None`, and the event loop's `.filter_map(|c| c.ifname)` skips it).

To ensure the cache is populated from the start, the monitor sends an `RTM_GETLINK` dump request (`NLM_F_REQUEST | NLM_F_DUMP`) on the same `netlink_sys::Socket` during initialization. This is done while the socket is still in blocking mode (before `set_non_blocking(true)`), after `bind_auto()` but before `add_membership()`. Each response message is parsed with the existing `parse_link_message()` function to extract `(ifindex, ifname)` pairs into the cache.

The startup sequence is: `bind_auto()` → `RTM_GETLINK` dump (blocking) → `set_non_blocking(true)` → `add_membership()` for multicast groups.

If the dump request fails, the monitor logs a warning and continues with an empty cache. This is non-fatal — the cache will still be populated by subsequent `RTM_NEWLINK` messages during normal operation, though address-only changes on interfaces that haven't had a link event will be missed until one occurs.

### Netlink multicast groups

Subscribe to:
- `RTNLGRP_LINK` (group 1) -- link creation, deletion, attribute changes (mtu, state, flags)
- `RTNLGRP_IPV4_IFADDR` (group 5) -- IPv4 address additions and removals
- `RTNLGRP_IPV4_ROUTE` (group 6) -- IPv4 route additions and removals

These groups provide real-time notifications for all the fields netfyr manages on ethernet interfaces.

### Debounce

Network changes often arrive as bursts -- for example, bringing an interface up may generate link state, address, and route notifications in rapid succession. The monitor debounces notifications:

1. On receiving a netlink message, record the interface index and change kind.
2. Start a 500ms timer (or reset it if already running).
3. When the timer expires, emit all accumulated changes as a single `Vec<NetlinkChange>`.
4. This coalesces burst changes into one journal entry per interface per event.

The debounce window is hardcoded at 500ms. This is long enough to coalesce burst changes but short enough to feel responsive in the history output.

### Self-change exclusion

When the daemon applies changes via `reconcile_and_apply()`, it generates netlink notifications for its own changes. These must be excluded from external change detection to avoid spurious journal entries.

The reconciler exposes an `applying` flag:

```rust
// In Reconciler
pub fn set_applying(&self, applying: bool);
pub fn is_applying(&self) -> bool;
```

The event loop flow:
1. Before calling `reconcile_and_apply()`, set `applying = true`.
2. `reconcile_and_apply()` applies diffs and writes a journal entry.
3. After it returns, set `applying = false`.
4. In the netlink monitor branch, if `reconciler.is_applying()`, discard the notification.

The flag uses an `AtomicBool` for thread safety (the monitor task runs on a separate tokio task).

### Event loop integration

Add a new branch to the daemon's `tokio::select!` loop in `serve_varlink()`:

```rust
// New branch in tokio::select!
Some(changes) = netlink_monitor.next_change() => {
    if reconciler.is_applying() {
        continue; // Discard self-changes
    }

    // Query current state of changed interfaces
    let mut changed_entities = Vec::new();
    for change in &changes {
        let ifname = match &change.ifname {
            Some(name) => name.clone(),
            None => continue, // Can't identify interface without a name
        };

        let actual = backend_registry.query_one("ethernet", &ifname).await;
        if let Ok(Some(current_state)) = actual {
            // Compare against last known state from journal
            if let Some(ref journal) = journal {
                let last = journal.latest_state_for(&ifname)?;
                if let Some(last_state) = last {
                    let diff = compute_external_diff(&last_state, &current_state);
                    if !diff.operations.is_empty() {
                        changed_entities.push(ifname);
                    }
                }
            }
        }
    }

    if !changed_entities.is_empty() {
        if let Some(ref mut journal) = journal {
            // Query full system state for the snapshot
            let actual_state = backend_registry.query_all().await?;
            let entry = JournalEntry {
                seq: 0,
                timestamp: Utc::now(),
                trigger: Trigger::ExternalChange { changed_entities: changed_entities.clone() },
                active_policies: summarize_policies(policy_store.policies()),
                diff: compute_full_external_diff(&journal, &actual_state),
                state_after: SerializableStateSet::from(&actual_state),
                outcome: ApplyOutcome::Observed,
            };
            if let Err(e) = journal.append(entry) {
                tracing::warn!("Failed to write external change journal entry: {}", e);
            } else {
                tracing::info!("External change detected on: {}", changed_entities.join(", "));
            }
        }
    }
}
```

### Computing external diffs

To record what changed externally, the daemon compares the current system state against the last known state from the journal. The `compute_external_diff` function:

1. For each entity in the current state, find its last snapshot in the journal.
2. Compare field values. Any field that differs produces a `SerializableFieldChange` with `change_kind: "set"` and both `current` (old value from journal) and `desired` (new value from system). This applies to both scalar fields (e.g., `mtu`) and list fields (e.g., `addresses`, `routes`). The backend query (SPEC-102) returns routes via `RTM_GETROUTE` dump filtered by output interface, so the current state always includes the `routes` field for managed interfaces.
3. Read-only fields (`carrier`, `speed`, `mac`, `driver`, `name`, `operstate` — the same set defined in `READONLY_FIELDS` in `crates/netfyr-backend/src/netlink/apply.rs`) are excluded from the comparison. These fields are reported by the backend query but are never set by policies, so the journal's `state_after` (which stores the desired state on policy-apply entries) will not contain them. Comparing them against the live query would produce spurious diffs (e.g., `+driver` every time).
4. Entities in the journal but not in the current state produce "remove" operations.
5. Entities in the current state but not in the journal are ignored (they might be unmanaged).

## Depends on

- SPEC-351 (journal infrastructure -- provides Journal, JournalEntry, Trigger::ExternalChange, and latest_state_for)
- SPEC-102 (rtnetlink query -- backend query used to get current state of changed interfaces)
- SPEC-403 (daemon -- the tokio::select! event loop where the monitor branch is added)

## Acceptance criteria

```gherkin
Feature: Netlink monitor
  Scenario: Monitor detects MTU change
    Given the daemon is running and has applied mtu=9000 to veth-e2e0
    When `ip link set veth-e2e0 mtu 1500` is run externally
    Then a journal entry is recorded with trigger "external_change"
    And the entry's diff shows mtu: 9000 -> 1500
    And the entry's outcome is "observed"
    And the entry's changed_entities includes "veth-e2e0"

  Scenario: Monitor detects address addition
    Given the daemon is running with veth-e2e0 having address 10.99.0.1/24
    When `ip addr add 10.99.0.2/24 dev veth-e2e0` is run externally
    Then a journal entry is recorded with trigger "external_change"
    And the entry's diff shows the address addition

  Scenario: Monitor detects address removal
    Given the daemon is running with veth-e2e0 having addresses 10.99.0.1/24 and 10.99.0.2/24
    When `ip addr del 10.99.0.2/24 dev veth-e2e0` is run externally
    Then a journal entry is recorded with trigger "external_change"
    And the entry's diff shows the address removal

  Scenario: Self-changes are excluded
    Given the daemon is running
    When a policy is submitted that changes mtu on veth-e2e0
    Then exactly one journal entry is recorded with trigger "policy_apply"
    And no journal entry is recorded with trigger "external_change"

  Scenario: Burst changes are coalesced
    Given the daemon is running
    When `ip link set veth-e2e0 mtu 1500` and `ip addr add 10.99.0.2/24 dev veth-e2e0` are run in quick succession
    Then a single journal entry is recorded (not two)
    And the entry's diff includes both the mtu and address changes

  Scenario: External changes do not trigger re-reconciliation
    Given the daemon is running with a policy setting mtu=9000 on veth-e2e0
    When `ip link set veth-e2e0 mtu 1500` is run externally
    Then the daemon records the change but does not re-apply mtu=9000
    And veth-e2e0 retains mtu=1500

  Scenario: Address change detected after daemon restart
    Given the daemon is running and has applied address 10.99.0.1/24 to veth-e2e0
    And the daemon is restarted (without any link attribute changes occurring)
    When `ip addr add 10.99.0.2/24 dev veth-e2e0` is run externally
    Then a journal entry is recorded with trigger "external_change"
    And the entry's diff shows the address addition

  Scenario: Monitor detects route addition
    Given the daemon is running with veth-e2e0 managed
    When `ip route add 10.99.1.0/24 via 10.99.0.254 dev veth-e2e0` is run externally
    Then a journal entry is recorded with trigger "external_change"
    And the entry's diff shows the route addition

  Scenario: Monitor detects route removal
    Given veth-e2e0 has route 10.99.1.0/24
    When `ip route del 10.99.1.0/24 dev veth-e2e0` is run externally
    Then a journal entry is recorded with trigger "external_change"
    And the entry's diff shows the route removal

  Scenario: Route and address changes are coalesced
    Given the daemon is running with veth-e2e0 managed
    When `ip addr add 10.99.0.3/24 dev veth-e2e0` and `ip route add 10.99.3.0/24 dev veth-e2e0` are run in quick succession
    Then a single journal entry is recorded with trigger "external_change"
    And the entry's diff shows both the address addition and the route addition

  Scenario: Read-only fields are excluded from external diffs
    Given the daemon is running and has applied mtu=9000 to veth-e2e0
    When `ip link set veth-e2e0 mtu 1500` is run externally
    Then the journal entry's diff does not mention "driver", "carrier", "speed", "mac", or "name"

  Scenario: Monitor ignores unmanaged interfaces
    Given the daemon is running with policies only for veth-e2e0
    When `ip link set veth-other0 mtu 1500` is run on an unmanaged interface
    Then no journal entry is recorded for veth-other0
```
