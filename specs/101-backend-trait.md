# SPEC-101: Backend Trait and Registry

## What
Define the `NetworkBackend` trait in the `netfyr-backend` crate, along with supporting types `ApplyReport`, `DryRunReport`, and `BackendRegistry`. The trait provides a uniform interface for querying system network state, applying configuration changes, performing dry-run simulations, and advertising supported entity types. The `BackendRegistry` maps entity types to their concrete backend implementations so the reconciliation engine can dispatch operations to the correct backend.

## Why
The architecture requires a backend abstraction layer that is the only layer performing I/O to the kernel. Multiple backends must coexist (e.g., rtnetlink for interfaces, nft for firewall rules). A trait-based design allows swappable implementations, testability via mock backends, and clean separation between the pure state/reconciliation layers and the system-touching layer. The registry enables entity-type routing so that a single reconciliation pass can drive multiple backends transparently.

## User interaction
Users do not interact with the backend trait directly. The reconciliation engine and CLI commands (`apply`, `query`, `diff`) use the `BackendRegistry` to look up the appropriate `NetworkBackend` implementation for each entity type. Backend implementors implement the trait for their specific kernel interface (rtnetlink, nftables, wpa_supplicant, etc.).

## Implementation details
- Crate: `netfyr-backend`
- Files: `src/lib.rs`, `src/trait_.rs`, `src/report.rs`, `src/registry.rs`
- Dependencies (external crates): none (pure trait definitions; `async-trait` for async methods, `thiserror` for error types)

### Type definitions

**`NetworkBackend` trait** (`src/trait_.rs`):
```rust
#[async_trait]
pub trait NetworkBackend: Send + Sync {
    /// Query entities of a specific type, optionally filtered by selector.
    /// Returns a StateSet containing the current system state for matching entities.
    async fn query(
        &self,
        entity_type: &EntityType,
        selector: Option<&Selector>,
    ) -> Result<StateSet, BackendError>;

    /// Query all entities supported by this backend.
    /// Returns a StateSet containing the current system state for all supported entity types.
    async fn query_all(&self) -> Result<StateSet, BackendError>;

    /// Apply a StateDiff to the system. Executes each operation (add, modify, remove)
    /// and returns a report detailing successes, failures, and skips.
    async fn apply(&self, diff: &StateDiff) -> Result<ApplyReport, BackendError>;

    /// Simulate applying a StateDiff without making changes.
    /// Returns a report of what would happen.
    async fn dry_run(&self, diff: &StateDiff) -> Result<DryRunReport, BackendError>;

    /// Return the list of entity types this backend can handle.
    fn supported_entities(&self) -> &[EntityType];
}
```

**`ApplyReport`** (`src/report.rs`):
- `succeeded: Vec<AppliedOperation>` -- operations that completed successfully, each containing the operation kind (Add/Modify/Remove), entity type, selector, and list of fields changed.
- `failed: Vec<FailedOperation>` -- operations that failed, each containing the operation kind, entity type, selector, the error (as a `BackendError`), and which fields were involved.
- `skipped: Vec<SkippedOperation>` -- operations that were skipped (e.g., entity already in desired state, or skipped due to a failed dependency), each containing the operation kind, entity type, selector, and reason string.
- Helper methods: `is_success() -> bool` (no failures), `is_partial() -> bool` (some succeeded, some failed), `is_total_failure() -> bool` (all failed), `summary() -> String`.

**`DryRunReport`** (`src/report.rs`):
- `changes: Vec<PlannedChange>` -- list of changes that would be made, each containing the operation kind (Add/Modify/Remove), entity type, selector, and a list of `FieldChange` items.
- `FieldChange` contains: field name, current value (Option), desired value (Option), change kind (Set/Unset/Modify).
- Helper method: `is_empty() -> bool`, `summary() -> String`.

**`BackendRegistry`** (`src/registry.rs`):
- Holds a `HashMap<EntityType, Arc<dyn NetworkBackend>>`.
- `register(&mut self, backend: Arc<dyn NetworkBackend>)` -- registers a backend for all its `supported_entities()`. Returns error if an entity type is already registered to a different backend.
- `get(&self, entity_type: &EntityType) -> Option<Arc<dyn NetworkBackend>>` -- look up backend by entity type.
- `query(&self, entity_type: &EntityType, selector: Option<&Selector>) -> Result<StateSet, BackendError>` -- convenience method that looks up and delegates.
- `query_all(&self) -> Result<StateSet, BackendError>` -- queries all registered backends and merges results into one StateSet.
- `apply(&self, diff: &StateDiff) -> Result<ApplyReport, BackendError>` -- partitions the diff by entity type, dispatches to appropriate backends, and merges ApplyReports.
- `supported_entities(&self) -> Vec<EntityType>` -- returns all registered entity types.

**`BackendError`** (`src/lib.rs`):
- Enum with variants: `UnsupportedEntityType(EntityType)`, `QueryFailed { entity_type: EntityType, source: Box<dyn Error> }`, `ApplyFailed { operation: String, source: Box<dyn Error> }`, `NotFound { entity_type: EntityType, selector: Selector }`, `PermissionDenied(String)`, `Internal(String)`.

## Depends on
- SPEC-002 (EntityType, Selector, and core state types defined in netfyr-state)
- SPEC-004 (StateSet, StateDiff, and set operations defined in netfyr-state)

## Acceptance criteria
```gherkin
Feature: NetworkBackend trait definition
  Scenario: A backend implements all required trait methods
    Given a struct "MockBackend" that implements NetworkBackend
    When the struct provides implementations for query, query_all, apply, dry_run, and supported_entities
    Then the code compiles successfully
    And the struct can be used as "dyn NetworkBackend"

  Scenario: Backend query returns a StateSet
    Given a MockBackend that supports entity type "ethernet"
    When query is called with entity_type "ethernet" and no selector
    Then the result is Ok containing a StateSet
    And the StateSet contains zero or more State entries of type "ethernet"

  Scenario: Backend query with selector filters results
    Given a MockBackend with two ethernet entities "eth0" and "eth1"
    When query is called with entity_type "ethernet" and selector name="eth0"
    Then the result is Ok containing a StateSet with exactly one entity
    And the entity has selector name "eth0"

  Scenario: Backend query for unsupported entity type
    Given a MockBackend that supports only entity type "ethernet"
    When query is called with entity_type "bond"
    Then the result is Err with BackendError::UnsupportedEntityType

  Scenario: Backend apply returns a detailed ApplyReport
    Given a MockBackend and a StateDiff with 3 operations
    When apply is called with the diff
    And 2 operations succeed and 1 fails
    Then the ApplyReport has 2 entries in succeeded and 1 entry in failed
    And is_partial() returns true
    And is_success() returns false

  Scenario: ApplyReport with all operations successful
    Given an ApplyReport where all operations are in succeeded
    And failed is empty and skipped is empty
    Then is_success() returns true
    And is_partial() returns false
    And is_total_failure() returns false

  Scenario: ApplyReport with all operations failed
    Given an ApplyReport where succeeded is empty and all operations are in failed
    Then is_total_failure() returns true
    And is_success() returns false

  Scenario: ApplyReport with skipped operations
    Given a StateDiff with 3 operations
    When apply is called and 1 succeeds, 1 fails, and 1 is skipped
    Then the ApplyReport has 1 succeeded, 1 failed, and 1 skipped
    And each skipped entry has a reason string explaining why it was skipped

  Scenario: Backend dry_run returns a DryRunReport
    Given a MockBackend and a StateDiff with add, modify, and remove operations
    When dry_run is called with the diff
    Then the result is Ok containing a DryRunReport
    And the DryRunReport lists each planned change with operation kind and field changes
    And no system state is modified

  Scenario: DryRunReport field changes show before and after values
    Given a StateDiff that modifies the mtu field of entity "eth0" from 1500 to 9000
    When dry_run is called
    Then the DryRunReport contains a PlannedChange for "eth0"
    And the PlannedChange has a FieldChange with field="mtu", current=Some(1500), desired=Some(9000)

  Scenario: DryRunReport is empty when no changes needed
    Given a StateDiff with zero operations
    When dry_run is called
    Then the DryRunReport has an empty changes list
    And is_empty() returns true

Feature: BackendRegistry routes entity types to backends
  Scenario: Register a backend and look it up by entity type
    Given an empty BackendRegistry
    And a MockBackend that supports entity types ["ethernet", "bond"]
    When register is called with the MockBackend
    Then get("ethernet") returns Some(backend)
    And get("bond") returns Some(backend)
    And get("vlan") returns None

  Scenario: Register two backends for different entity types
    Given an empty BackendRegistry
    And a NetlinkBackend that supports ["ethernet", "bond", "vlan"]
    And an NftBackend that supports ["firewall-rule"]
    When both backends are registered
    Then get("ethernet") returns the NetlinkBackend
    And get("firewall-rule") returns the NftBackend

  Scenario: Registering a conflicting entity type fails
    Given a BackendRegistry with a NetlinkBackend registered for "ethernet"
    When register is called with another backend that also supports "ethernet"
    Then the register call returns an error indicating the conflict
    And the original registration is preserved

  Scenario: Registry query_all queries all registered backends
    Given a BackendRegistry with two backends registered
    When query_all is called
    Then both backends are queried
    And the results are merged into a single StateSet

  Scenario: Registry apply partitions diff by entity type
    Given a BackendRegistry with NetlinkBackend for "ethernet" and NftBackend for "firewall-rule"
    And a StateDiff containing operations for both entity types
    When apply is called on the registry
    Then ethernet operations are dispatched to NetlinkBackend
    And firewall-rule operations are dispatched to NftBackend
    And the ApplyReports are merged into a single report

  Scenario: Registry apply with unknown entity type in diff
    Given a BackendRegistry with only "ethernet" registered
    And a StateDiff containing an operation for entity type "wifi"
    When apply is called on the registry
    Then the wifi operation is reported as failed with UnsupportedEntityType error
    And ethernet operations are still applied normally

  Scenario: Registry supported_entities returns all registered types
    Given a BackendRegistry with backends registered for ["ethernet", "bond", "vlan", "firewall-rule"]
    When supported_entities is called
    Then the result contains all four entity types
```
