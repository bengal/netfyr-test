# SPEC-351: Journal Infrastructure

## What

Add a `netfyr-journal` crate that records every network state change in an append-only journal. Each journal entry captures: what triggered the change (policy apply, DHCP event, external change, daemon startup, revert), which policies were active, the diff that was applied, a full state snapshot after the change, and the apply outcome.

The journal is written in both standalone and daemon modes:
- **Standalone**: `netfyr apply` appends an entry after a successful apply.
- **Daemon**: the reconciler appends an entry after each `reconcile_and_apply()`, with the trigger variant set by the caller.

This spec creates the journal crate, defines the entry types, implements NDJSON file I/O with rotation, and integrates journal writes into the existing apply paths.

## Why

Network configuration changes are high-stakes operations. When something goes wrong ("who changed the MTU on eth0?", "what did the last apply actually do?", "when did we lose that address?"), operators need an audit trail. Today netfyr applies changes but keeps no record of what it did, when, or why. The journal provides this audit trail, and is the foundation for the history inspection (SPEC-352), external change detection (SPEC-353), and state revert (SPEC-354) features.

## User interaction

Users do not interact with the journal directly in this spec. The journal is an internal infrastructure component. Users will interact with it through `netfyr history` (SPEC-352) and `netfyr revert` (SPEC-354).

However, the journal files are intentionally human-readable (NDJSON format) so that operators can inspect them with standard Unix tools:

```bash
# View the last 5 journal entries
tail -5 /var/lib/netfyr/journal/current.ndjson | jq .

# Count entries by trigger type
jq -r '.trigger.type' /var/lib/netfyr/journal/current.ndjson | sort | uniq -c

# Find all changes to eth0
grep '"eth0"' /var/lib/netfyr/journal/current.ndjson | jq .
```

## Implementation details

- Crate: `netfyr-journal`
- Files:
  - `crates/netfyr-journal/Cargo.toml`
  - `crates/netfyr-journal/src/lib.rs`
  - `crates/netfyr-journal/src/entry.rs` -- journal entry types
  - `crates/netfyr-journal/src/journal.rs` -- file I/O, append, read, rotation
  - `crates/netfyr-journal/src/serializable.rs` -- serializable wrappers for diff and state types
- Modified files:
  - `Cargo.toml` (workspace) -- add `netfyr-journal` to members
  - `crates/netfyr-cli/src/apply.rs` -- append journal entry after apply
  - `crates/netfyr-daemon/src/reconciler.rs` -- add `journal: Option<Journal>` field, append entry after reconcile_and_apply
- Dependencies (external crates): `serde`, `serde_json`, `chrono`, `flate2` (gzip for archive compression). All except `flate2` are already in the workspace.

### Journal entry types

```rust
// crates/netfyr-journal/src/entry.rs

/// Monotonically increasing sequence number within a journal.
pub type SequenceId = u64;

/// A single journal entry recording one state transition.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JournalEntry {
    /// Monotonic sequence number, 1-based, global across rotations.
    pub seq: SequenceId,
    /// Wall-clock timestamp (UTC).
    pub timestamp: DateTime<Utc>,
    /// What triggered the state change.
    pub trigger: Trigger,
    /// Policies active at the time of this entry.
    pub active_policies: Vec<PolicySummary>,
    /// The diff that was applied.
    pub diff: SerializableDiff,
    /// Full state snapshot of all managed entities after the change.
    pub state_after: SerializableStateSet,
    /// Apply outcome.
    pub outcome: ApplyOutcome,
}

/// What caused a journal entry to be recorded.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum Trigger {
    /// User ran `netfyr apply`.
    PolicyApply {
        /// Where policies were loaded from (file paths or "daemon").
        source: String,
    },
    /// A DHCP factory event caused re-reconciliation.
    DhcpEvent {
        policy_name: String,
        /// One of: "lease_acquired", "lease_renewed", "lease_expired".
        event_kind: String,
    },
    /// The daemon detected an external change on the system.
    ExternalChange {
        /// Entity names that were observed to have changed.
        changed_entities: Vec<String>,
    },
    /// The daemon started up and ran initial reconciliation.
    DaemonStartup,
    /// User explicitly reverted to a previous state.
    Revert {
        target_seq: SequenceId,
    },
}

/// Compact policy summary stored in each entry.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PolicySummary {
    pub name: String,
    pub factory_type: String,
    pub priority: u32,
}

/// The outcome of applying changes.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum ApplyOutcome {
    Applied {
        succeeded: u32,
        failed: u32,
        skipped: u32,
    },
    /// Observation-only (external change detected, no action taken).
    Observed,
}
```

### Serializable diff and state types

```rust
// crates/netfyr-journal/src/serializable.rs

/// Serializable representation of a StateDiff.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SerializableDiff {
    pub operations: Vec<SerializableDiffOp>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SerializableDiffOp {
    /// "add", "modify", or "remove".
    pub kind: String,
    pub entity_type: String,
    pub entity_name: String,
    pub field_changes: Vec<SerializableFieldChange>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SerializableFieldChange {
    pub field_name: String,
    /// "set", "unset", or "unchanged".
    pub change_kind: String,
    /// Previous value (None for new fields).
    pub current: Option<serde_json::Value>,
    /// New value (None for removed fields).
    pub desired: Option<serde_json::Value>,
}

/// Serializable representation of a StateSet snapshot.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SerializableStateSet {
    pub entities: Vec<SerializableState>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SerializableState {
    pub entity_type: String,
    pub selector_name: String,
    /// Flat JSON object of field values (no provenance).
    pub fields: serde_json::Value,
}
```

Implement `From<&StateDiff>` for `SerializableDiff` and `From<&StateSet>` for `SerializableStateSet` to convert from internal types. Field values are serialized via `serde_json::Value` to decouple the on-disk format from internal type evolution.

### Journal file I/O

```rust
// crates/netfyr-journal/src/journal.rs

pub struct Journal {
    dir: PathBuf,
    current_path: PathBuf,
    seq: SequenceId,
    entry_count: usize,
}

impl Journal {
    /// Open or create the journal at the default or configured directory.
    /// Reads NETFYR_JOURNAL_DIR env var, defaults to /var/lib/netfyr/journal/.
    pub fn open_default() -> Result<Self>;

    /// Open or create the journal at a specific directory.
    pub fn open(dir: &Path) -> Result<Self>;

    /// Append a journal entry. Handles rotation if thresholds are exceeded.
    pub fn append(&mut self, entry: JournalEntry) -> Result<()>;

    /// Read entries from the journal, most recent first.
    /// Supports optional count limit, time filter, and trigger filter.
    pub fn read_recent(&self, count: usize) -> Result<Vec<JournalEntry>>;

    /// Read a specific entry by sequence ID.
    pub fn read_entry(&self, seq: SequenceId) -> Result<Option<JournalEntry>>;

    /// Get the latest state snapshot for a given entity.
    pub fn latest_state_for(&self, entity_name: &str) -> Result<Option<SerializableState>>;

    /// Rotate the current journal to the archive directory (gzip compressed).
    fn rotate(&mut self) -> Result<()>;

    /// Remove archived journals older than the retention period.
    pub fn cleanup_archives(&self, retention_days: u64) -> Result<()>;
}
```

**File layout:**
```
$NETFYR_JOURNAL_DIR/            # default: /var/lib/netfyr/journal/
    current.ndjson              # active journal file (one JSON object per line)
    archive/
        journal-20260420T143000Z.ndjson.gz   # rotated + gzip compressed
    .seq                        # last assigned sequence number (text file)
```

**Rotation:** when `current.ndjson` exceeds 10,000 entries or 50 MB, rename to `archive/journal-{timestamp}.ndjson`, gzip compress it, start a new `current.ndjson`. Thresholds configurable via `NETFYR_JOURNAL_MAX_ENTRIES` and `NETFYR_JOURNAL_MAX_SIZE` env vars.

**Retention:** default 90 days. Configurable via `NETFYR_JOURNAL_RETENTION_DAYS`. Cleanup runs on journal open and can be called explicitly.

**Sequence numbers:** stored in `.seq` file. Read-then-increment on each append. Global across rotations — entries always have monotonically increasing IDs.

**Concurrency:** in daemon mode, journal writes are serialized by the `tokio::select!` main loop. In standalone mode, use `fcntl` advisory locking on `current.ndjson` for concurrent `netfyr apply` invocations (unlikely but safe).

### Integration: standalone apply

In `crates/netfyr-cli/src/apply.rs`, in `run_apply()`, after the apply step and before displaying results:

```rust
// After: let report = registry.apply(&diff).await?;
// Before: display_apply_report(&report, ...);

match Journal::open_default() {
    Ok(mut journal) => {
        let entry = JournalEntry {
            seq: 0, // assigned by journal.append()
            timestamp: Utc::now(),
            trigger: Trigger::PolicyApply { source: paths_display },
            active_policies: summarize_policies(&policy_set),
            diff: SerializableDiff::from(&state_diff),
            state_after: SerializableStateSet::from(&reconciliation.effective_state),
            outcome: ApplyOutcome::Applied {
                succeeded: report.succeeded.len() as u32,
                failed: report.failed.len() as u32,
                skipped: report.skipped.len() as u32,
            },
        };
        if let Err(e) = journal.append(entry) {
            tracing::warn!("Failed to write journal entry: {}", e);
        }
    }
    Err(e) => {
        tracing::warn!("Failed to open journal: {}", e);
    }
}
```

Journal write failure is a warning, not an error. The apply already succeeded.

### Integration: daemon reconciler

In `crates/netfyr-daemon/src/reconciler.rs`, add a `journal: Option<Journal>` field to the `Reconciler` struct. After `reconcile_and_apply()` completes, append a journal entry. The trigger is passed as a parameter:

```rust
pub async fn reconcile_and_apply(
    &self,
    policy_store: &PolicyStore,
    factory_manager: &FactoryManager,
    trigger: Trigger,
) -> Result<ApplyResult> {
    // ... existing reconciliation and apply logic ...

    if let Some(ref mut journal) = self.journal {
        let entry = JournalEntry {
            seq: 0,
            timestamp: Utc::now(),
            trigger,
            active_policies: summarize_policies(policy_store.policies()),
            diff: SerializableDiff::from(&state_diff),
            state_after: SerializableStateSet::from(&effective_state),
            outcome: ApplyOutcome::Applied { ... },
        };
        if let Err(e) = journal.append(entry) {
            tracing::warn!("Failed to write journal entry: {}", e);
        }
    }

    Ok(apply_result)
}
```

Callers set the trigger variant:
- `SubmitPolicies` handler: `Trigger::PolicyApply { source: "daemon".into() }`
- DHCP event handler: `Trigger::DhcpEvent { policy_name, event_kind }`
- Startup: `Trigger::DaemonStartup`

## Depends on

- SPEC-001 (workspace setup -- adds new crate to workspace members)
- SPEC-002 (state types -- State, StateSet, Value, FieldValue are serialized in journal entries)
- SPEC-004 (stateset operations -- StateSet snapshot serialization)
- SPEC-203 (diff generation -- StateDiff, DiffOperation, FieldChange are serialized in journal entries)
- SPEC-301 (CLI apply -- integration point for standalone journal writes)
- SPEC-403 (daemon -- integration point for daemon journal writes via reconciler)

## Acceptance criteria

```gherkin
Feature: Journal entry types
  Scenario: JournalEntry serializes to and from JSON
    Given a JournalEntry with trigger PolicyApply, a diff with one Modify operation, and a state snapshot
    When serialized to JSON and deserialized back
    Then the round-tripped entry equals the original

  Scenario: All trigger variants serialize correctly
    Given JournalEntry instances with each Trigger variant (PolicyApply, DhcpEvent, ExternalChange, DaemonStartup, Revert)
    When each is serialized to JSON
    Then each produces valid JSON with the correct "type" discriminator

Feature: Journal file I/O
  Scenario: Append and read entries
    Given an empty journal directory
    When 5 entries are appended
    Then read_recent(5) returns all 5 entries in reverse chronological order
    And each entry has a unique, monotonically increasing seq number

  Scenario: Sequence numbers persist across restarts
    Given a journal with 3 entries (seq 1, 2, 3)
    When the journal is closed and reopened
    And a new entry is appended
    Then the new entry has seq 4

  Scenario: Rotation triggers at entry count threshold
    Given NETFYR_JOURNAL_MAX_ENTRIES=100
    When 101 entries are appended
    Then current.ndjson has 1 entry
    And archive/ contains one gzipped file with 100 entries

  Scenario: Rotation triggers at file size threshold
    Given NETFYR_JOURNAL_MAX_SIZE=1024 (1 KB for testing)
    When entries are appended until current.ndjson exceeds 1 KB
    Then a rotation occurs and archive/ contains one gzipped file

  Scenario: Archive cleanup respects retention period
    Given archived journals from 100 days ago and 10 days ago
    And NETFYR_JOURNAL_RETENTION_DAYS=90
    When cleanup_archives is called
    Then the 100-day-old archive is deleted
    And the 10-day-old archive is kept

  Scenario: Read specific entry by sequence ID
    Given a journal with entries seq 1 through 10
    When read_entry(5) is called
    Then it returns the entry with seq 5

  Scenario: Journal directory is configurable
    Given NETFYR_JOURNAL_DIR is set to a temp directory
    When Journal::open_default() is called
    Then the journal is created in the temp directory

Feature: Standalone apply writes journal entry
  Scenario: netfyr apply records a journal entry
    Given a namespace with a veth pair and no daemon running
    When `netfyr apply policy.yaml` sets mtu=1400 on veth-e2e0
    Then a journal entry exists in current.ndjson
    And the entry has trigger type "policy_apply"
    And the entry's diff shows mtu changed to 1400
    And the entry's state_after includes the entity with mtu=1400
    And the entry's outcome is "applied" with succeeded=1

  Scenario: Journal write failure does not affect apply exit code
    Given NETFYR_JOURNAL_DIR points to a read-only directory
    When `netfyr apply policy.yaml` is run
    Then the apply succeeds (exit code 0)
    And a warning is logged about the journal write failure

Feature: Daemon writes journal entries
  Scenario: Daemon records journal entry on policy submission
    Given the daemon is running with a journal directory
    When policies are submitted via netfyr apply (daemon mode)
    Then a journal entry is recorded with trigger "policy_apply" and source "daemon"

  Scenario: Daemon records journal entry on DHCP event
    Given the daemon is running with a DHCP policy
    When a DHCP lease is acquired
    Then a journal entry is recorded with trigger "dhcp_event" and event_kind "lease_acquired"

  Scenario: Daemon records journal entry on startup
    Given persisted policies exist from a previous run
    When the daemon starts and runs initial reconciliation
    Then a journal entry is recorded with trigger "daemon_startup"

Feature: Serializable conversions
  Scenario: StateDiff converts to SerializableDiff
    Given a StateDiff with Add, Modify, and Remove operations
    When converted to SerializableDiff
    Then all operations are present with correct kind, entity, and field changes

  Scenario: StateSet converts to SerializableStateSet
    Given a StateSet with 3 ethernet entities
    When converted to SerializableStateSet
    Then all 3 entities are present with correct type, name, and fields
    And provenance information is not included (only field values)
```
