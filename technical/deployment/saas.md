# SaaS Deployment

## Overview

CloudVLAN SaaS is the managed offering where we operate the control plane and relay infrastructure. Customers get the simplicity of Tailscale with the protocol flexibility of CloudVLAN.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CloudVLAN SaaS Infrastructure                    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                      Control Plane                              │ │
│  │                                                                 │ │
│  │    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │ │
│  │    │ API GW  │  │ API GW  │  │ API GW  │  │ API GW  │         │ │
│  │    └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘         │ │
│  │         └────────────┼────────────┼────────────┘              │ │
│  │                      │            │                            │ │
│  │    ┌─────────────────┼────────────┼─────────────────────┐     │ │
│  │    │                 ▼            ▼                     │     │ │
│  │    │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │     │ │
│  │    │  │controller│ │controller│ │controller│           │     │ │
│  │    │  │    1     │ │    2     │ │    N     │           │     │ │
│  │    │  └──────────┘ └──────────┘ └──────────┘           │     │ │
│  │    │                      │                             │     │ │
│  │    │                      ▼                             │     │ │
│  │    │             ┌──────────────┐                       │     │ │
│  │    │             │  PostgreSQL  │                       │     │ │
│  │    │             │  (Primary)   │                       │     │ │
│  │    │             └──────────────┘                       │     │ │
│  │    └────────────────────────────────────────────────────┘     │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    Global Relay Network                         │ │
│  │                                                                 │ │
│  │   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐       │ │
│  │   │ Region  │   │ Region  │   │ Region  │   │ Region  │       │ │
│  │   │  US-E   │   │  US-W   │   │   EU    │   │  APAC   │       │ │
│  │   │ ┌─────┐ │   │ ┌─────┐ │   │ ┌─────┐ │   │ ┌─────┐ │       │ │
│  │   │ │relay│ │   │ │relay│ │   │ │relay│ │   │ │relay│ │       │ │
│  │   │ │relay│ │   │ │relay│ │   │ │relay│ │   │ │relay│ │       │ │
│  │   │ └─────┘ │   │ └─────┘ │   │ └─────┘ │   │ └─────┘ │       │ │
│  │   └─────────┘   └─────────┘   └─────────┘   └─────────┘       │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ Internet
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
   ┌────────────┐          ┌────────────┐          ┌────────────┐
   │  Tenant A  │          │  Tenant B  │          │  Tenant C  │
   │   nodes    │          │   nodes    │          │   nodes    │
   └────────────┘          └────────────┘          └────────────┘
```

## Multi-Tenancy Model

### Tenant Isolation

| Layer | Isolation Method |
|-------|------------------|
| Network Map | Per-tenant, no cross-tenant visibility |
| Policies | Scoped to tenant |
| Data | Row-level security in database |
| Relays | Shared infrastructure, encrypted traffic |
| Logs | Tenant-tagged, access controlled |

### Database Schema

```sql
-- All tables have tenant_id
CREATE TABLE nodes (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    public_key BYTEA,
    ...
);

-- Row-level security
ALTER TABLE nodes ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON nodes
    USING (tenant_id = current_setting('app.tenant_id')::UUID);
```

### Tenant Configuration

```json
{
  "tenant_id": "t-abc123",
  "name": "Acme Corp",
  "settings": {
    "ip_range": "100.64.0.0/16",
    "dns_suffix": "acme.cvlan.net",
    "allowed_auth_methods": ["oidc"],
    "oidc_config": {
      "issuer": "https://acme.okta.com",
      "client_id": "..."
    }
  },
  "subscription": {
    "plan": "business",
    "node_limit": 1000,
    "features": ["ipsec", "vrouter"]
  }
}
```

## Relay Network

### Global Distribution

Deploy relays in major cloud regions:

| Region | Provider | Relays | Notes |
|--------|----------|--------|-------|
| us-east-1 | AWS/Vultr | 3 | Primary US East |
| us-west-2 | AWS/Vultr | 3 | Primary US West |
| eu-west-1 | AWS/Hetzner | 3 | Primary EU |
| eu-central-1 | Hetzner | 2 | EU backup |
| ap-southeast-1 | AWS/Vultr | 2 | APAC |
| ap-northeast-1 | AWS | 2 | Japan |

### Relay Selection

Nodes select home relay based on:
1. Latency measurement to all relays
2. Choose lowest latency
3. Report selection to controller
4. Periodically re-evaluate

### Relay Mesh

Within each region, relays mesh for redundancy:

```
Region: us-east-1
┌─────────┐     ┌─────────┐     ┌─────────┐
│ relay-1 │◄───►│ relay-2 │◄───►│ relay-3 │
└────┬────┘     └────┬────┘     └────┬────┘
     │               │               │
     └───────────────┼───────────────┘
                     │
            Regional mesh
```

## Managed vRouters

### vRouter as a Service

Customers can provision managed vRouters:

```
POST /api/v1/vrouters
{
  "name": "hq-gateway",
  "size": "medium",
  "region": "us-east-1",
  "config": {
    "ipsec_peers": [...],
    "advertised_routes": ["10.0.0.0/8"]
  }
}
```

### Deployment Options

| Option | Description | Use Case |
|--------|-------------|----------|
| CloudVLAN-hosted | We run it in our infrastructure | Simplest |
| Customer cloud | We deploy to their AWS/GCP/Azure | Data locality |
| Hybrid | Management via SaaS, vRouter on-prem | Compliance |

### Pricing

| Size | Capacity | Monthly |
|------|----------|---------|
| Small | 100 tunnels, 500 Mbps | $50 |
| Medium | 500 tunnels, 2 Gbps | $150 |
| Large | 2000 tunnels, 10 Gbps | $400 |

## API & Admin Console

### API

RESTful API for all operations:

```bash
# Create network
curl -X POST https://api.cloudvlan.io/v1/networks \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name": "production"}'

# Add node (returns auth key)
curl -X POST https://api.cloudvlan.io/v1/nodes \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name": "server-1", "tags": ["web"]}'

# Update policies
curl -X PUT https://api.cloudvlan.io/v1/policies \
  -H "Authorization: Bearer $TOKEN" \
  -d @policies.json
```

### Admin Console

Web UI for:
- Network visualization
- Node management
- Policy editor
- Usage analytics
- Billing management

### CLI

```bash
# Login
cvlanctl login

# List nodes
cvlanctl nodes list

# Add node (interactive)
cvlanctl nodes add --name server-1

# Apply policies from file
cvlanctl policies apply -f policies.hjson
```

## Onboarding Flow

### Self-Service

1. Sign up at cloudvlan.io
2. Connect identity provider (Okta, Google, etc.)
3. Download cvlan-agent on first device
4. Authenticate and join network
5. Add more devices

### Enterprise

1. Contact sales / request trial
2. SSO configuration assistance
3. Network design consultation
4. Pilot deployment
5. Production rollout
6. Ongoing support

## SLAs

| Tier | Control Plane | Relay Network | Support |
|------|---------------|---------------|---------|
| Free | Best effort | Best effort | Community |
| Pro | 99.9% | 99.9% | Email, 24h response |
| Business | 99.95% | 99.95% | Email, 4h response |
| Enterprise | 99.99% | 99.99% | Dedicated, 1h response |

## Data Residency

For customers with data residency requirements:

| Region | Control Plane | Relays | Database |
|--------|---------------|--------|----------|
| US | us-east-1 | US only | US |
| EU | eu-west-1 | EU only | EU (GDPR compliant) |
| APAC | ap-southeast-1 | APAC only | Singapore |

## Security & Compliance

### Certifications (Target)

- SOC 2 Type II
- ISO 27001
- GDPR compliant
- HIPAA BAA available

### Security Measures

| Measure | Implementation |
|---------|----------------|
| Encryption at rest | AES-256 for database |
| Encryption in transit | TLS 1.3 everywhere |
| Access control | RBAC, MFA required for admin |
| Audit logging | All API calls logged |
| Penetration testing | Annual third-party assessment |
| Bug bounty | Responsible disclosure program |

## Cost Structure (Internal)

### Infrastructure Costs (at scale)

| Component | Monthly Cost | Per 10k nodes |
|-----------|--------------|---------------|
| Control plane (K8s) | $500 | $50 |
| Database (managed) | $200 | $20 |
| Relay network (20 servers) | $400 | $40 |
| Bandwidth | Variable | ~$100 |
| Monitoring/logging | $100 | $10 |
| **Total** | ~$1,200 | ~$220 |

Target margin: 70%+ on subscription revenue
