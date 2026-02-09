# cvlan-controller: Control Plane

## Overview

The control plane is the coordination hub for CloudVLAN. It handles authentication, authorization, node registration, key distribution, and policy management.

**Key principle**: The control plane coordinates but never sees plaintext traffic. It distributes public keys and policies; private keys never leave nodes.

## Responsibilities

| Function | Description |
|----------|-------------|
| Node Authentication | Verify node identity via SSO/OIDC or local credentials |
| Node Registration | Accept new nodes, assign IPs, track state |
| Key Distribution | Share public keys between authorized nodes |
| Network Map | Generate topology for each node's view of the network |
| Policy Management | Store and distribute ACLs |
| Configuration | Push settings, protocol preferences, relay assignments |

## Architecture

```
                    ┌─────────────────────────────────────┐
                    │          Load Balancer              │
                    │       (nginx / HAProxy)             │
                    └─────────────────┬───────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
              ▼                       ▼                       ▼
    ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
    │ cvlan-controller │   │ cvlan-controller │   │ cvlan-controller │
    │    (worker 1)    │   │    (worker 2)    │   │    (worker N)    │
    └────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘
             │                      │                      │
             └──────────────────────┼──────────────────────┘
                                    │
                                    ▼
                          ┌──────────────────┐
                          │    Database      │
                          │ (Postgres/SQLite)│
                          └──────────────────┘
```

## API Surface

### Node API (for cvlan-agent)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/machine/register` | POST | Register new node |
| `/api/v1/machine/key` | POST | Submit public key |
| `/api/v1/machine/map` | GET/Stream | Get network map |
| `/api/v1/machine/policy` | GET | Get applicable ACLs |

### Admin API (for cvlanctl / UI)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/admin/nodes` | GET/POST/DELETE | Manage nodes |
| `/api/v1/admin/policies` | GET/POST/PUT | Manage ACLs |
| `/api/v1/admin/users` | GET/POST/DELETE | Manage users |
| `/api/v1/admin/networks` | GET/POST | Manage networks/tenants |

## Data Model

### Node

```
Node {
    id: UUID
    name: String
    public_key_wg: Option<X25519PublicKey>
    public_key_ipsec: Option<Certificate>
    assigned_ip: IpAddr
    endpoints: Vec<Endpoint>  // Discovered external IPs/ports
    last_seen: Timestamp
    capabilities: Capabilities  // WG, IPsec, relay, etc.
    tags: Vec<String>  // For policy matching
    user_id: UUID
    created_at: Timestamp
}
```

### Network Map

```
NetworkMap {
    node_id: UUID  // Recipient
    peers: Vec<Peer>  // Nodes this node can see
    dns_config: DnsConfig
    relay_map: RelayMap
    collected_at: Timestamp
}

Peer {
    id: UUID
    name: String
    public_key: PublicKey
    allowed_ips: Vec<IpNetwork>
    endpoints: Vec<Endpoint>
    relay_home: Option<RelayId>
}
```

### Policy

```
Policy {
    id: UUID
    name: String
    rules: Vec<Rule>
    priority: u32
}

Rule {
    sources: Vec<Matcher>  // Users, groups, tags, IPs
    destinations: Vec<Matcher>
    ports: Vec<PortRange>
    action: Allow | Deny
}
```

## Authentication

### SSO/OIDC (Connected Environments)

```
┌────────┐     ┌────────────┐     ┌──────────────┐
│  Node  │────►│ Controller │────►│   IdP        │
│        │     │            │     │ (Okta, etc.) │
└────────┘     └────────────┘     └──────────────┘
    │                                    │
    │◄─────── OIDC Token ───────────────►│
    │                                    │
    └──────── Validated ─────────────────┘
```

Supported:
- Any OIDC-compliant IdP
- SAML via OIDC bridge
- Google, Okta, Azure AD, Auth0, etc.

### Local Auth (Air-Gapped)

For environments without external IdP:
- Local user database
- Pre-shared keys for nodes
- Certificate-based authentication

## Network Map Generation

The controller generates a per-node view of the network:

1. Query all nodes the requesting node can communicate with (based on ACLs)
2. Include peer public keys, endpoints, relay assignments
3. Filter based on policy—nodes only see authorized peers
4. Sign the map to prevent tampering

### Incremental Updates

Full map sync is expensive at scale. Use incremental updates:
- Track map version per node
- Send deltas (added/removed/changed peers)
- Periodic full sync as fallback

## Offline Resilience

When controller is unavailable:
- Nodes retain cached network map
- Existing tunnels continue working
- ACLs enforced from cache
- New connections may fail (no peer discovery)

Recovery:
- Nodes reconnect automatically
- Receive updated map
- Re-establish any failed tunnels

## Multi-Tenancy

For SaaS deployment:

```
Tenant {
    id: UUID
    name: String
    domain: String
    settings: TenantSettings
    subscription: SubscriptionInfo
}
```

Isolation:
- Nodes belong to exactly one tenant
- Network maps scoped to tenant
- Policies scoped to tenant
- Database: row-level or schema-per-tenant

## Scaling Considerations

| Aspect | Approach |
|--------|----------|
| API workers | Stateless, scale horizontally |
| Database | Single primary for consistency |
| Network map | Cache aggressively, incremental updates |
| WebSocket connections | Sticky sessions or shared state |

### Estimated Capacity

| Scale | Controller Instances | Database |
|-------|---------------------|----------|
| 100 nodes | 1 | SQLite |
| 1,000 nodes | 2-3 | PostgreSQL |
| 10,000 nodes | 3-5 | PostgreSQL |
| 100,000 nodes | 5-10 | PostgreSQL (tuned) |

## Implementation Notes

### Recommended Crates

- `axum` or `actix-web` for HTTP API
- `tokio-tungstenite` for WebSocket
- `sqlx` for database access
- `snow` for Noise protocol
- `jsonwebtoken` for JWT handling

### Key Design Decisions

1. **Stateless workers**: No in-memory state, all in database
2. **Optimistic locking**: Handle concurrent updates gracefully
3. **Eventual consistency**: Network maps can be slightly stale
4. **Fail open for reads**: Return cached data if database slow
