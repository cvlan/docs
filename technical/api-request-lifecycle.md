# API Request Lifecycle

How an HTTP request flows through cvlan-api from arrival to response.

## Request Pipeline

![Request Pipeline](../images/request-pipeline.svg)

Every request passes through this middleware stack:

```
HTTP Request
  │
  ├─ 1. Rate Limiter (per-IP, per-endpoint)
  │     └─ 429 Too Many Requests if exceeded
  │
  ├─ 2. Authentication
  │     ├─ JWT Bearer token (management API)
  │     ├─ API Key header (CLI/automation)
  │     ├─ mTLS certificate (node poll)
  │     ├─ Ed25519 token (node registration)
  │     └─ None (discovery endpoint)
  │
  ├─ 3. Tenant Extraction
  │     ├─ From JWT claims (tenant_id in token)
  │     ├─ From certificate CN (mTLS)
  │     └─ From request body (registration)
  │
  ├─ 4. Authorization
  │     ├─ Role check (superadmin/tenantadmin/member/readonly)
  │     ├─ Scope check (API key permissions)
  │     └─ 403 Forbidden if insufficient
  │
  ├─ 5. Request Validation
  │     ├─ JSON body parsing (serde)
  │     ├─ Path parameter validation
  │     └─ 400 Bad Request if invalid
  │
  ├─ 6. Handler (business logic)
  │     ├─ Database queries (SQLx, tenant-scoped)
  │     ├─ IP allocation (per-CVLAN mutex)
  │     ├─ Certificate operations (step-ca)
  │     └─ Token verification (Ed25519)
  │
  ├─ 7. Audit Log (write-after-success)
  │     └─ For all state-changing operations
  │
  └─ 8. Response
        ├─ 200/201/204 on success
        ├─ Structured JSON body
        └─ Pagination headers (for list endpoints)
```

## Rate Limiting

Per-IP limits with burst multiplier (5x):

| Endpoint | Rate | Burst |
|----------|------|-------|
| `GET /client/discover` | 100/s | 500 |
| `POST /client/register` | 10/s | 50 |
| `POST /client/poll` | 50/s | 250 |
| Management API | Varies | — |

Rate limiting uses a token bucket algorithm. Exceeding the limit returns `429 Too Many Requests` with a `Retry-After` header.

## Authentication Details

### JWT (Management API)

```
Authorization: Bearer eyJ...

Claims:
  sub: "user-uuid"        (user ID)
  iss: "cvlan"             (must be exactly "cvlan")
  role: "tenantadmin"      (no underscores)
  token_type: "access"
  scopes: []               (empty for full access)
  iat: 1707400000
  exp: 1707486400
```

Algorithm: HS256. Secret is per-environment (see JWT secrets in root CLAUDE.md).

### API Key

```
X-API-Key: cvlan_key_...
```

Looked up in database, Argon2-verified. Scoped to a specific tenant with configurable permissions.

### mTLS (Session Endpoints)

The client presents its X.509 certificate during TLS handshake. The server extracts the node ID from the certificate CN and verifies the certificate chain against the step-ca CA.

## Error Responses

All errors follow a consistent JSON format:

```json
{
  "error": "not_found",
  "message": "Node with ID abc123 not found",
  "status": 404
}
```

| Status | Meaning |
|--------|---------|
| 400 | Invalid request body or parameters |
| 401 | Missing or invalid authentication |
| 403 | Insufficient permissions for this operation |
| 404 | Resource not found (tenant-scoped) |
| 409 | Conflict (duplicate, CIDR exhausted) |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

## Pagination

List endpoints support cursor-based pagination:

```
GET /api/v1/nodes?limit=50&offset=0

Response headers:
  X-Total-Count: 1234
  X-Page-Size: 50
```

## Database Queries

All queries are tenant-scoped. A typical handler:

```rust
// Pseudo-code
async fn list_nodes(tenant_id: Uuid, params: ListParams) -> Result<Vec<Node>> {
    sqlx::query_as!(Node,
        "SELECT * FROM nodes WHERE tenant_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3",
        tenant_id, params.limit, params.offset
    )
    .fetch_all(&pool)
    .await
}
```

SQLx verifies these queries at compile time against a real PostgreSQL database, catching type mismatches and SQL errors before runtime.
