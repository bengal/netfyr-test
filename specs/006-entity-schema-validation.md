# SPEC-006: Entity Schema Validation

## What
Implement JSON Schema-based validation for entity types in `netfyr-state`. Schemas are embedded in the binary at compile time and define which fields are valid for each entity type, their types and constraints, whether they are required or optional, and whether they are writable (can be set in policies) or read-only (only returned by queries).

### Components

1. **`SchemaRegistry`** -- A registry that holds all entity type schemas and provides validation. Methods:
   - `new() -> SchemaRegistry` -- creates a registry pre-loaded with all embedded schemas
   - `validate(state: &State) -> Result<(), ValidationErrors>` -- validates an State against the schema for its entity_type. Returns all validation errors (not just the first).
   - `validate_writable(state: &State) -> Result<(), ValidationErrors>` -- like `validate()`, but also rejects any read-only fields that are present. Used when validating user-provided policy state (users should not set read-only fields).
   - `get_schema(entity_type: &str) -> Option<&EntitySchema>` -- get the parsed schema for an entity type
   - `entity_types() -> Vec<&str>` -- list all known entity types
   - `field_info(entity_type: &str, field: &str) -> Option<FieldSchemaInfo>` -- get metadata about a specific field (type, constraints, required, writable)

2. **`EntitySchema`** -- Parsed representation of a JSON Schema for one entity type. Contains the raw JSON Schema for validation and parsed field metadata for programmatic access.

3. **`FieldSchemaInfo`** -- Metadata about a single field:
   - `field_type: FieldType` -- the expected type (String, U32, U64, Bool, Array, Object, IpAddress, IpNetwork, MacAddress)
   - `required: bool`
   - `writable: bool` -- false means read-only
   - `constraints: Option<FieldConstraints>` -- min/max for integers, pattern for strings, item schema for arrays
   - `description: Option<String>`

4. **`ValidationErrors`** -- A collection of `ValidationError` values, each containing:
   - `field: String` -- the field path (e.g., `"mtu"`, `"routes[0].gateway"`)
   - `message: String` -- human-readable error description
   - `kind: ValidationErrorKind` -- enum: `InvalidType`, `OutOfRange`, `UnknownField`, `MissingRequired`, `ReadOnlyField`, `InvalidFormat`, `ConstraintViolation`

### Ethernet schema

The first schema to implement, embedded as a JSON Schema file:

| Field | Type | Constraints | Required | Writable | Description |
|---|---|---|---|---|---|
| `mtu` | integer | min: 68, max: 65535 | no | yes | Maximum transmission unit |
| `addresses` | array of strings | each item: CIDR format (e.g., "10.0.1.50/24") | no | yes | IP addresses assigned to the interface |
| `mac` | string | MAC address format (XX:XX:XX:XX:XX:XX) | no | **read-only** | Hardware MAC address |
| `carrier` | boolean | — | no | **read-only** | Whether the link has carrier |
| `speed` | integer | min: 0 | no | **read-only** | Link speed in Mbps |
| `routes` | array of objects | see below | no | yes | Routes via this interface |

Route object schema:
| Field | Type | Constraints | Required |
|---|---|---|---|
| `destination` | string | CIDR format | yes |
| `gateway` | string | IP address format | no |
| `metric` | integer | min: 0 | no |

### Schema embedding

Schemas are embedded in the binary using `include_str!()` on JSON files stored in the source tree. The `SchemaRegistry::new()` constructor loads and compiles all embedded schemas at initialization.

```rust
const ETHERNET_SCHEMA: &str = include_str!("schemas/ethernet.json");
```

## Why
Schema validation catches configuration errors before any system changes are made. Without validation, a typo like `mtt: 1500` (instead of `mtu`) would silently be ignored or cause confusing backend errors. The writable/read-only distinction prevents users from trying to set hardware properties (like MAC address or carrier status) in their policies -- these fields only appear in query output. Embedding schemas in the binary ensures they are always available without external file dependencies, and the JSON Schema standard allows potential future tooling integration (editor autocompletion, documentation generation).

## User interaction
Users benefit from validation implicitly -- when they run `netfyr apply` or `netfyr apply --dry-run`, their YAML is validated against the schema before any system changes. Validation errors are reported with clear messages:
```
Error: validation failed for ethernet/eth0:
  - field "mtt": unknown field (did you mean "mtu"?)
  - field "mtu": value 99999 exceeds maximum 65535
  - field "mac": read-only field cannot be set in a policy
```

## Implementation details
- Crate: `netfyr-state`
- Files:
  - `src/schema.rs` -- `SchemaRegistry`, `EntitySchema`, `FieldSchemaInfo`, `ValidationErrors`, `ValidationError`, `ValidationErrorKind`, `FieldType`, `FieldConstraints`
  - `src/schemas/ethernet.json` -- JSON Schema for the ethernet entity type
  - `src/lib.rs` -- update to re-export schema types
- Dependencies (external crates):
  - `jsonschema` -- JSON Schema validation library
  - `serde_json` -- needed for JSON Schema handling (already a dependency from SPEC-002)

### JSON Schema file format

The schema file is a standard JSON Schema (draft 2020-12 or 07) with custom extensions for netfyr-specific metadata:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "ethernet",
  "description": "Ethernet interface entity",
  "type": "object",
  "properties": {
    "mtu": {
      "type": "integer",
      "minimum": 68,
      "maximum": 65535,
      "description": "Maximum transmission unit",
      "x-netfyr-writable": true
    },
    "mac": {
      "type": "string",
      "pattern": "^([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}$",
      "description": "Hardware MAC address",
      "x-netfyr-writable": false
    }
  },
  "additionalProperties": false
}
```

The `x-netfyr-writable` extension property indicates whether a field can be set in policies (true) or is read-only (false, default for query-only fields). The `additionalProperties: false` setting ensures unknown fields are rejected.

### Validation flow
1. Look up schema for `entity_state.entity_type`
2. If no schema exists, return error `UnknownEntityType`
3. Convert `State.fields` to a JSON value
4. Run JSON Schema validation via `jsonschema` crate
5. Collect all validation errors
6. If `validate_writable()`, additionally check each field against `x-netfyr-writable` metadata

### Converting Value to JSON for validation
The `Value` enum must implement `Into<serde_json::Value>` for schema validation:
- `Value::String(s)` -> `json!(s)`
- `Value::U64(n)` -> `json!(n)`
- `Value::I64(n)` -> `json!(n)`
- `Value::Bool(b)` -> `json!(b)`
- `Value::IpAddr(ip)` -> `json!(ip.to_string())`
- `Value::IpNetwork(net)` -> `json!(net.to_string())`
- `Value::List(v)` -> JSON array
- `Value::Map(m)` -> JSON object

## Depends on
- SPEC-002
- SPEC-005

## Acceptance criteria
```gherkin
Feature: Schema registry initialization
  Scenario: Registry loads with embedded schemas
    Given a new SchemaRegistry is created
    When entity_types() is called
    Then it includes "ethernet"
    And get_schema("ethernet") returns Some

  Scenario: Unknown entity type returns None
    Given a new SchemaRegistry
    When get_schema("nonexistent") is called
    Then it returns None

Feature: Ethernet schema validation
  Scenario: Valid ethernet state passes validation
    Given an State with entity_type "ethernet" and fields:
      | field     | value          |
      | mtu       | 1500           |
      | addresses | ["10.0.1.50/24"] |
    When validate() is called
    Then it returns Ok

  Scenario: MTU below minimum is rejected
    Given an State with entity_type "ethernet" and field mtu=10
    When validate() is called
    Then it returns ValidationErrors
    And the errors include an OutOfRange error for field "mtu"
    And the message mentions the minimum is 68

  Scenario: MTU above maximum is rejected
    Given an State with entity_type "ethernet" and field mtu=99999
    When validate() is called
    Then it returns ValidationErrors
    And the errors include an OutOfRange error for field "mtu"
    And the message mentions the maximum is 65535

  Scenario: Unknown field is rejected
    Given an State with entity_type "ethernet" and field "mtt"=1500
    When validate() is called
    Then it returns ValidationErrors
    And the errors include an UnknownField error for field "mtt"

  Scenario: Read-only field in writable validation
    Given an State with entity_type "ethernet" and fields mtu=1500 and mac="aa:bb:cc:dd:ee:ff"
    When validate_writable() is called
    Then it returns ValidationErrors
    And the errors include a ReadOnlyField error for field "mac"

  Scenario: Read-only field in regular validation passes
    Given an State with entity_type "ethernet" and fields mtu=1500 and mac="aa:bb:cc:dd:ee:ff"
    When validate() (not validate_writable) is called
    Then it returns Ok (read-only fields are valid in query results)

  Scenario: Route object validation
    Given an State with entity_type "ethernet" and fields:
      | field  | value |
      | routes | [{"destination": "0.0.0.0/0", "gateway": "10.0.1.1", "metric": 100}] |
    When validate() is called
    Then it returns Ok

  Scenario: Route without required destination is rejected
    Given an State with entity_type "ethernet" and fields:
      | field  | value |
      | routes | [{"gateway": "10.0.1.1"}] |
    When validate() is called
    Then it returns ValidationErrors
    And the errors include a MissingRequired error for "routes[0].destination"

  Scenario: Multiple validation errors are collected
    Given an State with entity_type "ethernet" and fields mtu=99999 and an unknown field "foo"="bar"
    When validate() is called
    Then it returns ValidationErrors with at least 2 errors
    And one error is for field "mtu" and another for field "foo"

Feature: Field info queries
  Scenario: Query field metadata
    Given a new SchemaRegistry
    When field_info("ethernet", "mtu") is called
    Then it returns Some with field_type integer, writable true, required false
    And constraints include min=68 and max=65535

  Scenario: Query read-only field metadata
    Given a new SchemaRegistry
    When field_info("ethernet", "carrier") is called
    Then it returns Some with field_type boolean, writable false

  Scenario: Query unknown field returns None
    Given a new SchemaRegistry
    When field_info("ethernet", "nonexistent") is called
    Then it returns None

Feature: Unknown entity type handling
  Scenario: Validate state with unknown entity type
    Given an State with entity_type "nonexistent"
    When validate() is called
    Then it returns a ValidationErrors containing an error about unknown entity type
```
