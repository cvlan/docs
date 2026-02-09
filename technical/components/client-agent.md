# cvlan-agent: Client Agent

## Overview

cvlan-agent is the software that runs on every endpoint (laptop, server, container, VM). It establishes tunnels, enforces policies, and maintains connectivity to the CloudVLAN network.

## Responsibilities

| Function | Description |
|----------|-------------|
| Control Plane Sync | Connect to controller, receive network map |
| Tunnel Management | Create/destroy WireGuard and IPsec tunnels |
| Peer Discovery | Track peer endpoints, detect changes |
| NAT Traversal | UDP hole punching, STUN |
| Relay Fallback | Route through cvlan-relay when direct fails |
| Policy Enforcement | Apply ACLs to incoming connections |
| DNS Resolution | Resolve network hostnames locally |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        cvlan-agent                          │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Control    │  │   State     │  │    Local DNS        │ │
│  │  Client     │  │   Machine   │  │    Resolver         │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
│         │                │                     │            │
│         ▼                ▼                     ▼            │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    Engine Core                          ││
│  │  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐ ││
│  │  │   WireGuard   │  │    IPsec      │  │    NAT      │ ││
│  │  │    Stack      │  │    Stack      │  │  Traversal  │ ││
│  │  └───────┬───────┘  └───────┬───────┘  └──────┬──────┘ ││
│  └──────────┼──────────────────┼─────────────────┼────────┘│
│             │                  │                 │          │
│             ▼                  ▼                 ▼          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    TUN Device                           ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    Operating System Network Stack
```

## State Machine

```
                    ┌───────────────┐
                    │  Disconnected │
                    └───────┬───────┘
                            │ start()
                            ▼
                    ┌───────────────┐
                    │ Authenticating│◄────────────┐
                    └───────┬───────┘             │
                            │ authenticated       │ auth_expired
                            ▼                     │
                    ┌───────────────┐             │
                    │   Syncing     │─────────────┘
                    └───────┬───────┘
                            │ map_received
                            ▼
                    ┌───────────────┐
              ┌────►│   Running     │◄────┐
              │     └───────┬───────┘     │
              │             │             │
    map_update│             │ error       │ reconnected
              │             ▼             │
              │     ┌───────────────┐     │
              └─────│  Reconnecting │─────┘
                    └───────────────┘
```

## Control Client

Maintains connection to cvlan-controller:

### Connection

```rust
struct ControlClient {
    controller_url: Url,
    noise_keys: NoiseKeys,
    node_key: PrivateKey,
    state: ConnectionState,
}

impl ControlClient {
    async fn connect(&mut self) -> Result<()>;
    async fn register(&self, info: MachineInfo) -> Result<Registration>;
    async fn stream_map(&self) -> Result<MapStream>;
    async fn update_endpoints(&self, endpoints: Vec<Endpoint>) -> Result<()>;
}
```

### Authentication Flow

1. Generate node keypair (first run) or load existing
2. Connect to controller via HTTPS
3. Upgrade to Noise protocol for mutual auth
4. Exchange node info and receive network map
5. Maintain long-lived connection for updates

## Tunnel Management

### WireGuard

Using boringtun (userspace) or kernel WireGuard:

```rust
struct WireGuardManager {
    device: TunDevice,
    peers: HashMap<PublicKey, WgPeer>,
}

impl WireGuardManager {
    fn add_peer(&mut self, peer: PeerConfig) -> Result<()>;
    fn remove_peer(&mut self, key: &PublicKey) -> Result<()>;
    fn update_endpoint(&mut self, key: &PublicKey, endpoint: SocketAddr) -> Result<()>;
}
```

### IPsec

Integration with system IPsec or strongSwan:

```rust
struct IpsecManager {
    connections: HashMap<String, IpsecConnection>,
}

impl IpsecManager {
    fn establish(&mut self, peer: IpsecPeerConfig) -> Result<()>;
    fn terminate(&mut self, peer_id: &str) -> Result<()>;
}
```

### Protocol Selection

Decision tree for each peer:

```
Is peer IPsec-only?
  └─ Yes → Use IPsec
  └─ No → Does local policy require IPsec?
            └─ Yes → Use IPsec
            └─ No → Use WireGuard (default)
```

## NAT Traversal

### Endpoint Discovery

1. Query STUN servers for reflexive address
2. Report discovered endpoints to controller
3. Receive peer endpoints from network map

### Connection Establishment

```
1. Try direct connection to peer's endpoints
2. If failed, attempt UDP hole punching:
   a. Both sides send probes via relay
   b. Exchange observed addresses
   c. Attempt direct connection
3. If still failed, fall back to relay
```

### Relay Fallback

When direct connection impossible:

```rust
struct RelayConnection {
    relay_addr: SocketAddr,
    peer_key: PublicKey,
    tunnel: EncryptedTunnel,
}
```

Relay sees only encrypted WireGuard/IPsec packets.

## Policy Enforcement

ACLs enforced locally on packet ingress:

```rust
struct PolicyEngine {
    rules: Vec<Rule>,
}

impl PolicyEngine {
    fn evaluate(&self, packet: &Packet) -> Action {
        for rule in &self.rules {
            if rule.matches(packet) {
                return rule.action;
            }
        }
        Action::Deny  // Default deny
    }
}
```

## Local DNS

Resolve network hostnames without external DNS:

```
┌──────────────────────────────────────────────┐
│            Local DNS Resolver                │
│                                              │
│  Query: server1.cvlan.local                  │
│                    │                         │
│                    ▼                         │
│  ┌────────────────────────────────────────┐  │
│  │  Network Map Cache                      │  │
│  │  server1 → 100.64.1.15                  │  │
│  └────────────────────────────────────────┘  │
│                    │                         │
│                    ▼                         │
│  Response: 100.64.1.15                       │
└──────────────────────────────────────────────┘
```

For external domains: forward to configured upstream DNS.

## Platform Support

| Platform | TUN | WireGuard | IPsec | Notes |
|----------|-----|-----------|-------|-------|
| Linux | Native | Kernel or userspace | strongSwan | Primary target |
| macOS | utun | Userspace | Native or strongSwan | |
| Windows | Wintun | Userspace | Native IKEv2 | |
| FreeBSD | tun | Kernel | Native | |

## CLI Interface

```bash
# Start agent (daemon mode)
cvlan-agent up

# Status
cvlan-agent status

# Show peers
cvlan-agent peers

# Show routes
cvlan-agent routes

# Disconnect
cvlan-agent down

# Debug
cvlan-agent debug ping <peer>
cvlan-agent debug trace <peer>
```

## Configuration

```toml
# /etc/cvlan/agent.toml

[controller]
url = "https://controller.example.com"
# For air-gapped: can specify multiple fallback controllers

[auth]
# SSO login (interactive)
method = "oidc"

# Or pre-shared key (non-interactive)
# method = "key"
# key_file = "/etc/cvlan/node.key"

[network]
# Prefer WireGuard, fall back to IPsec
preferred_protocol = "wireguard"

# Accept incoming connections on these ports
listen_port = 41641

[dns]
# Enable local DNS resolver
enabled = true
# Magic DNS suffix
suffix = "cvlan.local"

[logging]
level = "info"
```

## Resource Usage

Target footprint:

| Metric | Target |
|--------|--------|
| Memory (idle) | < 20 MB |
| Memory (1k peers) | < 50 MB |
| CPU (idle) | < 1% |
| Binary size | < 10 MB |
