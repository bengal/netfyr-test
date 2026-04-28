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
trap 'kill "${DAEMON_PID:-}" 2>/dev/null; kill_dnsmasq; cleanup; rm -rf "$TMPDIR_TEST"' EXIT

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
2. Configure `veth-dhcp1` with address `10.99.1.1/24` and start dnsmasq serving `10.99.1.100-10.99.1.200`. The `start_dnsmasq` helper must record the PID so `kill_dnsmasq` can stop it in the EXIT trap.
3. Write two policy files: a static policy for `veth-static0` (mtu=1400, address `10.99.0.1/24`) and a DHCP policy for `veth-dhcp0`.
4. Start the daemon and apply both policies.
5. Wait for the DHCP lease (poll up to 10 s).
6. Verify `veth-static0` has mtu=1400 and address `10.99.0.1/24`.
7. Verify `veth-dhcp0` has an address in the `10.99.1.0/24` range.
8. Verify neither interface's configuration interferes with the other.
9. Set `NETFYR_JOURNAL_DIR` to a temp directory (or ensure it was set before step 4).
10. Run `netfyr history -n 5` and verify that a DHCP lease entry shows `dhcp-acquire` in the TRIGGER column.

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

#### 16. Journal entry after apply (`600-e2e-journal-apply.sh`)
Verifies that `netfyr apply` records a journal entry:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Apply a static policy setting mtu=1400 on `veth-e2e0`.
4. Verify `current.ndjson` exists in the journal directory.
5. Parse the last line of `current.ndjson` with `jq`.
6. Verify the entry has `trigger.type` = `"policy_apply"`.
7. Verify the entry's diff mentions `veth-e2e0` and mtu.
8. Verify the entry's `state_after` includes `veth-e2e0` with mtu=1400.
9. Verify the entry's `outcome.kind` = `"applied"` with `succeeded` >= 1.

#### 17. Journal sequence numbers increment (`600-e2e-journal-seq.sh`)
Verifies sequence numbers are monotonic across multiple applies:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Apply policy A setting mtu=1400.
4. Apply policy B setting mtu=1300.
5. Read all lines from `current.ndjson`.
6. Verify there are 2 entries with seq=1 and seq=2.
7. Verify seq=1 timestamp is earlier than seq=2 timestamp.

#### 18. History lists journal entries (`600-e2e-history-list.sh`)
Verifies `netfyr history` shows recorded entries with proper formatting:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Apply policy A (named `veth-e2e0-a`) setting mtu=1400 and address `10.99.0.1/24` on `veth-e2e0`.
4. Apply policy B (named `veth-e2e0-b`) setting mtu=1300 on `veth-e2e0`.
5. Run `netfyr history -n 5`.
6. Verify the output contains 2 entries in reverse chronological order (most recent first).
7. Verify each row shows SEQ, TIMESTAMP, TRIGGER, ENTITIES, and CHANGES columns (no OUTCOME column).
8. Verify the CHANGES column for the second apply shows `mtu 1400→1300` (scalar change with old→new values) and the address removal by value.
9. Verify the TIMESTAMP column uses relative format (entries are recent, so should show durations like "N min ago").
10. Verify column widths are dynamically sized — the SEQ column should not be wider than the header when values are single-digit.
11. Verify the TRIGGER column shows `apply (veth-e2e0-b)` for the most recent entry (policy name in parentheses).
12. Verify the ENTITIES column shows `veth-e2e0` without a `+` or `-` prefix (entity was modified, not created or removed).
13. Run `netfyr history --absolute-timestamps -n 5`.
14. Verify the TIMESTAMP column shows full `YYYY-MM-DD HH:MM:SS` format instead of relative durations.

#### 19. History show detail (`600-e2e-history-show.sh`)
Verifies `netfyr history --show` displays full entry detail:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Apply a policy setting mtu=1400 on `veth-e2e0`.
4. Run `netfyr history --show 1`.
5. Verify the output shows the trigger type.
6. Verify the output shows the diff (mtu change).
7. Verify the output shows the outcome.
8. Verify the "State after" section uses the same YAML format as `netfyr query`:
   - Contains `- type: ethernet` (YAML sequence element with type field).
   - Contains `  name: veth-e2e0`.
   - Contains `  mtu: 1400`.
   - Addresses (if present) are listed as a YAML block sequence (`- addr` per line), not a JSON array.
   - Does NOT contain JSON-style inline arrays like `["..."]` or inline objects like `{"..."}`.

#### 19b. History show state-after format matches query (`600-e2e-history-state-format.sh`)
Verifies that `netfyr history --show` state-after section matches `netfyr query` YAML output:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Add address `10.99.0.1/24` to `veth-e2e0`.
3. Set `NETFYR_JOURNAL_DIR` to a temp directory.
4. Apply a policy setting mtu=1400 and addresses=`["10.99.0.1/24"]` on `veth-e2e0`.
5. Run `netfyr query -s name=veth-e2e0` and capture the YAML output.
6. Run `netfyr history --show 1` and extract the "State after" section (everything after the `State after:` line).
7. Verify the state-after YAML structure matches the query output: same field names, same block-style formatting for lists, same field ordering.
8. Specifically verify addresses appear as `- 10.99.0.1/24` (block sequence), not `["10.99.0.1/24"]` (JSON array).

#### 20. History JSON output (`600-e2e-history-json.sh`)
Verifies `netfyr history -o json` produces valid JSON:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Apply two policies sequentially.
4. Run `netfyr history -n 5 -o json`.
5. Pipe the output through `jq` to validate it is a JSON array.
6. Verify the array has 2 elements.
7. Verify each element has `seq`, `timestamp`, `trigger`, and `outcome` fields.

#### 21. History filter by entity (`600-e2e-history-filter.sh`)
Verifies `netfyr history` filtering works:
1. Create two veth pairs: `veth-a0`/`veth-a1` and `veth-b0`/`veth-b1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Apply a policy for `veth-a0` (mtu=1400).
4. Apply a policy for `veth-b0` (mtu=1300).
5. Run `netfyr history -s name=veth-a0`.
6. Verify only the entry for `veth-a0` is shown.

#### 22. Revert to previous state (`600-e2e-revert.sh`)
Verifies `netfyr revert` restores a previous state:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Apply policy A setting mtu=1400 on `veth-e2e0`. Note the seq number (1).
4. Apply policy B setting mtu=1300 on `veth-e2e0`.
5. Verify mtu is now 1300.
6. Run `netfyr revert 1`.
7. Verify mtu is now 1400.
8. Verify the revert created a new journal entry with trigger type `"revert"`.
9. Run `netfyr history -n 1` and verify the TRIGGER column shows `revert (1)` (target seq in parentheses).

#### 23. Revert dry-run (`600-e2e-revert-dry-run.sh`)
Verifies `netfyr revert --dry-run` previews without applying:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Apply policy A setting mtu=1400.
4. Apply policy B setting mtu=1300.
5. Run `netfyr revert 1 --dry-run`.
6. Verify the output mentions `mtu: 1300 -> 1400`.
7. Verify mtu is still 1300 (unchanged).
8. Verify no new journal entry was created (still only 2 entries).

#### 24. Revert to nonexistent entry (`600-e2e-revert-noent.sh`)
Verifies `netfyr revert` fails gracefully for missing entries:
1. Set `NETFYR_JOURNAL_DIR` to a temp directory.
2. Run `netfyr revert 9999`.
3. Verify the exit code is 1.
4. Verify the output contains "not found".

#### 25. Revert with address changes (`600-e2e-revert-addr.sh`)
Verifies revert handles address restoration:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Apply policy A with addresses=["10.99.0.1/24", "10.99.0.2/24"]. Note seq (1).
4. Apply policy B with addresses=["10.99.0.3/24"].
5. Verify `veth-e2e0` has only `10.99.0.3/24`.
6. Run `netfyr revert 1`.
7. Verify `veth-e2e0` has addresses `10.99.0.1/24` and `10.99.0.2/24`.
8. Verify `veth-e2e0` does not have `10.99.0.3/24`.
9. Run `netfyr history -n 1` and verify the CHANGES column shows the restored addresses by value (e.g., `+10.99.0.1/24`, `+10.99.0.2/24`) and the removal by value (`-10.99.0.3/24`).

#### 26. External change detection (`600-e2e-external-change.sh`)
Verifies the daemon detects and journals external network changes without reverting them:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Start the daemon.
4. Apply a policy setting mtu=1400 on `veth-e2e0`. Note journal entry count.
5. **External MTU change**: run `ip link set veth-e2e0 mtu 1500`.
6. Sleep 1s (wait for debounce window).
7. Run `netfyr history -n 1 -o json` and verify the latest entry has trigger type `"external_change"`.
8. Verify the entry's diff shows mtu changed from 1400 to 1500.
9. Verify `ip link show veth-e2e0` still shows mtu=1500 (daemon did not re-reconcile).
10. **External address additions**: run `ip addr add 10.99.0.1/24 dev veth-e2e0` and `ip addr add 10.99.0.2/24 dev veth-e2e0`.
11. Sleep 1s.
12. Verify a new journal entry with trigger `"external_change"` was recorded.
13. Verify the entry's diff shows the address additions.
14. Verify `ip addr show veth-e2e0` shows both 10.99.0.1/24 and 10.99.0.2/24.
15. **External address removal**: run `ip addr del 10.99.0.1/24 dev veth-e2e0`.
16. Sleep 1s.
17. Verify a new journal entry with trigger `"external_change"` was recorded.
18. Verify the entry's diff shows the address removal.
19. Verify `ip addr show veth-e2e0` shows only 10.99.0.2/24.
20. **External route addition**: run `ip route add 10.99.0.0/24 dev veth-e2e0`.
21. Sleep 1s.
22. Verify a new journal entry with trigger `"external_change"` was recorded.
23. Verify the entry's diff shows the route addition.
24. **External route removal**: run `ip route del 10.99.0.0/24 dev veth-e2e0`.
25. Sleep 1s.
26. Verify a new journal entry with trigger `"external_change"` was recorded.
27. Verify the entry's diff shows the route removal.
28. **Multiple routes added at once**: run `ip route add 10.99.1.0/24 dev veth-e2e0` and `ip route add 10.99.2.0/24 dev veth-e2e0`.
29. Sleep 1s.
30. Verify a single journal entry with trigger `"external_change"` is recorded (debounce coalesces).
31. Verify the entry's diff shows both route additions.
32. **Mixed address and route changes**: run `ip addr add 10.99.0.3/24 dev veth-e2e0` and `ip route add 10.99.3.0/24 dev veth-e2e0` in quick succession.
33. Sleep 1s.
34. Verify a journal entry with trigger `"external_change"` is recorded.
35. Verify the entry's diff shows both the address addition and the route addition.
36. **Mixed additions and removals**: run `ip route del 10.99.1.0/24 dev veth-e2e0` and `ip route add 10.99.4.0/24 dev veth-e2e0` in quick succession.
37. Sleep 1s.
38. Verify a journal entry with trigger `"external_change"` is recorded.
39. Verify the entry's diff shows the route removal and the route addition.
40. Verify the daemon did not re-apply the original policy state at any point during the test.
41. **History text output shows inline values**: run `netfyr history -n 3` (text format).
42. Verify the CHANGES column shows `mtu 1400→1500` (not `~mtu`) for the MTU change entry.
43. Verify the CHANGES column shows actual addresses for address change entries (e.g., `+10.99.0.1/24`).
44. Verify the TRIGGER column shows `external` for the external change entries.
45. Verify the ENTITIES column shows `veth-e2e0` without a `+` or `-` prefix (external changes modify existing entities).

#### 26b. Daemon-startup separator in history (`600-e2e-history-separator.sh`)
Verifies that `netfyr history` shows visual separators between daemon lifecycle sessions:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Start the daemon.
4. Apply a policy setting mtu=1400 on `veth-e2e0`.
5. Kill the daemon and restart it with the same policy directory.
6. Wait for daemon socket.
7. Apply a policy setting mtu=1300 on `veth-e2e0`.
8. Run `netfyr history -n 10`.
9. Verify the output contains a `──── daemon restart ────` separator line between the entries from the two daemon sessions.
10. Verify the separator appears between the daemon-startup entry and the entry above it.
11. Verify the TRIGGER column shows `daemon-startup` for the daemon-startup entry.

#### 28. Debug logging covers key decision points (`600-e2e-debug-logging.sh`)
Verifies that debug-level messages appear for the main event flow:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Start the daemon with `RUST_LOG=netfyr_daemon=debug`, capturing stderr to a file.
4. Apply a policy setting mtu=1400 on `veth-e2e0`.
5. Run `ip link set veth-e2e0 mtu 1500` externally.
6. Sleep 1s (debounce).
7. Grep the daemon's stderr for:
   - `RTM_GETLINK dump` (startup cache)
   - `netlink event parsed` (message parsing)
   - `debounce timer fired` (debounce drain)
   - `recording external changes` (change recording)
   - `diff computed` (reconciliation)
8. Verify all 5 patterns are present.

#### 29. Diagnose detects configuration drift (`600-e2e-diagnose-drift.sh`)
Verifies that `netfyr diagnose` detects when external changes have drifted the system from policy:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Start the daemon.
4. Apply a policy setting mtu=1400 on `veth-e2e0`.
5. Run `ip link set veth-e2e0 mtu 1500` externally.
6. Sleep 1s (debounce).
7. Run `netfyr diagnose -s name=veth-e2e0`.
8. Verify the output shows "configuration drift" with severity "warning".
9. Verify the output mentions mtu=1400 (policy) vs mtu=1500 (system).
10. Verify the output suggests `netfyr apply` to re-converge.
11. Verify the exit code is 1 (warning).

#### 30. Diagnose reports healthy when no drift (`600-e2e-diagnose-healthy.sh`)
Verifies that `netfyr diagnose` reports healthy when system matches policy:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Start the daemon.
4. Apply a policy setting mtu=1400 on `veth-e2e0`.
5. Run `netfyr diagnose -s name=veth-e2e0`.
6. Verify the output shows "healthy".
7. Verify the exit code is 0.

#### 31. Diagnose detects carrier loss (`600-e2e-diagnose-carrier.sh`)
Verifies that `netfyr diagnose` detects carrier loss:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Start the daemon.
4. Apply a policy setting mtu=1400 on `veth-e2e0`.
5. Bring down the peer end: `ip link set veth-e2e1 down`.
6. Sleep 1s (debounce).
7. Run `netfyr diagnose -s name=veth-e2e0`.
8. Verify the output shows "carrier loss" with severity "critical".
9. Verify the exit code is 2 (critical).

#### 32. Diagnose JSON output (`600-e2e-diagnose-json.sh`)
Verifies that `netfyr diagnose -o json` produces valid structured output:
1. Create veth pair `veth-e2e0`/`veth-e2e1`.
2. Set `NETFYR_JOURNAL_DIR` to a temp directory.
3. Start the daemon.
4. Apply a policy setting mtu=1400 on `veth-e2e0`.
5. Run `ip link set veth-e2e0 mtu 1500` externally.
6. Sleep 1s.
7. Run `netfyr diagnose -o json`.
8. Pipe through `jq` to validate it is a JSON array.
9. Verify the array contains an element with `"pattern": "configuration_drift"`.
10. Verify the element has `entity`, `severity`, `summary`, `details`, `suggested_actions`, and `related_entries` fields.

#### 33. DHCP lease expiry and re-acquisition (`600-e2e-dhcp-lease-expiry.sh`)
Verifies that the daemon removes the DHCP-assigned address when the lease expires and re-acquires a lease when the DHCP server becomes available again:
1. Create veth pair `veth-dhcp0`/`veth-dhcp1`.
2. Configure `veth-dhcp1` with address `10.99.1.1/24`.
3. Start dnsmasq on `veth-dhcp1` serving `10.99.1.100-10.99.1.200` with a 120-second lease (the minimum dnsmasq allows).
4. Start the daemon with `NETFYR_JOURNAL_DIR` set to a temp directory.
5. Write a DHCP policy for `veth-dhcp0` and run `netfyr apply`.
6. Wait for the DHCP address to appear on `veth-dhcp0` (poll up to 10 s).
7. Kill the dnsmasq process so that T1 renewal and T2 rebind requests fail silently.
8. Wait for the address to disappear from `veth-dhcp0` (poll up to 150 s — the 120 s lease plus margin for event processing and reconciliation).
9. Verify `veth-dhcp0` has no IPv4 address in the `10.99.1.` range.
10. Restart dnsmasq on `veth-dhcp1` with the same DHCP range and lease time.
11. Wait for a DHCP address to reappear on `veth-dhcp0` (poll up to 30 s — the client restarts discovery from scratch after expiry, so initial backoff is short).
12. Verify `veth-dhcp0` has an address in the `10.99.1.` range.

Note: this test takes approximately 2–3 minutes because the minimum dnsmasq lease time is 120 seconds. The polling helper `wait_for_no_address IFACE PREFIX TIMEOUT_SECONDS` (analogous to the existing `wait_for_address`) should be added to `helpers.sh` for this test.

#### 27. Unmanaged interfaces are untouched (`600-e2e-unmanaged.sh`)
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
- SPEC-351 (journal infrastructure — needed for journal and revert tests)
- SPEC-352 (history CLI — needed for history tests)
- SPEC-355 (diagnose CLI — needed for diagnose tests)
- SPEC-353 (external change detection — needed for external change test)
- SPEC-354 (state revert — needed for revert tests)
- SPEC-401 (DHCP factory — needed for DHCP tests)
- SPEC-402 (policy store — needed for persistence test)
- SPEC-403 (daemon — needed for all daemon-mode tests)
- SPEC-405 (daemon debug logging — needed for debug logging test)

## Integration test infrastructure
This story produces only shell test scripts — no Rust code. All tests are shell scripts in `tests/`.

Shell test script rules (from SPEC-001):
- **Naming**: `600-e2e-description.sh` — the Makefile discovers tests via `tests/[0-9]*.sh`.
- **No skip**: if a prerequisite is missing (binary, `unshare`, `dnsmasq`), the script must `echo "FAIL: ..." >&2; exit 1`. Never `exit 0` on failure. Never wrap `netns_setup` or `start_dnsmasq` with `|| exit 0`.
- **Binary path**: locate binaries via `SCRIPT_DIR/../target/debug/netfyr` and `SCRIPT_DIR/../target/debug/netfyr-daemon` (overridable with `NETFYR_BIN` and `NETFYR_DAEMON_BIN` env vars). Check with `[[ ! -x "$NETFYR_BIN" ]]` and `exit 1` if missing.
- **Helpers**: source `tests/helpers.sh` which provides `netns_setup`, `create_veth`, `add_address`, `start_dnsmasq`, `kill_dnsmasq`, `cleanup`, and assertion functions.
- **No process leaks**: every test script must kill all background processes it started (daemon, dnsmasq, etc.) in its EXIT trap. The `kill_dnsmasq` helper kills all dnsmasq processes started by the test. Leaked dnsmasq processes hold pipes open and cause the test runner to hang indefinitely. The EXIT trap must call `kill_dnsmasq` before `cleanup`.

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
    And `netfyr history` shows a DHCP lease entry with TRIGGER "dhcp-acquire"

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

Feature: End-to-end journal entry after apply
  Scenario: Apply records a journal entry with correct metadata
    Given a namespace with a veth pair and NETFYR_JOURNAL_DIR set
    When `netfyr apply policy.yaml` sets mtu=1400 on veth-e2e0
    Then current.ndjson contains an entry with trigger "policy_apply"
    And the entry's diff references veth-e2e0 and mtu
    And the entry's state_after includes veth-e2e0 with mtu=1400
    And the entry's outcome is "applied" with succeeded >= 1

Feature: End-to-end journal sequence numbers
  Scenario: Multiple applies produce incrementing sequence numbers
    Given a namespace with a veth pair and NETFYR_JOURNAL_DIR set
    When two policies are applied sequentially
    Then current.ndjson contains 2 entries with seq=1 and seq=2
    And seq=1 has an earlier timestamp than seq=2

Feature: End-to-end history list
  Scenario: History shows recorded entries with inline values and dynamic widths
    Given a namespace with a veth pair and NETFYR_JOURNAL_DIR set
    And policy A named "veth-e2e0-a" (mtu=1400, address 10.99.0.1/24) was applied
    And policy B named "veth-e2e0-b" (mtu=1300) was applied
    When `netfyr history -n 5` is run
    Then the output contains 2 entries in reverse chronological order
    And each row shows SEQ, TIMESTAMP, TRIGGER, ENTITIES, and CHANGES (no OUTCOME column)
    And the CHANGES column for the second apply shows "mtu 1400→1300" (old→new notation)
    And the TIMESTAMP column shows relative durations (e.g., "N min ago")
    And column widths are dynamically sized (no excessive padding)
    And the TRIGGER column shows "apply (veth-e2e0-b)" for the most recent entry
    And the ENTITIES column shows "veth-e2e0" without lifecycle prefix (modified, not created)

  Scenario: Absolute timestamps override relative format
    Given a namespace with a veth pair and NETFYR_JOURNAL_DIR set
    And a policy has been applied
    When `netfyr history --absolute-timestamps -n 5` is run
    Then the TIMESTAMP column shows full "YYYY-MM-DD HH:MM:SS" format

Feature: End-to-end history show
  Scenario: History --show displays full entry detail
    Given a namespace with a veth pair and a journal with one entry
    When `netfyr history --show 1` is run
    Then the output shows the trigger, diff, and outcome
    And the "State after" section contains "- type: ethernet" and "  name: veth-e2e0"
    And the "State after" section does not contain JSON-style inline arrays or objects

  Scenario: History --show state-after format matches query output
    Given a namespace with a veth pair, address 10.99.0.1/24, and a journal with one apply entry
    When `netfyr query -s name=veth-e2e0` YAML output is captured
    And `netfyr history --show 1` is run
    Then the "State after" section uses the same YAML block-style format as the query output
    And addresses appear as a YAML block sequence ("- 10.99.0.1/24"), not a JSON array

Feature: End-to-end history JSON
  Scenario: History -o json produces valid JSON
    Given a namespace with a veth pair and two journal entries
    When `netfyr history -n 5 -o json` is run
    Then the output is a valid JSON array with 2 elements
    And each element has seq, timestamp, trigger, and outcome fields

Feature: End-to-end history filter
  Scenario: History filters by entity name
    Given a namespace with two veth pairs and one apply per pair
    When `netfyr history -s name=veth-a0` is run
    Then only the entry affecting veth-a0 is shown

Feature: End-to-end revert
  Scenario: Revert restores a previous network state
    Given a namespace with a veth pair and NETFYR_JOURNAL_DIR set
    And policy A (mtu=1400) was applied as seq=1
    And policy B (mtu=1300) was applied as seq=2
    When `netfyr revert 1` is run
    Then veth-e2e0 has mtu=1400
    And a new journal entry with trigger "revert" exists
    And `netfyr history -n 1` shows TRIGGER "revert (1)"

Feature: End-to-end revert dry-run
  Scenario: Revert --dry-run previews without applying
    Given a namespace with a veth pair and NETFYR_JOURNAL_DIR set
    And policy A (mtu=1400) at seq=1 and policy B (mtu=1300) at seq=2
    When `netfyr revert 1 --dry-run` is run
    Then the output mentions "mtu: 1300 -> 1400"
    And veth-e2e0 still has mtu=1300
    And the journal still has only 2 entries

Feature: End-to-end revert nonexistent entry
  Scenario: Revert to missing entry fails with clear error
    When `netfyr revert 9999` is run
    Then the exit code is 1
    And the output contains "not found"

Feature: End-to-end revert with addresses
  Scenario: Revert restores address set from a previous state
    Given a namespace with a veth pair and NETFYR_JOURNAL_DIR set
    And policy A (addresses 10.99.0.1/24, 10.99.0.2/24) at seq=1
    And policy B (addresses 10.99.0.3/24) at seq=2
    When `netfyr revert 1` is run
    Then veth-e2e0 has addresses 10.99.0.1/24 and 10.99.0.2/24
    And veth-e2e0 does not have address 10.99.0.3/24
    And `netfyr history -n 1` CHANGES shows addresses by value (e.g., "+10.99.0.1/24", "-10.99.0.3/24")

Feature: End-to-end external change detection
  Scenario: Daemon detects external MTU change
    Given a namespace with a veth pair, the daemon running, and NETFYR_JOURNAL_DIR set
    And a policy setting mtu=1400 on veth-e2e0 has been applied
    When `ip link set veth-e2e0 mtu 1500` is run externally
    And 1 second passes (debounce window)
    Then the latest journal entry has trigger "external_change"
    And the entry's diff shows mtu: 1400 -> 1500
    And veth-e2e0 still has mtu=1500 (daemon did not re-reconcile)

  Scenario: Daemon detects external address additions
    Given the daemon is running with veth-e2e0 managed
    When `ip addr add 10.99.0.1/24 dev veth-e2e0` and `ip addr add 10.99.0.2/24 dev veth-e2e0` are run
    And 1 second passes
    Then a journal entry with trigger "external_change" is recorded
    And the entry's diff shows the address additions
    And veth-e2e0 has both 10.99.0.1/24 and 10.99.0.2/24

  Scenario: Daemon detects external address removal
    Given veth-e2e0 has addresses 10.99.0.1/24 and 10.99.0.2/24
    When `ip addr del 10.99.0.1/24 dev veth-e2e0` is run
    And 1 second passes
    Then a journal entry with trigger "external_change" is recorded
    And the entry's diff shows the address removal
    And veth-e2e0 has only 10.99.0.2/24

  Scenario: Daemon detects external route addition
    Given the daemon is running with veth-e2e0 managed
    When `ip route add 10.99.0.0/24 dev veth-e2e0` is run externally
    And 1 second passes
    Then a journal entry with trigger "external_change" is recorded
    And the entry's diff shows the route addition

  Scenario: Daemon detects external route removal
    Given veth-e2e0 has route 10.99.0.0/24
    When `ip route del 10.99.0.0/24 dev veth-e2e0` is run externally
    And 1 second passes
    Then a journal entry with trigger "external_change" is recorded
    And the entry's diff shows the route removal

  Scenario: Daemon detects multiple routes added at once
    Given the daemon is running with veth-e2e0 managed
    When `ip route add 10.99.1.0/24 dev veth-e2e0` and `ip route add 10.99.2.0/24 dev veth-e2e0` are run in quick succession
    And 1 second passes
    Then a single journal entry with trigger "external_change" is recorded (coalesced)
    And the entry's diff shows both route additions

  Scenario: Daemon detects mixed address and route changes
    Given the daemon is running with veth-e2e0 managed
    When `ip addr add 10.99.0.3/24 dev veth-e2e0` and `ip route add 10.99.3.0/24 dev veth-e2e0` are run in quick succession
    And 1 second passes
    Then a journal entry with trigger "external_change" is recorded
    And the entry's diff shows both the address addition and the route addition

  Scenario: Daemon detects mixed route additions and removals
    Given veth-e2e0 has routes 10.99.1.0/24 and 10.99.2.0/24
    When `ip route del 10.99.1.0/24 dev veth-e2e0` and `ip route add 10.99.4.0/24 dev veth-e2e0` are run in quick succession
    And 1 second passes
    Then a journal entry with trigger "external_change" is recorded
    And the entry's diff shows the route removal and the route addition

  Scenario: Daemon does not re-apply policy after external changes
    Given a policy setting mtu=1400 was applied via the daemon
    When mtu, addresses, and routes are changed externally across multiple steps
    Then the daemon records each change but never re-applies the original state

  Scenario: History text output shows inline values for external changes
    Given external changes have been recorded (mtu 1400→1500, address additions)
    When `netfyr history -n 3` is run (text format)
    Then the CHANGES column shows "mtu 1400→1500" (not "~mtu")
    And the CHANGES column shows actual address values (e.g., "+10.99.0.1/24")
    And the TRIGGER column shows "external"
    And the ENTITIES column shows "veth-e2e0" without lifecycle prefix

Feature: End-to-end daemon-startup separator in history
  Scenario: History shows separator between daemon sessions
    Given a namespace with a veth pair and NETFYR_JOURNAL_DIR set
    And the daemon was started, a policy applied, then restarted
    When `netfyr history -n 10` is run
    Then the output contains a "──── daemon restart ────" separator line
    And the separator appears between the daemon-startup entry and the previous session's entries
    And the TRIGGER column shows "daemon-startup" for the daemon-startup entry

Feature: End-to-end debug logging
  Scenario: Debug messages appear for external change flow
    Given a namespace with a veth pair and the daemon started with RUST_LOG=netfyr_daemon=debug
    And a policy setting mtu=1400 on veth-e2e0 has been applied
    When `ip link set veth-e2e0 mtu 1500` is run externally
    And 1 second passes (debounce window)
    Then the daemon's stderr contains "RTM_GETLINK dump"
    And the daemon's stderr contains "netlink event parsed"
    And the daemon's stderr contains "debounce timer fired"
    And the daemon's stderr contains "recording external changes"
    And the daemon's stderr contains "diff computed"

Feature: End-to-end diagnose drift
  Scenario: Diagnose detects configuration drift from external changes
    Given a namespace with a veth pair, the daemon running, and NETFYR_JOURNAL_DIR set
    And a policy setting mtu=1400 on veth-e2e0 has been applied
    When `ip link set veth-e2e0 mtu 1500` is run externally
    And 1 second passes
    And `netfyr diagnose -s name=veth-e2e0` is run
    Then the output shows "configuration drift" with severity "warning"
    And the output mentions mtu=1400 (policy) vs mtu=1500 (system)
    And the output suggests `netfyr apply`
    And the exit code is 1

  Scenario: Diagnose reports healthy when system matches policy
    Given a namespace with a veth pair, the daemon running, and NETFYR_JOURNAL_DIR set
    And a policy setting mtu=1400 on veth-e2e0 has been applied (no external changes)
    When `netfyr diagnose -s name=veth-e2e0` is run
    Then the output shows "healthy"
    And the exit code is 0

Feature: End-to-end diagnose carrier loss
  Scenario: Diagnose detects carrier loss via peer link down
    Given a namespace with a veth pair, the daemon running, and NETFYR_JOURNAL_DIR set
    And a policy setting mtu=1400 on veth-e2e0 has been applied
    When the peer `ip link set veth-e2e1 down` is run
    And 1 second passes
    And `netfyr diagnose -s name=veth-e2e0` is run
    Then the output shows "carrier loss" with severity "critical"
    And the exit code is 2

Feature: End-to-end diagnose JSON output
  Scenario: Diagnose -o json produces valid structured output
    Given a namespace with a veth pair, the daemon running, and NETFYR_JOURNAL_DIR set
    And a policy setting mtu=1400 has been applied and mtu externally changed to 1500
    When `netfyr diagnose -o json` is run
    Then the output is a valid JSON array
    And an element has "pattern": "configuration_drift"
    And the element has entity, severity, summary, details, suggested_actions, and related_entries fields

Feature: End-to-end DHCP lease expiry and re-acquisition
  Scenario: Address is removed when DHCP lease expires and restored when server returns
    Given a namespace with veth pair "veth-dhcp0"/"veth-dhcp1"
    And dnsmasq serving 10.99.1.100-10.99.1.200 with a 120-second lease on veth-dhcp1
    And the daemon is running with a DHCP policy for veth-dhcp0
    And veth-dhcp0 has acquired an address in the 10.99.1.0/24 range
    When dnsmasq is killed
    And the 120-second lease expires without renewal or rebind
    Then veth-dhcp0 has no address in the 10.99.1.0/24 range
    When dnsmasq is restarted with the same configuration
    Then veth-dhcp0 acquires a new address in the 10.99.1.0/24 range within 30 seconds

Feature: End-to-end unmanaged interfaces
  Scenario: Interfaces without policies are not modified
    Given a namespace with three veth pairs and the daemon running
    And veth-other0 is manually configured with mtu=1400 and address 10.99.2.1/24
    And policies exist only for veth-managed0 (static) and veth-dhcp0 (DHCP)
    When policies are applied and the DHCP lease is acquired
    Then veth-other0 still has mtu=1400 and address 10.99.2.1/24
```
