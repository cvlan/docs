# Self-Hosted Deployment

## Overview

Self-hosted is a **first-class deployment model** for CloudVLAN, not an afterthought. The same binaries, same features, same quality.

## Architecture Options

### Minimal (Small Teams)

Single-node deployment for small teams or testing:

```
┌─────────────────────────────────────────┐
│            Single Server                │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │       cvlan-controller          │   │
│  │       (+ embedded SQLite)       │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │        cvlan-relay              │   │
│  │        (optional, local)        │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘

Requirements:
- 1 VM/server
- 2 GB RAM
- 2 vCPU
- Public IP (or NAT with port forwarding)
```

### Standard (Production)

Separate components for reliability:

```
┌─────────────────┐     ┌─────────────────┐
│  Load Balancer  │     │    Database     │
│  (nginx/HAProxy)│     │   (PostgreSQL)  │
└────────┬────────┘     └────────┬────────┘
         │                       │
         ▼                       │
┌─────────────────────────────────┴────────┐
│           Controller Cluster             │
│                                          │
│  ┌──────────────┐  ┌──────────────┐     │
│  │ controller-1 │  │ controller-2 │     │
│  └──────────────┘  └──────────────┘     │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│           Relay Servers                  │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │ relay-1  │  │ relay-2  │  │relay-3 │ │
│  │ (region1)│  │ (region2)│  │(region3│ │
│  └──────────┘  └──────────┘  └────────┘ │
└──────────────────────────────────────────┘

Requirements:
- 3-5 VMs for controller + DB
- N relay servers (1 per region)
- Total: ~$100-200/month
```

### High Availability

For mission-critical deployments:

```
                    ┌─────────────────┐
                    │  Global LB      │
                    │  (DNS/Anycast)  │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   Region A    │    │   Region B    │    │   Region C    │
│               │    │               │    │               │
│ ┌───────────┐ │    │ ┌───────────┐ │    │ ┌───────────┐ │
│ │controllers│ │    │ │controllers│ │    │ │controllers│ │
│ │  (2-3)    │ │    │ │  (2-3)    │ │    │ │  (2-3)    │ │
│ └───────────┘ │    │ └───────────┘ │    │ └───────────┘ │
│ ┌───────────┐ │    │ ┌───────────┐ │    │ ┌───────────┐ │
│ │  relays   │ │    │ │  relays   │ │    │ │  relays   │ │
│ │  (2-3)    │ │    │ │  (2-3)    │ │    │ │  (2-3)    │ │
│ └───────────┘ │    │ └───────────┘ │    │ └───────────┘ │
│ ┌───────────┐ │    │ ┌───────────┐ │    │ ┌───────────┐ │
│ │ postgres  │ │    │ │ postgres  │ │    │ │ postgres  │ │
│ │ (replica) │◄┼────┼►│ (primary) │◄┼────┼►│ (replica) │ │
│ └───────────┘ │    │ └───────────┘ │    │ └───────────┘ │
└───────────────┘    └───────────────┘    └───────────────┘
```

## Installation

### Single Binary

```bash
# Download
curl -LO https://releases.cloudvlan.io/latest/cvlan-controller-linux-amd64
curl -LO https://releases.cloudvlan.io/latest/cvlan-relay-linux-amd64

# Install
chmod +x cvlan-*
sudo mv cvlan-* /usr/local/bin/

# Initialize
cvlan-controller init --data-dir /var/lib/cvlan
```

### Container

```bash
# Controller
docker run -d \
  --name cvlan-controller \
  -p 443:443 \
  -v /var/lib/cvlan:/data \
  cloudvlan/controller:latest

# Relay
docker run -d \
  --name cvlan-relay \
  -p 443:443 \
  -p 3478:3478/udp \
  cloudvlan/relay:latest
```

### Kubernetes (Helm)

```bash
helm repo add cloudvlan https://charts.cloudvlan.io
helm install cvlan cloudvlan/cloudvlan \
  --set controller.replicas=2 \
  --set relay.replicas=3 \
  --set postgresql.enabled=true
```

## Configuration

### Controller

```toml
# /etc/cvlan/controller.toml

[server]
listen_addr = "0.0.0.0:443"
tls_cert = "/etc/cvlan/certs/server.crt"
tls_key = "/etc/cvlan/certs/server.key"

[database]
# SQLite for simple deployments
type = "sqlite"
path = "/var/lib/cvlan/cvlan.db"

# Or PostgreSQL for production
# type = "postgres"
# url = "postgres://user:pass@localhost/cvlan"

[auth]
# Local auth (for air-gapped, see air-gapped.md)
local_enabled = true

# OIDC (for connected environments)
oidc_enabled = true
oidc_issuer = "https://auth.example.com"
oidc_client_id = "cvlan"
oidc_client_secret = "secret"

[network]
# IP range for overlay network
ip_range = "100.64.0.0/10"

# DNS suffix
dns_suffix = "cvlan.local"
```

### Relay

```toml
# /etc/cvlan/relay.toml

[server]
listen_addr = "0.0.0.0:443"
stun_addr = "0.0.0.0:3478"

[controller]
url = "https://controller.example.com"
api_key = "relay-registration-key"

[mesh]
# Other relays in this region
peers = []
```

## Operations

### Backup

```bash
# SQLite
sqlite3 /var/lib/cvlan/cvlan.db ".backup /backup/cvlan-$(date +%Y%m%d).db"

# PostgreSQL
pg_dump cvlan > /backup/cvlan-$(date +%Y%m%d).sql
```

### Monitoring

Prometheus endpoints:
- Controller: `https://controller:443/metrics`
- Relay: `https://relay:443/metrics`

Key metrics to alert on:
- `cvlan_controller_nodes_total` - Node count
- `cvlan_relay_clients_connected` - Connected clients
- `cvlan_controller_db_latency_seconds` - Database health

### Upgrades

```bash
# Rolling upgrade (Kubernetes)
kubectl set image deployment/cvlan-controller \
  controller=cloudvlan/controller:v1.2.0

# Single node
systemctl stop cvlan-controller
curl -LO https://releases.cloudvlan.io/v1.2.0/cvlan-controller-linux-amd64
mv cvlan-controller-linux-amd64 /usr/local/bin/cvlan-controller
systemctl start cvlan-controller
```

## Sizing Guide

| Scale | Controller | Database | Relays | Est. Cost |
|-------|------------|----------|--------|-----------|
| 100 nodes | 1x 2GB | SQLite | 1x 512MB | ~$15/mo |
| 1,000 nodes | 2x 4GB | Postgres (2GB) | 2x 1GB | ~$80/mo |
| 10,000 nodes | 3x 4GB | Postgres (4GB) | 5x 1GB | ~$200/mo |
| 100,000 nodes | 5x 8GB | Postgres (8GB) | 20x 1GB | ~$800/mo |

## Cloud-Neutral Requirements

CloudVLAN runs on any infrastructure with:

| Requirement | Minimum |
|-------------|---------|
| Linux kernel | 4.19+ |
| Architecture | amd64, arm64 |
| Network | Public IP or NAT with port forward |
| Storage | 10GB+ (depends on node count) |
| TLS certificates | Let's Encrypt or self-managed |

**No cloud-specific services required:**
- ❌ AWS ALB, RDS, Lambda
- ❌ GCP Cloud SQL, Cloud Run
- ❌ Azure App Service, CosmosDB
- ✅ Any Linux VM
- ✅ Any container runtime
- ✅ Any Kubernetes cluster
