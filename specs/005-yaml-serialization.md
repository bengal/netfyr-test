# SPEC-005: YAML Serialization

## What
Implement YAML serialization and deserialization for `State` and `StateSet` in the `netfyr-state` crate. Support two YAML formats, multi-document YAML files, and directory-based loading.

### Flat YAML format

The YAML format uses a flat structure where matching properties (`type`, `name`, `driver`, `mac`, `pci_path`) and configuration properties (`mtu`, `addresses`, `routes`, etc.) coexist at the same level. The parser separates them based on the entity schema — matching properties are mapped to `entity_type` and `Selector`, everything else is mapped to `fields`.

1. **Bare state shorthand** -- A YAML document with no `kind:` field. The simplest format for users:
   ```yaml
   type: ethernet
   name: eth0
   mtu: 1500
   addresses:
     - 10.0.1.50/24
   ```

2. **Explicit format** -- A YAML document with `kind: state` (or wrapped inside a policy, handled by SPEC-007). Contains the same flat structure but explicitly marked:
   ```yaml
   kind: state
   type: ethernet
   name: eth0
   mtu: 1500
   ```

Both formats deserialize to the same `State`. The distinction is: if `kind` is absent or `kind: state`, it is a bare state. The `kind` field is not stored on `State` -- it is only used during parsing to determine format.

### Property classification

The parser classifies top-level YAML keys as follows:
- **Entity type**: `type` → maps to `State.entity_type`
- **Selector properties**: `name`, `driver`, `mac`, `pci_path` → maps to `State.selector`
- **Meta properties**: `kind` → used for format detection, not stored
- **Configuration properties**: everything else → maps to `State.fields`

The set of selector property names is fixed and known to the parser. The entity schema (SPEC-006) validates that configuration properties are valid for the given entity type.

### Multi-document YAML

A single `.yaml` or `.yml` file may contain multiple YAML documents separated by `---`. Each document is parsed independently, producing one `State` per document. This allows declaring multiple entities in a single file.

### Directory loading

A function `load_dir(path: &Path) -> Result<StateSet>` recursively loads all `.yaml` and `.yml` files from a directory, parses all documents from each file, and collects them into a `StateSet`.

### Serialization

`State` serializes to the flat format by default (no `kind:` field). The serializer flattens the internal `State` structure back into a flat YAML mapping with `type`, selector fields, and configuration fields at the same level.

### Field value YAML mapping

The `Value` enum maps to YAML as follows:
- `Value::String` -> YAML string
- `Value::U64` / `Value::I64` -> YAML integer
- `Value::Bool` -> YAML boolean
- `Value::IpAddr` -> YAML string (e.g., `"10.0.1.1"`)
- `Value::IpNetwork` -> YAML string (e.g., `"10.0.1.0/24"`)
- `Value::List` -> YAML sequence
- `Value::Map` -> YAML mapping

Since YAML does not distinguish between IP addresses and strings, deserialization uses heuristics: a string value is first attempted as `IpNetwork`, then as `IpAddr`, and falls back to `String`. Numeric values are parsed as `U64` if non-negative, `I64` if negative. Booleans map directly. When schema validation is available (SPEC-006), the schema provides type hints to disambiguate.

### FieldValue serialization

In YAML, fields are serialized as plain values (without provenance/metadata wrapping). Provenance is not included in the YAML output by default -- it is runtime metadata. When round-tripping through YAML, deserialized `FieldValue` instances get `Provenance::UserConfigured` with the policy_ref from the parent context (or an empty string if standalone).

### Metadata handling

`StateMetadata` is not serialized to YAML in the bare format. When deserializing, a fresh `StateMetadata` is generated via `StateMetadata::new()`. Metadata is only preserved in internal/API serialization formats (JSON), not in user-facing YAML.

## Why
YAML is the primary user-facing configuration format for netfyr. The flat format is the simplest way for users to declare network state -- just write the interface identity and the desired configuration in a natural, readable structure. Multi-document support allows grouping related configurations in a single file (e.g., an interface and its DNS settings). Directory loading enables the `/etc/netfyr/policies/` pattern where each file describes one or more entities and `netfyr apply /etc/netfyr/policies/` loads everything.

## User interaction
Users write YAML files in either format and run `netfyr apply <file-or-dir>`. The CLI and policy layer use `load_dir()` and `parse_yaml()` to convert files into `StateSet` values for reconciliation.

## Implementation details
- Crate: `netfyr-state`
- Files:
  - `src/serde.rs` -- custom serde implementations for flat YAML format handling, `Value` deserialization heuristics, property classification logic
  - `src/loader.rs` -- `parse_yaml(input: &str) -> Result<Vec<State>>` for multi-document parsing, `load_file(path: &Path) -> Result<Vec<State>>`, `load_dir(path: &Path) -> Result<StateSet>`
  - `src/lib.rs` -- update to re-export parsing/loading functions
- Dependencies (external crates):
  - `serde_yaml` -- YAML parsing and emission
  - `walkdir` -- recursive directory traversal for `load_dir`

### Property classification constants

```rust
/// Properties that map to the Selector (matching properties).
const SELECTOR_KEYS: &[&str] = &["name", "driver", "mac", "pci_path"];

/// Properties with special meaning (not stored in State).
const META_KEYS: &[&str] = &["kind", "type"];
```

### parse_yaml pseudocode
```
fn parse_yaml(input: &str) -> Result<Vec<State>> {
    let mut results = Vec::new();
    for document in serde_yaml::Deserializer::from_str(input) {
        let raw: serde_yaml::Value = Deserialize::deserialize(document)?;
        // Check if "kind" field exists and its value
        // If absent or "state": parse as State
        // If other kind: return error (or skip, depending on context)
        let entity = parse_raw_to_state(raw)?;
        results.push(entity);
    }
    Ok(results)
}

fn parse_raw_to_state(raw: serde_yaml::Value) -> Result<State> {
    let map = raw.as_mapping().ok_or("expected a YAML mapping")?;

    // Extract entity type (required)
    let entity_type = map.get("type")
        .and_then(|v| v.as_str())
        .ok_or("missing 'type' field")?
        .to_string();

    // Extract selector properties
    let mut selector = Selector::default();
    for key in SELECTOR_KEYS {
        if let Some(value) = map.get(*key) {
            match *key {
                "name" => selector.name = Some(value.as_str().unwrap().to_string()),
                "driver" => selector.driver = Some(value.as_str().unwrap().to_string()),
                "mac" => selector.mac = Some(value.as_str().unwrap().to_string()),
                "pci_path" => selector.pci_path = Some(value.as_str().unwrap().to_string()),
                _ => {}
            }
        }
    }

    // Everything else goes to fields
    let mut fields = IndexMap::new();
    for (key, value) in map {
        let key_str = key.as_str().unwrap();
        if META_KEYS.contains(&key_str) || SELECTOR_KEYS.contains(&key_str) {
            continue; // skip type, kind, and selector keys
        }
        fields.insert(key_str.to_string(), deserialize_value(value)?);
    }

    Ok(State {
        entity_type,
        selector,
        fields,
        metadata: StateMetadata::new(),
        policy_ref: None,
        priority: 100,
    })
}
```

### Value deserialization heuristic
When deserializing a YAML value into `Value` without schema context:
1. YAML boolean -> `Value::Bool`
2. YAML integer >= 0 -> `Value::U64`
3. YAML integer < 0 -> `Value::I64`
4. YAML string -> try parse as `IpNetwork`, then `IpAddr`, then keep as `Value::String`
5. YAML sequence -> `Value::List` (recurse for each element)
6. YAML mapping -> `Value::Map` (recurse for each value)

### Serialization pseudocode
```
fn serialize_state(state: &State) -> serde_yaml::Value {
    let mut map = serde_yaml::Mapping::new();

    // Write type
    map.insert("type".into(), state.entity_type.clone().into());

    // Write selector properties
    if let Some(name) = &state.selector.name {
        map.insert("name".into(), name.clone().into());
    }
    if let Some(driver) = &state.selector.driver {
        map.insert("driver".into(), driver.clone().into());
    }
    // ... other selector fields

    // Write configuration fields
    for (key, field_value) in &state.fields {
        map.insert(key.clone().into(), serialize_value(&field_value.value));
    }

    serde_yaml::Value::Mapping(map)
}
```

### load_dir behavior
- Recursively walk the directory
- Include only files with `.yaml` or `.yml` extensions
- Skip hidden files (starting with `.`)
- Parse each file, collecting all `State` values
- Insert each into a `StateSet` (key conflicts within a directory are errors)
- Return the final `StateSet`

## Depends on
- SPEC-002
- SPEC-003

## Acceptance criteria
```gherkin
Feature: YAML deserialization of states
  Scenario: Parse flat bare state
    Given a YAML string:
      """
      type: ethernet
      name: eth0
      mtu: 1500
      addresses:
        - 10.0.1.50/24
      """
    When parse_yaml() is called
    Then it returns one State
    And entity_type is "ethernet"
    And selector.name is Some("eth0")
    And fields contains "mtu" with Value::U64(1500)
    And fields contains "addresses" with Value::List containing one IpNetwork value

  Scenario: Parse bare state with driver selector
    Given a YAML string:
      """
      type: ethernet
      driver: ixgbe
      mtu: 9000
      """
    When parse_yaml() is called
    Then it returns one State
    And entity_type is "ethernet"
    And selector.driver is Some("ixgbe")
    And selector.name is None
    And fields contains "mtu" with Value::U64(9000)

  Scenario: Parse explicit format with kind: state
    Given a YAML string:
      """
      kind: state
      type: ethernet
      name: eth0
      mtu: 9000
      """
    When parse_yaml() is called
    Then it returns one State with mtu=9000
    And the result is identical to the bare format (kind is not stored)

  Scenario: Parse multi-document YAML
    Given a YAML string with two documents separated by "---":
      """
      type: ethernet
      name: eth0
      mtu: 1500
      ---
      type: ethernet
      name: eth1
      mtu: 9000
      """
    When parse_yaml() is called
    Then it returns two State values
    And the first has selector.name "eth0" and mtu 1500
    And the second has selector.name "eth1" and mtu 9000

  Scenario: Parse route objects in fields
    Given a YAML string:
      """
      type: ethernet
      name: eth0
      routes:
        - destination: 0.0.0.0/0
          gateway: 10.0.1.1
          metric: 100
      """
    When parse_yaml() is called
    Then fields contains "routes" as a Value::List
    And the first element is a Value::Map with keys "destination", "gateway", "metric"

  Scenario: Selector properties are not in fields
    Given a YAML string:
      """
      type: ethernet
      name: eth0
      driver: e1000
      mtu: 1500
      """
    When parse_yaml() is called
    Then selector.name is Some("eth0")
    And selector.driver is Some("e1000")
    And fields does NOT contain "name" or "driver"
    And fields contains only "mtu"

  Scenario: String values that look like IPs are parsed as IpAddr
    Given a YAML field value "10.0.1.1"
    When deserialized as a Value
    Then it becomes Value::IpAddr(10.0.1.1)

  Scenario: String values that look like CIDR are parsed as IpNetwork
    Given a YAML field value "10.0.1.0/24"
    When deserialized as a Value
    Then it becomes Value::IpNetwork(10.0.1.0/24)

  Scenario: Plain strings remain as strings
    Given a YAML field value "802.3ad"
    When deserialized as a Value
    Then it becomes Value::String("802.3ad")

  Scenario: Boolean values are parsed correctly
    Given a YAML field value true
    When deserialized as a Value
    Then it becomes Value::Bool(true)

  Scenario: Negative integers are parsed as I64
    Given a YAML field value -1
    When deserialized as a Value
    Then it becomes Value::I64(-1)

Feature: YAML serialization of states
  Scenario: Serialize State to flat format
    Given a State with entity_type "ethernet", selector name "eth0", and field mtu=1500
    When serialized to YAML
    Then the output contains "type: ethernet"
    And the output contains "name: eth0" at the top level
    And the output contains "mtu: 1500" at the top level
    And the output does not contain "kind:"
    And the output does not contain "selector:" or "fields:"

  Scenario: Serialize State with explicit kind
    Given a State with entity_type "ethernet"
    When serialized with the explicit format option
    Then the output contains "kind: state" as the first field

  Scenario: Round-trip serialization preserves data
    Given a State with various field types (string, integer, boolean, IP, list, map)
    When serialized to YAML and then deserialized back
    Then the deserialized State has the same entity_type, selector, and field values
    And metadata is regenerated (not preserved through YAML round-trip)

Feature: Directory loading
  Scenario: Load all YAML files from a directory
    Given a directory containing:
      - eth0.yaml (one ethernet state)
      - dns.yaml (one dns state)
      - bond.yml (one bond state)
    When load_dir() is called on the directory
    Then the resulting StateSet contains 3 entities
    And it includes ethernet/eth0, dns, and bond states

  Scenario: Load multi-document file from directory
    Given a directory containing:
      - interfaces.yaml (two documents: eth0 and eth1)
    When load_dir() is called on the directory
    Then the resulting StateSet contains 2 entities

  Scenario: Skip hidden files in directory
    Given a directory containing:
      - eth0.yaml (one ethernet state)
      - .backup.yaml (should be skipped)
    When load_dir() is called on the directory
    Then the resulting StateSet contains 1 entity
    And .backup.yaml is not loaded

  Scenario: Error on duplicate entity keys within a directory
    Given a directory containing:
      - file1.yaml (ethernet/eth0 with mtu=1500)
      - file2.yaml (ethernet/eth0 with mtu=9000)
    When load_dir() is called on the directory
    Then it returns an error indicating a duplicate key for ethernet/eth0

  Scenario: Error on invalid YAML
    Given a directory containing a file with invalid YAML syntax
    When load_dir() is called on the directory
    Then it returns an error with the filename and parse error details

  Scenario: Empty directory returns empty StateSet
    Given an empty directory
    When load_dir() is called on the directory
    Then the resulting StateSet is empty and has len() 0
```
