# SPEC-007: Policy Types and Static Factory

## What
Define the policy data model and implement the static factory in the `netfyr-policy` crate. A policy is a factory that produces a `StateSet`. The static factory is the simplest factory type -- it takes an inline state definition and produces a `StateSet` directly.

### Types

1. **`Policy`** -- The top-level policy type. Fields:
   - `name: String` -- unique policy name (e.g., `"eth0"`, `"eth0-dhcp"`)
   - `factory_type: FactoryType` -- which factory produces the state
   - `priority: u32` -- numeric priority that propagates to all generated fields (default: 100)
   - `state: Option<State>` -- inline state definition (used by Static factory for single-entity policies)
   - `states: Option<Vec<State>>` -- inline state definitions (used when a policy produces multiple entities)
   - `selector: Option<Selector>` -- target selector (used by Dhcpv4 factory to identify which interface to run DHCP on; for Static factory, the matching properties are flat inside the state/states)

2. **`FactoryType`** enum -- Enumerates all factory types:
   - `Static` -- produces state from inline YAML definition
   - `Dhcpv4` -- produces state from DHCPv4 lease acquisition (runs inside the daemon)

   Serializes to/from lowercase strings in YAML (e.g., `"static"`, `"dhcpv4"`).

3. **`StateFactory` trait** -- The trait all factories implement:
   ```rust
   pub trait StateFactory {
       fn produce(&self, policy: &Policy) -> Result<StateSet, FactoryError>;
   }
   ```

4. **`StaticFactory`** -- Implements `StateFactory`. Takes the inline `state` or `states` from the `Policy` and wraps them into a `StateSet`, applying the policy's priority and setting `policy_ref` on each entity state.

5. **`PolicySet`** -- A collection of `Policy` values keyed by name. Methods:
   - `insert(policy: Policy)` -- add or replace a policy
   - `get(name: &str) -> Option<&Policy>` -- look up by name
   - `remove(name: &str) -> Option<Policy>`
   - `iter() -> impl Iterator<Item = &Policy>`
   - `len() -> usize` and `is_empty() -> bool`
   - `produce_all_static(&self) -> Result<StateSet, FactoryError>` -- run all static factories and union the results. Skips non-static policies.

6. **`FactoryError`** -- Error type for factory failures:
   - `MissingState { policy_name }` -- Static factory but no state/states defined
   - `InvalidFactory { policy_name, factory_type, reason }` -- factory misconfiguration
   - `ConflictError(ConflictError)` -- wraps StateSet union conflict from SPEC-004
   - `Other { message }` -- catch-all

### YAML format for policies

Static policy:
```yaml
kind: policy
name: eth0-static
factory: static
priority: 100
state:
  type: ethernet
  name: eth0
  mtu: 1500
  addresses:
    - 10.0.1.50/24
```

Multi-entity static policy:
```yaml
kind: policy
name: server-network
factory: static
priority: 100
states:
  - type: ethernet
    name: eth0
    mtu: 1500
  - type: dns
    scope: global
    servers:
      - 10.0.1.2
```

DHCPv4 policy (zero config -- selector identifies the interface, factory acquires a lease):
```yaml
kind: policy
name: eth0-dhcp
factory: dhcpv4
priority: 100
selector:
  name: eth0
```

### YAML parsing

The `netfyr-policy` crate provides `parse_policy_yaml(input: &str) -> Result<Vec<Policy>>` which handles multi-document YAML where each document may be:
- `kind: policy` -- parsed as a Policy
- No `kind` field -- delegated to SPEC-008 (bare state shorthand auto-wrapping)
- `kind: state` -- delegated to SPEC-008 (explicit state auto-wrapping)
- Other `kind` values -- returned as an error

## Why
The policy layer is the "how to produce state" layer. Every piece of desired state in netfyr comes from a policy factory. The static factory is the simplest and most common -- it just takes what the user wrote and produces it verbatim. The DHCPv4 factory (SPEC-401) is more dynamic but follows the same trait interface. The `PolicySet` with `produce_all_static()` is used by the CLI in daemon-free mode to produce state from static policies only. When the daemon is running, it handles all factories including DHCP. Having `FactoryType` as an enum keeps the system predictable and ensures each factory type is explicitly supported.

## User interaction
Users write policy YAML files in `/etc/netfyr/policies/` and apply them with `netfyr apply`. The `kind: policy` wrapper gives users control over factory type, priority, and naming. For the simplest case (static state), users can skip the policy wrapper entirely (handled by SPEC-008).

## Implementation details
- Crate: `netfyr-policy`
- Files:
  - `src/lib.rs` -- public re-exports
  - `src/policy.rs` -- `Policy`, `FactoryType`, `PolicySet` types
  - `src/factory.rs` -- `StateFactory` trait, `FactoryError` error type
  - `src/static_factory.rs` -- `StaticFactory` implementation
- Dependencies (external crates):
  - `netfyr-state` (workspace dependency)
  - `serde` and `serde_yaml` for YAML parsing
  - `thiserror` for error types
  - `tracing` for logging

### StaticFactory::produce pseudocode
```rust
impl StateFactory for StaticFactory {
    fn produce(&self, policy: &Policy) -> Result<StateSet, FactoryError> {
        let mut set = StateSet::new();

        // Handle single state
        if let Some(state) = &policy.state {
            let mut s = state.clone();
            s.priority = policy.priority;
            s.policy_ref = Some(policy.name.clone());
            // Set provenance on all fields to UserConfigured
            for (_, field) in s.fields.iter_mut() {
                field.provenance = Provenance::UserConfigured {
                    policy_ref: policy.name.clone(),
                };
            }
            set.insert(s);
        }

        // Handle multiple states
        if let Some(states) = &policy.states {
            for state in states {
                let mut s = state.clone();
                s.priority = policy.priority;
                s.policy_ref = Some(policy.name.clone());
                for (_, field) in s.fields.iter_mut() {
                    field.provenance = Provenance::UserConfigured {
                        policy_ref: policy.name.clone(),
                    };
                }
                set.insert(s);
            }
        }

        // Neither state nor states defined
        if policy.state.is_none() && policy.states.is_none() {
            return Err(FactoryError::MissingState {
                policy_name: policy.name.clone(),
            });
        }

        Ok(set)
    }
}
```

### FactoryType YAML serialization
Use `#[serde(rename_all = "lowercase")]` or custom serde to map enum variants:
- `Static` <-> `"static"`
- `Dhcpv4` <-> `"dhcpv4"`

### PolicySet::produce_all_static pseudocode
```rust
fn produce_all_static(&self) -> Result<StateSet, FactoryError> {
    let factory = StaticFactory;
    let mut combined = StateSet::new();
    for policy in self.iter().filter(|p| p.factory_type == FactoryType::Static) {
        let state_set = factory.produce(policy)?;
        combined = union(&combined, &state_set)
            .map_err(FactoryError::ConflictError)?;
    }
    Ok(combined)
}
```

## Depends on
- SPEC-002
- SPEC-004
- SPEC-005

## Acceptance criteria
```gherkin
Feature: Policy type definitions
  Scenario: Create a static policy from YAML
    Given a YAML string:
      """
      kind: policy
      name: eth0-static
      factory: static
      priority: 150
      state:
        type: ethernet
        name: eth0
        mtu: 1500
      """
    When parse_policy_yaml() is called
    Then it returns one Policy
    And the policy name is "eth0-static"
    And factory_type is FactoryType::Static
    And priority is 150
    And state is Some with entity_type "ethernet"

  Scenario: Create a multi-entity static policy
    Given a YAML string:
      """
      kind: policy
      name: server-network
      factory: static
      priority: 100
      states:
        - type: ethernet
          name: eth0
          mtu: 1500
        - type: dns
          scope: global
          servers:
            - 10.0.1.2
      """
    When parse_policy_yaml() is called
    Then it returns one Policy with states containing 2 States

  Scenario: Create a DHCPv4 policy from YAML
    Given a YAML string:
      """
      kind: policy
      name: eth0-dhcp
      factory: dhcpv4
      priority: 100
      selector:
        name: eth0
      """
    When parse_policy_yaml() is called
    Then it returns one Policy
    And factory_type is FactoryType::Dhcpv4
    And selector is Some with name "eth0"
    And state is None and states is None

  Scenario: FactoryType serialization
    Given a Policy with factory_type FactoryType::Dhcpv4
    When serialized to YAML
    Then the factory field is "dhcpv4"

  Scenario: FactoryType deserialization
    Given a YAML policy with factory: "dhcpv4"
    When deserialized
    Then factory_type is FactoryType::Dhcpv4

  Scenario: Default priority is 100
    Given a YAML policy with no priority field specified
    When deserialized
    Then priority is 100

Feature: Static factory produces StateSet
  Scenario: Static factory with single state
    Given a Policy named "eth0" with factory_type Static, priority 200, and a state for ethernet/eth0 with mtu=1500
    When StaticFactory.produce() is called
    Then it returns a StateSet containing one State
    And the entity's priority is 200
    And the entity's policy_ref is Some("eth0")
    And the mtu field has Provenance::UserConfigured with policy_ref "eth0"

  Scenario: Static factory with multiple states
    Given a Policy named "server" with factory_type Static, priority 100, and states for ethernet/eth0 and dns/global
    When StaticFactory.produce() is called
    Then it returns a StateSet containing two States
    And both entities have priority 100 and policy_ref "server"

  Scenario: Static factory with no state defined
    Given a Policy named "empty" with factory_type Static and neither state nor states
    When StaticFactory.produce() is called
    Then it returns Err(FactoryError::MissingState)
    And the error contains the policy name "empty"

  Scenario: Static factory preserves all field values
    Given a Policy with state containing fields mtu=9000, addresses=["10.0.1.50/24"], routes=[{destination: "0.0.0.0/0", gateway: "10.0.1.1"}]
    When StaticFactory.produce() is called
    Then all field values are preserved exactly in the output StateSet

Feature: PolicySet collection and produce_all_static
  Scenario: PolicySet collects and retrieves policies
    Given an empty PolicySet
    When a Policy named "eth0" is inserted
    Then get("eth0") returns the policy
    And len() returns 1

  Scenario: produce_all_static unions multiple static policy outputs
    Given a PolicySet with:
      - Policy "eth0" (static, priority 100, ethernet/eth0 mtu=1500)
      - Policy "dns" (static, priority 100, dns/global servers=["10.0.1.2"])
    When produce_all_static() is called
    Then it returns a StateSet with 2 entities (ethernet/eth0 and dns/global)

  Scenario: produce_all_static skips DHCP policies
    Given a PolicySet with:
      - Policy "eth0" (static, priority 100, ethernet/eth0 mtu=1500)
      - Policy "eth1-dhcp" (dhcpv4, priority 100, selector name=eth1)
    When produce_all_static() is called
    Then it returns a StateSet with 1 entity (only ethernet/eth0)
    And the DHCP policy is not processed

  Scenario: produce_all_static detects conflicts
    Given a PolicySet with:
      - Policy "a" (static, priority 100, ethernet/eth0 mtu=1500)
      - Policy "b" (static, priority 100, ethernet/eth0 mtu=9000)
    When produce_all_static() is called
    Then it returns Err(FactoryError::ConflictError)
    And the error identifies the conflicting field "mtu" on ethernet/eth0

  Scenario: produce_all_static resolves by priority
    Given a PolicySet with:
      - Policy "base" (static, priority 100, ethernet/eth0 mtu=1500)
      - Policy "override" (static, priority 200, ethernet/eth0 mtu=9000)
    When produce_all_static() is called
    Then it returns Ok with ethernet/eth0 mtu=9000
    And the mtu field's provenance references policy "override"

Feature: Multi-document policy YAML parsing
  Scenario: Parse multiple policies from one file
    Given a YAML string with two policy documents separated by "---"
    When parse_policy_yaml() is called
    Then it returns two Policy values
    And each has its own name, factory_type, and state
```
