# SPEC-355: Diagnose CLI

## What

Add a `netfyr diagnose` CLI subcommand that analyzes journal history and current system state to detect network problems, explain their cause, and suggest corrective actions. Instead of requiring operators to manually read through `netfyr history` entries and piece together what went wrong, `diagnose` runs pattern detectors that surface actionable findings.

## Why

`netfyr history` shows *what happened* but leaves the operator to interpret it. When something goes wrong — an interface lost carrier, DHCP stopped renewing, or someone changed the MTU outside netfyr — the operator must mentally correlate multiple journal entries, compare against policy, and figure out the fix. `diagnose` automates this: it reads the journal timeline, queries current system state, compares against the last applied policy state, and reports findings with severity, explanation, and suggested next steps.

## User interaction

```bash
# Diagnose all managed entities
netfyr diagnose

# Diagnose a specific entity
netfyr diagnose -s name=eth0

# Machine-readable output
netfyr diagnose -o json

# Look further back in time (default: 1h)
netfyr diagnose --since 24h

# Diagnose with color disabled
netfyr diagnose --color never
```

### Output format

#### Text (default)

```
enp7s0: configuration drift (warning)
  Policy wants mtu=1500, system has mtu=9000
  Changed externally 10 min ago (seq 49)
  3 addresses present that are not in policy
  → Run `netfyr apply` to re-converge
  → Or `netfyr revert 48` to restore last applied state

enp8s0: healthy
  Last apply: 2h ago (seq 44), no drift, carrier up

veth-dhcp0: DHCP lease lost (critical)
  Lease expired 5 min ago (seq 51), no renewal received
  Interface has no addresses
  → Check DHCP server reachability
  → Verify link connectivity with `ip link show veth-dhcp0`
```

Each entity gets a one-line status header (`entity: pattern (severity)`) followed by indented detail lines explaining the situation and `→` lines suggesting actions. Entities with no problems show a single `healthy` line. Healthy entities are listed last.

When color is enabled: `critical` is red, `warning` is yellow, `healthy` is green. The `→` action lines are cyan.

#### JSON (`-o json`)

```json
[
  {
    "entity": "enp7s0",
    "entity_type": "ethernet",
    "severity": "warning",
    "pattern": "configuration_drift",
    "summary": "Policy wants mtu=1500, system has mtu=9000",
    "details": [
      "Changed externally 10 min ago (seq 49)",
      "3 addresses present that are not in policy"
    ],
    "suggested_actions": [
      "Run `netfyr apply` to re-converge",
      "Or `netfyr revert 48` to restore last applied state"
    ],
    "related_entries": [49, 48]
  }
]
```

## Implementation details

- Crate: `netfyr-cli` (CLI command)
- Files:
  - `crates/netfyr-cli/src/main.rs` -- add `Diagnose` to the `Commands` enum
  - `crates/netfyr-cli/src/diagnose.rs` -- (new) diagnose command implementation and pattern detectors
- Dependencies (external crates): none beyond what existing specs add

### CLI arguments

```rust
#[derive(Args)]
struct DiagnoseArgs {
    /// Filter by entity name.
    #[arg(long, short = 's', value_parser = parse_selector)]
    selector: Vec<(String, String)>,

    /// How far back to scan journal entries. Default: 1h.
    /// Accepts relative durations (1h, 30m, 7d) or ISO 8601 timestamps.
    #[arg(long, default_value = "1h")]
    since: String,

    /// Output format: "text" (default), "json".
    #[arg(long, short = 'o', default_value = "text")]
    output: OutputFormat,
}
```

### Dual-mode operation

The diagnose command follows the same daemon-detection pattern as `history` and `query`:

1. Read `NETFYR_SOCKET_PATH` env var (default `/run/netfyr/netfyr.sock`).
2. Attempt connection.
3. If connected: retrieve journal entries via `GetHistory` and current state via Varlink query.
4. If not connected: read journal files directly, query system state via backend.

### Data collection

Diagnose needs two pieces of data:

1. **Journal entries** within the `--since` window — provides the timeline of what happened.
2. **Current system state** — a live backend query (same as `netfyr query`) to compare against what the system should look like.

The "desired state" is derived from the most recent journal entry with outcome `applied` (not `observed`). Its `state_after` snapshot represents the intended system state after the last successful apply. External change entries (`observed`) record drift but do not update the desired state.

### Pattern detectors

Each detector is a function that takes the collected data and returns zero or more findings. Detectors run independently and their findings are merged and sorted by severity.

#### 1. Configuration drift

Compares current system state against the `state_after` from the most recent `applied` entry for each entity.

- **Trigger**: any field difference between current state and last applied state.
- **Severity**: `warning`.
- **Details**: lists each differing field with policy value vs. system value, and the most recent external change entry that caused it.
- **Actions**: `netfyr apply` to re-converge, or `netfyr revert <seq>` to restore.

To detect drift, the detector:
1. Finds the most recent journal entry with outcome `applied` (trigger: `policy_apply`, `daemon_startup`, or `revert`).
2. Extracts `state_after` for each managed entity.
3. Queries current system state for the same entities.
4. Computes a diff (reusing `generate_diff` from `netfyr-reconcile`), excluding read-only fields.
5. Any non-empty diff = drift finding.

#### 2. Carrier loss

Detects that an interface has lost its physical link.

- **Trigger**: the most recent external change entry shows `carrier` changed to `false`, and current state still has `carrier: false`.
- **Severity**: `critical`.
- **Details**: when carrier was lost (timestamp and seq of the external change entry).
- **Actions**: check physical cable, switch port, or driver.

#### 3. DHCP lease lost

Detects that a DHCP-managed interface lost its lease without renewal.

- **Trigger**: a `dhcp-expire` entry exists in the window with no subsequent `dhcp-acquire` for the same policy.
- **Severity**: `critical`.
- **Details**: when the lease expired, which policy, current interface state (no addresses).
- **Actions**: check DHCP server reachability, verify link.

#### 4. Recurring flaps

Detects interfaces that are bouncing — carrier or addresses changing repeatedly.

- **Trigger**: more than 3 external change entries for the same entity within the `--since` window where the same field oscillates (e.g., carrier true→false→true→false).
- **Severity**: `warning`.
- **Details**: number of flaps, affected fields, time range.
- **Actions**: investigate physical link stability, upstream switch, or conflicting automation.

#### 5. Failed apply

Detects recent applies that had failures.

- **Trigger**: a `policy_apply` or `daemon_startup` entry with `outcome.failed > 0`.
- **Severity**: `warning`.
- **Details**: which entry, how many operations failed.
- **Actions**: `netfyr history --show <seq>` to inspect the failure details.

#### 6. Policy conflict

Detects that the most recent apply had field conflicts between policies.

- **Trigger**: a `policy_apply` entry where the diff shows conflicting fields (same field set by multiple policies at the same priority). This is detected by checking if the apply exit code was 1 (partial success) and the output mentioned conflicts.
- **Severity**: `warning`.
- **Details**: which fields on which entities had conflicts.
- **Actions**: review overlapping policies, adjust priorities.

Note: conflict detection relies on the journal entry recording conflict information. If the journal entry does not include conflict details (current `ApplyOutcome` only has succeeded/failed/skipped counts), this detector checks for the pattern: multiple consecutive applies on the same entity with the same field changing back and forth, which suggests competing policies. A future enhancement could add conflict details to the journal entry.

#### 7. No recent activity

Detects that an entity has no journal entries in the `--since` window.

- **Trigger**: a managed entity (appears in past journal entries) has no entries within the window.
- **Severity**: `info`.
- **Details**: when the last entry was recorded.
- **Actions**: no action needed, informational only.

### Finding type

```rust
struct Finding {
    entity_type: String,
    entity_name: String,
    severity: Severity,
    pattern: PatternKind,
    summary: String,
    details: Vec<String>,
    suggested_actions: Vec<String>,
    related_entries: Vec<SequenceId>,
}

#[derive(PartialOrd, Ord, PartialEq, Eq)]
enum Severity {
    Healthy,
    Info,
    Warning,
    Critical,
}

enum PatternKind {
    ConfigurationDrift,
    CarrierLoss,
    DhcpLeaseLost,
    RecurringFlaps,
    FailedApply,
    PolicyConflict,
    NoRecentActivity,
}
```

### Entity discovery

When no `--selector` is given, diagnose must determine which entities to analyze. It discovers managed entities from:

1. The most recent journal entry with outcome `applied` — its `state_after` lists all managed entities.
2. If no applied entries exist in the window, fall back to all entities mentioned in any journal entry within the window.

### Output ordering

Findings are grouped by entity and sorted:
1. Entities with `critical` findings first, then `warning`, then `info`, then `healthy`.
2. Within an entity, findings are sorted by severity (most severe first).
3. Healthy entities are listed last with a single summary line.

### Healthy status

An entity is `healthy` when no detector produced a finding for it. The healthy line includes:
- When the last apply happened (relative timestamp and seq).
- Whether carrier is up (for ethernet entities).
- A short confirmation: "no drift".

## Depends on

- SPEC-351 (journal infrastructure -- reads journal entries)
- SPEC-352 (history CLI -- same dual-mode pattern, reuses journal reading)
- SPEC-302 (CLI query -- live system state query for drift comparison)
- SPEC-203 (diff generation -- reused for drift detection)
- SPEC-102 (backend query -- system state retrieval in standalone mode)
- SPEC-301 (CLI apply -- follows same CLI patterns)

## Acceptance criteria

```gherkin
Feature: Diagnose detects configuration drift
  Scenario: Drift from external MTU change
    Given a journal with a policy-apply entry setting mtu=1500 on ethernet eth0
    And the current system state of eth0 has mtu=9000
    When `netfyr diagnose -s name=eth0` is run
    Then the output shows "eth0: configuration drift (warning)"
    And the output shows "Policy wants mtu=1500, system has mtu=9000"
    And the output suggests "Run `netfyr apply` to re-converge"

  Scenario: No drift when system matches policy
    Given a journal with a policy-apply entry setting mtu=1500 on ethernet eth0
    And the current system state of eth0 has mtu=1500
    When `netfyr diagnose -s name=eth0` is run
    Then the output shows "eth0: healthy"

  Scenario: Drift with multiple fields
    Given a journal with a policy-apply entry setting mtu=1500 and addresses=["10.0.1.50/24"] on eth0
    And the current system state has mtu=9000 and addresses=["10.0.1.50/24", "10.0.1.99/24"]
    When `netfyr diagnose -s name=eth0` is run
    Then the output shows drift for mtu
    And the output mentions extra addresses not in policy

Feature: Diagnose detects carrier loss
  Scenario: Interface lost carrier
    Given a journal with an external_change entry showing carrier changed to false on eth0
    And the current system state of eth0 has carrier=false
    When `netfyr diagnose -s name=eth0` is run
    Then the output shows "eth0: carrier loss (critical)"
    And the output shows when carrier was lost
    And the output suggests checking the physical cable or switch port

  Scenario: Carrier recovered
    Given a journal with an external_change entry showing carrier changed to false on eth0
    And a later external_change entry showing carrier changed to true
    And the current system state of eth0 has carrier=true
    When `netfyr diagnose -s name=eth0` is run
    Then the output does not show carrier loss (carrier has recovered)

Feature: Diagnose detects DHCP lease loss
  Scenario: DHCP lease expired without renewal
    Given a journal with a dhcp-expire entry for policy "eth0-dhcp"
    And no subsequent dhcp-acquire entry for the same policy
    When `netfyr diagnose` is run
    Then the output shows the entity as "DHCP lease lost (critical)"
    And the output suggests checking DHCP server reachability

  Scenario: DHCP lease expired then renewed
    Given a journal with a dhcp-expire entry followed by a dhcp-acquire entry for the same policy
    When `netfyr diagnose` is run
    Then the output does not show DHCP lease loss

Feature: Diagnose detects recurring flaps
  Scenario: Carrier flapping
    Given a journal with 5 external_change entries on eth0 within 30 minutes
    And the carrier field alternates between true and false across those entries
    When `netfyr diagnose -s name=eth0` is run
    Then the output shows "recurring flaps (warning)"
    And the output mentions the number of flaps and the affected field

  Scenario: Multiple changes without oscillation are not flaps
    Given a journal with 5 external_change entries on eth0
    And each entry adds a different address (no oscillation)
    When `netfyr diagnose -s name=eth0` is run
    Then the output does not show recurring flaps

Feature: Diagnose detects failed applies
  Scenario: Recent apply had failures
    Given a journal with a policy-apply entry with outcome failed=2
    When `netfyr diagnose` is run
    Then the output shows "failed apply (warning)"
    And the output suggests `netfyr history --show <seq>` to inspect

  Scenario: All recent applies succeeded
    Given a journal with policy-apply entries all having failed=0
    When `netfyr diagnose` is run
    Then the output does not show failed apply

Feature: Diagnose detects no recent activity
  Scenario: Entity has no recent entries
    Given a managed entity eth0 whose last journal entry is 2 hours ago
    And --since is 1h (default)
    When `netfyr diagnose -s name=eth0` is run
    Then the output shows "no recent activity (info)"
    And the output mentions when the last entry was recorded

Feature: Diagnose output formats
  Scenario: JSON output
    Given a journal with drift on eth0
    When `netfyr diagnose -o json` is run
    Then the output is a valid JSON array
    And each element has entity, severity, pattern, summary, details, suggested_actions, and related_entries fields

  Scenario: Text output groups by severity
    Given findings: eth0 is critical, eth1 is warning, eth2 is healthy
    When `netfyr diagnose` is run
    Then eth0 (critical) appears before eth1 (warning)
    And eth1 (warning) appears before eth2 (healthy)

Feature: Diagnose entity filtering
  Scenario: Filter by entity name
    Given managed entities eth0 (drifted) and eth1 (healthy)
    When `netfyr diagnose -s name=eth0` is run
    Then only eth0 findings are shown
    And eth1 is not mentioned

  Scenario: No selector shows all entities
    Given managed entities eth0 and eth1
    When `netfyr diagnose` is run
    Then findings for both eth0 and eth1 are shown

Feature: Diagnose with empty journal
  Scenario: No journal entries
    Given an empty journal
    When `netfyr diagnose` is run
    Then the output shows "No journal entries found. Run `netfyr apply` first."
    And the exit code is 0

Feature: Diagnose time window
  Scenario: Only entries within --since window are analyzed
    Given a journal with a carrier-loss entry from 2 hours ago
    And no entries within the last hour
    When `netfyr diagnose --since 1h` is run
    Then the carrier loss is not reported (outside the window)

  Scenario: Wider window catches older events
    Given a journal with a carrier-loss entry from 2 hours ago
    When `netfyr diagnose --since 3h` is run
    Then the carrier loss is reported

Feature: Diagnose exit codes
  Scenario: Exit code reflects worst severity
    Given findings with severity "critical"
    When `netfyr diagnose` is run
    Then the exit code is 2

  Scenario: Warning findings
    Given findings with severity "warning" (no critical)
    When `netfyr diagnose` is run
    Then the exit code is 1

  Scenario: All healthy
    Given all entities are healthy
    When `netfyr diagnose` is run
    Then the exit code is 0
```
