# SPEC-356: Show CLI

## What
Add a `netfyr show` subcommand that displays a system overview: daemon state (running or not, uptime) and all network interfaces with their active policies and DHCP lease details. This is the operator's dashboard -- a single command that answers "what is netfyr doing right now?" The command works both when the daemon is running (full details) and when it is not (basic interface listing).

## Why
The `netfyr query` command shows the current network state (addresses, routes, MTU) but says nothing about the daemon, which policies are active on each interface, or the DHCP lifecycle. Operators need a quick way to see: is the daemon running? Which interfaces are managed? What policies apply to each? Is the DHCP lease healthy? Without this, answering these questions requires cross-referencing `systemctl status`, `netfyr query`, and daemon logs. A dedicated show command provides a single, clear view organized by interface.

## User interaction

### Text output (default)

```
$ netfyr show
Daemon
  Status:  running
  Uptime:  2h 15m

Interfaces
  enp7s0
    Policies:  server-network (dhcpv4)
    DHCP:      running
    Lease:     3600s total, 54m 12s remaining

  enp1s0
    Policies:  mgmt (static)

  lo
```

Multiple policies on the same interface:

```
$ netfyr show
Daemon
  Status:  running
  Uptime:  2h 15m

Interfaces
  enp7s0
    Policies:  server-mtu (static), server-dhcp (dhcpv4)
    DHCP:      running
    Lease:     3600s total, 54m 12s remaining

  enp1s0
    Policies:  mgmt (static)

  lo
```

DHCP factory in waiting state:

```
$ netfyr show
Daemon
  Status:  running
  Uptime:  1m 3s

Interfaces
  enp7s0
    Policies:  server-dhcp (dhcpv4)
    DHCP:      waiting

  lo
```

Only static policies (no DHCP factories):

```
$ netfyr show
Daemon
  Status:  running
  Uptime:  5m 3s

Interfaces
  enp7s0
    Policies:  server-net (static)

  lo
```

### Daemon not running

When the daemon is not running, the command still succeeds (exit 0). It shows daemon status as "not running" and lists all system interfaces without policy or DHCP details (since those require the daemon).

```
$ netfyr show
Daemon
  Status:  not running

Interfaces
  enp7s0
  enp1s0
  lo
```

### JSON output

```
$ netfyr show -o json
{
  "daemon": {
    "status": "running",
    "uptime_seconds": 8103
  },
  "interfaces": [
    {
      "name": "enp7s0",
      "policies": [
        {"name": "server-network", "type": "dhcpv4"}
      ],
      "dhcp": {
        "state": "running",
        "lease_address": "192.168.122.63/24",
        "lease_time_secs": 3600,
        "lease_remaining_secs": 3252
      }
    },
    {
      "name": "enp1s0",
      "policies": [
        {"name": "mgmt", "type": "static"}
      ]
    },
    {
      "name": "lo"
    }
  ]
}
```

Interfaces with no policies omit the `policies` and `dhcp` fields from the JSON output (the fields are absent, not null). Factories in `"waiting"` state include a `dhcp` object with `"state": "waiting"` but omit `lease_address`, `lease_time_secs`, and `lease_remaining_secs`.

### JSON output when daemon is not running

```
$ netfyr show -o json
{
  "daemon": {
    "status": "not_running"
  },
  "interfaces": [
    {"name": "enp7s0"},
    {"name": "enp1s0"},
    {"name": "lo"}
  ]
}
```

When the daemon is not running, `daemon` has no `uptime_seconds` field and all interfaces are listed with only their `name`.

## Implementation details
- Crate: `netfyr-cli`
- Files: `src/show.rs` (new), `src/main.rs` (register subcommand), `src/lib.rs` (export)
- Dependencies (external crates): none new

### CLI arguments

```rust
/// Show system overview
///
/// Display daemon status, all network interfaces, and their active
/// policies and DHCP lease state. Works with or without the daemon.
#[derive(clap::Args)]
pub struct ShowArgs {
    /// Output format: text (default) or json.
    #[arg(short, long, default_value = "text")]
    pub output: OutputFormat,
}
```

### Human-readable duration format

Duration values (uptime, lease remaining) use a compact two-unit format:

| Range | Format | Example |
|-------|--------|---------|
| 0--59 s | `Xs` | `45s` |
| 1--59 min | `Xm Ys` | `15m 3s` |
| 1--23 h | `Xh Ym` | `2h 15m` |
| >= 1 d | `Xd Yh` | `3d 2h` |

The smaller unit is omitted when zero (e.g., `5m` not `5m 0s`).

### Lease line format

The `Lease:` line in text output shows the total lease time in seconds (matching DHCP protocol units) followed by remaining time in human-readable format:

```
    Lease:     120s total, 1m 45s remaining
```

### Policies line format

The `Policies:` line lists all policies active on the interface, each formatted as `name (type)`, comma-separated:

```
    Policies:  server-mtu (static), server-dhcp (dhcpv4)
```

### Varlink API usage

The `netfyr show` command uses the `GetShowInfo` method (SPEC-404) to retrieve system overview data when the daemon is running. The method returns a `ShowInfo` struct containing daemon status and per-interface details.

When the daemon is not running, the CLI falls back to querying the backend directly (via `EthernetBackend::query_all()`) to enumerate interface names, and constructs the response locally with `daemon.status = "not_running"` and bare interface entries.

### Factory lease timing data

The `Dhcpv4Factory` (SPEC-401) stores lease timing alongside the produced state:

```rust
struct LeaseTimingInfo {
    lease_time_secs: u32,
    acquired_at: Instant,
}
```

The background DHCP client task updates this whenever a lease is acquired or renewed. The daemon's `build_show_info()` reads it to compute `lease_time_secs` and `lease_remaining_secs = lease_time_secs - acquired_at.elapsed().as_secs()` (clamped to 0).

### Data flow

1. CLI attempts to connect to the daemon via Varlink.
2. If daemon is running: call `get_show_info()`. The daemon enumerates all system interfaces via `backend.query_all()`, cross-references with the policy store and factory manager, and returns `ShowInfo`.
3. If daemon is not running (connection refused or socket not found): CLI queries the backend directly for interface names and constructs `ShowInfo` with `daemon.status = "not_running"` and bare interface entries (no policies or DHCP data).
4. CLI formats the `ShowInfo` for display (text or JSON).

### Error handling

- Daemon not running: not an error. The CLI falls back to direct backend query and displays limited information.
- Backend query failure (e.g., netlink error): print `Error: <message>` to stderr, exit 1.
- Varlink protocol error: print `Error: <message>` to stderr, exit 1.

### Exit codes

| Code | Meaning |
|------|---------|
| 0 | Show completed successfully (including daemon-not-running case) |
| 1 | Error querying system state or Varlink protocol error |

## Depends on
- SPEC-102 (rtnetlink query -- needed for daemon-free interface enumeration)
- SPEC-301 (CLI infrastructure -- `Cli`, `Commands`, `daemon_socket_path`)
- SPEC-401 (DHCPv4 factory -- lease data source)
- SPEC-403 (daemon -- hosts factories and serves GetShowInfo)
- SPEC-404 (Varlink API -- GetShowInfo method and types)

## Verification
Run `cargo test -p netfyr-cli` for unit tests and `make integration-test` for shell integration tests. The `netfyr show` command must work both with and without a running daemon.

## Acceptance criteria
```gherkin
Feature: Show CLI
  Scenario: Show displays daemon status and interfaces when daemon is running
    Given the daemon is running with 2 static policies for "veth-e2e0" and "veth-e2e1"
    When `netfyr show` is run
    Then the output contains "Daemon" with a "Status:" line showing "running"
    And the output contains an "Uptime:" line with a duration
    And the "Interfaces" section lists managed interfaces with their policies
    And unmanaged interfaces appear as bare names
    And the exit code is 0

  Scenario: Show displays DHCP factory with lease timing
    Given the daemon is running with a DHCP policy for "veth-dhcp0"
    And the factory has acquired a lease with lease_time=120
    When `netfyr show` is run
    Then the interface entry for "veth-dhcp0" shows "Policies:" with "(dhcpv4)"
    And the DHCP line shows "running"
    And the Lease line shows "120s total" and a remaining time

  Scenario: Show displays factory in waiting state
    Given the daemon is running with a DHCP policy for an interface with no DHCP server
    When `netfyr show` is run
    Then the interface entry shows "DHCP:" with "waiting"
    And no Lease line is shown for that interface

  Scenario: Show displays static-only interfaces
    Given the daemon is running with only static policies
    When `netfyr show` is run
    Then managed interfaces show "Policies:" with "(static)"
    And no DHCP or Lease lines appear

  Scenario: Show works when daemon is not running
    Given the netfyr daemon is not running
    When `netfyr show` is run
    Then the exit code is 0
    And the output contains "Status:" with "not running"
    And the "Interfaces" section lists system interfaces as bare names
    And no Policies, DHCP, or Lease lines appear

  Scenario: Show lists all system interfaces including unmanaged
    Given the daemon is running with a policy for "veth-e2e0" only
    And "veth-e2e1" and "lo" exist but have no policies
    When `netfyr show` is run
    Then all three interfaces appear in the output
    And "veth-e2e0" has a Policies line
    And "veth-e2e1" and "lo" appear as bare names

  Scenario: Show JSON output when daemon is running
    Given the daemon is running with a DHCP lease on "veth-dhcp0"
    When `netfyr show -o json` is run
    Then the output is valid JSON
    And "daemon.status" is "running"
    And "daemon.uptime_seconds" is a non-negative integer
    And "interfaces" is an array
    And the entry for "veth-dhcp0" has a "policies" array and a "dhcp" object
    And "dhcp.state" is "running"
    And "dhcp.lease_time_secs" matches the server's lease time
    And "dhcp.lease_remaining_secs" is a non-negative integer

  Scenario: Show JSON output for waiting factory
    Given the daemon is running with a DHCP policy but no DHCP server
    When `netfyr show -o json` is run
    Then the interface entry has "dhcp.state": "waiting"
    And the dhcp object does not have "lease_time_secs" or "lease_remaining_secs"

  Scenario: Show JSON output when daemon is not running
    Given the netfyr daemon is not running
    When `netfyr show -o json` is run
    Then the output is valid JSON
    And "daemon.status" is "not_running"
    And "daemon" has no "uptime_seconds" field
    And "interfaces" is an array of objects with only "name" fields

  Scenario: Show with multiple policies on one interface
    Given the daemon is running with both a static and a DHCP policy for "veth-dhcp0"
    When `netfyr show` is run
    Then the Policies line for "veth-dhcp0" lists both policies comma-separated

  Scenario: Uptime format uses compact two-unit durations
    Given the daemon has been running for 7385 seconds
    When `netfyr show` is run
    Then the Uptime line shows "2h 3m"

  Scenario: Lease remaining format uses compact two-unit durations
    Given a factory with a 3600s lease acquired 3000 seconds ago
    When `netfyr show` is run
    Then the Lease line shows "3600s total, 10m remaining"
```
