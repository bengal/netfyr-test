# SPEC-354: State Revert

## What

Add a `netfyr revert <seq>` CLI command that restores system network state to match a journal snapshot. The command reads the target entry's `state_after` snapshot, computes the diff from the current system state to the target state, and applies it. A `--dry-run` flag previews the changes without applying. This is full-state revert only -- per-entity filtering is not supported in this version.

A corresponding `Revert` Varlink method is added for daemon-mode operation.

## Why

Operators need the ability to undo network configuration changes quickly. When a policy change breaks connectivity or causes unexpected behavior, the operator should be able to say "go back to how it was 10 minutes ago" without manually reconstructing the previous configuration. Combined with `netfyr history` (SPEC-352), this enables a workflow:

1. `netfyr history` -- see what changed and when
2. `netfyr revert 142 --dry-run` -- preview the rollback
3. `netfyr revert 142` -- execute the rollback

The revert is a direct state manipulation: it computes the diff from the current system state to the historical snapshot and applies it. It does not restore old policies. This is the right semantic because:
- It's what operators mean by "undo" -- restore the system state, not the policy set.
- In daemon mode, the current policy set may be correct but was applied with a bug that has since been fixed. Reverting to a good state while keeping the corrected policies is the desired behavior.
- Policy restoration would be a separate, more complex feature (e.g., "netfyr policies restore").

## User interaction

```bash
# Revert to the state recorded in journal entry #142
netfyr revert 142

# Preview what would change without applying
netfyr revert 142 --dry-run
```

### Example: undo a bad MTU change

```bash
$ netfyr history -n 3
SEQ  TIMESTAMP             TRIGGER         ENTITIES   OUTCOME        CHANGES
144  2026-04-20 15:30:00   policy-apply    eth0       applied (1 ok) ~mtu
143  2026-04-20 15:00:00   policy-apply    eth0       applied (1 ok) addr(+1)
142  2026-04-20 14:30:00   policy-apply    eth0       applied (2 ok) ~mtu, addr(+1)

$ netfyr revert 143 --dry-run
Reverting to state from entry #143 (2026-04-20 15:00:00 UTC)
Changes that would be applied:
  ~ ethernet eth0
      mtu: 9000 -> 1500

$ netfyr revert 143
Reverting to state from entry #143 (2026-04-20 15:00:00 UTC)
Applied 1 change (1 entity modified).
  ~ ethernet eth0: mtu 9000 -> 1500
Warning: the active policy set was not changed. The daemon may re-apply
the current desired state on the next reconciliation cycle.
```

### Example: nothing to revert

```bash
$ netfyr revert 144
Reverting to state from entry #144 (2026-04-20 15:30:00 UTC)
No changes needed. System is already in the target state.
```

## Implementation details

- Crate: `netfyr-cli` (CLI command), `netfyr-varlink` (Varlink API), `netfyr-daemon` (Varlink handler)
- Files:
  - `crates/netfyr-cli/src/main.rs` -- add `Revert` to the `Commands` enum
  - `crates/netfyr-cli/src/revert.rs` -- (new) revert command implementation
  - `crates/netfyr-varlink/src/io.netfyr.varlink` -- add `Revert` method
  - `crates/netfyr-daemon/src/server.rs` -- handle `Revert` Varlink method
- Dependencies (external crates): none new

### CLI arguments

```rust
#[derive(Args)]
struct RevertArgs {
    /// Sequence ID of the journal entry to revert to.
    /// System state will be restored to match this entry's state_after snapshot.
    pub target: u64,

    /// Preview the changes without applying.
    #[arg(long)]
    pub dry_run: bool,
}
```

### Revert flow (daemon-free mode)

```rust
pub async fn run_revert(args: RevertArgs) -> Result<ExitCode> {
    // 1. Open the journal
    let journal = Journal::open_default()?;

    // 2. Read the target entry
    let target_entry = journal.read_entry(args.target)?
        .ok_or_else(|| anyhow!("Entry #{} not found", args.target))?;

    println!("Reverting to state from entry #{} ({} UTC)",
        target_entry.seq, target_entry.timestamp);

    // 3. Convert the snapshot to a target StateSet
    let target_state = target_entry.state_after.to_state_set()?;

    // 4. Query the current system state
    let registry = create_backend_registry();
    let actual_state = registry.query_all().await?;

    // 5. Compute diff from current to target
    //    Use all entities in the target as "managed" for diff purposes
    let managed = target_state.entities().collect();
    let diff = generate_diff(&target_state, &actual_state, &managed);

    // 6. Dry-run: display and exit
    if args.dry_run {
        if diff.is_empty() {
            println!("No changes needed. System is already in the target state.");
            return Ok(ExitCode::from(0));
        }
        println!("Changes that would be applied:");
        display_diff_report(&diff);
        return Ok(ExitCode::from(0));
    }

    // 7. Apply
    if diff.is_empty() {
        println!("No changes needed. System is already in the target state.");
        return Ok(ExitCode::from(0));
    }
    let report = registry.apply(&diff).await?;

    // 8. Record revert in journal
    let mut journal = journal;
    let revert_entry = JournalEntry {
        seq: 0,
        timestamp: Utc::now(),
        trigger: Trigger::Revert { target_seq: args.target },
        active_policies: vec![], // no policy context in daemon-free mode
        diff: SerializableDiff::from(&diff),
        state_after: SerializableStateSet::from(&target_state),
        outcome: ApplyOutcome::Applied {
            succeeded: report.succeeded.len() as u32,
            failed: report.failed.len() as u32,
            skipped: report.skipped.len() as u32,
        },
    };
    if let Err(e) = journal.append(revert_entry) {
        tracing::warn!("Failed to write revert journal entry: {}", e);
    }

    // 9. Display results
    display_apply_report(&report);

    // 10. Exit code
    Ok(exit_code_from_report(&report))
}
```

### Revert flow (daemon mode)

When the daemon is detected (socket probe succeeds), the CLI sends a `Revert` Varlink request instead of applying directly:

```rust
if let Ok(client) = varlink_connect(&socket_path) {
    let response = client.revert(args.target, args.dry_run).await?;
    display_apply_report(&response.report);

    // Warn about policy drift
    eprintln!("Warning: the active policy set was not changed. The daemon may re-apply");
    eprintln!("the current desired state on the next reconciliation cycle.");

    return Ok(exit_code_from_report(&response.report));
}
```

The daemon handler for the `Revert` method:

1. Reads the target journal entry.
2. Converts the snapshot to a target `StateSet`.
3. Queries the current system state.
4. Computes the diff.
5. If `dry_run`: returns the diff as a report without applying.
6. Otherwise: applies the diff, records a revert journal entry, returns the report.

The daemon does NOT change its policy store on revert. The next DHCP event or policy submission will trigger reconciliation with the current (unchanged) policies, which may produce a different state than the reverted one. The warning message makes this explicit.

### Snapshot to StateSet conversion

The `SerializableStateSet` stored in journal entries must be convertible back to a `StateSet` for diff computation:

```rust
impl SerializableStateSet {
    /// Convert back to a StateSet for diff computation.
    /// Field values are deserialized from JSON using the same heuristics
    /// as YAML deserialization (SPEC-005): try Ipv4Network, then Ipv4Addr,
    /// then keep as String/U64/I64/Bool.
    pub fn to_state_set(&self) -> Result<StateSet>;
}
```

Provenance for all fields in the restored state is set to `Provenance::UserConfigured { policy_ref: "revert" }`.

### Varlink API addition

```varlink
method Revert(target_seq: int, dry_run: bool) -> (report: ApplyReport)
```

The `ApplyReport` type is already defined in the Varlink interface (used by `SubmitPolicies`). Revert reuses it.

### Warning about policy drift

In daemon mode, after a successful revert, the CLI always prints:

```
Warning: the active policy set was not changed. The daemon may re-apply
the current desired state on the next reconciliation cycle.
```

This warning is printed to stderr so it doesn't interfere with JSON output parsing.

## Depends on

- SPEC-351 (journal infrastructure -- provides Journal, JournalEntry, SerializableStateSet, and Trigger::Revert)
- SPEC-203 (diff generation -- generate_diff used to compute revert diff)
- SPEC-103 (rtnetlink apply -- backend apply used to execute the revert)
- SPEC-301 (CLI apply -- follows the same patterns for display, exit codes, daemon detection)
- SPEC-404 (Varlink API -- extends with Revert method)

## Acceptance criteria

```gherkin
Feature: State revert (daemon-free mode)
  Scenario: Revert to a previous state
    Given a namespace with a veth pair and no daemon running
    And policy A was applied setting mtu=1400 (journal entry seq=1)
    And policy B was applied setting mtu=1300 (journal entry seq=2)
    When `netfyr revert 1` is run
    Then veth-e2e0 has mtu=1400
    And a journal entry with trigger "revert" and target_seq=1 is recorded
    And the output shows "Applied 1 change"

  Scenario: Revert dry-run previews changes
    Given a namespace with a veth pair and no daemon running
    And policy A was applied setting mtu=1400 (journal entry seq=1)
    And policy B was applied setting mtu=1300 (journal entry seq=2)
    When `netfyr revert 1 --dry-run` is run
    Then the output shows "mtu: 1300 -> 1400"
    And veth-e2e0 still has mtu=1300 (unchanged)
    And no new journal entry is recorded

  Scenario: Revert when already at target state
    Given a namespace with a veth pair and no daemon running
    And policy A was applied setting mtu=1400 (journal entry seq=1)
    When `netfyr revert 1` is run (system already matches entry 1)
    Then the output shows "No changes needed. System is already in the target state."
    And the exit code is 0

  Scenario: Revert to nonexistent entry
    When `netfyr revert 9999` is run
    Then the output shows "Entry #9999 not found"
    And the exit code is 1

  Scenario: Revert with address changes
    Given a namespace with a veth pair
    And policy A was applied with addresses=["10.99.0.1/24", "10.99.0.2/24"] (seq=1)
    And policy B was applied with addresses=["10.99.0.3/24"] (seq=2)
    When `netfyr revert 1` is run
    Then veth-e2e0 has addresses 10.99.0.1/24 and 10.99.0.2/24
    And veth-e2e0 does not have address 10.99.0.3/24
    And the address order matches the original order from entry 1

  Scenario: Revert records journal entry
    Given a successful revert to entry seq=1
    When `netfyr history -n 1` is run
    Then the most recent entry has trigger "revert" with target_seq=1

Feature: State revert (daemon mode)
  Scenario: Revert via daemon
    Given the daemon is running with a journal
    And policies were applied resulting in journal entries
    When `netfyr revert <seq>` is run
    Then the revert is executed via the Varlink Revert method
    And a warning about policy drift is printed to stderr

  Scenario: Revert dry-run via daemon
    Given the daemon is running with a journal
    When `netfyr revert <seq> --dry-run` is run
    Then the diff is computed via the daemon
    And no changes are applied
    And no journal entry is recorded

Feature: Revert journal entry
  Scenario: Revert entry contains correct metadata
    Given a revert from entry seq=5
    When the revert journal entry is inspected
    Then its trigger is "revert" with target_seq=5
    And its diff shows the changes from current state to the target
    And its state_after matches the target entry's state_after
    And its outcome reflects the apply result

Feature: SerializableStateSet conversion
  Scenario: Snapshot round-trips through serialization
    Given a StateSet with 3 entities including addresses and routes
    When converted to SerializableStateSet, serialized to JSON, deserialized, and converted back to StateSet
    Then the resulting StateSet has the same entities with the same field values
```
