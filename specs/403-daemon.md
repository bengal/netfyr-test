# SPEC-403: Daemon Architecture

## What
Implement the `netfyr-daemon` binary (or `netfyrd`). The daemon is a long-running process that listens on a Varlink socket, accepts policy submissions, manages DHCPv4 factory lifecycles, runs reconciliation, and applies the effective state to the system. It integrates with systemd for readiness notification and socket activation.

## Why
Static policies can be applied directly by the CLI without a daemon. However, dynamic factories like DHCPv4 require a long-running process to manage lease lifecycles and re-reconcile when leases change. The daemon centralizes policy management so that multiple `netfyr apply` invocations and DHCP lease events are reconciled together into a single coherent system state. The daemon also enables replace-all semantics: each `netfyr apply` replaces the daemon's entire policy set, ensuring the system converges to exactly what the user specified.

## User interaction

```bash
# Start the daemon via systemd
sudo systemctl start netfyr

# Check daemon status
sudo systemctl status netfyr

# Submit policies (automatically routes through daemon)
netfyr apply /etc/netfyr/policies/

# The daemon persists policies -- they survive daemon restart
sudo systemctl restart netfyr

# Stop the daemon
sudo systemctl stop netfyr
```

## Implementation details
- Crate: `netfyr-daemon` (binary)
- Files: `src/main.rs`, `src/server.rs`, `src/factory_manager.rs`, `src/reconciler.rs`
- Dependencies (external crates): `tokio` (async runtime), `varlink` (Varlink server), `anyhow` (error handling), `tracing` (structured logging)

### Environment variables

The daemon reads these environment variables to allow overriding default paths (essential for integration tests which run in user namespaces where system paths are not writable):

| Variable | Default | Description |
|---|---|---|
| `NETFYR_SOCKET_PATH` | `/run/netfyr/netfyr.sock` | Varlink socket path |
| `NETFYR_POLICY_DIR` | `/var/lib/netfyr/policies/` | Persistent policy storage directory |

### Daemon startup flow

```rust
#[tokio::main]
async fn main() -> Result<()> {
    // 1. Initialize logging
    tracing_subscriber::init();

    // 2. Read paths from environment (with defaults for production)
    let socket_path = std::env::var("NETFYR_SOCKET_PATH")
        .unwrap_or_else(|_| "/run/netfyr/netfyr.sock".to_string());
    let policy_dir = std::env::var("NETFYR_POLICY_DIR")
        .unwrap_or_else(|_| "/var/lib/netfyr/policies/".to_string());

    // 3. Create the socket directory
    ensure_socket_dir(Path::new(&socket_path).parent().unwrap())?;

    // 4. Load persisted policies from disk
    let policy_store = PolicyStore::load(&policy_dir)?;

    // 5. Start DHCP factories for any DHCPv4 policies
    let factory_manager = FactoryManager::new();
    factory_manager.start_factories(&policy_store.policies()).await?;

    // 6. Run initial reconciliation and apply
    let reconciler = Reconciler::new();
    reconciler.reconcile_and_apply(&policy_store, &factory_manager).await?;

    // 7. Notify systemd that we're ready
    sd_notify::notify(true, &[sd_notify::NotifyState::Ready])?;

    // 8. Start Varlink server and factory event loop
    tokio::select! {
        result = serve_varlink(&socket_path, policy_store, factory_manager, reconciler) => {
            result?;
        }
        _ = tokio::signal::ctrl_c() => {
            info!("Shutting down...");
        }
    }

    // 8. Graceful shutdown: release DHCP leases
    factory_manager.stop_all().await?;

    Ok(())
}
```

### FactoryManager (`src/factory_manager.rs`)

Manages the lifecycle of DHCPv4 factories.

```rust
pub struct FactoryManager {
    factories: HashMap<PolicyId, Dhcpv4Factory>,
    event_rx: mpsc::Receiver<FactoryEvent>,
    event_tx: mpsc::Sender<FactoryEvent>,
}

impl FactoryManager {
    pub fn new() -> Self;

    /// Start factories for all DHCPv4 policies. Idempotent for already-running factories.
    pub async fn start_factories(&mut self, policies: &[Policy]) -> Result<()>;

    /// Stop factories for policies that are no longer in the set.
    pub async fn stop_removed(&mut self, current_policies: &[Policy]) -> Result<()>;

    /// Sync factories to match the current policy set:
    /// stop removed, start added, leave unchanged running.
    pub async fn sync(&mut self, policies: &[Policy]) -> Result<()>;

    /// Get the produced state from all active factories.
    pub fn produced_states(&self) -> Vec<(PolicyId, State)>;

    /// Stop all factories (for shutdown).
    pub async fn stop_all(&mut self) -> Result<()>;

    /// Receive the next factory event (lease acquired/renewed/expired).
    pub async fn next_event(&mut self) -> Option<FactoryEvent>;
}
```

### Reconciler (`src/reconciler.rs`)

Coordinates reconciliation and apply.

```rust
pub struct Reconciler {
    backend_registry: BackendRegistry,
}

impl Reconciler {
    pub fn new() -> Self;

    /// Run full reconciliation: merge static policy states with factory-produced states,
    /// query actual state, generate diff, and apply.
    pub async fn reconcile_and_apply(
        &self,
        policy_store: &PolicyStore,
        factory_manager: &FactoryManager,
    ) -> Result<ApplyResult>;

    /// Dry-run: reconcile and diff, but don't apply.
    pub async fn dry_run(
        &self,
        policy_store: &PolicyStore,
        factory_manager: &FactoryManager,
    ) -> Result<StateDiff>;
}
```

The reconciliation flow inside `reconcile_and_apply`:
1. Collect static policy states via `produce_all_static()`.
2. Collect factory-produced states from `factory_manager.produced_states()`. Note: DHCPv4 factories that haven't acquired a lease yet still produce a pending state with `operstate: up` (see SPEC-401).
3. Merge all into `Vec<PolicyInput>`.
4. Run reconciliation engine (SPEC-201) to get effective state.
5. **Compute managed entities**: build a `HashSet<EntityKey>` from the selectors of all policies in the policy store. This includes entities targeted by DHCP policies even if no lease exists yet.
6. Query actual system state via backend.
7. Generate diff via `generate_diff(desired, actual, managed_entities)` (SPEC-203). Only managed entities can be removed; unmanaged system entities are left untouched.
8. Apply diff via backend (SPEC-103).
9. Return `ApplyResult` with report.

### Event loop (`src/server.rs`)

The main event loop handles two sources of events concurrently:

```rust
async fn serve_varlink(
    socket_path: &str,
    mut policy_store: PolicyStore,
    mut factory_manager: FactoryManager,
    reconciler: Reconciler,
) -> Result<()> {
    let listener = VarlinkListener::bind(socket_path)?;

    loop {
        tokio::select! {
            // Handle Varlink requests from CLI
            Some(request) = listener.accept() => {
                match request {
                    VarlinkRequest::SubmitPolicies(policies) => {
                        // Replace all policies
                        policy_store.replace_all(policies)?;
                        // Sync factories (stop removed DHCPs, start new ones)
                        factory_manager.sync(policy_store.policies()).await?;
                        // Reconcile and apply
                        let result = reconciler.reconcile_and_apply(
                            &policy_store, &factory_manager
                        ).await?;
                        request.reply(result)?;
                    }
                    VarlinkRequest::Query(selector) => {
                        let state = reconciler.query(selector).await?;
                        request.reply(state)?;
                    }
                    VarlinkRequest::DryRun(policies) => {
                        // Temporarily compute what would happen with these policies
                        let temp_store = PolicyStore::ephemeral(policies);
                        let diff = reconciler.dry_run(
                            &temp_store, &factory_manager
                        ).await?;
                        request.reply(diff)?;
                    }
                    VarlinkRequest::GetShowInfo => {
                        let info = build_show_info(
                            &policy_store, &factory_manager, &reconciler
                        ).await?;
                        request.reply(info)?;
                    }
                }
            }

            // Handle factory events (DHCP lease changes)
            Some(event) = factory_manager.next_event() => {
                match event {
                    FactoryEvent::LeaseAcquired { .. } |
                    FactoryEvent::LeaseRenewed { .. } |
                    FactoryEvent::LeaseExpired { .. } => {
                        // Re-reconcile with updated factory state
                        let _result = reconciler.reconcile_and_apply(
                            &policy_store, &factory_manager
                        ).await?;
                    }
                    FactoryEvent::Error { policy_id, error } => {
                        warn!(%policy_id, %error, "Factory error");
                    }
                }
            }
        }
    }
}
```

### Systemd integration

**Service unit** (`netfyr.service`):
```ini
[Unit]
Description=Netfyr declarative network configuration daemon
After=network-pre.target
Before=network.target
Wants=network-pre.target

[Service]
Type=notify
ExecStart=/usr/bin/netfyr-daemon
Restart=on-failure
RuntimeDirectory=netfyr
StateDirectory=netfyr

[Install]
WantedBy=multi-user.target
```

**Varlink socket unit** (`netfyr.socket`):
```ini
[Unit]
Description=Netfyr Varlink socket

[Socket]
ListenStream=/run/netfyr/netfyr.sock
SocketMode=0666

[Install]
WantedBy=sockets.target
```

- `Type=notify`: The daemon calls `sd_notify(READY=1)` after initial reconciliation.
- `RuntimeDirectory=netfyr`: systemd creates `/run/netfyr/` for the socket.
- `StateDirectory=netfyr`: systemd creates `/var/lib/netfyr/` for persisted policies.
- Socket activation: when enabled, systemd creates the socket and starts the daemon on first connection.

### Graceful shutdown

On `SIGTERM` (systemd stop):
1. Stop accepting new Varlink connections.
2. Stop all DHCP factories, releasing leases.
3. Leave applied network configuration in place (do not deconfigure interfaces on shutdown -- the system should keep working).
4. Exit.

### Error handling

- Policy persistence failure: log error, continue operating with in-memory state. Return error to CLI.
- Factory start failure: log error, report in SubmitPolicies response. Other policies still reconciled.
- Reconciliation failure: log error, return error to CLI. Previous state remains applied.
- Backend apply failure: return partial failure report to CLI (continue-and-report).

## Depends on
- SPEC-007 (Policy types and parsing)
- SPEC-101 (Backend trait and registry)
- SPEC-201 (Reconciliation engine)
- SPEC-203 (Diff generation)
- SPEC-401 (DHCPv4 factory)
- SPEC-402 (Policy store)
- SPEC-404 (Varlink API)

## Integration test infrastructure
Integration tests are shell scripts in `tests/`. Each script uses `unshare --user --net` to create an unprivileged network namespace, starts the daemon as a subprocess, submits policies via `netfyr apply`, and verifies the kernel state with `ip` commands. No root required.

Shell test script rules (from SPEC-001):
- **Naming**: `403-description.sh` — the Makefile discovers tests via `tests/[0-9]*.sh`.
- **No skip**: if a prerequisite is missing (binary, `unshare`, `dnsmasq`), the script must `echo "FAIL: ..." >&2; exit 1`. Never `exit 0` on failure. Never wrap `netns_setup` or `start_dnsmasq` with `|| exit 0`.
- **Binary path**: locate binaries via `SCRIPT_DIR/../target/debug/netfyr` and `SCRIPT_DIR/../target/debug/netfyr-daemon` (overridable with `NETFYR_BIN` and `NETFYR_DAEMON_BIN` env vars). Check with `[[ ! -x "$NETFYR_BIN" ]]` and `exit 1` if missing.
- **Helpers**: source `tests/helpers.sh` which provides `netns_setup`, `create_veth`, `add_address`, `start_dnsmasq`, `cleanup`, and assertion functions.

## Verification
In addition to `cargo test` and `cargo clippy`, run `make integration-test SPEC=403` to execute this story's shell integration tests. All tests must pass. This is a required verification step — the story is not complete until shell tests pass.

## Acceptance criteria
```gherkin
Feature: Daemon core lifecycle
  Scenario: Daemon starts and listens on Varlink socket
    When the daemon is started
    Then it creates the Varlink socket at /run/netfyr/netfyr.sock
    And it sends sd_notify READY=1

  Scenario: Daemon loads persisted policies on startup
    Given policies were previously persisted to /var/lib/netfyr/policies/
    When the daemon starts
    Then it loads all persisted policies
    And runs initial reconciliation and apply

  Scenario: Daemon shuts down gracefully
    Given the daemon is running with DHCP factories
    When SIGTERM is received
    Then all DHCP leases are released
    And the daemon exits cleanly
    And applied network configuration is left in place

Feature: Policy submission
  Scenario: Submit policies replaces entire set
    Given the daemon is running with policies A and B
    When SubmitPolicies is called with policies C and D
    Then the daemon's policy set is now C and D only
    And policies A and B are removed from persistence
    And reconciliation runs with only C and D

  Scenario: Submit policies starts new DHCP factories
    Given the daemon is running with no policies
    When SubmitPolicies is called with a DHCPv4 policy for eth0
    Then a Dhcpv4Factory is started for eth0
    And the factory immediately produces a pending state with operstate=up
    And reconciliation runs, bringing eth0 up (so DHCP discovery can proceed)
    And once a lease is acquired, re-reconciliation applies the full DHCP state

  Scenario: Submit policies stops removed DHCP factories
    Given the daemon is running with a DHCPv4 policy for eth0
    When SubmitPolicies is called with only static policies (no DHCP)
    Then the Dhcpv4Factory for eth0 is stopped
    And the DHCP lease is released
    And reconciliation runs without DHCP state

Feature: DHCP re-reconciliation
  Scenario: Lease acquisition triggers reconciliation
    Given the daemon is running with a DHCPv4 policy for eth0
    And a static policy for eth0 with mtu=9000
    When the DHCP factory acquires a lease with IP 10.0.1.50/24
    Then reconciliation runs merging static and DHCP state
    And eth0 gets mtu=9000 (from static) and address 10.0.1.50/24 (from DHCP)

  Scenario: Lease renewal triggers reconciliation
    Given the daemon has an active DHCP lease for eth0
    When the lease is renewed (same IP)
    Then reconciliation runs
    And the system state remains unchanged

  Scenario: Lease expiry triggers reconciliation
    Given the daemon has an active DHCP lease for eth0
    When the lease expires
    Then reconciliation runs without the DHCP state
    And the DHCP-acquired address is removed from eth0

Feature: Query via daemon
  Scenario: Query returns current system state
    Given the daemon is running
    When a Query request is received with selector type=ethernet
    Then the daemon queries the backend and returns the state

  Scenario: Dry-run computes diff without applying
    Given the daemon is running with existing policies
    When a DryRun request is received with new policies
    Then the daemon computes the diff that would result
    And returns the StateDiff without applying
    And the current system state is unchanged

  Scenario: GetShowInfo returns system overview
    Given the daemon is running with 3 policies and 1 DHCP factory
    And the system has 3 network interfaces
    When a GetShowInfo request is received
    Then the response includes daemon status "running" with uptime
    And interfaces includes all 3 system interfaces
    And the DHCP-managed interface has policies and dhcp fields

Feature: Integration tests for daemon (shell scripts)
  Scenario: Daemon applies static policy in namespace
    Given a shell script running inside `unshare --user --net`
    And a veth pair "veth-test0"/"veth-test1"
    And the daemon is started as a background process inside the namespace
    When `netfyr apply policy.yaml` is run with a policy setting veth-test0 mtu=1400
    Then `ip link show veth-test0` shows mtu 1400

  Scenario: Daemon handles DHCP policy in namespace
    Given a namespace with a veth pair "veth-dhcp0"/"veth-dhcp1"
    And "veth-dhcp1" has address "10.99.0.1/24" and is link up
    And dnsmasq is running on "veth-dhcp1" serving 10.99.0.100-10.99.0.200
    And the daemon is started inside the namespace
    When `netfyr apply dhcp-policy.yaml` is run with a DHCPv4 policy for "veth-dhcp0"
    And the script waits up to 10 seconds for a lease
    Then `ip addr show veth-dhcp0` includes an address in the range 10.99.0.0/24
    And `ip link show veth-dhcp0` shows UP

  Scenario: DHCP policy does not tear down other interfaces
    Given a namespace with veth pairs "veth-dhcp0"/"veth-dhcp1" and "veth-other0"/"veth-other1"
    And "veth-other0" is link up with mtu=1400
    And dnsmasq is running on "veth-dhcp1"
    And the daemon is started inside the namespace
    When `netfyr apply dhcp-policy.yaml` is run with only a DHCPv4 policy for "veth-dhcp0"
    Then `ip link show veth-other0` still shows UP and mtu 1400 (unmanaged, untouched)
    And "veth-dhcp0" acquires a DHCP lease

  Scenario: Replace-all removes old policies in namespace
    Given the daemon is running in a namespace with a policy for veth-test0 (mtu=1400)
    When `netfyr apply new-policy.yaml` is run with veth-test0 mtu=1300
    Then `ip link show veth-test0` shows mtu 1300
```
