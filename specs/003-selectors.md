# SPEC-003: Selectors

## What
Implement the `Selector` type in `netfyr-state` that identifies which system entity a state targets. Selectors support multi-field matching with AND logic -- all specified fields must match for a selector to match an entity.

Selector fields:
- `name: Option<String>` -- exact interface/entity name (e.g., `"eth0"`, `"bond0.100"`)
- `entity_type: Option<String>` -- entity type filter (e.g., `"ethernet"`, `"wifi"`) -- note: this is separate from `State.entity_type` and allows a selector to match across entity types if unset
- `driver: Option<String>` -- kernel driver name (e.g., `"ixgbe"`, `"mlx5_core"`)
- `pci_path: Option<String>` -- PCI device path for stable hardware identification (e.g., `"0000:03:00.0"`)
- `mac: Option<MacAddr>` -- MAC address (6-byte hardware address)
- `labels: HashMap<String, String>` -- user-defined key-value labels; all specified labels must be present on the target with matching values

The `Selector` type must implement:
- `matches(&self, other: &Selector) -> bool` -- returns true if all fields specified in `self` match the corresponding fields in `other`. Unspecified fields (None / empty) in `self` match anything. For labels, every label in `self.labels` must exist in `other.labels` with the same value (subset matching).
- `is_specific(&self) -> bool` -- returns true if the selector has a `name` set (meaning it targets a single known entity rather than a class of entities).
- `key(&self) -> String` -- returns a stable string key used for indexing in a StateSet (typically the name, or a hash of the selector fields if no name is set).

The `MacAddr` type should be a newtype wrapping `[u8; 6]` with:
- `Display` formatting as `"aa:bb:cc:dd:ee:ff"` (lowercase hex with colons)
- `FromStr` parsing that accepts `"AA:BB:CC:DD:EE:FF"` (case-insensitive, colon-separated)
- `Serialize`/`Deserialize` as a string in the above format

## Why
Selectors are how netfyr connects a declared state to a real system entity. Simple use cases (like Ana's server setup) use `name` alone. Advanced use cases require multi-field selectors for device-agnostic policies -- for example, a policy targeting "any ixgbe ethernet device" uses `driver: ixgbe` + `entity_type: ethernet` without specifying a name. The AND logic ensures specificity: when multiple fields are provided, all must match, preventing accidental over-matching. The `matches()` method is used by the reconciliation engine to resolve which system entity a state applies to, and by the StateSet to group states by entity.

## User interaction
Users specify selectors in YAML:
```yaml
selector:
  name: eth0
```
or for device-agnostic matching:
```yaml
selector:
  driver: ixgbe
  labels:
    role: uplink
```

## Implementation details
- Crate: `netfyr-state`
- Files:
  - `src/selector.rs` -- `Selector` struct, `MacAddr` newtype, and all impls
  - `src/lib.rs` -- update to re-export `Selector` and `MacAddr`
- Dependencies (external crates): none beyond what SPEC-002 already added (serde, etc.)

### Selector struct definition

```rust
#[derive(Clone, Debug, Default, Serialize, Deserialize, PartialEq)]
pub struct Selector {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub name: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub entity_type: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub driver: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub pci_path: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub mac: Option<MacAddr>,
    #[serde(default, skip_serializing_if = "HashMap::is_empty")]
    pub labels: HashMap<String, String>,
}
```

### matches() logic
```
For each field in self:
  If self.field is Some(v):
    If other.field is None or other.field != Some(v):
      return false
For self.labels:
  For each (k, v) in self.labels:
    If other.labels.get(k) != Some(v):
      return false
return true
```

### key() logic
If `name` is `Some(n)`, return `n.clone()`. Otherwise, produce a deterministic string from all set fields sorted alphabetically, e.g., `"driver=ixgbe;entity_type=ethernet;labels.role=uplink"`.

## Depends on
- SPEC-001

## Acceptance criteria
```gherkin
Feature: Selectors for entity matching
  Scenario: Exact name matching
    Given a selector with name "eth0"
    And a target selector with name "eth0" and driver "ixgbe"
    When matches() is called
    Then it returns true

  Scenario: Name mismatch
    Given a selector with name "eth0"
    And a target selector with name "eth1"
    When matches() is called
    Then it returns false

  Scenario: Multi-field AND matching succeeds
    Given a selector with driver "ixgbe" and entity_type "ethernet"
    And a target selector with name "eth0", driver "ixgbe", and entity_type "ethernet"
    When matches() is called
    Then it returns true

  Scenario: Multi-field AND matching fails on one mismatch
    Given a selector with driver "ixgbe" and entity_type "wifi"
    And a target selector with name "eth0", driver "ixgbe", and entity_type "ethernet"
    When matches() is called
    Then it returns false

  Scenario: Unspecified fields match anything
    Given a selector with only driver "ixgbe" (all other fields unset)
    And a target selector with name "eth0", driver "ixgbe", entity_type "ethernet", and pci_path "0000:03:00.0"
    When matches() is called
    Then it returns true

  Scenario: Empty selector matches everything
    Given an empty selector (all fields unset, no labels)
    And any target selector
    When matches() is called
    Then it returns true

  Scenario: Label subset matching succeeds
    Given a selector with labels {"role": "uplink"}
    And a target selector with labels {"role": "uplink", "env": "prod"}
    When matches() is called
    Then it returns true

  Scenario: Label subset matching fails on missing label
    Given a selector with labels {"role": "uplink", "env": "staging"}
    And a target selector with labels {"role": "uplink"}
    When matches() is called
    Then it returns false

  Scenario: Label value mismatch
    Given a selector with labels {"role": "downlink"}
    And a target selector with labels {"role": "uplink"}
    When matches() is called
    Then it returns false

  Scenario: MAC address matching
    Given a selector with mac "aa:bb:cc:dd:ee:ff"
    And a target selector with mac "AA:BB:CC:DD:EE:FF" and name "eth0"
    When matches() is called
    Then it returns true because MAC comparison is case-insensitive

  Scenario: MacAddr parsing and formatting
    Given the string "AA:BB:CC:DD:EE:FF"
    When it is parsed into a MacAddr
    Then parsing succeeds
    And Display formatting produces "aa:bb:cc:dd:ee:ff"

  Scenario: MacAddr rejects invalid format
    Given the string "not-a-mac"
    When it is parsed into a MacAddr
    Then parsing fails with an error

  Scenario: is_specific returns true for named selectors
    Given a selector with name "eth0"
    When is_specific() is called
    Then it returns true

  Scenario: is_specific returns false for unnamed selectors
    Given a selector with only driver "ixgbe"
    When is_specific() is called
    Then it returns false

  Scenario: key produces stable identifier from name
    Given a selector with name "eth0"
    When key() is called
    Then it returns "eth0"

  Scenario: key produces deterministic identifier without name
    Given a selector with driver "ixgbe" and entity_type "ethernet"
    When key() is called twice
    Then both calls return the same string
    And the string contains both "driver=ixgbe" and "entity_type=ethernet"

  Scenario: Selector serializes to YAML with only set fields
    Given a selector with name "eth0" and no other fields set
    When serialized to YAML
    Then the YAML contains "name: eth0"
    And the YAML does not contain "driver", "pci_path", "mac", or "labels"
```
