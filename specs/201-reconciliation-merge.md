# SPEC-201: Reconciliation Engine -- Per-Field Priority Merge

## What
Implement the core reconciliation engine in the `netfyr-reconcile` crate. The engine takes a `Vec<(PolicyId, StateSet)>` representing the state produced by all active policies, groups fields by entity (identified by entity type + selector), and merges fields from all policies using per-field priority. For each field on each entity, the value from the highest-priority policy wins. The result is the effective `StateSet` -- the single, merged desired state of the entire system.

## Why
Multiple policies can target the same entity. For example, a base policy might set MTU on `eth0` while a DHCP policy provides addresses and routes. The reconciliation engine is the central component that resolves which values take effect. Per-field priority (as opposed to per-entity or per-policy) gives maximum flexibility: different policies can contribute different fields to the same entity without conflict, and higher-priority policies can override specific fields from lower-priority ones. This is a core design decision of netfyr's architecture.

## User interaction
Users do not invoke reconciliation directly. It runs automatically during:
- `netfyr apply <path>...` -- reconcile all active policies, then apply the effective state.
- `netfyr apply --dry-run <path>...` -- reconcile and show what would change.
- Daemon mode -- the daemon reconciles when policies are submitted or DHCP leases change.

Users control reconciliation outcomes through policy priority values: higher priority numbers win per field.

## Implementation details
- Crate: `netfyr-reconcile`
- Files: `src/lib.rs`, `src/engine.rs`, `src/merge.rs`
- Dependencies (external crates): none (operates purely on netfyr-state types)

### Core types

**`PolicyInput`** (`src/engine.rs`):
```rust
pub struct PolicyInput {
    pub policy_id: PolicyId,
    pub priority: u32,
    pub state_set: StateSet,
}
```

**`ReconciliationResult`** (`src/engine.rs`):
```rust
pub struct ReconciliationResult {
    pub effective_state: StateSet,
    pub field_sources: HashMap<(EntityKey, FieldName), PolicyId>,
    pub conflicts: ConflictReport, // from SPEC-202
}
```

**`EntityKey`** -- a tuple of `(EntityType, Selector)` that uniquely identifies an entity across all policies.

### Merge algorithm (`src/merge.rs`)

```
fn merge(inputs: Vec<PolicyInput>) -> ReconciliationResult:
    1. Build entity map: HashMap<EntityKey, Vec<(PolicyId, Priority, FieldName, FieldValue)>>
       - For each PolicyInput, iterate its StateSet's entities.
       - For each entity, iterate its fields.
       - Add each field to the entity map keyed by (entity_type, selector).

    2. For each entity in the entity map:
       a. Group fields by field name.
       b. For each field name:
          - Collect all (PolicyId, Priority, Value) tuples.
          - Sort by priority descending.
          - If the highest priority is unique (only one policy at that priority for this field):
            → The value from that policy wins. Record in field_sources.
          - If multiple policies share the highest priority:
            → If all provide the same value: that value wins (no conflict).
            → If values differ: this is a conflict (handled by SPEC-202).
       c. Build the effective State from all winning field values.

    3. Collect all effective States into the effective StateSet.
    4. Return ReconciliationResult with effective state, field sources, and conflicts.
```

### Priority semantics

- Priority is a `u32` value. Higher numbers mean higher priority.
- Default priority is 100.
- Priority is set at the policy level and applies to all fields produced by that policy.
- A policy with priority 200 overrides fields from a policy with priority 100.
- If a field is only provided by one policy, priority is irrelevant (it wins by default).

### Field source tracking

The `field_sources` map records which policy provided each winning field value. This enables:
- Provenance display (showing "this MTU came from policy X" during debugging).
- Debugging merge decisions.
- Conflict reporting (which policies are in conflict).

### Edge cases

- **Empty input**: `merge(vec![])` returns an empty `ReconciliationResult` with an empty effective StateSet.
- **Single policy**: `merge(vec![single])` returns that policy's StateSet as the effective state (no merging needed).
- **Disjoint entities**: Policies targeting different entities never interact; their states are simply unioned.
- **Selector normalization**: Selectors must be compared canonically (e.g., MAC addresses case-insensitive, whitespace-trimmed names).

## Depends on
- SPEC-002 (EntityType, Selector, State, FieldValue types)
- SPEC-004 (StateSet type and its operations)

## Acceptance criteria
```gherkin
Feature: Reconciliation per-field priority merge
  Scenario: Single policy produces effective state unchanged
    Given a single policy "eth0-config" with priority 100
    And it produces a StateSet with entity ethernet/eth0 with fields mtu=1500, addresses=["10.0.1.50/24"]
    When reconciliation is run
    Then the effective StateSet contains ethernet/eth0 with mtu=1500 and addresses=["10.0.1.50/24"]
    And field_sources maps (ethernet/eth0, mtu) to "eth0-config"
    And field_sources maps (ethernet/eth0, addresses) to "eth0-config"

  Scenario: Two policies contribute different fields to the same entity
    Given policy "eth0-base" with priority 100 provides ethernet/eth0 with mtu=1500
    And policy "eth0-dhcp" with priority 100 provides ethernet/eth0 with addresses=["10.0.1.50/24"]
    When reconciliation is run
    Then the effective state for ethernet/eth0 has both mtu=1500 and addresses=["10.0.1.50/24"]
    And field_sources maps (ethernet/eth0, mtu) to "eth0-base"
    And field_sources maps (ethernet/eth0, addresses) to "eth0-dhcp"

  Scenario: Higher priority policy overrides a field from lower priority
    Given policy "eth0-base" with priority 100 provides ethernet/eth0 with mtu=1500
    And policy "eth0-override" with priority 200 provides ethernet/eth0 with mtu=9000
    When reconciliation is run
    Then the effective state for ethernet/eth0 has mtu=9000
    And field_sources maps (ethernet/eth0, mtu) to "eth0-override"

  Scenario: Higher priority overrides only conflicting fields, not all fields
    Given policy "eth0-base" with priority 100 provides ethernet/eth0 with mtu=1500 and addresses=["10.0.1.50/24"]
    And policy "eth0-override" with priority 200 provides ethernet/eth0 with mtu=9000
    When reconciliation is run
    Then the effective state has mtu=9000 (from override) and addresses=["10.0.1.50/24"] (from base)

  Scenario: Three policies with cascading priorities
    Given policy "default" with priority 50 provides ethernet/eth0 with mtu=1500, addresses=["10.0.0.1/24"]
    And policy "team" with priority 100 provides ethernet/eth0 with mtu=9000
    And policy "emergency" with priority 200 provides ethernet/eth0 with addresses=["192.168.1.1/24"]
    When reconciliation is run
    Then the effective state has mtu=9000 (from team, priority 100 > 50)
    And addresses=["192.168.1.1/24"] (from emergency, priority 200 > 50)

  Scenario: Policies targeting different entities do not interact
    Given policy "eth0-config" provides ethernet/eth0 with mtu=1500
    And policy "eth1-config" provides ethernet/eth1 with mtu=9000
    When reconciliation is run
    Then the effective StateSet contains two entities
    And ethernet/eth0 has mtu=1500
    And ethernet/eth1 has mtu=9000

  Scenario: Same priority, same value is not a conflict
    Given policy "policy-a" with priority 100 provides ethernet/eth0 with mtu=1500
    And policy "policy-b" with priority 100 provides ethernet/eth0 with mtu=1500
    When reconciliation is run
    Then the effective state for ethernet/eth0 has mtu=1500
    And no conflict is reported for the mtu field

  Scenario: Empty policy input produces empty effective state
    Given no policies are provided
    When reconciliation is run
    Then the effective StateSet is empty
    And field_sources is empty
    And conflicts are empty

  Scenario: Policy with multiple entities
    Given policy "network-config" with priority 100 provides:
      | entity         | fields                              |
      | ethernet/eth0  | mtu=1500, addresses=["10.0.1.50/24"] |
      | ethernet/eth1  | mtu=9000                             |
      | dns/global     | servers=["10.0.1.2"]                 |
    When reconciliation is run
    Then the effective StateSet contains all 3 entities with their respective fields

  Scenario: Lower priority policy fields are included when not overridden
    Given policy "base" with priority 50 provides ethernet/eth0 with mtu=1500, addresses=["10.0.1.50/24"], routes=[default via 10.0.1.1]
    And policy "overlay" with priority 100 provides ethernet/eth0 with mtu=9000
    When reconciliation is run
    Then the effective state has mtu=9000 (overridden), addresses=["10.0.1.50/24"] (from base), routes=[default via 10.0.1.1] (from base)
    And field_sources maps mtu to "overlay" and addresses and routes to "base"
```
