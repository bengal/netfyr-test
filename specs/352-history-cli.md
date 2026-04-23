# SPEC-352: History CLI and Varlink API

## What

Add a `netfyr history` CLI subcommand that displays journal entries recorded by the journal infrastructure (SPEC-351). The command supports filtering by time, trigger type, and entity, and outputs in text, JSON, or YAML formats. In daemon mode, the command delegates to new Varlink API methods.

## Why

The journal (SPEC-351) records state changes but provides no user-facing way to inspect them beyond manually reading NDJSON files. `netfyr history` gives operators a structured, filterable view of what happened, when, and why -- essential for incident investigation, change auditing, and understanding system behavior over time.

## User interaction

```bash
# List the last 20 changes (default)
netfyr history

# List the last 50 changes
netfyr history -n 50

# Show changes from the last hour
netfyr history --since 1h

# Filter by trigger type
netfyr history --trigger external
netfyr history --trigger apply
netfyr history --trigger dhcp

# Filter by entity
netfyr history -s name=eth0

# Show full details for a specific entry
netfyr history --show 142

# JSON output (for piping to jq)
netfyr history -o json

# Combine filters
netfyr history -s name=eth0 --since 7d --trigger apply -o json
```

### Output formats

#### Text list (default)

```
SEQ  TIMESTAMP             TRIGGER         ENTITIES       OUTCOME          CHANGES
142  2026-04-20 14:30:00   policy-apply    eth0           applied          ~mtu
141  2026-04-20 14:15:32   dhcp-lease      eth0           applied          addr(+1)
140  2026-04-20 14:10:00   external        eth1           observed         ~mtu
139  2026-04-20 14:00:00   daemon-startup  eth0, eth1     applied          +2 entities
138  2026-04-20 13:45:00   policy-apply    eth0           applied (1 fail) ~mtu, addr(+1)
```

The `OUTCOME` column in list view shows only the outcome kind (`applied`, `observed`). A failure count is appended only when failures occurred: `applied (N fail)`. Success and skip counts are omitted — they add noise in the common case and the detail view (`--show`) has the full breakdown. The column has a fixed width of 17 characters, enough for `applied (N fail)` with reasonable entity counts. If the text exceeds the column width (e.g., very large failure counts), it is truncated with `…`.

The CHANGES column is placed last so it can use all remaining terminal width. When the line would exceed the terminal width (or 120 characters if stdout is not a TTY), the CHANGES value is truncated with `...`. Full details are available via `--show <seq>`.

#### Text detail (`--show <seq>`)

```
Entry #142 at 2026-04-20 14:30:00 UTC
Trigger: policy-apply (source: /etc/netfyr/policies/)
Active policies:
  - eth0-config (static, priority 100)
  - eth0-dhcp (dhcpv4, priority 100)
Diff:
  ~ ethernet eth0
      -mtu: 1500
      +mtu: 9000
Outcome: applied (1 succeeded, 0 failed, 0 skipped)
State after:
  ethernet eth0:
    mtu: 9000
    addresses: [10.0.1.50/24]
    carrier: true
```

The diff section uses unified-diff style (see SPEC-203 for the full format definition):
- Scalar field changes show separate red (`-old`) and green (`+new`) lines.
- List field changes (addresses, routes) show a field-name header followed by per-element `+`/`-` lines. Only changed elements are shown; unchanged elements are omitted. Route elements use the format `destination metric N` (omitting metric when 0).

Example with list field changes:
```
Diff:
  ~ ethernet enp7s0
      addresses:
        +172.25.14.22/32
      routes:
        +172.25.14.22/32 metric 0
```

#### JSON output (`-o json`)

Each entry is output as a JSON object (same format as the NDJSON journal file). When listing multiple entries, they are output as a JSON array.

## Implementation details

- Crate: `netfyr-cli` (CLI command), `netfyr-varlink` (Varlink API additions), `netfyr-daemon` (Varlink handler)
- Files:
  - `crates/netfyr-cli/src/main.rs` -- add `History` to the `Commands` enum
  - `crates/netfyr-cli/src/history.rs` -- (new) history command implementation
  - `crates/netfyr-varlink/src/io.netfyr.varlink` -- add `GetHistory` and `GetJournalEntry` methods
  - `crates/netfyr-daemon/src/server.rs` -- handle new Varlink methods
- Dependencies (external crates): none beyond what SPEC-351 already adds

### CLI arguments

```rust
#[derive(Args)]
struct HistoryArgs {
    /// Number of entries to show (most recent first). Default: 20.
    #[arg(long, short = 'n', default_value = "20")]
    count: usize,

    /// Show entries since this time. Accepts relative durations (1h, 30m, 7d)
    /// or ISO 8601 timestamps.
    #[arg(long)]
    since: Option<String>,

    /// Filter by trigger type: "apply", "dhcp", "external", "startup", "revert".
    #[arg(long)]
    trigger: Option<String>,

    /// Filter by entity name (shows only entries that touched this entity).
    #[arg(long, short = 's', value_parser = parse_selector)]
    selector: Vec<(String, String)>,

    /// Show full detail for a single entry by sequence ID.
    #[arg(long)]
    show: Option<u64>,

    /// Output format: "text" (default), "json".
    #[arg(long, short = 'o', default_value = "text")]
    output: OutputFormat,
}
```

### Dual-mode operation

The history command follows the same daemon-detection pattern as `apply` and `query`:

1. Read `NETFYR_SOCKET_PATH` env var (default `/run/netfyr/netfyr.sock`).
2. Attempt connection.
3. If connected: delegate to Varlink (`GetHistory` or `GetJournalEntry`).
4. If not connected: read journal files directly via `Journal::open_default()`.

In daemon-free mode, the CLI reads journal files directly using the `Journal` struct from `netfyr-journal`. This works because the journal is just NDJSON files on disk.

### Duration parsing

The `--since` flag accepts:
- Relative durations: `30s`, `5m`, `1h`, `7d` (seconds, minutes, hours, days)
- ISO 8601 timestamps: `2026-04-20T14:00:00Z`

Implement a `parse_duration` function in the history module that handles both formats.

### Filtering logic

Filters are applied after reading entries:
1. `--since`: compare entry timestamp against the computed cutoff time.
2. `--trigger`: match the trigger variant name (case-insensitive, partial match: "apply" matches "policy_apply").
3. `--selector name=X`: check if any entity in the diff operations matches the selector.
4. `--count`: limit the number of results returned (applied last).

Filters combine with AND logic.

### Text formatting

The list format uses fixed-width columns for all columns except CHANGES, which is placed last and uses all remaining terminal width. The column order is: SEQ, TIMESTAMP, TRIGGER, ENTITIES, OUTCOME, CHANGES.

The `ENTITIES` column has a fixed width of 25 characters — enough for one maximum-length Linux interface name (15 chars) plus the `+N more` suffix. It shows a comma-separated list of entity names from the diff operations. If the text would exceed the column width, it is truncated with `+N more` (e.g., `enp7s0, +2 more`).

The `CHANGES` column shows a compact summary using this notation:
- **Scalar fields** (mtu, carrier, state, driver, mac, name): `+field` (set), `~field` (changed), `-field` (removed).
- **List fields** (addresses, routes): `addr(+2)` (2 added), `addr(-1)` (1 removed), `addr(+1,-1)` (1 added, 1 removed), `routes(+3)` (3 added). Zero counts are omitted. List fields only have `+N` and `-N` counts, never `~`, because individual list elements are either present or absent.
- **Entity-level operations**: `+entity` (new entity), `-entity` (entity removed).

When color is enabled (see SPEC-301 for the global `--color` flag), the entire `+` line is green, the entire `-` line is red, and `~` lines are yellow. Colors reinforce but never replace the textual indicators — the output is unambiguous without color.

The CHANGES value is truncated with `...` when it would exceed the terminal width (or 120 characters if stdout is not a TTY).

The detail format (`--show`) renders the full `JournalEntry` in a human-readable layout matching the format shown in the User Interaction section. The diff section uses unified-diff style as defined in SPEC-203: scalar fields show `-old` / `+new` line pairs; list fields show a header followed by per-element `+`/`-` lines (unchanged elements omitted). When color is enabled, the entire `+` line is green and the entire `-` line is red (not just the `+`/`-` prefix character — the field name and value are colored too).

### Varlink API additions

```varlink
method GetHistory(count: ?int, since: ?string, trigger: ?string, selector_name: ?string) -> (entries: []JournalEntry)
method GetJournalEntry(seq: int) -> (entry: ?JournalEntry)

type JournalEntry (
    seq: int,
    timestamp: string,
    trigger: Trigger,
    active_policies: []PolicySummary,
    diff: DiffSummary,
    state_after: []StateSummary,
    outcome: ApplyOutcome
)

type Trigger (
    type: string,
    source: ?string,
    policy_name: ?string,
    event_kind: ?string,
    changed_entities: ?[]string,
    target_seq: ?int
)

type PolicySummary (
    name: string,
    factory_type: string,
    priority: int
)

type DiffSummary (
    operations: []DiffOpSummary
)

type DiffOpSummary (
    kind: string,
    entity_type: string,
    entity_name: string,
    field_changes: []FieldChangeSummary
)

type FieldChangeSummary (
    field_name: string,
    change_kind: string,
    current: ?string,
    desired: ?string
)

type StateSummary (
    entity_type: string,
    selector_name: string,
    fields: object
)

type ApplyOutcome (
    kind: string,
    succeeded: ?int,
    failed: ?int,
    skipped: ?int
)
```

## Depends on

- SPEC-351 (journal infrastructure -- provides `Journal`, `JournalEntry`, and file I/O)
- SPEC-302 (CLI query -- follows the same CLI patterns: clap args, output formats, daemon detection)
- SPEC-404 (Varlink API -- extends the existing Varlink interface with new methods)

## Acceptance criteria

```gherkin
Feature: History list command
  Scenario: List recent journal entries
    Given a journal with 30 entries
    When `netfyr history` is run
    Then the output shows the 20 most recent entries in reverse chronological order
    And each row shows SEQ, TIMESTAMP, TRIGGER, ENTITIES, OUTCOME, and CHANGES

  Scenario: Limit entry count
    Given a journal with 30 entries
    When `netfyr history -n 5` is run
    Then exactly 5 entries are shown

  Scenario: Filter by time
    Given journal entries from 2 hours ago, 30 minutes ago, and 5 minutes ago
    When `netfyr history --since 1h` is run
    Then only the entries from 30m and 5m ago are shown

  Scenario: Filter by trigger type
    Given journal entries with triggers: policy_apply, dhcp_event, external_change
    When `netfyr history --trigger external` is run
    Then only the external_change entry is shown

  Scenario: Filter by entity name
    Given journal entries affecting eth0 and eth1
    When `netfyr history -s name=eth0` is run
    Then only entries that include eth0 in their diff are shown

  Scenario: Combine filters
    Given multiple journal entries
    When `netfyr history -s name=eth0 --trigger apply --since 1h` is run
    Then only entries matching all three filters are shown

Feature: History detail command
  Scenario: Show full detail for an entry
    Given a journal with entry seq=42
    When `netfyr history --show 42` is run
    Then the output shows the full entry including trigger details, active policies, diff, outcome, and state snapshot

  Scenario: Detail diff shows scalar change as unified-diff lines
    Given a journal entry where mtu changed from 1500 to 9000 on ethernet eth0
    When `netfyr history --show <seq>` is run
    Then the diff section contains a line "      -mtu: 1500"
    And the diff section contains a line "      +mtu: 9000"

  Scenario: Detail diff colors entire lines, not just the prefix
    Given a journal entry where mtu changed from 1500 to 9000 on ethernet eth0
    When `netfyr history --show <seq> --color always` is run
    Then the "-mtu: 1500" line is entirely red (ANSI red wraps the full line including field name and value)
    And the "+mtu: 9000" line is entirely green (ANSI green wraps the full line including field name and value)
    And the color does not apply only to the "+" or "-" character

  Scenario: Detail diff shows list field additions as per-element lines
    Given a journal entry where address 172.25.14.22/32 was added to ethernet enp7s0
    When `netfyr history --show <seq>` is run
    Then the diff section contains a line "      addresses:"
    And the diff section contains a line "        +172.25.14.22/32"
    And unchanged addresses are not shown

  Scenario: Detail diff shows route changes with readable format
    Given a journal entry where route 10.0.0.0/8 metric 100 was added to ethernet eth0
    When `netfyr history --show <seq>` is run
    Then the diff section contains a line "      routes:"
    And the diff section contains a line "        +10.0.0.0/8 metric 100"

  Scenario: Show nonexistent entry
    When `netfyr history --show 9999` is run
    Then the output shows "Entry #9999 not found"
    And the exit code is 1

Feature: JSON output
  Scenario: List entries in JSON format
    Given a journal with 5 entries
    When `netfyr history -n 5 -o json` is run
    Then the output is a valid JSON array with 5 elements
    And each element has the same structure as a JournalEntry

  Scenario: Show entry detail in JSON
    Given a journal with entry seq=42
    When `netfyr history --show 42 -o json` is run
    Then the output is a valid JSON object representing the entry

Feature: Daemon-mode history via Varlink
  Scenario: History delegates to daemon when running
    Given the daemon is running and has journal entries
    When `netfyr history` is run
    Then the entries are retrieved via the GetHistory Varlink method
    And the output matches what would be shown from direct file reading

  Scenario: GetJournalEntry Varlink method
    Given the daemon is running with a journal entry seq=10
    When a Varlink GetJournalEntry(seq: 10) request is sent
    Then the response contains the full journal entry

Feature: Empty journal
  Scenario: No entries exist
    Given an empty journal
    When `netfyr history` is run
    Then the output shows "No journal entries found."

Feature: History with no journal directory
  Scenario: Journal directory does not exist
    When `netfyr history` is run and no journal directory exists
    Then the output shows "No journal found at /var/lib/netfyr/journal/"
    And the exit code is 1
```
