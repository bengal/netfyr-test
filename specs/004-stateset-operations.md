# SPEC-004: StateSet Operations

## What
Implement the `StateSet` type (a collection of `State` values keyed by entity type + selector) and four set operations that work **per-field**, not per-entity:

1. **`StateSet`** -- A collection type holding zero or more `State` values, keyed by a composite key of `(entity_type, selector.key())`. Provides:
   - `insert(state: State)` -- add or replace an entity state
   - `get(entity_type: &str, selector_key: &str) -> Option<&State>` -- look up by key
   - `remove(entity_type: &str, selector_key: &str) -> Option<State>` -- remove by key
   - `iter() -> impl Iterator<Item = &State>` -- iterate over all states
   - `len() -> usize` and `is_empty() -> bool`
   - `entities() -> Vec<(String, String)>` -- list all (entity_type, selector_key) pairs

2. **`union(a: &StateSet, b: &StateSet) -> Result<StateSet, ConflictError>`** -- Merge two state sets per-field:
   - For entities present in only one set: include as-is.
   - For entities present in both: merge fields. For each field:
     - Present in only one: include it.
     - Present in both with different priorities: the higher priority field wins.
     - Present in both with equal priority and same value: include either (they agree).
     - Present in both with equal priority and different values: return `ConflictError` listing the conflicting entity, field name, and both values.

3. **`intersection(a: &StateSet, b: &StateSet) -> StateSet`** -- Fields present in both sets with the same value (regardless of priority/provenance). Entities with no common matching fields are excluded from the result.

4. **`complement(a: &StateSet, b: &StateSet) -> StateSet`** -- Fields in `a` that are not in `b`. For each entity in `a`, include only the fields that do not appear in `b` for the same entity. Entities with no remaining fields are excluded. This is the mechanism for deletion detection: `complement(system_state, desired_state)` yields what needs to be removed.

5. **`diff(from: &StateSet, to: &StateSet) -> StateDiff`** -- Compute the operations needed to transform `from` into `to`. Returns a `StateDiff` containing a list of `DiffOp`:
   - `Add { entity_type, selector, fields }` -- entity exists in `to` but not in `from`
   - `Modify { entity_type, selector, changed_fields, removed_fields }` -- entity exists in both but fields differ. `changed_fields` are fields with different values; `removed_fields` are fields in `from` but not in `to`.
   - `Remove { entity_type, selector }` -- entity exists in `from` but not in `to`

6. **`ConflictError`** -- Error type containing a list of `Conflict { entity_type, selector_key, field, value_a, value_b, priority }`.

7. **`StateDiff`** -- A container for `Vec<DiffOp>` with:
   - `ops() -> &[DiffOp]` -- access the operations
   - `is_empty() -> bool` -- true if no changes
   - `summary() -> String` -- human-readable summary (e.g., "2 added, 1 modified, 0 removed")

## Why
Set operations are the mathematical foundation of netfyr's reconciliation model. The reconciliation engine collects StateSets from all active policies and performs union to merge them (with priority-based conflict resolution). The complement operation (`system_state \ desired_state`) identifies entities/fields to remove -- this is how netfyr achieves deletion without `state:absent` markers. The diff operation produces the concrete list of changes the backend must apply. All operations work per-field (not per-entity) because different policies may contribute different fields to the same entity at different priorities.

## User interaction
Users do not interact with these operations directly. They are used internally by:
- The reconciliation engine (SPEC not yet written) to merge policy outputs and compute effective state
- The `netfyr apply --dry-run` command to show what would change

## Implementation details
- Crate: `netfyr-state`
- Files:
  - `src/set.rs` -- `StateSet` struct and `union`, `intersection`, `complement` functions
  - `src/diff.rs` -- `diff` function, `StateDiff`, `DiffOp` types
  - `src/lib.rs` -- update to re-export new types
- Dependencies (external crates): `thiserror` for error types

### StateSet internal storage
Use an `IndexMap<(String, String), State>` where the key is `(entity_type, selector.key())`. IndexMap preserves insertion order for deterministic output.

### Union field merge pseudocode
```
for each entity key in a.keys() union b.keys():
  if only in a: result.insert(a[key].clone())
  if only in b: result.insert(b[key].clone())
  if in both:
    merged_fields = IndexMap::new()
    for each field in a[key].fields union b[key].fields:
      if only in a: merged_fields.insert(field_from_a)
      if only in b: merged_fields.insert(field_from_b)
      if in both:
        if a_field.value == b_field.value: merged_fields.insert(field_from_a)
        elif a_priority > b_priority: merged_fields.insert(field_from_a)
        elif b_priority > a_priority: merged_fields.insert(field_from_b)
        else: conflicts.push(Conflict{...})
    if conflicts.is_empty():
      result.insert(State with merged_fields)
    else:
      return Err(ConflictError)
```

### Priority source for union
Priority comes from `State.priority`. When merging, each field carries the priority of its parent State. The winning field retains its original provenance.

## Depends on
- SPEC-002
- SPEC-003

## Acceptance criteria
```gherkin
Feature: StateSet collection operations
  Scenario: Insert and retrieve an entity state
    Given an empty StateSet
    When an State for ethernet/eth0 with field mtu=1500 is inserted
    Then get("ethernet", "eth0") returns that State
    And len() returns 1

  Scenario: Insert replaces existing entity with same key
    Given a StateSet containing ethernet/eth0 with mtu=1500
    When a new State for ethernet/eth0 with mtu=9000 is inserted
    Then get("ethernet", "eth0") returns the state with mtu=9000
    And len() returns 1

  Scenario: Remove an entity state
    Given a StateSet containing ethernet/eth0
    When remove("ethernet", "eth0") is called
    Then it returns Some(the removed state)
    And len() returns 0

Feature: Union operation (per-field merge)
  Scenario: Union of disjoint sets
    Given StateSet A containing ethernet/eth0 with mtu=1500
    And StateSet B containing bond/bond0 with mode="802.3ad"
    When union(A, B) is computed
    Then the result contains both ethernet/eth0 and bond/bond0
    And each entity retains its original fields

  Scenario: Union merges fields from same entity
    Given StateSet A containing ethernet/eth0 with field mtu=1500 at priority 100
    And StateSet B containing ethernet/eth0 with field addresses=["10.0.1.50/24"] at priority 100
    When union(A, B) is computed
    Then the result contains ethernet/eth0 with both mtu and addresses fields

  Scenario: Union resolves conflicts by higher priority
    Given StateSet A containing ethernet/eth0 with mtu=1500 at priority 100
    And StateSet B containing ethernet/eth0 with mtu=9000 at priority 200
    When union(A, B) is computed
    Then the result contains ethernet/eth0 with mtu=9000 (from B, higher priority)
    And the provenance of mtu is preserved from B

  Scenario: Union reports error on equal-priority field conflict
    Given StateSet A containing ethernet/eth0 with mtu=1500 at priority 100
    And StateSet B containing ethernet/eth0 with mtu=9000 at priority 100
    When union(A, B) is computed
    Then it returns a ConflictError
    And the error lists field "mtu" on entity "ethernet/eth0" with values 1500 and 9000

  Scenario: Union allows equal-priority fields with same value
    Given StateSet A containing ethernet/eth0 with mtu=1500 at priority 100
    And StateSet B containing ethernet/eth0 with mtu=1500 at priority 100
    When union(A, B) is computed
    Then it succeeds
    And the result contains ethernet/eth0 with mtu=1500

Feature: Intersection operation
  Scenario: Intersection of overlapping sets
    Given StateSet A containing ethernet/eth0 with mtu=1500, addresses=["10.0.1.50/24"]
    And StateSet B containing ethernet/eth0 with mtu=1500, routes=[...]
    When intersection(A, B) is computed
    Then the result contains ethernet/eth0 with only mtu=1500
    And addresses and routes are excluded

  Scenario: Intersection of disjoint entities
    Given StateSet A containing ethernet/eth0
    And StateSet B containing bond/bond0
    When intersection(A, B) is computed
    Then the result is empty

  Scenario: Intersection excludes fields with different values
    Given StateSet A containing ethernet/eth0 with mtu=1500
    And StateSet B containing ethernet/eth0 with mtu=9000
    When intersection(A, B) is computed
    Then the result contains no fields for ethernet/eth0
    And the entity is excluded from the result (no remaining fields)

Feature: Complement operation
  Scenario: Complement yields fields only in A
    Given StateSet A containing ethernet/eth0 with mtu=1500 and addresses=["10.0.1.50/24"]
    And StateSet B containing ethernet/eth0 with mtu=1500
    When complement(A, B) is computed
    Then the result contains ethernet/eth0 with only addresses=["10.0.1.50/24"]
    And mtu is excluded (it exists in B)

  Scenario: Complement of identical sets is empty
    Given StateSet A and B both containing ethernet/eth0 with identical fields
    When complement(A, B) is computed
    Then the result is empty

  Scenario: Complement yields entire entities not in B
    Given StateSet A containing ethernet/eth0 and bond/bond0
    And StateSet B containing only ethernet/eth0
    When complement(A, B) is computed
    Then the result contains bond/bond0 with all its fields
    And ethernet/eth0 is excluded or has only fields not in B

Feature: Diff operation
  Scenario: Diff detects added entities
    Given StateSet "from" is empty
    And StateSet "to" contains ethernet/eth0 with mtu=1500
    When diff(from, to) is computed
    Then the result contains an Add operation for ethernet/eth0 with mtu=1500

  Scenario: Diff detects removed entities
    Given StateSet "from" contains ethernet/eth0 with mtu=1500
    And StateSet "to" is empty
    When diff(from, to) is computed
    Then the result contains a Remove operation for ethernet/eth0

  Scenario: Diff detects modified fields
    Given StateSet "from" contains ethernet/eth0 with mtu=1500
    And StateSet "to" contains ethernet/eth0 with mtu=9000
    When diff(from, to) is computed
    Then the result contains a Modify operation for ethernet/eth0
    And changed_fields includes mtu changing from 1500 to 9000

  Scenario: Diff detects added and removed fields on same entity
    Given StateSet "from" contains ethernet/eth0 with mtu=1500 and addresses=["10.0.1.50/24"]
    And StateSet "to" contains ethernet/eth0 with mtu=1500 and routes=[...]
    When diff(from, to) is computed
    Then the result contains a Modify operation for ethernet/eth0
    And removed_fields includes "addresses"
    And changed_fields includes "routes" being added

  Scenario: Diff of identical sets is empty
    Given StateSet "from" and "to" contain identical entities and fields
    When diff(from, to) is computed
    Then the result is empty
    And is_empty() returns true

  Scenario: StateDiff summary formatting
    Given a StateDiff with 2 Add ops, 1 Modify op, and 1 Remove op
    When summary() is called
    Then it returns a string like "2 added, 1 modified, 1 removed"
```
