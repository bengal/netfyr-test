# SPEC-405: Daemon Debug Logging

## What

Add debug-level log messages to the daemon's key decision points so that
operators can troubleshoot event processing, change detection, and
reconciliation by setting `RUST_LOG=netfyr_daemon=debug`.

## Why

The daemon currently logs at info and warn only. When an external change
is silently dropped (e.g., address events after a restart), there is no
way to trace the event flow without rebuilding with additional
instrumentation. Debug-level messages at each filtering and decision
point let operators diagnose problems by enabling
`RUST_LOG=netfyr_daemon=debug` without recompiling.

## Implementation details

- Crate: `netfyr-daemon`
- Files:
  - `crates/netfyr-daemon/src/netlink_monitor.rs`
  - `crates/netfyr-daemon/src/server.rs`
  - `crates/netfyr-daemon/src/reconciler.rs`
  - `crates/netfyr-daemon/src/policy_store.rs`
- Dependencies: none new — `tracing` is already a dependency

All messages below use `tracing::debug!`. No existing log levels are
changed. No new dependencies are added.

### netlink_monitor.rs

1. **`dump_link_names` (RTM_GETLINK startup dump):** After the dump
   completes successfully, log the number of cached names and list them:
   `debug!("RTM_GETLINK dump: cached {} names: {:?}", count, names)`

2. **`process_buffer` (message parsing):** For each successfully parsed
   message, log the extracted fields:
   `debug!(ifindex, ?ifname, ?kind, "netlink event parsed")`

3. **`monitor_task` (debounce drain):** When the debounce timer fires
   and changes are emitted, log the count:
   `debug!(count, "debounce timer fired, emitting changes")`

4. **`parse_message`:** When a message type is not RTM_NEWLINK,
   RTM_DELLINK, RTM_NEWADDR, or RTM_DELADDR, log the ignored type:
   `debug!(nlmsg_type, "ignoring netlink message type")`

5. **`parse_link_message`:** When ifindex is invalid (<=0), log the
   rejection: `debug!(ifindex, "rejecting link message: invalid ifindex")`

6. **`parse_addr_message`:** When the buffer is too small, log it:
   `debug!(len, expected, "rejecting addr message: buffer too small")`

### server.rs

1. **Netlink change branch — self-change filtering:** When
   `is_applying()` is true and events are discarded, log:
   `debug!(count, "discarding netlink events during self-apply")`

2. **Netlink change branch — name resolution:** When a change has
   `ifname: None`, log:
   `debug!(ifindex, "dropping change: ifname not resolved")`

3. **Netlink change branch — unmanaged interface filtering:** When an
   interface is not in the managed set, log:
   `debug!(ifname, "dropping change: interface not managed")`

4. **Netlink change branch — all filtered:** When all changes in a batch
   are filtered out (empty `changed_names`), log:
   `debug!("all netlink changes filtered, no journal entry")`

5. **Netlink change branch — recording:** When external changes are
   recorded, log the names:
   `debug!(?changed_names, "recording external changes")`

### reconciler.rs

1. **`reconcile_and_apply` — inputs:** At the start, log the count of
   policy inputs and managed entities:
   `debug!(policy_count, entity_count, "starting reconciliation")`

2. **`reconcile_and_apply` — diff result:** After diff computation, log
   operation counts:
   `debug!(adds, modifies, removes, "diff computed")`

3. **`reconcile_and_apply` — apply result:** After backend apply, log
   the outcome:
   `debug!(succeeded, failed, skipped, "apply completed")`

4. **`record_external_change` — per-entity:** For each entity checked,
   log whether it changed:
   `debug!(entity, changed, field_count, "external change check")`

5. **`record_external_change` — result:** Log whether a journal entry
   was written or all entities were unchanged:
   `debug!(entities, "external change recorded")` or
   `debug!("no external field changes detected")`

### policy_store.rs

1. **`load` — directory scan:** After loading, log the count:
   `debug!(count, "loaded policies from directory")`

2. **`replace_all` — completion:** After the atomic write, log counts:
   `debug!(written, removed, "policies persisted")`

## Verification

Run `cargo test -p netfyr-daemon` to verify no regressions. Debug
messages do not need dedicated unit tests — they are observational and
do not affect behavior. Integration verification is covered by SPEC-600
test scenario 28 (`600-e2e-debug-logging.sh`).

## Depends on

- SPEC-353 (external change detection — netlink monitor, event loop integration)
- SPEC-403 (daemon — event loop, reconciliation)
- SPEC-402 (policy store)

## Acceptance criteria

This story has no standalone acceptance criteria. Debug messages are
observational and do not change behavior. Correctness is verified by:
- `cargo test -p netfyr-daemon` (no regressions)
- SPEC-600 test scenario 28 (`600-e2e-debug-logging.sh`) verifies that
  debug messages appear for the key event flow
