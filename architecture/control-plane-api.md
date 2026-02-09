# Control Plane API (cvlan-api)

cvlan-api is the central server. It handles tenant management, node registration, policy enforcement, IP allocation, audit logging, and the background job scheduler. Everything goes through its REST API.

## Request Pipeline

![Request Pipeline](../images/request-pipeline.svg)

Every HTTP request passes through:

1. **Rate limiter** — Per-IP limits (configurable per endpoint)
2. **Authentication** — JWT bearer token, API key, or mTLS certificate depending on endpoint
3. **Tenant extraction** — From JWT claims or certificate CN
4. **Authorization** — Role-based: superadmin > tenantadmin > member > readonly
5. **Handler** — Business logic
6. **Database** — PostgreSQL (or SQLite for single-box)
7. **Audit log** — Write-after-success for state-changing operations
8. **Response** — JSON with appropriate status codes

## Server Modes

cvlan-api supports running in different modes, each exposing a subset of endpoints:

```
--mode=combined      # All endpoints (default, development)
--mode=controller    # Full REST API (admin, JWT/API-key auth)
--mode=discovery     # GET /client/discover only (public, rate-limited)
--mode=registration  # POST /client/register only (token-verified)
--mode=session       # POST /client/poll only (mTLS required)
```

In production, you'd run separate instances behind different load balancers:
- Discovery on port 443 (public internet)
- Registration behind a WAF
- Session on an internal network (mTLS only)
- Controller behind VPN/bastion

## Multi-Tenant RBAC

Four roles with hierarchical permissions:

| Role | Scope | Can Manage |
|------|-------|------------|
| **superadmin** | Global | Tenants, regions, relays, everything |
| **tenantadmin** | Tenant | Users, groups, CVLANs, nodes, policies, API keys, audit logs |
| **member** | Tenant | CVLANs, nodes, policies |
| **readonly** | Tenant | View only |

All queries are tenant-scoped. A tenantadmin in Tenant A cannot see or modify anything in Tenant B. Superadmin can operate across tenants.

## IP Allocation

![IP Allocation Flow](../images/ip-allocation-flow.svg)

Each CVLAN has its own CIDR and a per-CVLAN mutex allocator:

- **Fast Path 1** (O(1)): Reuse a gap from a deleted node (`first_free` pointer)
- **Fast Path 2** (O(1)): Increment the high-water mark (`next_offset`)
- **Slow Path** (O(n)): Full gap scan when watermark exhausted

This achieves 21,000 ops/sec at 200 concurrency. Reserved IPs (first + last usable per CIDR) are automatically skipped.

CIDR auto-allocation follows RFC 1918 priority: 10.0.0.0/8 → 172.16.0.0/12 → 192.168.0.0/16.

## Background Job Scheduler

Database-backed scheduler for periodic tasks:

- **Audit log cleanup** — Purge entries older than 90 days (configurable)
- **Node health checks** — Mark nodes as offline if not seen within timeout
- **Certificate expiry warnings** — Alert before mTLS certs expire

Jobs are stored in PostgreSQL with leader election (single-writer pattern). Configurable intervals, retry logic, and dead-letter handling.

## Audit Logging

Every state-changing operation creates an audit log entry:

```json
{
  "id": 12345,
  "tenant_id": "uuid",
  "user_id": "uuid",
  "action": "node.create",
  "resource_type": "node",
  "resource_id": "uuid",
  "details": { "name": "laptop-01", "cvlan": "prod-network" },
  "created_at": "2026-02-09T10:00:00Z"
}
```

Retention: 90 days by default (configurable). Export via API for compliance archival.

## Key Endpoints

### Management API (`/api/v1/`)
- Tenants: CRUD + signing key management
- CVLANs: CRUD + CIDR allocation + IP state
- Nodes: CRUD + bulk operations
- Users: CRUD + role assignment
- Groups: CRUD + membership
- Policies: CRUD + group-based ACLs
- VRouters: CRUD + region assignment + stats
- Regions: CRUD
- Relays: CRUD
- API Keys: Create/revoke + scoped permissions
- Audit Logs: List/filter/export

### Node Coordination
- `GET /client/discover?tenant=<slug>` — Find controller URL (public)
- `POST /client/register` — Register with Ed25519 token (returns cert + IP + peers)
- `POST /client/poll` — Delta updates (mTLS, returns changed peers/policies)

### VRouter Coordination
- `POST /vrouter/register` — Register with join token (returns cert + config)
- `POST /vrouter/poll` — Poll for config changes + report stats (mTLS)

## Database

Primary: PostgreSQL (SQLx 0.8 with compile-time query verification).
Alternative: SQLite for single-box self-hosted deployments.

Both backends share the same schema (with SQLite-specific adaptations for UUID storage as BLOB and timestamp handling).
