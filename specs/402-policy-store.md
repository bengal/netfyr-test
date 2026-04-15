# SPEC-402: Policy Store

## What
Implement the `PolicyStore` module that persists submitted policies to disk and provides replace-all semantics. The policy store manages the `/var/lib/netfyr/policies/` directory, writing individual YAML files for each policy and loading them on startup. It is the single source of truth for which policies the daemon is currently managing.

## Why
The daemon needs to survive restarts without losing submitted policies. Persisting policies to disk as individual YAML files ensures that the daemon can reload its full policy set on startup. Replace-all semantics simplify the model: each `netfyr apply` replaces the entire policy set rather than merging with existing policies, so the on-disk state always reflects exactly what the user last submitted. Individual files per policy (rather than a single monolithic file) make the store inspectable with standard tools (`ls`, `cat`, `diff`) and reduce the blast radius of corruption — a single malformed file does not prevent loading the rest.

## User interaction
Users do not interact with the policy store directly. The daemon uses it internally when processing `SubmitPolicies` requests and on startup. Administrators may inspect the persisted files for debugging:

```bash
# List persisted policies
ls /var/lib/netfyr/policies/

# Inspect a specific policy
cat /var/lib/netfyr/policies/office-network.yaml

# The daemon owns this directory — manual edits are overwritten on next apply
```

## Implementation details
- Crate: `netfyr-daemon` (library module)
- Files: `src/policy_store.rs`
- Dependencies (external crates): `serde_yaml` (YAML serialization), `anyhow` (error handling), `tracing` (structured logging)

### PolicyStore struct

```rust
pub struct PolicyStore {
    /// Directory where policy files are stored.
    /// None for ephemeral stores (dry-run).
    dir: Option<PathBuf>,
    /// Current set of policies, in submission order.
    policies: Vec<Policy>,
}
```

### File naming

Each policy is persisted as `{policy_name}.yaml` in the store directory. The policy name comes from `Policy.name` (defined in SPEC-007). Policy names are validated to be safe filenames: lowercase alphanumeric, hyphens, and underscores only. Names that fail validation are sanitized by replacing invalid characters with underscores.

Examples:
```
/var/lib/netfyr/policies/office-network.yaml
/var/lib/netfyr/policies/dhcp-eth0.yaml
/var/lib/netfyr/policies/datacenter_spine.yaml
```

### File format

Each file contains a single policy serialized as YAML using the same format as user-written policy files (SPEC-007). This means an administrator can copy a persisted file to `/etc/netfyr/policies/` and use it directly with `netfyr apply`.

```yaml
kind: policy
name: office-network
factory: static
priority: 100
states:
  - type: ethernet
    name: eth0
    mtu: 1500
    addresses:
      - 10.0.1.50/24
```

### Constructor and load

```rust
impl PolicyStore {
    /// Create a new store backed by the given directory.
    /// Creates the directory if it does not exist.
    /// Loads all existing .yaml files from the directory.
    pub fn load(dir: &Path) -> Result<Self>;

    /// Create an ephemeral store (for dry-run). Not backed by disk.
    pub fn ephemeral(policies: Vec<Policy>) -> Self;
}
```

**Load algorithm:**
1. Ensure the directory exists (create with `fs::create_dir_all` if needed — the daemon's `StateDirectory=netfyr` typically handles this, but the store is defensive).
2. Scan for all `*.yaml` files in the directory (non-recursive, skip subdirectories).
3. Sort filenames lexicographically for deterministic ordering.
4. For each file:
   a. Read the file contents.
   b. Parse as a `Policy` using the SPEC-007 parser.
   c. On parse error: log a warning with the filename and error, skip the file. Do not abort the entire load — partial recovery is better than failing to start.
5. Return the `PolicyStore` with the successfully loaded policies.

### Replace-all

```rust
impl PolicyStore {
    /// Replace all policies with a new set. Persists to disk.
    /// Returns the previous policy set.
    pub fn replace_all(&mut self, policies: Vec<Policy>) -> Result<Vec<Policy>>;
}
```

**Replace-all algorithm:**
1. Compute the set of filenames for the new policies: `{name}.yaml` for each.
2. Write each new policy to a temporary file (`{name}.yaml.tmp`) in the store directory.
3. Rename (atomic `fs::rename`) each `.tmp` file to its final `.yaml` name. This ensures each individual file is written atomically — a crash mid-write leaves a `.tmp` file, not a corrupt `.yaml` file.
4. Compute the set of old filenames that are not in the new set.
5. Remove each stale file.
6. Clean up any leftover `.tmp` files from prior incomplete writes.
7. Update `self.policies` to the new set.
8. Return the previous policy set.

**Crash safety:**
- A crash during step 2–3 (writing new files): Some `.tmp` files may remain. The next `load` ignores `.tmp` files (it only reads `*.yaml`). The old `.yaml` files are still intact. Result: the daemon restarts with the previous policy set — stale but valid.
- A crash during step 5 (removing old files): Extra `.yaml` files remain from the previous set. The next `load` reads all `.yaml` files, producing a superset of old + new policies. Result: stale but valid. The next successful `replace_all` cleans them up.
- A crash during step 6 (cleaning `.tmp` files): Harmless leftover `.tmp` files. The next `replace_all` cleans them up in step 6.

### Accessors

```rust
impl PolicyStore {
    /// Get the current policy set.
    pub fn policies(&self) -> &[Policy];

    /// Check if the store is empty.
    pub fn is_empty(&self) -> bool;

    /// Number of policies in the store.
    pub fn len(&self) -> usize;
}
```

### Error handling

| Failure | Behavior |
|---------|----------|
| Store directory missing and cannot be created | Return error from `load`. The daemon fails to start. |
| Individual `.yaml` file fails to parse | Log warning, skip file. Other policies are loaded. |
| Write failure during `replace_all` (disk full, permissions) | Return error. In-memory state is **not** updated — the store retains the previous policy set. The daemon reports the error to the CLI. |
| Rename failure during `replace_all` | Return error. Some `.tmp` files may remain; cleaned up on next `replace_all` or restart. |

### Concurrency

The `PolicyStore` is accessed only from the daemon's main event loop (single-threaded access pattern within `tokio::select!`). No locking is needed. The daemon serializes all `SubmitPolicies` requests — a second request waits until the first completes.

## Depends on
- SPEC-007 (Policy types and parsing)

## Acceptance criteria
```gherkin
Feature: Policy store initialization
  Scenario: Load from directory with existing policies
    Given 3 valid YAML policy files exist in /var/lib/netfyr/policies/
    When PolicyStore::load is called
    Then all 3 policies are loaded into the store
    And policies() returns them in lexicographic filename order

  Scenario: Load from empty directory
    Given /var/lib/netfyr/policies/ exists but is empty
    When PolicyStore::load is called
    Then the store has 0 policies
    And is_empty() returns true

  Scenario: Load creates directory if missing
    Given /var/lib/netfyr/policies/ does not exist
    When PolicyStore::load is called
    Then the directory is created
    And the store has 0 policies

  Scenario: Load skips malformed files with warning
    Given /var/lib/netfyr/policies/ contains valid.yaml and corrupt.yaml
    And corrupt.yaml contains invalid YAML
    When PolicyStore::load is called
    Then the store contains only the policy from valid.yaml
    And a warning is logged mentioning corrupt.yaml

  Scenario: Load ignores .tmp files
    Given /var/lib/netfyr/policies/ contains policy-a.yaml and policy-b.yaml.tmp
    When PolicyStore::load is called
    Then the store contains only policy-a
    And policy-b.yaml.tmp is ignored

Feature: Replace-all operation
  Scenario: Replace-all persists new policies and removes old
    Given a PolicyStore with policies A and B
    When replace_all is called with policies C and D
    Then the store contains only C and D
    And C.yaml and D.yaml exist in the store directory
    And A.yaml and B.yaml have been removed

  Scenario: Replace-all returns previous policy set
    Given a PolicyStore with policies A and B
    When replace_all is called with policies C and D
    Then the return value contains policies A and B

  Scenario: Replace-all writes atomically per file
    Given a PolicyStore with policies A
    When replace_all is called with policies B
    Then B.yaml.tmp is written first
    And then renamed to B.yaml
    And then A.yaml is removed

  Scenario: Replace-all with empty set clears all policies
    Given a PolicyStore with policies A, B, and C
    When replace_all is called with an empty Vec
    Then the store is empty
    And no .yaml files remain in the store directory

  Scenario: Replace-all does not update in-memory state on write failure
    Given a PolicyStore with policies A and B
    And the store directory is read-only
    When replace_all is called with policies C and D
    Then it returns an error
    And policies() still returns A and B

  Scenario: Replace-all cleans up leftover .tmp files
    Given the store directory contains stale.yaml.tmp from a prior crash
    When replace_all is called with policies A
    Then stale.yaml.tmp is removed
    And A.yaml exists

Feature: Crash recovery
  Scenario: Crash during write leaves previous state intact
    Given a PolicyStore with policies A and B on disk
    And a replace_all with policies C and D crashes after writing C.yaml.tmp
    When the daemon restarts and calls PolicyStore::load
    Then the store contains policies A and B (the previous set)
    And C.yaml.tmp is ignored

  Scenario: Crash after rename but before cleanup
    Given a PolicyStore with policies A and B on disk
    And a replace_all with policies C and D crashes after renaming C.yaml and D.yaml but before removing A.yaml
    When the daemon restarts and calls PolicyStore::load
    Then the store contains policies A, B, C, and D (superset, stale but valid)
    And the next successful replace_all cleans up the extra files

Feature: File naming
  Scenario: Policy name maps to filename
    Given a policy with name "office-network"
    When it is persisted to the store
    Then the file is named office-network.yaml

  Scenario: Invalid characters in policy name are sanitized
    Given a policy with name "my policy/v2"
    When it is persisted to the store
    Then the file is named my_policy_v2.yaml

Feature: Ephemeral store
  Scenario: Ephemeral store holds policies without disk I/O
    Given an ephemeral PolicyStore created with 2 policies
    Then policies() returns the 2 policies
    And no files are written to disk

  Scenario: Replace-all on ephemeral store updates in-memory only
    Given an ephemeral PolicyStore with policies A
    When replace_all is called with policies B
    Then policies() returns B
    And no files are written to disk

Feature: Persisted file format
  Scenario: Persisted files use standard policy YAML format
    Given a PolicyStore with a static policy for eth0 with mtu=1500
    When the policy is persisted
    Then the .yaml file can be parsed by the SPEC-007 policy parser
    And contains kind: policy, the policy name, factory, priority, and states
```
