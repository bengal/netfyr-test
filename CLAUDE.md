# netfyr specs

This repository contains user story specifications for the netfyr network configuration tool. The factory at `~/work/factory2` processes these specs to generate a Rust project.

## Spec structure

Every spec should have these sections:

- `## What` -- what is being implemented
- `## Why` -- rationale for the design
- `## Implementation details` -- crate, files, dependencies
- `## Acceptance criteria` -- Gherkin-style scenarios (Feature/Scenario/Given/When/Then)
- `## Depends on` -- list of dependencies as `SPEC-NNN (description)`

Optional: `## Verification`, `## User interaction`, `## Integration test infrastructure`.

## Numbering ranges

| Range | Layer | Examples |
|-------|-------|---------|
| 0xx | State types, YAML, schema, policy wrapping | 001-workspace-setup through 008-bare-state-shorthand |
| 1xx | Backend (rtnetlink, DHCP) | 101-backend-trait, 102-query, 103-apply |
| 2xx | Reconciliation (merge, conflicts, diff) | 201-203 |
| 3xx | CLI and user features (apply, query, history, revert, journal) | 301-354 |
| 4xx | Daemon, factories, varlink, policy store | 401-405 |
| 5xx | Packaging and documentation | 501-503 |
| 6xx | End-to-end integration tests | 600 |

Dependencies only point backward (lower numbers). No forward references.

## Where to put test scenarios

**Individual story specs** (0xx-5xx) describe behavior and implementation. Their acceptance criteria verify the feature in isolation (unit tests, single-crate integration tests).

**Spec 600 (end-to-end tests)** is the comprehensive integration test suite. It exercises cross-cutting user workflows involving the daemon, CLI, reconciliation, and journal together. It has a strict **no-skip policy**: if a prerequisite is missing (`unshare`, `dnsmasq`, binaries), the test must print `FAIL: ...` to stderr and `exit 1`. Never `exit 0` on failure.

**Rule: tests that require the daemon, network namespaces, or multiple components interacting belong in spec 600, not in individual story specs.** Individual specs may describe the behavior scenarios, but the executable test scenarios that cannot be skipped must be in 600-end-to-end-tests.md. For example, "external route change detection" is described in spec 353 but the integration test scenario belongs in spec 600.

## Crate mapping

| Crate | Specs | Purpose |
|-------|-------|---------|
| netfyr-state | 001-008 | Core types, StateSet, YAML, schema validation |
| netfyr-policy | 007-008 | Policy types, static/state factories |
| netfyr-backend | 101-103 | Backend trait, rtnetlink query/apply |
| netfyr-reconcile | 201-203 | Merge, conflict detection, diff generation |
| netfyr-journal | 351 | Journal infrastructure |
| netfyr-cli | 301, 302, 352, 354 | CLI subcommands (apply, query, history, revert) |
| netfyr-daemon | 353, 401-405 | Daemon, DHCPv4 factory, policy store, varlink |
| netfyr-varlink | 404 | Varlink protocol definitions |

## Read-only fields

Ethernet entity fields marked `x-netfyr-writable: false` in the JSON schema: `mac`, `carrier`, `speed`, `driver`, `name`, `operstate`. These appear in query results but are excluded from diffs and cannot be set by policies.

## Common pitfalls

- Never use sysfs (`/sys/class/net/`) for namespace-aware queries -- use netlink
- Never skip integration tests with `exit 0` when prerequisites are missing
- External changes are recorded but never trigger re-reconciliation -- the operator decides via `netfyr apply` or `netfyr revert`
- Remove operations on physical ethernet interfaces only deconfigure them, never delete the interface
- Prefer standard library over external crates; minimize transitive dependencies
