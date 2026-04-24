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

# Show the most recent entry
netfyr history --show -1

# Show the third-to-last entry
netfyr history --show -3

# Show full timestamps instead of relative
netfyr history --absolute-timestamps

# JSON output (for piping to jq)
netfyr history -o json

# Combine filters
netfyr history -s name=eth0 --since 7d --trigger apply -o json
```

### Output formats

#### Text list (default)

```
SEQ  TIMESTAMP        TRIGGER               ENTITIES        CHANGES
142  30 min ago       apply (eth0-static)   eth0            mtu 1500→9000
141  45 min ago       dhcp-acquire          eth0            +10.0.1.50/24
140  50 min ago       external              eth1            mtu 1400→1500
139  1h ago           daemon-startup        +eth0, +eth1    +eth0, +eth1
138  1h ago           apply (eth0-st…, +1)  eth0            FAIL mtu 1400→9000, +10.0.1.50/24
──── daemon restart ────
137  yesterday 12:30  external              eth0, sys:dns   +192.168.1.100/24, +ns 8.8.8.8
136  yesterday 12:28  external              eth0            mtu 1400→1500, +10.0.1.51/24
135  yesterday 12:27  external              eth0            +10.0.1.50/24, +3 routes
134  yesterday 12:25  external              eth0            -10.0.1.99/24
133  yesterday 12:22  external              eth0            +10.0.1.51/24 (+2 addrs)
132  yesterday 12:20  external              eth0            +10.0.1.50/24
131  yesterday 12:15  daemon-startup        eth0            -10.0.1.99/24, -3 routes
```

**Relative timestamps**: entries from today show relative durations (`2 min ago`, `1h ago`), yesterday's entries show `yesterday HH:MM`, and older entries show the full date (`2026-04-18 14:30`).

**Daemon-startup separators**: a `──── daemon restart ────` line is inserted after each entry whose trigger is `daemon-startup`, visually separating daemon lifecycle sessions. The separator appears between the daemon-startup row and the row above it (which belongs to the previous session).

**Dynamic column widths**: each column width is computed as `max(header_width, widest_value)`, capped at a per-column maximum. Values exceeding the cap are truncated with `…`. The CHANGES column (rightmost) has no maximum and uses all remaining terminal width. See the Text formatting section for max width values. Columns are separated by two spaces for readability.

There is no OUTCOME column in the list view. The outcome is fully determined by the trigger: `ExternalChange` always produces `Observed`, and all other triggers produce `Applied`. When an apply has failures, the CHANGES column is prefixed with `FAIL`: e.g., `FAIL mtu 1400→9000`. The detail view (`--show`) still shows the full outcome breakdown.

The CHANGES column is placed last so it can use all remaining terminal width. When the line would exceed the terminal width (or 120 characters if stdout is not a TTY), the CHANGES value is truncated with `…`. Full details are available via `--show <seq>`.

#### Text detail (`--show <seq>`)

The `<seq>` argument accepts a positive sequence number or a negative offset from the end: `-1` is the most recent entry, `-2` is the second-to-last, etc.

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
- type: ethernet
  name: eth0
  mtu: 9000
  addresses:
    - 10.0.1.50/24
  carrier: true
```

The "State after" section uses exactly the same YAML format as `netfyr query` output (see SPEC-302): a YAML sequence where each element has `type`, selector properties, and configuration fields at the top level. This means the same serialization code used for `netfyr query` must be reused for the state-after section of `--show`.

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
    /// Positive values are absolute sequence numbers.
    /// Negative values count from the end: -1 is the most recent entry.
    #[arg(long, allow_hyphen_values = true)]
    show: Option<i64>,

    /// Output format: "text" (default), "json".
    #[arg(long, short = 'o', default_value = "text")]
    output: OutputFormat,

    /// Show full timestamps (YYYY-MM-DD HH:MM:SS) instead of relative/abbreviated.
    #[arg(long)]
    absolute_timestamps: bool,
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

#### Dynamic column widths

The list format uses dynamically computed column widths. For each column, the width is `max(header_width, widest_value_in_column)`, capped at a per-column maximum. Values exceeding the cap are truncated with `…` (Unicode ellipsis). The CHANGES column is always last and has no maximum — it uses all remaining terminal width.

Column order and maximum widths:

| Column    | Max width | Notes |
|-----------|-----------|-------|
| SEQ       | 7         | Enough for 7-digit sequence numbers |
| TIMESTAMP | 20        | Enough for `yesterday HH:MM` and relative durations |
| TRIGGER   | 24        | Enough for `apply (policy-name)` without truncation in most cases |
| ENTITIES  | 24        | Enough for one max-length interface name (15) + lifecycle prefix + `(+N more)` |
| CHANGES   | unlimited | Uses all remaining terminal width |

Columns are separated by two spaces. When the CHANGES value would exceed the terminal width (or 120 characters if stdout is not a TTY), it is truncated with `…`.

#### TRIGGER column

The TRIGGER column shows a human-readable label for what caused the change, with context where useful:

| Trigger variant | Display format | Examples |
|---|---|---|
| `PolicyApply` | `apply (<policy>)` | `apply (eth0-static)`, `apply (eth0-static, +2)` |
| `DhcpEvent` | `dhcp-acquire`, `dhcp-renew`, `dhcp-expire` | `dhcp-acquire` |
| `ExternalChange` | `external` | `external` |
| `DaemonStartup` | `daemon-startup` | `daemon-startup` |
| `Revert` | `revert (<seq>)` | `revert (42)` |

For `apply`, the parenthesized value shows the first policy name from `active_policies`. If multiple policies were applied, append `+N`: `apply (eth0-static, +2)`. The policy name is always used — never the source path or `"daemon"`, since the user cares about what was applied, not how it was submitted.

For DHCP events, the `event_kind` field (`lease_acquired`, `lease_renewed`, `lease_expired`) maps directly to `dhcp-acquire`, `dhcp-renew`, `dhcp-expire`.

For `revert`, the parenthesized value is the target sequence number being reverted to.

When the trigger text exceeds the column's max width (24 chars), it is truncated with `…`.

#### Timestamps

By default, timestamps use relative or abbreviated formats depending on age:
- **Today**: relative duration — `2 min ago`, `30 min ago`, `1h ago`, `5h ago`.
- **Yesterday**: `yesterday HH:MM` (24-hour format).
- **Older**: full date — `2026-04-18 14:30`.

When `--absolute-timestamps` is passed, all list-view timestamps use the full format `YYYY-MM-DD HH:MM:SS` instead.

The `--show` detail view always uses the full ISO 8601 timestamp (`2026-04-20 14:30:00 UTC`) regardless of the flag.

#### ENTITIES column

The ENTITIES column shows which entities were affected by the change. Entity names are prefixed with lifecycle indicators:
- `+entity` — entity was created (Add operation in the diff).
- `-entity` — entity was removed (Remove operation in the diff).
- `entity` (no prefix) — entity was modified.

System-wide resources (not network interfaces) use the `sys:` prefix to avoid ambiguity with interface names: `sys:dns`, `sys:hostname`, `sys:ntp`. Interface names are shown bare.

When the entity list exceeds the column width, it is capped with a count suffix:
- **1–3 entities**: show all.
- **4–6 entities**: show the first 2, then `(+N more)`.
- **7+ entities**: show counts by operation — `+4, ~2, -1 entities`.

When capping, prioritize entities with lifecycle events (created/removed) over modifications, and entities with the largest change sets.

#### CHANGES column

The CHANGES column shows a compact summary of what changed, using actual values instead of opaque counts.

**Scalar field changes** use `old→new` notation: `mtu 1500→9000`, `state up→down`. New fields show just the value prefixed with `+`: `+mtu: 1500`. Removed fields show `-field`.

**Address changes** show actual addresses inline, with `+`/`-` prefixes:
- 1–2 addresses changed: show all by value — `+192.168.1.100/24`, `-10.0.0.42/24`.
- 3–8 addresses changed: show the first 2 by value, count the rest — `+192.168.1.100/24 +10.0.0.50/24 (+3 addrs), -10.0.0.42/24 (-2 addrs)`.
- 9+ addresses changed: show only counts — `+10 addrs, -10 addrs`.

When selecting which addresses to show by value, prioritize non-link-local addresses over `fe80::`/`169.254.` addresses.

**Route changes** always show the default route by value when it changes: `+dflt via 10.0.0.1`, `-dflt via 192.168.1.1`. Non-default routes use counts: `+3 routes`, `-1 route`. The default route is always shown regardless of the address cap.

**DNS changes** show nameserver values with the `ns` shorthand: `+ns 8.8.8.8`, `-ns 10.0.0.1`. Search domain changes use scalar notation: `search example.com→corp.local`.

**Entity-level operations**: `+entity` (new entity created), `-entity` (entity removed).

When color is enabled (see SPEC-301 for the global `--color` flag), added values are green, removed values are red, and changed scalars are yellow. Colors reinforce but never replace the textual indicators — the output is unambiguous without color.

#### Daemon-startup separators

A visual separator `──── daemon restart ────` is inserted between daemon lifecycle sessions. The separator appears between a `daemon-startup` entry and the entry above it (which belongs to the previous session). No separator is shown above the oldest visible entry.

The detail format (`--show`) renders the full `JournalEntry` in a human-readable layout matching the format shown in the User Interaction section. The diff section uses unified-diff style as defined in SPEC-203: scalar fields show `-old` / `+new` line pairs; list fields show a header followed by per-element `+`/`-` lines (unchanged elements omitted). When color is enabled, the entire `+` line is green and the entire `-` line is red (not just the `+`/`-` prefix character — the field name and value are colored too).

The "State after" section must use exactly the same YAML serialization as `netfyr query` (SPEC-302): a YAML block-style sequence where each element has `type`, selector properties, and configuration fields at the top level. Lists (addresses, routes) use YAML block sequences, never JSON-style inline arrays or objects. The implementation must reuse the same serialization code path as `netfyr query` to guarantee format consistency.

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
    And each row shows SEQ, TIMESTAMP, TRIGGER, ENTITIES, and CHANGES

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

Feature: Dynamic column widths
  Scenario: Column widths adapt to content
    Given a journal with entries where SEQ values are 1-9 (single digit)
    And entity names are all "eth0" (4 chars)
    When `netfyr history` is run
    Then the SEQ column is 3 characters wide (matching header width)
    And the ENTITIES column is 8 characters wide (matching header width)
    And columns are not padded to their maximum width

  Scenario: Column widths respect maximum caps
    Given a journal entry with 10 entities whose names total over 30 characters
    When `netfyr history` is run
    Then the ENTITIES column does not exceed 24 characters
    And the overflowing entity list is truncated with "…"

  Scenario: CHANGES column uses remaining terminal width
    Given a journal entry with a long CHANGES value
    When `netfyr history` is run on a 120-column terminal
    Then the CHANGES column extends to fill the remaining terminal width
    And values exceeding the available width are truncated with "…"

Feature: Relative timestamps
  Scenario: Entries from today show relative durations
    Given a journal entry from 5 minutes ago and one from 2 hours ago
    When `netfyr history` is run
    Then the recent entry shows "5 min ago"
    And the older entry shows "2h ago"

  Scenario: Entries from yesterday show abbreviated date
    Given a journal entry from yesterday at 15:26
    When `netfyr history` is run
    Then the entry shows "yesterday 15:26"

  Scenario: Older entries show full date
    Given a journal entry from 3 days ago at 14:30
    When `netfyr history` is run
    Then the entry shows the date in "YYYY-MM-DD HH:MM" format

  Scenario: Detail view always shows full timestamp
    Given a journal entry from 5 minutes ago
    When `netfyr history --show <seq>` is run
    Then the timestamp is shown in full ISO 8601 format with UTC suffix

  Scenario: Absolute timestamps flag overrides relative format
    Given a journal entry from 5 minutes ago
    When `netfyr history --absolute-timestamps` is run
    Then the TIMESTAMP column shows "YYYY-MM-DD HH:MM:SS" (full format, not "5 min ago")

Feature: CHANGES column inline values
  Scenario: Scalar change shows old and new values
    Given a journal entry where mtu changed from 1500 to 9000 on eth0
    When `netfyr history` is run
    Then the CHANGES column shows "mtu 1500→9000"

  Scenario: Single address addition shows actual value
    Given a journal entry where address 192.168.1.100/24 was added
    When `netfyr history` is run
    Then the CHANGES column shows "+192.168.1.100/24"

  Scenario: Two address changes show both values
    Given a journal entry where 10.0.0.50/24 was added and 10.0.0.42/24 was removed
    When `netfyr history` is run
    Then the CHANGES column shows "+10.0.0.50/24, -10.0.0.42/24"

  Scenario: Many address changes cap at two shown values
    Given a journal entry where 5 addresses were added and 3 removed
    When `netfyr history` is run
    Then the CHANGES column shows the first 2 added addresses by value
    And shows "(+3 addrs)" for the remaining additions
    And shows the first removed address by value
    And shows "(-2 addrs)" for the remaining removals

  Scenario: Very many address changes show only counts
    Given a journal entry where 10 addresses were added and 10 removed
    When `netfyr history` is run
    Then the CHANGES column shows "+10 addrs, -10 addrs"

  Scenario: Default route change always shown by value
    Given a journal entry where the default route changed and 5 other routes were added
    When `netfyr history` is run
    Then the CHANGES column shows "+dflt via 10.0.0.1" (the default route by value)
    And shows "+5 routes" for the non-default routes

  Scenario: Non-default route changes show counts only
    Given a journal entry where 3 non-default routes were added
    When `netfyr history` is run
    Then the CHANGES column shows "+3 routes"

  Scenario: Address priority prefers non-link-local
    Given a journal entry where fe80::1/64 and 192.168.1.100/24 were both added
    When `netfyr history` is run
    Then the CHANGES column shows 192.168.1.100/24 (non-link-local is shown first)

  Scenario: DNS nameserver changes show ns shorthand
    Given a journal entry where nameserver 8.8.8.8 was added on sys:dns
    When `netfyr history` is run
    Then the CHANGES column shows "+ns 8.8.8.8"

  Scenario: DNS search domain change shows scalar notation
    Given a journal entry where the search domain changed from example.com to corp.local
    When `netfyr history` is run
    Then the CHANGES column shows "search example.com→corp.local"

Feature: ENTITIES column formatting
  Scenario: Created entity shows + prefix
    Given a journal entry with a diff Add operation for ethernet eth0
    When `netfyr history` is run
    Then the ENTITIES column shows "+eth0"

  Scenario: Removed entity shows - prefix
    Given a journal entry with a diff Remove operation for vlan bond0.200
    When `netfyr history` is run
    Then the ENTITIES column shows "-bond0.200"

  Scenario: Modified entity shows no prefix
    Given a journal entry with a diff Modify operation for ethernet eth0
    When `netfyr history` is run
    Then the ENTITIES column shows "eth0" (no prefix)

  Scenario: System entity uses sys: prefix
    Given a journal entry affecting dns/global
    When `netfyr history` is run
    Then the ENTITIES column shows "sys:dns"

  Scenario: Mixed interface and system entities
    Given a journal entry affecting ethernet eth0 and dns/global
    When `netfyr history` is run
    Then the ENTITIES column shows "eth0, sys:dns"

  Scenario: Many entities are capped
    Given a journal entry affecting 6 entities
    When `netfyr history` is run
    Then the ENTITIES column shows the first 2 entities and "(+4 more)"

Feature: Daemon-startup separators
  Scenario: Separator between daemon sessions
    Given journal entries: seq=5 (policy-apply), seq=4 (daemon-startup), seq=3 (external)
    When `netfyr history` is run
    Then a "──── daemon restart ────" separator appears between seq=5 and seq=3
    And the separator is placed after the daemon-startup entry (between it and the previous session)

  Scenario: No separator above oldest visible entry
    Given a journal with 3 entries where seq=1 is daemon-startup
    When `netfyr history` is run
    Then no separator appears below seq=1 (nothing older to separate from)

Feature: History detail command
  Scenario: Show full detail for an entry
    Given a journal with entry seq=42
    When `netfyr history --show 42` is run
    Then the output shows the full entry including trigger details, active policies, diff, outcome, and state snapshot
    And the "State after" section uses the same YAML format as `netfyr query` output (SPEC-302)

  Scenario: State-after format matches query output
    Given a journal entry for ethernet eth0 with mtu=9000, addresses=["10.0.1.50/24"], carrier=true
    When `netfyr history --show <seq>` is run
    Then the "State after" section contains "- type: ethernet"
    And it contains "  name: eth0"
    And it contains "  mtu: 9000"
    And addresses are listed as a YAML block sequence (one "- addr" per line), not a JSON array
    And routes are listed as a YAML block sequence with destination/gateway/metric fields, not JSON objects

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

  Scenario: Show most recent entry with negative offset
    Given a journal with 30 entries (seq 1 through 30)
    When `netfyr history --show -1` is run
    Then the output shows the full detail for entry seq=30

  Scenario: Show Nth-to-last entry with negative offset
    Given a journal with 30 entries (seq 1 through 30)
    When `netfyr history --show -3` is run
    Then the output shows the full detail for entry seq=28

  Scenario: Negative offset beyond journal size
    Given a journal with 5 entries
    When `netfyr history --show -10` is run
    Then the output shows "Entry not found"
    And the exit code is 1

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
