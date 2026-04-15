# SPEC-401: DHCPv4 Factory

## What
Implement the DHCPv4 factory in the `netfyr-backend` crate. This factory runs a DHCPv4 client on the interface identified by the policy's selector, acquires a lease, and produces an `State` containing the lease-acquired configuration (addresses, routes, DNS servers). On lease renewal or change, the factory notifies the daemon to trigger re-reconciliation. The DHCPv4 factory only runs inside the daemon -- the CLI cannot run DHCP directly.

## Why
DHCP is the most common dynamic network configuration method. Many environments require a mix of static and dynamic configuration: for example, a server might have a statically configured management interface alongside a DHCP-configured production interface. The DHCPv4 factory allows netfyr to manage both through the same policy model. By modeling DHCP as a factory that produces state, DHCP-acquired configuration participates in the same reconciliation and conflict resolution as static configuration, enabling predictable field-level merging between static and dynamic sources.

## User interaction
Users define a DHCPv4 policy in YAML:

```yaml
kind: policy
name: eth0-dhcp
factory: dhcpv4
selector:
  name: eth0
```

This is the complete configuration -- zero additional options. The factory acquires a lease and produces state automatically. The policy is submitted to the daemon via `netfyr apply`:

```bash
# Submit DHCP policy (daemon must be running)
netfyr apply /etc/netfyr/policies/eth0-dhcp.yaml

# Verify acquired state
netfyr query ethernet --selector name=eth0
```

If the daemon is not running, `netfyr apply` fails with a clear error:
```
Error: policy "eth0-dhcp" uses factory "dhcpv4" which requires the netfyr daemon.
Start the daemon with: systemctl start netfyr
```

## Implementation details
- Crate: `netfyr-backend`
- Files: `src/dhcp/mod.rs`, `src/dhcp/client.rs`, `src/dhcp/lease.rs`
- Dependencies (external crates): `dhcpv4` or raw socket implementation, `tokio` (async runtime)

### Factory interface

```rust
pub struct Dhcpv4Factory {
    interface: String,
    lease: Option<DhcpLease>,
    state_tx: mpsc::Sender<FactoryEvent>,
}

pub enum FactoryEvent {
    LeaseAcquired { policy_id: PolicyId, state: State },
    LeaseRenewed { policy_id: PolicyId, state: State },
    LeaseExpired { policy_id: PolicyId },
    Error { policy_id: PolicyId, error: String },
}

impl Dhcpv4Factory {
    /// Start the DHCP client on the given interface.
    /// Returns immediately; lease acquisition runs asynchronously.
    /// Sends FactoryEvent messages via state_tx when lease state changes.
    pub async fn start(
        interface: &str,
        policy_id: PolicyId,
        state_tx: mpsc::Sender<FactoryEvent>,
    ) -> Result<Self>;

    /// Stop the DHCP client and release the lease.
    pub async fn stop(&mut self) -> Result<()>;

    /// Get the current lease state, if any.
    pub fn current_state(&self) -> Option<&State>;
}
```

### Lease lifecycle

1. **Discover**: Send DHCPDISCOVER broadcast on the interface.
2. **Offer**: Receive DHCPOFFER from a server.
3. **Request**: Send DHCPREQUEST for the offered address.
4. **Acknowledge**: Receive DHCPACK. Lease is now active.
5. **Renew** (at T1, typically 50% of lease time): Send unicast DHCPREQUEST to the server.
6. **Rebind** (at T2, typically 87.5% of lease time): Send broadcast DHCPREQUEST.
7. **Expire**: If no response by lease end, lease expires. Send LeaseExpired event.
8. **Release**: On stop(), send DHCPRELEASE to the server.

### Produced State

When a lease is acquired, the factory produces an `State` with:

```rust
fn lease_to_state(lease: &DhcpLease, interface: &str) -> State {
    let mut fields = BTreeMap::new();

    // IP address from the lease
    fields.insert(
        FieldName::from("addresses"),
        FieldValue::List(vec![
            FieldValue::String(format!("{}/{}", lease.ip, lease.subnet_mask_to_prefix())),
        ]),
    );

    // Default gateway (option 3)
    if let Some(gateway) = &lease.gateway {
        fields.insert(
            FieldName::from("routes"),
            FieldValue::List(vec![
                FieldValue::Map(btreemap! {
                    "destination".into() => FieldValue::String("0.0.0.0/0".into()),
                    "gateway".into() => FieldValue::String(gateway.to_string()),
                }),
            ]),
        );
    }

    // DNS servers (option 6) -- stored for reconciliation with dns entity
    if !lease.dns_servers.is_empty() {
        fields.insert(
            FieldName::from("dns_servers"),
            FieldValue::List(
                lease.dns_servers.iter()
                    .map(|s| FieldValue::String(s.to_string()))
                    .collect()
            ),
        );
    }

    State {
        entity_type: EntityType::Ethernet,
        selector: Selector { name: Some(interface.to_string()), ..Default::default() },
        fields,
    }
}
```

### DhcpLease type

```rust
pub struct DhcpLease {
    pub ip: Ipv4Addr,
    pub subnet_mask: Ipv4Addr,
    pub gateway: Option<Ipv4Addr>,
    pub dns_servers: Vec<Ipv4Addr>,
    pub lease_time: u32,         // seconds
    pub renewal_time: u32,       // T1, seconds
    pub rebind_time: u32,        // T2, seconds
    pub server_id: Ipv4Addr,
    pub acquired_at: Instant,
}

impl DhcpLease {
    pub fn subnet_mask_to_prefix(&self) -> u8;
    pub fn is_expired(&self) -> bool;
    pub fn time_until_renewal(&self) -> Duration;
    pub fn time_until_rebind(&self) -> Duration;
}
```

### Interaction with daemon

The daemon (SPEC-403) manages factory lifecycles:
1. When a DHCPv4 policy is submitted, the daemon starts a `Dhcpv4Factory` for that policy.
2. The factory sends `FactoryEvent` messages on a channel the daemon monitors.
3. On `LeaseAcquired` or `LeaseRenewed`, the daemon updates the policy's produced state and triggers re-reconciliation.
4. On `LeaseExpired`, the daemon removes the policy's produced state and triggers re-reconciliation.
5. When a policy is removed (due to replace-all), the daemon calls `factory.stop()` to release the lease.

### Error handling

- Interface not found: `FactoryEvent::Error` with "interface not found" message.
- No DHCP server responds (timeout): `FactoryEvent::Error` with "DHCP discovery timeout" message. Factory retries with exponential backoff.
- Lease acquisition failure: `FactoryEvent::Error`. Factory retries.
- Network errors during renewal: Fall back to rebind. If rebind fails, lease expires normally.

## Depends on
- SPEC-002 (State, FieldValue types)
- SPEC-007 (Policy types -- DHCPv4 factory type)
- SPEC-101 (Backend trait)

## Integration test infrastructure
Integration tests use `netfyr-test-utils` (SPEC-001) to create unprivileged user + network namespaces with a veth pair. One end runs a dnsmasq DHCP server, the other runs the DHCPv4 factory. No root required.

## Acceptance criteria
```gherkin
Feature: DHCPv4 factory
  Scenario: Factory acquires a DHCP lease
    Given a Dhcpv4Factory started on interface "eth0"
    And a DHCP server is running on the network
    When the factory completes the DHCP handshake
    Then a LeaseAcquired event is sent
    And the produced State contains the leased IP address
    And the produced State contains the default gateway route

  Scenario: Lease produces correct State fields
    Given a DHCP lease with IP 10.0.1.50, mask 255.255.255.0, gateway 10.0.1.1, DNS 10.0.1.2
    When lease_to_state is called
    Then the State has addresses=["10.0.1.50/24"]
    And the State has routes with destination="0.0.0.0/0" gateway="10.0.1.1"
    And the State has dns_servers=["10.0.1.2"]

  Scenario: Factory sends LeaseRenewed on renewal
    Given a factory with an active lease
    When the lease renewal timer fires and renewal succeeds
    Then a LeaseRenewed event is sent with the updated state

  Scenario: Factory sends LeaseExpired when lease expires
    Given a factory with an active lease
    When the lease expires without successful renewal or rebind
    Then a LeaseExpired event is sent

  Scenario: Factory releases lease on stop
    Given a factory with an active lease
    When stop() is called
    Then a DHCPRELEASE is sent to the server
    And the factory stops cleanly

  Scenario: Factory retries on discovery timeout
    Given a factory started on an interface with no DHCP server
    When the discovery timeout elapses
    Then the factory retries with exponential backoff
    And a FactoryEvent::Error is sent

  Scenario: current_state returns None before lease
    Given a newly started factory
    When current_state() is called before any lease is acquired
    Then it returns None

  Scenario: current_state returns state after lease
    Given a factory that has acquired a lease
    When current_state() is called
    Then it returns Some(State) with the lease data

  Scenario: DhcpLease expiry detection
    Given a DhcpLease with lease_time=3600 acquired 3601 seconds ago
    When is_expired() is called
    Then it returns true

  Scenario: Subnet mask to prefix conversion
    Given a DhcpLease with subnet_mask=255.255.255.0
    When subnet_mask_to_prefix() is called
    Then it returns 24

Feature: Integration tests for DHCPv4 factory (unprivileged netns)
  Scenario: Acquire DHCP lease in unprivileged namespace
    Given an unprivileged user + network namespace with a veth pair "veth-dhcp0"/"veth-dhcp1"
    And "veth-dhcp1" has address "10.99.0.1/24" and is link up
    And dnsmasq is running on "veth-dhcp1" serving range 10.99.0.100-10.99.0.200
    And "veth-dhcp0" is link up with no addresses
    When a Dhcpv4Factory is started on "veth-dhcp0"
    Then a LeaseAcquired event is received within 10 seconds
    And the leased IP is in the range 10.99.0.100-10.99.0.200
    And the gateway is 10.99.0.1

  Scenario: Lease renewal in namespace
    Given an active DHCP lease in an unprivileged namespace
    And the DHCP server is configured with a short lease time (e.g., 30 seconds)
    When the renewal timer fires
    Then a LeaseRenewed event is received
    And the lease IP is unchanged (or updated if server changed it)

  Scenario: Factory stop releases lease in namespace
    Given an active DHCP lease in an unprivileged namespace
    When stop() is called on the factory
    Then the factory exits cleanly
    And the DHCP server's lease table shows the lease as released
```
