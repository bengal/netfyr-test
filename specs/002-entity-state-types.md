# SPEC-002: State Types

## What
Define the core state types in the `netfyr-state` crate that represent the desired or current configuration of a network entity. These types form the foundational data model for the entire system.

Types to implement:

1. **`State`** -- The top-level type representing one network entity's configuration. Fields:
   - `entity_type: String` -- the kind of entity (e.g., `"ethernet"`, `"bond"`, `"vlan"`)
   - `selector: Selector` -- identifies which system entity this targets (defined in SPEC-003, use a placeholder for now)
   - `fields: IndexMap<String, FieldValue>` -- ordered key-value configuration fields
   - `metadata: StateMetadata` -- identity and tracking information
   - `policy_ref: Option<String>` -- name of the policy that produced this state (set by the factory during production)
   - `priority: u32` -- numeric priority for field-level conflict resolution (higher wins, default 100)

2. **`FieldValue`** -- A field's value paired with its provenance. Fields:
   - `value: Value` -- the actual value
   - `provenance: Provenance` -- where this value came from

3. **`Value`** enum -- The set of possible value types:
   - `String(String)`
   - `U64(u64)`
   - `I64(i64)`
   - `Bool(bool)`
   - `IpAddr(std::net::IpAddr)`
   - `IpNetwork(ipnetwork::IpNetwork)`
   - `List(Vec<Value>)`
   - `Map(IndexMap<String, Value>)`

4. **`Provenance`** enum -- Tracks where a field value originated:
   - `UserConfigured { policy_ref: String }` -- explicitly set by a user in a policy
   - `KernelDefault` -- never changed, reflects kernel initial value
   - `ExternalTool { tool: String, detected_at: chrono::DateTime<chrono::Utc> }` -- change detected from iproute2, NetworkManager, etc.
   - `Derived { reason: String }` -- computed by netfyr (e.g., auto-calculated broadcast address)

5. **`StateMetadata`** -- Identity and tracking metadata. Fields:
   - `id: uuid::Uuid` -- UUIDv7 (time-ordered) unique identifier for this state instance
   - `timeline_id: uuid::Uuid` -- stable across versions of the same logical entity
   - `created_at: chrono::DateTime<chrono::Utc>` -- when this state was created
   - `labels: HashMap<String, String>` -- user-defined key-value labels
   - `description: Option<String>` -- optional human-readable description

All types must derive or implement: `Clone`, `Debug`, `Serialize`, `Deserialize`, `PartialEq`.

## Why
The state layer is the foundation of netfyr's architecture. Every layer above -- policies, reconciliation, backend, daemon -- operates on these types. Per-field provenance tracking is a key differentiator that allows answering "where did this value come from?" at any time. The `Value` enum must support all data types found in network configuration (strings, integers, booleans, IP addresses, CIDR networks, nested lists, and maps for complex structures like routes). Using `IndexMap` for fields and map values preserves insertion order, which matters for deterministic YAML serialization and user-facing output.

## User interaction
These types are internal library types. Other crates (netfyr-policy, netfyr-reconcile, netfyr-backend, netfyr-daemon) depend on them to construct, manipulate, and serialize network state. Users interact with them indirectly through YAML files (SPEC-005) and CLI output (SPEC-302).

## Implementation details
- Crate: `netfyr-state`
- Files:
  - `src/lib.rs` -- public re-exports of all types
  - `src/entity.rs` -- `State` struct
  - `src/field.rs` -- `FieldValue` struct
  - `src/value.rs` -- `Value` enum with Display impl and convenience constructors
  - `src/provenance.rs` -- `Provenance` enum
  - `src/metadata.rs` -- `StateMetadata` struct with a `new()` constructor that generates a UUIDv7 and sets `created_at` to now
- Dependencies (external crates):
  - `serde` (with `derive` feature) -- serialization framework
  - `serde_json` -- needed for Value conversions and testing
  - `chrono` (with `serde` feature) -- timestamps
  - `uuid` (with `v7` and `serde` features) -- UUIDv7 generation
  - `ipnetwork` (with `serde` feature) -- CIDR network representation
  - `indexmap` (with `serde` feature) -- ordered maps

### Value enum details

The `Value` enum must implement:
- `From<String>`, `From<&str>`, `From<u64>`, `From<i64>`, `From<bool>`, `From<IpAddr>`, `From<IpNetwork>` -- ergonomic construction
- `Display` -- human-readable formatting (e.g., IP addresses as dotted notation, lists as `[a, b, c]`)
- A `as_str()` -> `Option<&str>` accessor, and similar typed accessors (`as_u64`, `as_bool`, etc.)

### StateMetadata::new()

```rust
impl StateMetadata {
    pub fn new() -> Self {
        Self {
            id: Uuid::now_v7(),
            timeline_id: Uuid::now_v7(),
            created_at: Utc::now(),
            labels: HashMap::new(),
            description: None,
        }
    }
}
```

### Placeholder for Selector

Until SPEC-003 is implemented, `Selector` can be a simple struct with a `name: Option<String>` field. It will be expanded in SPEC-003.

## Depends on
- SPEC-001

## Acceptance criteria
```gherkin
Feature: Entity state types
  Scenario: Create an State with all fields populated
    Given the netfyr-state crate is available
    When an State is constructed with entity_type "ethernet", a selector with name "eth0", fields containing "mtu" = U64(1500) with UserConfigured provenance, metadata with labels {"role": "uplink"}, policy_ref "eth0-policy", and priority 100
    Then the State can be cloned
    And the clone is equal to the original via PartialEq
    And Debug formatting produces a non-empty string

  Scenario: FieldValue pairs a value with provenance
    Given a Value of U64(9000) and Provenance of UserConfigured with policy_ref "bond0"
    When a FieldValue is constructed from them
    Then field_value.value equals Value::U64(9000)
    And field_value.provenance equals Provenance::UserConfigured { policy_ref: "bond0".into() }

  Scenario: Value enum supports all required types
    Given the Value enum
    When values are created for each variant: String("eth0"), U64(1500), I64(-1), Bool(true), IpAddr(10.0.1.1), IpNetwork(10.0.1.0/24), List([String("a"), String("b")]), Map({"key": String("val")})
    Then each value can be constructed without errors
    And each value implements Clone, Debug, PartialEq

  Scenario: Value From trait conversions work
    Given the From implementations on Value
    When Value::from("hello") is called
    Then it produces Value::String("hello".to_string())
    When Value::from(42u64) is called
    Then it produces Value::U64(42)
    When Value::from(true) is called
    Then it produces Value::Bool(true)

  Scenario: Value typed accessors return correct results
    Given a Value::U64(1500)
    When as_u64() is called
    Then it returns Some(1500)
    When as_str() is called
    Then it returns None
    When as_bool() is called
    Then it returns None

  Scenario: Provenance enum captures all source types
    Given the Provenance enum
    When a UserConfigured variant is created with policy_ref "my-policy"
    Then its policy_ref field is "my-policy"
    When a KernelDefault variant is created
    Then it has no additional fields
    When an ExternalTool variant is created with tool "iproute2" and a timestamp
    Then its tool field is "iproute2" and detected_at is the provided timestamp
    When a Derived variant is created with reason "auto-broadcast"
    Then its reason field is "auto-broadcast"

  Scenario: StateMetadata generates unique IDs
    Given the StateMetadata::new() constructor
    When two StateMetadata instances are created
    Then their id fields are different UUIDv7 values
    And their timeline_id fields are different UUIDv7 values
    And their created_at fields are within 1 second of the current time
    And labels is an empty HashMap
    And description is None

  Scenario: All types serialize and deserialize with serde
    Given an State with populated fields
    When it is serialized to JSON via serde_json
    Then the JSON string is valid
    When the JSON string is deserialized back to an State
    Then the deserialized value equals the original

  Scenario: State fields preserve insertion order
    Given an State with fields inserted in order: "mtu", "addresses", "routes"
    When the fields are iterated
    Then they appear in order: "mtu", "addresses", "routes"
```
