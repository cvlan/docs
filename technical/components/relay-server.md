# cvlan-relay: Relay Server

## Overview

cvlan-relay provides packet relay for nodes that cannot establish direct connections due to NAT or firewall restrictions. It's the fallback path when UDP hole punching fails.

**First implementation target**: This is the smallest, most self-contained component—ideal for proving out the Rust stack.

## Responsibilities

| Function | Description |
|----------|-------------|
| Packet Relay | Forward encrypted packets between nodes |
| Home Registration | Track which nodes consider this relay "home" |
| Health Reporting | Report status to controller |
| Regional Mesh | Forward to other relays in region |

## Design Principles

1. **Minimal**: Do one thing well—forward packets
2. **Untrusted**: Cannot decrypt traffic, only forwards ciphertext
3. **Stateless-ish**: Connection state only, no persistent storage
4. **Lightweight**: Run on smallest VPS instances

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     cvlan-relay                         │
│                                                         │
│  ┌─────────────────┐      ┌─────────────────────────┐  │
│  │  Client Handler │      │    Relay Mesh Handler   │  │
│  │                 │      │    (inter-relay)        │  │
│  └────────┬────────┘      └────────────┬────────────┘  │
│           │                            │               │
│           ▼                            ▼               │
│  ┌──────────────────────────────────────────────────┐  │
│  │              Packet Router                        │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────────┐ │  │
│  │  │           Client Registry                    │ │  │
│  │  │   PublicKey → ClientConnection              │ │  │
│  │  └─────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Protocol

### Client Connection

Clients connect via:
- TCP with TLS (HTTPS upgrade, firewall-friendly)
- Or raw TCP on dedicated port
- Or QUIC (future)

### Frame Format

```
┌─────────────────────────────────────────────────────┐
│  Frame Type (1 byte)                                │
├─────────────────────────────────────────────────────┤
│  Destination Key (32 bytes) - for SendPacket        │
├─────────────────────────────────────────────────────┤
│  Payload Length (2 bytes, big-endian)               │
├─────────────────────────────────────────────────────┤
│  Payload (variable)                                 │
└─────────────────────────────────────────────────────┘
```

### Frame Types

| Type | Value | Description |
|------|-------|-------------|
| ClientInfo | 0x01 | Client announces its public key |
| SendPacket | 0x02 | Forward packet to destination key |
| RecvPacket | 0x03 | Packet received from peer |
| KeepAlive | 0x04 | Connection keepalive |
| PeerGone | 0x05 | Notify that a peer disconnected |

### Connection Flow

```
Client                          Relay
   │                              │
   │──── TLS Handshake ──────────►│
   │                              │
   │──── ClientInfo(pubkey) ─────►│
   │                              │  Register client
   │◄─── ServerInfo ──────────────│
   │                              │
   │──── SendPacket(dest, data) ─►│
   │                              │  Lookup dest
   │                              │  Forward to dest
   │                              │
   │◄─── RecvPacket(src, data) ───│  From another client
   │                              │
```

## Implementation

### Core Structures

```rust
use std::collections::HashMap;
use tokio::sync::mpsc;

type PublicKey = [u8; 32];

struct RelayServer {
    clients: HashMap<PublicKey, ClientHandle>,
    mesh_peers: Vec<MeshPeer>,
    config: RelayConfig,
}

struct ClientHandle {
    key: PublicKey,
    sender: mpsc::Sender<Frame>,
    connected_at: Instant,
    bytes_relayed: AtomicU64,
}

struct Frame {
    frame_type: FrameType,
    dest_key: Option<PublicKey>,
    payload: Bytes,
}
```

### Packet Routing

```rust
impl RelayServer {
    async fn route_packet(&self, from: PublicKey, to: PublicKey, data: Bytes) {
        // Try local client first
        if let Some(client) = self.clients.get(&to) {
            let frame = Frame::recv_packet(from, data);
            let _ = client.sender.send(frame).await;
            return;
        }

        // Try mesh peers (other relays in region)
        for peer in &self.mesh_peers {
            if peer.has_client(&to) {
                peer.forward(from, to, data).await;
                return;
            }
        }

        // Client not found - drop packet
        // (Client will retry or fall back)
    }
}
```

## Regional Mesh

Multiple relays in a region mesh together for redundancy:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  relay-1    │◄───►│  relay-2    │◄───►│  relay-3    │
│  (region-us)│     │  (region-us)│     │  (region-us)│
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
              Clients choose one as "home"
              Others can forward if needed
```

### Mesh Protocol

Relays discover each other via:
- Configuration (static list)
- Or controller provides relay map

Inter-relay auth via pre-shared key or mTLS.

## Configuration

```toml
# /etc/cvlan/relay.toml

[server]
listen_addr = "0.0.0.0:443"
listen_addr_stun = "0.0.0.0:3478"

[tls]
cert_file = "/etc/cvlan/relay.crt"
key_file = "/etc/cvlan/relay.key"

[mesh]
# Other relays in this region
peers = [
    "relay-2.example.com:443",
    "relay-3.example.com:443",
]
mesh_key = "base64-encoded-psk"

[controller]
# Report health to controller
url = "https://controller.example.com"
api_key = "relay-api-key"

[limits]
max_clients = 10000
max_packet_size = 65535
client_timeout_secs = 60
```

## Resource Usage

Relays are designed to be extremely lightweight:

| Metric | Target | Notes |
|--------|--------|-------|
| Memory per client | ~1 KB | Connection state only |
| Memory (1k clients) | ~10 MB | Plus base overhead |
| Memory (10k clients) | ~50 MB | |
| CPU | Minimal | Just copying bytes |
| Bandwidth | Pass-through | No amplification |

### Cost Estimate

| VPS Size | Capacity | Monthly Cost |
|----------|----------|--------------|
| 512 MB RAM | ~2,000 clients | ~$5 |
| 1 GB RAM | ~5,000 clients | ~$10 |
| 2 GB RAM | ~15,000 clients | ~$20 |

## Deployment

### Single Binary

```bash
# Run relay
cvlan-relay --config /etc/cvlan/relay.toml

# Or with env vars
CVLAN_LISTEN_ADDR=0.0.0.0:443 cvlan-relay
```

### Container

```dockerfile
FROM scratch
COPY cvlan-relay /cvlan-relay
ENTRYPOINT ["/cvlan-relay"]
```

### Health Check

```
GET /health

{
  "status": "healthy",
  "clients": 1234,
  "uptime_secs": 86400,
  "bytes_relayed": 1234567890
}
```

## Monitoring

Expose Prometheus metrics:

```
# HELP cvlan_relay_clients_connected Current connected clients
cvlan_relay_clients_connected 1234

# HELP cvlan_relay_bytes_relayed_total Total bytes relayed
cvlan_relay_bytes_relayed_total 1234567890

# HELP cvlan_relay_packets_relayed_total Total packets relayed
cvlan_relay_packets_relayed_total 98765432

# HELP cvlan_relay_packets_dropped_total Packets dropped (dest not found)
cvlan_relay_packets_dropped_total 123
```

## Security Considerations

| Concern | Mitigation |
|---------|------------|
| DDoS amplification | No amplification—1:1 relay only |
| Unauthorized clients | Optional: require auth token from controller |
| Traffic analysis | Can see metadata (who talks to who), not content |
| Resource exhaustion | Connection limits, timeouts |

## Implementation Priority

This is the **first component to implement** because:

1. Self-contained—no dependencies on other cvlan components
2. Simple—just packet forwarding
3. Proves the Rust async networking stack
4. Useful standalone for testing
5. Can be tested with existing WireGuard clients
