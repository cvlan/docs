# Self-Hosted Deployment

Self-hosted is a **first-class deployment model** for CloudVLAN, not an afterthought. Same binaries, same features, same quality as SaaS.

## Architecture Options

### Minimal (Small Teams)

Single-node deployment with embedded SQLite:

```
┌─────────────────────────────────────────┐
│            Single Server                │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │       cvlan-api (combined mode) │   │
│  │       + embedded SQLite         │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │       Management UI (optional)  │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘

Requirements: 1 VM, 2 GB RAM, 2 vCPU, public IP
```

In combined mode, cvlan-api runs all server functions (discovery, registration, session, controller) in a single process. SQLite eliminates the need for a separate database.

### Standard (Production)

Separate components for reliability:

```
┌─────────────────┐     ┌─────────────────┐
│  Load Balancer  │     │   PostgreSQL    │
│  (nginx/HAProxy)│     │                 │
└────────┬────────┘     └────────┬────────┘
         │                       │
┌────────┴───────────────────────┴────────┐
│  ┌──────────────┐  ┌──────────────┐     │
│  │  cvlan-api   │  │  cvlan-api   │     │
│  │  (instance 1)│  │  (instance 2)│     │
│  └──────────────┘  └──────────────┘     │
│                                          │
│  ┌──────────────┐  ┌──────────────┐     │
│  │  step-ca     │  │  Management  │     │
│  │  (mTLS CA)   │  │     UI       │     │
│  └──────────────┘  └──────────────┘     │
└──────────────────────────────────────────┘
```

### Tiered (Large Scale)

Separate instances per server mode for DDoS resilience:

```
Internet → Discovery (port 443, rate-limited, no auth)
         → Registration (WAF, token verification)
Internal → Session (mTLS only, registered nodes)
         → Controller (VPN/bastion, admin access)
```

See [Tiered Architecture](../../architecture/system-overview.md) for details.

## Deployment Artifacts

| Artifact | Format | Available |
|----------|--------|-----------|
| Docker images | OCI | cvlan-api, cvlan-ui, vrouter |
| Debian packages | .deb | cvlan-ctrl, cvland (in progress) |
| Static binaries | Linux x86_64 | Future |

## Configuration

### cvlan-api

Key environment variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL or SQLite path | `postgres://user:pass@host/db` or `sqlite:///data/cvlan.db` |
| `JWT_SECRET` | Token signing secret | Random 256-bit string |
| `STEP_CA_URL` | step-ca endpoint | `https://step-ca:9000` |
| `SERVER_MODE` | Server mode | `combined`, `controller`, `discovery`, etc. |
| `LISTEN_ADDR` | Bind address | `0.0.0.0:8080` |

### step-ca

Required for mTLS certificate issuance. Deploy alongside cvlan-api:

```bash
step ca init --name="CloudVLAN CA" --provisioner="admin"
step-ca --password-file=/etc/step/password $(step path)/config/ca.json
```

## Sizing Guide

| Scale | Nodes | cvlan-api | Database | RAM | CPU |
|-------|-------|-----------|----------|-----|-----|
| Small | < 100 | 1 instance | SQLite | 2 GB | 2 |
| Medium | < 1,000 | 2 instances | PostgreSQL | 4 GB | 4 |
| Large | < 10,000 | 4 instances | PostgreSQL HA | 8 GB | 8 |
| XL | < 100,000 | Tiered | PostgreSQL HA + read replicas | 16 GB | 16 |

## Related

- [Architecture: System Overview](../../architecture/system-overview.md)
- [Architecture: Control Plane API](../../architecture/control-plane-api.md)
- [Technical: Database Design](../../technical/database-design.md)
