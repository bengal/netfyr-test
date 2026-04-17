# netfyr

Declarative network configuration for Linux.

netfyr applies a desired network state described in YAML policies to the system
via netlink. Policies are merged by a per-field priority reconciliation engine,
so multiple policies can target the same interface without conflict. A daemon
manages dynamic factories (DHCPv4) and re-reconciles when leases change.

## Architecture

```
 Policy YAML ──> Static Factory ──> StateSet ──┐
                                                ├──> Reconcile ──> Diff ──> Backend (netlink)
 DHCP lease  ──> DHCPv4 Factory ──> StateSet ──┘         ▲
                                                         │
                                              System state (query)
```

Seven crates, layered by responsibility:

| Crate | Role |
|---|---|
| `netfyr-state` | Core types: State, Selector, StateSet, field-level set operations |
| `netfyr-policy` | Policy data model, static factory, YAML parsing, schema validation |
| `netfyr-reconcile` | Per-field priority merge, conflict detection, diff generation |
| `netfyr-backend` | Backend trait and rtnetlink implementation (query + apply) |
| `netfyr-varlink` | Varlink IPC protocol between CLI and daemon |
| `netfyr-cli` | `netfyr` binary: `apply` and `query` commands |
| `netfyr-daemon` | `netfyr-daemon` binary: Varlink server, DHCPv4 factory manager |

## Specifications

Each spec is a self-contained story with acceptance criteria, implementation
details, and dependencies.

### 0xx -- Core data model (`netfyr-state`, `netfyr-policy`)

| Spec | Title | Summary |
|---|---|---|
| [001](specs/001-workspace-setup.md) | Workspace Setup | Rust workspace structure, integration test infrastructure, Makefile |
| [002](specs/002-entity-state-types.md) | State Types | State, FieldValue, Value, Provenance |
| [003](specs/003-selectors.md) | Selectors | Multi-field entity matching with AND logic |
| [004](specs/004-stateset-operations.md) | StateSet Operations | Per-field union, intersection, complement, diff |
| [005](specs/005-yaml-serialization.md) | YAML Serialization | State and StateSet YAML format |
| [006](specs/006-entity-schema-validation.md) | Schema Validation | JSON Schema-based entity validation |
| [007](specs/007-policy-types-static-factory.md) | Policy Types | Policy model and static factory |
| [008](specs/008-bare-state-shorthand.md) | Bare State Shorthand | Auto-wrap YAML without `kind:` into static policies |

### 1xx -- Backend (`netfyr-backend`)

| Spec | Title | Summary |
|---|---|---|
| [101](specs/101-backend-trait.md) | Backend Trait | NetworkBackend trait and BackendRegistry |
| [102](specs/102-rtnetlink-query-ethernet.md) | rtnetlink Query | Query ethernet interfaces via netlink |
| [103](specs/103-rtnetlink-apply-ethernet.md) | rtnetlink Apply | Apply StateDiff operations via netlink |

### 2xx -- Reconciliation (`netfyr-reconcile`)

| Spec | Title | Summary |
|---|---|---|
| [201](specs/201-reconciliation-merge.md) | Reconciliation Merge | Per-field priority merge across policies |
| [202](specs/202-conflict-detection.md) | Conflict Detection | Same-priority same-field different-value detection |
| [203](specs/203-diff-generation.md) | Diff Generation | Desired vs. actual state diff with Add/Modify/Remove |

### 3xx -- CLI (`netfyr-cli`)

| Spec | Title | Summary |
|---|---|---|
| [301](specs/301-cli-apply.md) | CLI Apply | `netfyr apply` with daemon-free and daemon modes |
| [302](specs/302-cli-query.md) | CLI Query | `netfyr query` with selector filtering and YAML/JSON output |

### 4xx -- Daemon (`netfyr-daemon`, `netfyr-varlink`)

| Spec | Title | Summary |
|---|---|---|
| [401](specs/401-dhcpv4-factory.md) | DHCPv4 Factory | DHCP client with dual-socket architecture (packet + UDP) |
| [402](specs/402-policy-store.md) | Policy Store | Persistent policy storage with replace-all semantics |
| [403](specs/403-daemon.md) | Daemon | Varlink server, factory manager, reconciliation loop |
| [404](specs/404-varlink-api.md) | Varlink API | IPC protocol: SubmitPolicies, Query, DryRun, GetStatus |

### 5xx -- Packaging and documentation

| Spec | Title | Summary |
|---|---|---|
| [501](specs/501-man-pages.md) | Man Pages | Generated and hand-written man pages |
| [502](specs/502-rpm-packaging.md) | RPM Packaging | Fedora RPM spec file |
| [503](specs/503-man-page-yaml-reference.md) | YAML Reference | netfyr.yaml(5) man page |

### 6xx -- Integration

| Spec | Title | Summary |
|---|---|---|
| [600](specs/600-end-to-end-tests.md) | End-to-End Tests | Full-pipeline shell integration tests |

## License

See [LICENSE](LICENSE).
