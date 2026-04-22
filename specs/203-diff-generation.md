# SPEC-203: Diff Generation Between Desired and Actual State

## What
Generate a `StateDiff` by comparing the effective `StateSet` (desired state, from reconciliation) against the current system `StateSet` (actual state, from backend query). For each entity: if present in desired but not actual, generate an Add operation. If present in actual but not desired, generate a Remove operation. If present in both, compare per-field and generate a Modify operation listing field-level changes. Also produce a human-readable diff report for dry-run output showing additions (+), removals (-), and modifications (~) with field-level detail.

## Why
The diff is the bridge between the pure declarative world (desired state from reconciliation) and the imperative world (operations that change the system). Without accurate diff generation, netfyr cannot know what changes are needed -- it would have to blindly re-apply everything. Per-field comparison ensures minimal changes: only fields that actually differ are modified, reducing unnecessary system churn and making dry-run output meaningful and trustworthy.

## User interaction
- `netfyr apply --dry-run <path>...` -- runs reconciliation, queries system state, generates diff, and displays the human-readable report without applying.
- `netfyr apply <path>...` -- generates the diff internally and passes it to the backend for application.

### Diff output format (human-readable)
```
+ ethernet eth1                          # Add: entity exists in desired, not in actual
+   mtu: 1500
+   addresses: [10.0.2.50/24]

- vlan bond0.200                         # Remove: entity exists in actual, not in desired
-   vlan-id: 200
-   parent: bond0

~ ethernet eth0                          # Modify: entity exists in both, fields differ
    -mtu: 1500                           # Scalar: separate lines for old/new
    +mtu: 9000
    addresses:                           # List: per-element unified diff
      +10.0.1.51/24                      #   element added
      -10.0.1.99/24                      #   element removed
       10.0.1.50/24                      #   element unchanged (context)

No changes: bond bond0, dns global       # Entities that match desired state
```

## Implementation details
- Crate: `netfyr-reconcile`
- Files: `src/diff.rs`, `src/report.rs`
- Dependencies (external crates): none

### Core types (`src/diff.rs`)

```rust
/// A single operation in the diff.
pub struct DiffOperation {
    pub kind: DiffKind,
    pub entity_type: EntityType,
    pub selector: Selector,
    pub field_changes: Vec<FieldChange>,
}

pub enum DiffKind {
    Add,     // Entity exists in desired, not in actual
    Remove,  // Entity exists in actual, not in desired
    Modify,  // Entity exists in both, fields differ
}

pub struct FieldChange {
    pub field_name: FieldName,
    pub change: FieldChangeKind,
}

pub enum FieldChangeKind {
    /// Field is being set (new value). current is None for Add, or the old value for Modify.
    Set { current: Option<FieldValue>, desired: FieldValue },
    /// Field is being removed. Only in Remove operations or when a field is dropped.
    Unset { current: FieldValue },
    /// Field value is unchanged (for context in reports, not included in apply operations).
    Unchanged { value: FieldValue },
}
```

**`StateDiff`** (`src/diff.rs`):
```rust
pub struct StateDiff {
    pub operations: Vec<DiffOperation>,
}

impl StateDiff {
    pub fn is_empty(&self) -> bool;
    pub fn additions(&self) -> impl Iterator<Item = &DiffOperation>;
    pub fn removals(&self) -> impl Iterator<Item = &DiffOperation>;
    pub fn modifications(&self) -> impl Iterator<Item = &DiffOperation>;
    pub fn len(&self) -> usize;
}
```

### Diff algorithm (`src/diff.rs`)

```rust
pub fn generate_diff(
    desired: &StateSet,
    actual: &StateSet,
    managed_entities: &HashSet<EntityKey>,
) -> StateDiff;
```

The `managed_entities` set contains all entity keys that are explicitly targeted by at least one active policy (via its selector). This set is computed by the caller (the daemon's reconciler) from the full policy list — not from the produced states, since a policy may manage an entity before it produces any state (e.g., a DHCPv4 factory that hasn't acquired a lease yet). **Only managed entities can generate Remove operations.** Unmanaged entities in the system are left untouched.

Algorithm:
1. Build lookup maps for both StateSets keyed by `EntityKey` (entity_type + selector).
2. **Additions**: For each entity in `desired` not in `actual`:
   - Create a `DiffOperation` with `kind: Add`.
   - All fields become `FieldChange::Set { current: None, desired: value }`.
3. **Removals**: For each entity in `actual` not in `desired`:
   - **Skip if the entity is not in `managed_entities`** — the entity is not managed by any policy and must be left as-is.
   - If managed: create a `DiffOperation` with `kind: Remove`.
   - All fields become `FieldChange::Unset { current: value }`.
4. **Modifications**: For each entity in both `desired` and `actual`:
   - Compare fields. Collect:
     - Fields in desired but not actual: `Set { current: None, desired }` (field added).
     - Fields in actual but not desired: `Unset { current }` (field removed).
     - Fields in both with different values: `Set { current: Some(old), desired: new }` (field changed).
     - Fields in both with same values: `Unchanged { value }` (for reporting only).
   - If any non-Unchanged changes exist, create a `DiffOperation` with `kind: Modify`.
   - If all fields are Unchanged, the entity is not included in the diff (no operation needed).

### Entity matching

Entities are matched between desired and actual by `EntityKey` (entity_type + selector). The selector comparison must be canonical:
- Interface names: exact string match.
- MAC addresses: case-insensitive comparison.
- For selectors with `name` field, `name` is the primary match key.
- For selectors without `name` (e.g., device-agnostic policies matching by driver), the backend query should resolve to concrete names before diffing.

### Human-readable report (`src/report.rs`)

```rust
pub struct DiffReport {
    pub operations: Vec<DiffOperation>,
    pub unchanged_entities: Vec<EntityKey>,
}

impl DiffReport {
    /// Format as human-readable diff text (for CLI output).
    pub fn format_text(&self) -> String;
    /// Format as YAML.
    pub fn format_yaml(&self) -> String;
    /// Format as JSON.
    pub fn format_json(&self) -> String;
}
```

The text format uses colored unified-diff style:
- `+` prefix for additions (green in terminal).
- `-` prefix for removals (red in terminal).
- `~` prefix for modified entities (yellow in terminal).
- A leading space (no prefix) for unchanged elements shown for context.
- Indentation for fields within an entity.
- Scalar field changes show separate `-old` / `+new` lines (e.g., `-mtu: 1500` / `+mtu: 9000`).
- List field changes (addresses, routes) show a field-name header followed by per-element lines prefixed with `+` (added), `-` (removed), or ` ` (unchanged context). Only changed elements are required; unchanged elements may be shown for context. Route elements use the format `destination metric N` (omitting metric when 0).

### Edge cases

- **Empty desired state with managed entities**: Only managed entities become Remove operations. Unmanaged entities are ignored.
- **Empty desired state with empty managed set**: No Remove operations at all — nothing is managed, so nothing is removed.
- **Empty actual state**: All desired entities become Add operations (everything is new).
- **Both empty**: Empty diff, `is_empty()` returns true.
- **Identical states**: Empty diff, all entities listed in `unchanged_entities`.
- **Read-only fields in actual but not in desired**: These are excluded from diff. Read-only fields (carrier, speed) are informational and not part of the desired state. They appear in query output but should not generate Modify operations.
- **Unmanaged entities**: Entities present in the system but not targeted by any policy are completely invisible to the diff — they generate no operations and do not appear in `unchanged_entities`.

## Depends on
- SPEC-004 (StateSet type, State, FieldValue comparison)
- SPEC-201 (Reconciliation engine that produces the effective/desired StateSet)

## Acceptance criteria
```gherkin
Feature: Diff generation between desired and actual state
  Scenario: Entity exists in desired but not actual generates Add
    Given desired StateSet contains ethernet/eth0 with mtu=1500, addresses=["10.0.1.50/24"]
    And actual StateSet does not contain ethernet/eth0
    When generate_diff is called
    Then the StateDiff contains 1 operation with kind=Add for ethernet/eth0
    And the operation has field changes: mtu Set(None→1500), addresses Set(None→["10.0.1.50/24"])

  Scenario: Managed entity in actual but not desired generates Remove
    Given desired StateSet does not contain ethernet/eth0
    And actual StateSet contains ethernet/eth0 with mtu=1500, addresses=["10.0.1.50/24"]
    And managed_entities contains ethernet/eth0
    When generate_diff is called
    Then the StateDiff contains 1 operation with kind=Remove for ethernet/eth0
    And the operation has field changes: mtu Unset(1500), addresses Unset(["10.0.1.50/24"])

  Scenario: Unmanaged entity in actual but not desired is left alone
    Given desired StateSet does not contain ethernet/eth1
    And actual StateSet contains ethernet/eth1 with mtu=1500, operstate=up
    And managed_entities does NOT contain ethernet/eth1
    When generate_diff is called
    Then the StateDiff contains no operations for ethernet/eth1
    And ethernet/eth1 is not in unchanged_entities either (completely ignored)

  Scenario: Entity in both with different field values generates Modify
    Given desired ethernet/eth0 with mtu=9000, addresses=["10.0.1.50/24"]
    And actual ethernet/eth0 with mtu=1500, addresses=["10.0.1.50/24"]
    When generate_diff is called
    Then the StateDiff contains 1 operation with kind=Modify for ethernet/eth0
    And the operation has field change: mtu Set(Some(1500)→9000)
    And addresses is Unchanged

  Scenario: Entity in both with identical fields generates no operation
    Given desired ethernet/eth0 with mtu=1500, addresses=["10.0.1.50/24"]
    And actual ethernet/eth0 with mtu=1500, addresses=["10.0.1.50/24"]
    When generate_diff is called
    Then the StateDiff is empty (no operations)

  Scenario: Field added to existing entity
    Given desired ethernet/eth0 with mtu=1500, addresses=["10.0.1.50/24"]
    And actual ethernet/eth0 with mtu=1500 (no addresses)
    When generate_diff is called
    Then the Modify operation has: addresses Set(None→["10.0.1.50/24"]), mtu Unchanged

  Scenario: Field removed from existing entity
    Given desired ethernet/eth0 with mtu=1500
    And actual ethernet/eth0 with mtu=1500, addresses=["10.0.1.50/24"]
    When generate_diff is called
    Then the Modify operation has: addresses Unset(["10.0.1.50/24"]), mtu Unchanged

  Scenario: Multiple entities with mixed operations
    Given desired contains: ethernet/eth0 (mtu=9000), ethernet/eth2 (mtu=1500)
    And actual contains: ethernet/eth0 (mtu=1500), ethernet/eth1 (mtu=1500), ethernet/eth3 (mtu=1500)
    And managed_entities contains ethernet/eth0, ethernet/eth1, ethernet/eth2
    When generate_diff is called
    Then the StateDiff has 3 operations:
      | kind   | entity        |
      | Modify | ethernet/eth0 |
      | Remove | ethernet/eth1 |
      | Add    | ethernet/eth2 |
    And ethernet/eth3 is not in the diff (unmanaged)

  Scenario: Empty desired state only removes managed entities
    Given desired StateSet is empty
    And actual StateSet contains ethernet/eth0, ethernet/eth1, and ethernet/eth2
    And managed_entities contains only ethernet/eth0 and ethernet/eth1
    When generate_diff is called
    Then the StateDiff has 2 Remove operations (eth0 and eth1)
    And ethernet/eth2 is not in the diff (unmanaged)

  Scenario: Empty actual state adds everything
    Given desired StateSet contains ethernet/eth0 and ethernet/eth1
    And actual StateSet is empty
    When generate_diff is called
    Then the StateDiff has 2 Add operations

  Scenario: Both states empty produces empty diff
    Given desired StateSet is empty
    And actual StateSet is empty
    When generate_diff is called
    Then the StateDiff is empty
    And is_empty() returns true

  Scenario: Human-readable format shows additions with + prefix
    Given a StateDiff with an Add operation for ethernet/eth0 with mtu=1500
    When format_text() is called on the DiffReport
    Then the output contains "+ ethernet eth0"
    And the output contains "+   mtu: 1500"

  Scenario: Human-readable format shows removals with - prefix
    Given a StateDiff with a Remove operation for vlan/bond0.200
    When format_text() is called
    Then the output contains "- vlan bond0.200"

  Scenario: Human-readable format shows scalar modifications as unified diff lines
    Given a StateDiff with a Modify operation changing mtu from 1500 to 9000
    When format_text() is called
    Then the output contains a red line "    -mtu: 1500"
    And the output contains a green line "    +mtu: 9000"

  Scenario: Human-readable format shows list field changes as per-element diff
    Given a StateDiff with a Modify operation where addresses changed
    And the old value is ["10.0.1.50/24","10.0.1.99/24"]
    And the new value is ["10.0.1.50/24","10.0.1.51/24"]
    When format_text() is called
    Then the output contains a field header "    addresses:"
    And the output contains a green line "      +10.0.1.51/24"
    And the output contains a red line "      -10.0.1.99/24"

  Scenario: Read-only fields from actual state are excluded from diff
    Given desired ethernet/eth0 with mtu=1500
    And actual ethernet/eth0 with mtu=1500, carrier=true, speed=1000
    When generate_diff is called
    Then the StateDiff is empty (carrier and speed are read-only, not diffed)

  Scenario: Diff accessors filter by operation kind
    Given a StateDiff with 2 Add, 1 Modify, and 1 Remove operations
    When additions() is called it returns 2 operations
    And removals() returns 1 operation
    And modifications() returns 1 operation
    And len() returns 4
```
