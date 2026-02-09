# CloudVLAN Architecture

## Overview

CloudVLAN follows a control plane / data plane separation:

- **Control Plane**: Centralized coordination (auth, policy, node management)
- **Data Plane**: Distributed mesh (direct node-to-node or via relay)

```
┌─────────────────────────────────────────────────────────────────┐
│                      CONTROL PLANE                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  cvlan-controller                        │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │    │
│  │  │   Auth   │ │  Policy  │ │   Node   │ │  Network   │  │    │
│  │  │  Engine  │ │  Engine  │ │ Registry │ │ Map Gen    │  │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                   Control Protocol (HTTPS + Noise)
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  cvlan-agent  │   │  cvlan-agent  │   │ cvlan-vrouter │
│   (laptop)    │   │   (server)    │   │  (gateway)    │
│               │   │               │   │               │
│ ┌───────────┐ │   │ ┌───────────┐ │   │ ┌───────────┐ │
│ │ WireGuard │ │   │ │ WireGuard │ │   │ │  IPsec    │ │
│ │   Stack   │ │   │ │   Stack   │ │   │ │Concentratr│ │
│ └───────────┘ │   │ └───────────┘ │   │ └───────────┘ │
│ ┌───────────┐ │   │ ┌───────────┐ │   │ ┌───────────┐ │
│ │  IPsec    │ │   │ │  IPsec    │ │   │ │ WireGuard │ │
│ │  Stack    │ │   │ │  Stack    │ │   │ │   Stack   │ │
│ └───────────┘ │   │ └───────────┘ │   │ └───────────┘ │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                    DATA PLANE (Mesh)
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
     Direct Connection            Via cvlan-relay
     (UDP hole punch)             (NAT fallback)
```

## Control Plane

### cvlan-controller

Central coordination server responsible for:

| Function | Description |
|----------|-------------|
| **Authentication** | Validate node identity via SSO/OIDC or local auth |
| **Authorization** | Determine what nodes can join, what they can access |
| **Node Registry** | Track all nodes, their keys, endpoints, capabilities |
| **Network Map** | Generate and distribute topology to all nodes |
| **Policy Distribution** | Push ACLs to nodes for local enforcement |
| **Key Coordination** | Distribute public keys (not private—those never leave nodes) |

**Characteristics:**
- Stateless workers (horizontal scaling)
- State in PostgreSQL or SQLite
- Minimal traffic (keys + config only)
- Can be offline temporarily—nodes cache state

### Control Protocol

Nodes communicate with controller via:
- HTTPS for API calls
- Noise protocol for authenticated, encrypted streaming updates
- Long-polling or WebSocket for real-time config updates

## Data Plane

### cvlan-agent

Client software on every endpoint:

| Function | Description |
|----------|-------------|
| **Tunnel Management** | Establish WireGuard or IPsec tunnels |
| **Peer Discovery** | Learn about other nodes from controller |
| **NAT Traversal** | UDP hole punching, STUN |
| **Relay Fallback** | Use cvlan-relay when direct fails |
| **Policy Enforcement** | Apply ACLs locally |
| **DNS** | Local resolver for network hostnames |

**Protocol Selection:**
- Default: WireGuard (faster, simpler)
- Fallback/Compliance: IPsec/IKEv2
- Per-connection or per-node policy

### cvlan-relay

Fallback relay servers for NAT traversal:

| Function | Description |
|----------|-------------|
| **Packet Forwarding** | Relay encrypted packets between nodes |
| **Home Selection** | Nodes choose nearest relay |
| **Mesh Within Region** | Relays mesh for redundancy |

**Characteristics:**
- Sees only encrypted traffic (can't decrypt)
- Lightweight (Rust, minimal memory)
- Geographically distributed
- Can be self-hosted for air-gapped

### cvlan-vrouter

Managed IPsec concentrator:

| Function | Description |
|----------|-------------|
| **IPsec Gateway** | Terminate IPsec tunnels from legacy devices |
| **Protocol Bridge** | Connect IPsec devices to WireGuard mesh |
| **Multi-tenant** | Isolated tunnels per tenant |
| **Cloud-Portable** | Deploy in any cloud or colo |

### VRouter Registration Flow

VRouters use a join token + mTLS certificate flow for secure registration:

```
Admin                    cvlan-api                 step-ca              VRouter
  │                          │                        │                    │
  │ 1. Create vrouter        │                        │                    │
  ├─────────────────────────►│                        │                    │
  │                          │                        │                    │
  │ 2. Get join token        │                        │                    │
  ├─────────────────────────►│                        │                    │
  │◄─────────────────────────┤                        │                    │
  │                          │                        │                    │
  │── 3. Configure vrouter (out of band) ─────────────────────────────────►│
  │                          │                        │                    │
  │                          │    4. Register with token                   │
  │                          │◄────────────────────────────────────────────┤
  │                          │                        │                    │
  │                          │ 5. Mint cert ─────────►│                    │
  │                          │◄───────────────────────┤                    │
  │                          │                        │                    │
  │                          │ 6. Return cert + config ───────────────────►│
  │                          │                        │                    │
  │                          │    7. Poll with mTLS   │                    │
  │                          │◄────────────────────────────────────────────┤
```

**Key Points:**
- Join tokens are single-use, short-lived (1 hour default)
- mTLS certificates are valid for 8 hours, auto-renewed
- VRouter identity established via certificate CN (vrouter_id)
- step-ca provides SPIFFE-compatible certificate minting

## Type System

CloudVLAN uses Protocol Buffers as the canonical source for type definitions, enabling consistent types across all components:

```
┌─────────────────────────────────────────────────────────────────┐
│                    protodefs/ (Single Source of Truth)          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  cloudvlan/v1/*.proto                                   │    │
│  │  - common.proto (Status, Capacity, Pagination)          │    │
│  │  - tenant.proto, cvlan.proto, node.proto, etc.          │    │
│  └─────────────────────────────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │ buf generate
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────┐
       │  Rust    │   │TypeScript│   │ (future) │
       │  (prost) │   │(ts-proto)│   │  OpenAPI │
       └──────────┘   └──────────┘   └──────────┘
              │              │
              ▼              ▼
       ┌──────────┐   ┌──────────┐
       │ cvlan-api│   │    ui    │
       │ vrouterd │   │  (React) │
       └──────────┘   └──────────┘
```

**Benefits:**
- Single source of truth eliminates type drift
- Breaking change detection via `buf breaking`
- Consistent serialization across Rust and TypeScript
- Proto enums map to i32 in Rust (efficient, exhaustive matching)

**Code Generation:**
```bash
# Generate all types (run in cvlan-build container)
docker compose -f build/docker-compose.yml run --rm -w /app/protodefs dev make generate

# Sync TypeScript to UI repo
make sync-ui
```

**Proto Structure:**
| File | Contents |
|------|----------|
| `common.proto` | Status, Capacity, Pagination, ListMeta |
| `tenant.proto` | Tenant, TenantStats, requests |
| `cvlan.proto` | Cvlan, CvlanStats, requests |
| `node.proto` | Node, NodeStatus, NodeRoute, NodeEndpoint |
| `user.proto` | User, UserRole, UserStatus |
| `group.proto` | Group, GroupMember, GroupWithCount |
| `policy.proto` | Policy, PolicyRule, PolicyTarget, PolicyAction |
| `vrouter.proto` | VRouter, VRouterStats, VRouterConnection, join token types |
| `region.proto` | Region, requests |
| `relay.proto` | Relay, RelayStats |
| `api_key.proto` | ApiKey, ApiKeyStatus, ApiScope |
| `audit_log.proto` | AuditLog, ActorType, ExportFormat |
| `tag.proto` | Tag, TagSummary, TagValues |
| `discovery_server.proto` | DiscoveryServer |

## Certificate Infrastructure

CloudVLAN uses step-ca as the Certificate Authority for mTLS authentication:

```
┌─────────────────────────────────────────────────────────────────┐
│                       step-ca (CA Service)                       │
│  - Mints mTLS certificates for vrouters                         │
│  - Short-lived certs (8 hours) for security                     │
│  - SPIFFE-compatible for future workload identity               │
└──────────────────────────────┬──────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
       ┌──────────┐     ┌──────────┐     ┌──────────┐
       │ vrouter  │     │ vrouter  │     │  (future │
       │  cert    │     │  cert    │     │  agents) │
       └──────────┘     └──────────┘     └──────────┘
```

**Deployment:**
- step-ca runs as a separate container/service
- cvlan-api connects to step-ca to request certificates
- Production: Consider SPIRE for full workload attestation

## Network Model

### Addressing

- Private IP range for overlay (e.g., 100.64.0.0/10 like Tailscale, or configurable)
- Each node gets stable IP within the overlay
- Controller assigns IPs, tracks allocations

### Connectivity

```
Node A ◄──────────────────────────────► Node B
         │                         │
         │  1. Try direct UDP      │
         │  2. Try UDP hole punch  │
         │  3. Fall back to relay  │
         │                         │
         └─────── cvlan-relay ─────┘
```

### Protocol Negotiation

```
┌─────────┐         ┌─────────┐
│ Node A  │         │ Node B  │
│ (WG+IPsec)        │ (WG only)│
└────┬────┘         └────┬────┘
     │                   │
     │◄─ Capabilities ──►│
     │                   │
     │   Negotiate: WG   │
     │◄═════════════════►│
     │   (common protocol)│
```

## Security Model

### Trust Boundaries

| Boundary | Trust Level |
|----------|-------------|
| Controller | Trusted (but can't decrypt traffic) |
| Relay | Untrusted (sees only ciphertext) |
| Nodes | Trust established via controller |

### Key Management

- **Private keys**: Generated on node, never leave node
- **Public keys**: Distributed via controller
- **Session keys**: Negotiated per-tunnel (WireGuard/IPsec handles)

### Policy Enforcement

- Policies defined centrally in controller
- Distributed to all nodes
- Enforced locally on each node (deny-by-default)
- Controller unavailability doesn't break existing connections

## Scalability

### Control Plane Scaling

| Component | Strategy |
|-----------|----------|
| Controller API | Stateless, horizontal scale behind LB |
| Database | Single primary (Postgres) or embedded (SQLite) |
| Network Map | Incremental updates, not full sync |

### Data Plane Scaling

| Component | Strategy |
|-----------|----------|
| Relays | Add more relays, geographic distribution |
| Agents | No central bottleneck—mesh is distributed |
| vRouters | Scale per tenant, isolated |

### Target Scale

- 100,000 nodes
- < $2,000/month infrastructure
- Sub-second network map updates
