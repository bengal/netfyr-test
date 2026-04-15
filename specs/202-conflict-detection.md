# SPEC-202: Conflict Detection During Reconciliation

## What
During the reconciliation merge (SPEC-201), detect and report conflicts: situations where two or more policies provide the same field for the same entity at the same priority level but with different values. Conflicting fields are excluded from the effective state (they are not applied to the system). Non-conflicting fields proceed normally. A `ConflictReport` is produced listing all detected conflicts with full details including the entity, field name, the conflicting policies, their priorities, and their respective values.

## Why
The architecture mandates that equal-priority field conflicts are errors, not silently resolved. This prevents surprises where an arbitrary policy "wins" without the user's knowledge. By excluding conflicting fields from the effective state and reporting them clearly, users are forced to make an explicit decision (e.g., changing one policy's priority). This design principle -- "require explicit decisions, no surprises" -- is a key design decision documented in the architecture.

## User interaction
- `netfyr apply` -- if conflicts are detected, the conflicting fields are skipped and the conflicts are printed to stderr as warnings. The apply still proceeds for non-conflicting fields. Exit code is 1 (partial) if conflicts exist.
- `netfyr apply --dry-run` -- shows conflicts in the output with a dedicated conflicts section.

Users resolve conflicts by:
1. Changing one policy's priority to be higher or lower than the other.
2. Removing the conflicting field from one of the policies.
3. Ensuring both policies provide the same value (same value at same priority is not a conflict).

## Implementation details
- Crate: `netfyr-reconcile`
- Files: `src/conflict.rs`
- Dependencies (external crates): none

### Types

**`Conflict`** (`src/conflict.rs`):
```rust
pub struct Conflict {
    /// The entity where the conflict occurs.
    pub entity_key: EntityKey,
    /// The field name that is in conflict.
    pub field_name: FieldName,
    /// The priority level at which the conflict occurs.
    pub priority: u32,
    /// The conflicting contributions, one per policy.
    pub contributions: Vec<ConflictContribution>,
}

pub struct ConflictContribution {
    pub policy_id: PolicyId,
    pub value: FieldValue,
}
```

**`ConflictReport`** (`src/conflict.rs`):
```rust
pub struct ConflictReport {
    pub conflicts: Vec<Conflict>,
}

impl ConflictReport {
    pub fn is_empty(&self) -> bool;
    pub fn len(&self) -> usize;
    /// Group conflicts by entity for display purposes.
    pub fn by_entity(&self) -> HashMap<EntityKey, Vec<&Conflict>>;
    /// Format a human-readable summary of all conflicts.
    pub fn summary(&self) -> String;
}
```

### Integration with merge algorithm

The conflict detection is integrated into the merge algorithm from SPEC-201, step 2b:

```
For each field name on each entity:
    Collect all (PolicyId, Priority, Value) tuples.
    Sort by priority descending.
    Let max_priority = tuples[0].priority
    Let top_tier = tuples where priority == max_priority

    If top_tier.len() == 1:
        → Winner: top_tier[0].value
    Else if top_tier all have the same value:
        → Winner: that shared value (no conflict)
    Else:
        → CONFLICT: Create a Conflict entry.
        → This field is EXCLUDED from the effective state.
        → The field does not appear in the effective State.
```

### Conflict display format

When formatting conflicts for CLI output:
```
CONFLICTS:
  ethernet eth0:
    mtu: policy "eth0-team-a" sets 9000, policy "eth0-team-b" sets 1500 (both priority 100)
  bond bond0:
    mode: policy "bond-ops" sets "802.3ad", policy "bond-dev" sets "active-backup" (both priority 100)
```

### Edge cases

- **More than two policies conflicting**: All policies at the highest priority with differing values are listed in the conflict report. All their values are excluded.
- **Conflict at lower priority tier**: If policies at priority 200 agree on a value, but policies at priority 100 conflict on the same field, there is no conflict -- the priority 200 value wins.
- **All policies agree**: Even if 5 policies provide the same field at the same priority, if the value is identical across all of them, there is no conflict.
- **Conflict on complex field types**: List fields (e.g., `addresses`) and map fields (e.g., `routes`) use deep equality comparison. `["10.0.1.50/24", "10.0.1.51/24"]` and `["10.0.1.51/24", "10.0.1.50/24"]` are compared as sets (order-independent) for list fields to avoid false conflicts.

## Depends on
- SPEC-201 (Reconciliation merge algorithm that conflict detection is embedded in)

## Acceptance criteria
```gherkin
Feature: Conflict detection during reconciliation merge
  Scenario: Two policies conflict on same field at same priority
    Given policy "team-a" with priority 100 provides ethernet/eth0 with mtu=9000
    And policy "team-b" with priority 100 provides ethernet/eth0 with mtu=1500
    When reconciliation is run
    Then a Conflict is reported for entity ethernet/eth0, field "mtu", priority 100
    And the conflict lists "team-a" with value 9000 and "team-b" with value 1500
    And the effective state for ethernet/eth0 does NOT include the mtu field

  Scenario: Same value at same priority is not a conflict
    Given policy "team-a" with priority 100 provides ethernet/eth0 with mtu=1500
    And policy "team-b" with priority 100 provides ethernet/eth0 with mtu=1500
    When reconciliation is run
    Then no conflict is reported for the mtu field
    And the effective state for ethernet/eth0 has mtu=1500

  Scenario: Higher priority resolves what would otherwise be a conflict
    Given policy "base" with priority 100 provides ethernet/eth0 with mtu=1500
    And policy "override" with priority 200 provides ethernet/eth0 with mtu=9000
    When reconciliation is run
    Then no conflict is reported
    And the effective state has mtu=9000

  Scenario: Conflict on one field does not affect other fields
    Given policy "team-a" with priority 100 provides ethernet/eth0 with mtu=9000, addresses=["10.0.1.50/24"]
    And policy "team-b" with priority 100 provides ethernet/eth0 with mtu=1500
    When reconciliation is run
    Then a conflict is reported for mtu
    And the effective state for ethernet/eth0 has addresses=["10.0.1.50/24"] (no conflict on this field)
    And the effective state for ethernet/eth0 does NOT include mtu

  Scenario: Three-way conflict at same priority
    Given policy "a" with priority 100 provides ethernet/eth0 with mtu=1500
    And policy "b" with priority 100 provides ethernet/eth0 with mtu=9000
    And policy "c" with priority 100 provides ethernet/eth0 with mtu=4500
    When reconciliation is run
    Then a Conflict is reported with 3 contributions
    And the conflict lists all three policies and their values
    And the effective state does NOT include mtu

  Scenario: Conflict at lower priority does not matter when higher priority exists
    Given policy "low-a" with priority 100 provides ethernet/eth0 with mtu=1500
    And policy "low-b" with priority 100 provides ethernet/eth0 with mtu=9000
    And policy "high" with priority 200 provides ethernet/eth0 with mtu=4500
    When reconciliation is run
    Then no conflict is reported (priority 200 wins outright)
    And the effective state has mtu=4500

  Scenario: Conflicts on list fields use set comparison
    Given policy "a" with priority 100 provides ethernet/eth0 with addresses=["10.0.1.50/24", "10.0.1.51/24"]
    And policy "b" with priority 100 provides ethernet/eth0 with addresses=["10.0.1.51/24", "10.0.1.50/24"]
    When reconciliation is run
    Then no conflict is reported (same values, different order)
    And the effective state has addresses containing both "10.0.1.50/24" and "10.0.1.51/24"

  Scenario: Conflict on list fields with genuinely different values
    Given policy "a" with priority 100 provides ethernet/eth0 with addresses=["10.0.1.50/24"]
    And policy "b" with priority 100 provides ethernet/eth0 with addresses=["10.0.2.50/24"]
    When reconciliation is run
    Then a conflict is reported for addresses
    And the effective state does NOT include addresses

  Scenario: Multiple conflicts on different entities
    Given policy "a" with priority 100 provides ethernet/eth0 with mtu=9000 and ethernet/eth1 with mtu=9000
    And policy "b" with priority 100 provides ethernet/eth0 with mtu=1500 and ethernet/eth1 with mtu=1500
    When reconciliation is run
    Then 2 conflicts are reported: one for ethernet/eth0 mtu, one for ethernet/eth1 mtu
    And ConflictReport.len() returns 2

  Scenario: ConflictReport summary produces readable output
    Given a ConflictReport with 2 conflicts
    When summary() is called
    Then the output includes the entity, field name, conflicting policy names, and their values for each conflict

  Scenario: ConflictReport by_entity groups correctly
    Given a ConflictReport with conflicts on ethernet/eth0 (mtu and addresses) and ethernet/eth1 (mtu)
    When by_entity() is called
    Then the result has 2 keys: ethernet/eth0 and ethernet/eth1
    And ethernet/eth0 has 2 conflicts
    And ethernet/eth1 has 1 conflict

  Scenario: No conflicts produces empty ConflictReport
    Given policies that do not conflict on any field
    When reconciliation is run
    Then ConflictReport.is_empty() returns true
    And ConflictReport.len() returns 0
```
