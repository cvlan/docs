# CloudVLAN (cvlan) - Project Context for Claude Code

> **Purpose**: This document provides full project context for Claude Code sessions. Read this first in any new session.

## What This Is

CloudVLAN is a next-generation overlay network that combines:
- The simplicity of Tailscale
- The compliance of IPsec (IKEv2)
- First-class self-hosting (not an afterthought)
- Air-gapped network support
- Managed cloud vRouters as IPsec concentrators

It targets enterprises that need secure, compliant network connectivity but are underserved by expensive SD-WAN solutions or limited by WireGuard-only tools like Tailscale.

## Project Nature

- **Part-time product building experiment** with AI-assisted development (Claude Code)
- **No fixed timeline or VC pressure**
- **Success outcomes**: Sell the company, or open-source with a blog post about the journey
- **Solo founder** with prior SDN experience (built multiple SDN solutions before)

## Key Decisions (Settled - Do Not Revisit)

### Technical
| Decision | Rationale |
|----------|-----------|
| **Rust for data plane** | No GC = predictable memory, no cost spikes at scale |
| **No Go/Java for high-volume paths** | GC creates 2-3x memory overhead, unpredictable costs |
| **Dual protocol: WireGuard + IKEv2/IPsec** | WireGuard for simplicity, IPsec for compliance |
| **Cloud-neutral infrastructure** | No AWS/GCP-specific services, runs anywhere |
| **Self-hosted is primary, not secondary** | Unlike Tailscale/Headscale relationship |
| **Server-authoritative model** | Server assigns CVLANs, IPs, policies. Clients don't choose. |
| **Ed25519 registration tokens** | Tenant signs offline, server verifies and assigns CVLAN |
| **mTLS via step-ca** | Lightweight CA, auto-renewal, no manual cert management |
| **Protocol Buffers for types** | Single source of truth, generates Rust + TypeScript |
| **SQLite dual-backend** | Self-hosted single-box deploys without Postgres dependency |

### Business
| Decision | Rationale |
|----------|-----------|
| **Scale target: 100k clients < $2k/month infra** | ~$0.02/client, proves efficiency |
| **Break-even: few hundred nodes** | Low fixed costs, sustainable from early stage |
| **No VC model** | Build solid product, fair pricing, no ARR pressure |
| **Target: large enterprises** | Struggling with SD-WAN costs, compliance needs |

## Component Naming

| Component | Binary | Repo | Description |
|-----------|--------|------|-------------|
| Control plane API | cvlan-api | cvlan/ | REST API + node registration + policy |
| Admin CLI | cvlanctl | cvlan/ | Tenant/CVLAN/node management |
| VPP Gateway | vrouterd | vrouter/ | VPP 24.10 + WireGuard + IPsec |
| Gateway CLI | vrouterctl | vrouter/ | VPP debug/introspection |
| Client control plane | cvlan-ctrl | client/ | Discovery, registration, poll loop |
| Client data plane | cvland | client/ | WireGuard tunnels (stub) |
| Client CLI | cvlancli | client/ | Debug/introspection |
| Management UI | — | ui/ | React 18 + TypeScript |
| Relay | — | — | Not started |

## Architecture Summary

```
CONTROL PLANE
┌─────────────────────────────────────────────────────────────────┐
│  cvlan-api (REST + node registration + policy + audit)          │
│  Multi-tenant RBAC │ Background scheduler │ IP allocation       │
│  Server modes: controller / discovery / registration / session  │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS / mTLS
DATA PLANE                 │
┌──────────────────────────┴──────────────────────────────────────┐
│  vrouterd         cvlan-ctrl+cvland   relay (not started)       │
│  (VPP gateway)    (client node)       (NAT traversal)           │
│  [vrouter repo]   [client repo]                                 │
└─────────────────────────────────────────────────────────────────┘
        │               │               │
        └───────────────┴───────────────┘
              Direct mesh (WireGuard) or via relay
```

## Current Phase

**Phase**: Client data plane (critical path to first connection)
**What's built**: Control plane API, VPP gateway, management UI, client control plane daemon, regression suite
**What's next**: cvland (boringtun WireGuard tunnels), node-VRouter connectivity, end-to-end test
**Status**: See [STATUS.md](STATUS.md)

## Implementation Path (Actual)

1. **cvlan-api** — Control plane with multi-tenancy, RBAC, audit logging (Week 1)
2. **Management UI** — React SPA for all entity management (Week 1-2)
3. **vrouterd** — VPP gateway with mTLS registration + DiffSync poll (Week 3)
4. **Protocol Buffers** — Single source of truth for type definitions (Week 3)
5. **cvlan-ctrl** — Client control plane daemon (Week 4)
6. **cvland** — Client data plane with boringtun (next)
7. **Relay** — NAT traversal (future)

## Code Conventions

- **Language**: Rust for all data plane and performance-critical code
- **Async runtime**: Tokio
- **Error handling**: `thiserror` for libraries, `anyhow` for binaries
- **CLI parsing**: `clap`
- **Serialization**: `serde` with JSON for configs, YAML for config files
- **Database**: SQLx 0.8 (Postgres), rusqlite (bundled SQLite for local state)
- **Crypto**: x25519-dalek (Curve25519), Ed25519 (signing), Argon2 (password/key hashing)
- **No unsafe** unless absolutely necessary and well-documented
- **All builds in Docker** — never cargo/npm on host

## Session Instructions for Claude

1. **Read this file first** in any new session
2. **Check [STATUS.md](STATUS.md)** for current status and open items
3. **Do not add Claude as co-author** in git commits
4. **Respect settled decisions** listed above — don't re-debate them
5. **Keep code simple** — no over-engineering, no premature abstraction
6. **All builds happen in Docker containers** — never run cargo/npm on host

## Key Files

| File | Purpose |
|------|---------|
| `CLAUDE.md` | This file — project context |
| `STATUS.md` | Current progress, backlog, Headscale comparison |
| `README.md` | Landing page + component table + doc links |
| `architecture/` | System design documents |
| `technical/` | Deep dives (IP allocation, auth, database, etc.) |
| `operations/` | Build system, testing, deployment |
