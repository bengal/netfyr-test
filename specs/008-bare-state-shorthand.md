# SPEC-008: Bare State Shorthand

## What
When loading YAML files in the policy layer, if a document has no `kind:` field (or `kind: state`), automatically wrap it into a static policy with default priority (100). The policy name is derived from the source filename (without extension). Emit an info-level log message indicating the implicit wrapping so the behavior is transparent to the user.

### Wrapping rules

1. **No `kind:` field** -- The document is a bare state (e.g., `type: ethernet`, `name: eth0`, `mtu: 1500`). It is parsed as an `State` using the SPEC-005 parser, then wrapped into a `Policy` with:
   - `name`: derived from the filename (e.g., `eth0.yaml` -> `"eth0"`, `bond0-vlan100.yml` -> `"bond0-vlan100"`)
   - `factory_type`: `FactoryType::Static`
   - `priority`: 100 (default)
   - `state`: the parsed `State`

2. **`kind: state`** -- Same behavior as no `kind:` field (explicit state marker is equivalent to bare state for policy wrapping purposes).

3. **`kind: policy`** -- Not handled by this spec; passed through to the normal policy parser (SPEC-007).

4. **Multi-document files with bare states** -- If a file contains multiple bare state documents (separated by `---`), each is wrapped into its own policy. The policy name for each is derived from the filename with a numeric suffix: `"filename-1"`, `"filename-2"`, etc. If the file contains a single document, no suffix is added.

5. **Mixed files** -- A file may contain a mix of `kind: policy` documents and bare state documents. Each is handled according to its `kind` (or lack thereof).

### Logging

When a bare state is auto-wrapped, emit an info log:
```
Wrapping bare state from eth0.yaml as static policy "eth0" with priority 100
```

### Integration point

This behavior is implemented in the policy loader (`netfyr-policy/src/loader.rs`), which is the unified entry point for loading policy files. It:
1. Reads a YAML file
2. For each document, checks for `kind:`
3. Bare states -> wraps into Policy (this spec)
4. `kind: policy` -> parses as Policy (SPEC-007)
5. Other `kind` values -> returns an error
6. Returns `Vec<Policy>`

The loader also provides `load_policy_dir(path: &Path) -> Result<PolicySet>` which loads all `.yaml`/`.yml` files from a directory and returns a `PolicySet`.

## Why
The bare state shorthand is netfyr's primary usability feature for server administrators. Ana (Persona 1 in the user stories) writes simple YAML files without any `kind:` or factory boilerplate and runs `netfyr apply`. The auto-wrapping makes this work by transparently creating the policy machinery behind the scenes. This design follows the principle of progressive complexity: simple use cases stay simple, and users only encounter the policy concept when they need factory types, priorities, or other advanced features. The log message ensures the behavior is never surprising -- users can see exactly what netfyr does with their files.

## User interaction
Users write bare state files:
```yaml
# /etc/netfyr/policies/eth0.yaml
type: ethernet
name: eth0
mtu: 1500
addresses:
  - 10.0.1.50/24
```

And apply them:
```bash
$ netfyr apply /etc/netfyr/policies/eth0.yaml
# INFO: Wrapping bare state from eth0.yaml as static policy "eth0" with priority 100
Applied 1 state (1 policy).
```

This is functionally identical to writing:
```yaml
kind: policy
name: eth0
factory: static
priority: 100
state:
  type: ethernet
  name: eth0
  mtu: 1500
  addresses:
    - 10.0.1.50/24
```

## Implementation details
- Crate: `netfyr-policy`
- Files:
  - `src/loader.rs` -- `load_policy_file(path: &Path) -> Result<Vec<Policy>>`, `load_policy_dir(path: &Path) -> Result<PolicySet>`, and the auto-wrapping logic
  - `src/lib.rs` -- re-export loader functions
- Dependencies (external crates):
  - `netfyr-state` (workspace dependency, for `parse_yaml` and `State`)
  - `serde_yaml` -- YAML parsing
  - `tracing` or `log` -- for info-level log messages
  - `walkdir` -- directory traversal

### Filename-to-policy-name derivation

```rust
fn policy_name_from_path(path: &Path) -> String {
    path.file_stem()
        .and_then(|s| s.to_str())
        .unwrap_or("unnamed")
        .to_string()
}
```

### load_policy_file pseudocode

```rust
fn load_policy_file(path: &Path) -> Result<Vec<Policy>> {
    let content = std::fs::read_to_string(path)?;
    let base_name = policy_name_from_path(path);
    let mut policies = Vec::new();
    let mut doc_index = 0;

    for document in serde_yaml::Deserializer::from_str(&content) {
        let raw: serde_yaml::Value = Deserialize::deserialize(document)?;
        doc_index += 1;

        let kind = raw.get("kind").and_then(|v| v.as_str());

        match kind {
            None | Some("state") => {
                // Bare state -> auto-wrap
                let entity = parse_raw_to_entity_state(raw)?;
                let name = if doc_index == 1 && is_single_doc {
                    base_name.clone()
                } else {
                    format!("{}-{}", base_name, doc_index)
                };
                info!("Wrapping bare state from {} as static policy \"{}\" with priority 100",
                      path.file_name().unwrap().to_string_lossy(), name);
                policies.push(Policy {
                    name,
                    factory_type: FactoryType::Static,
                    priority: 100,
                    state: Some(entity),
                    states: None,
                });
            }
            Some("policy") => {
                let policy = parse_policy_from_raw(raw)?;
                policies.push(policy);
            }
            Some(other) => {
                return Err(anyhow!("Unknown kind '{}' in {}", other, path.display()));
            }
        }
    }
    Ok(policies)
}
```

### load_policy_dir pseudocode
```rust
fn load_policy_dir(path: &Path) -> Result<PolicySet> {
    let mut policy_set = PolicySet::new();
    for entry in WalkDir::new(path).into_iter().filter_ok(|e| {
        e.file_type().is_file()
            && !e.file_name().to_string_lossy().starts_with('.')
            && matches!(e.path().extension().and_then(|s| s.to_str()), Some("yaml" | "yml"))
    }) {
        let entry = entry?;
        let policies = load_policy_file(entry.path())?;
        for policy in policies {
            if policy_set.get(&policy.name).is_some() {
                return Err(anyhow!("Duplicate policy name '{}' (from {})", policy.name, entry.path().display()));
            }
            policy_set.insert(policy);
        }
    }
    Ok(policy_set)
}
```

## Depends on
- SPEC-005 (YAML parsing for State)
- SPEC-007 (Policy types and FactoryType::Static)

## Acceptance criteria
```gherkin
Feature: Bare state auto-wrapping into static policy
  Scenario: Single bare state file is wrapped into a policy
    Given a file "/tmp/policies/eth0.yaml" containing:
      """
      type: ethernet
      name: eth0
      mtu: 1500
      """
    When load_policy_file() is called on the file
    Then it returns one Policy
    And the policy name is "eth0"
    And factory_type is FactoryType::Static
    And priority is 100
    And state has entity_type "ethernet" with field mtu=1500

  Scenario: Explicit kind: state is treated same as bare state
    Given a file "/tmp/policies/eth0.yaml" containing:
      """
      kind: state
      type: ethernet
      name: eth0
      mtu: 1500
      """
    When load_policy_file() is called on the file
    Then it returns one Policy with the same structure as a bare state wrapping

  Scenario: Multi-document bare state file produces numbered policies
    Given a file "/tmp/policies/interfaces.yaml" containing:
      """
      type: ethernet
      name: eth0
      mtu: 1500
      ---
      type: ethernet
      name: eth1
      mtu: 9000
      """
    When load_policy_file() is called on the file
    Then it returns two Policies
    And the first policy name is "interfaces-1"
    And the second policy name is "interfaces-2"

  Scenario: kind: policy documents are not wrapped
    Given a file "/tmp/policies/custom.yaml" containing:
      """
      kind: policy
      name: custom-policy
      factory: static
      priority: 200
      state:
        type: ethernet
        name: eth0
        mtu: 9000
      """
    When load_policy_file() is called on the file
    Then it returns one Policy
    And the policy name is "custom-policy" (not derived from filename)
    And priority is 200 (not the default 100)

  Scenario: Mixed file with bare state and explicit policy
    Given a file "/tmp/policies/mixed.yaml" containing:
      """
      type: dns
      scope: global
      servers:
        - 10.0.1.2
      ---
      kind: policy
      name: eth0-override
      factory: static
      priority: 200
      state:
        type: ethernet
        name: eth0
        mtu: 9000
      """
    When load_policy_file() is called on the file
    Then it returns two Policies
    And the first is a wrapped bare state named "mixed-1" with priority 100
    And the second is the explicit policy named "eth0-override" with priority 200

  Scenario: Info log is emitted for wrapped bare states
    Given a file "/tmp/policies/eth0.yaml" with a bare state
    When load_policy_file() is called
    Then an info-level log message is emitted
    And the message contains "Wrapping bare state from eth0.yaml"
    And the message contains "static policy" and "priority 100"

  Scenario: Policy name derived from filename without extension
    Given a file "/tmp/policies/bond0-vlan100.yml" with a bare state
    When load_policy_file() is called
    Then the policy name is "bond0-vlan100"

  Scenario: Load all policies from a directory
    Given a directory "/tmp/policies/" containing:
      - eth0.yaml (bare state for ethernet/eth0)
      - dns.yaml (bare state for dns/global)
      - custom.yaml (kind: policy, name: "custom")
    When load_policy_dir() is called on the directory
    Then the resulting PolicySet contains 3 policies
    And it includes policies named "eth0", "dns", and "custom"

  Scenario: Duplicate policy names across files are rejected
    Given a directory containing:
      - file1.yaml (bare state, derives name "eth0")
      - eth0.yaml (bare state, also derives name "eth0")
    When load_policy_dir() is called
    Then it returns an error about duplicate policy name "eth0"

  Scenario: Hidden files are skipped during directory loading
    Given a directory containing:
      - eth0.yaml (bare state)
      - .backup.yaml (should be skipped)
    When load_policy_dir() is called
    Then the resulting PolicySet contains 1 policy
    And .backup.yaml is not loaded

  Scenario: Unknown kind value produces an error
    Given a file containing:
      """
      kind: unknown
      name: something
      """
    When load_policy_file() is called
    Then it returns an error mentioning "Unknown kind 'unknown'"
```
