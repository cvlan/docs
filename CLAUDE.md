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

### Business
| Decision | Rationale |
|----------|-----------|
| **Scale target: 100k clients < $2k/month infra** | ~$0.02/client, proves efficiency |
| **Break-even: few hundred nodes** | Low fixed costs, sustainable from early stage |
| **No VC model** | Build solid product, fair pricing, no ARR pressure |
| **Target: large enterprises** | Struggling with SD-WAN costs, compliance needs |

## USPs / Differentiators

1. **Dual protocol**: WireGuard + IPsec (compliance-ready, not WireGuard-only)
2. **First-class self-hosting**: Same product for SaaS and self-hosted
3. **Air-gapped support**: Fully offline operation possible
4. **Cloud vRouters**: Managed IPsec concentrators for tenants
5. **Radically lower cost**: 10-50x cheaper than traditional SD-WAN
6. **No VC extraction**: Sustainable pricing, focused product

## Constraints

| Constraint | Target |
|------------|--------|
| Scale | 100k clients |
| Infrastructure cost at scale | < $2,000/month |
| Break-even point | Few hundred nodes |
| Deployment | Any cloud, colo, or on-prem |
| Memory model | No GC languages for data plane |

## Component Naming

| Component | Binary/Reference |
|-----------|------------------|
| Control plane | cvlan-controller |
| Client agent | cvlan-agent |
| Relay server | cvlan-relay |
| vRouter | cvlan-vrouter |
| CLI tool | cvlanctl |
| Policy engine | cvlan-policy |

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    cvlan-controller                          │
│         (coordination, auth, policy, node mgmt)              │
└─────────────────────┬───────────────────────────────────────┘
                      │ Control Plane (HTTPS/Noise)
        ┌─────────────┼─────────────┬─────────────┐
        ▼             ▼             ▼             ▼
   ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────┐
   │cvlan-   │  │cvlan-   │  │cvlan-    │  │cvlan-    │
   │agent    │  │agent    │  │vrouter   │  │relay     │
   │(WG+IPsec)│ │(WG+IPsec)│ │(IPsec GW)│  │(fallback)│
   └────┬────┘  └────┬────┘  └────┬─────┘  └────┬─────┘
        │            │            │             │
        └────────────┴─────┬──────┴─────────────┘
                           │ Data Plane (WireGuard / IPsec)
                      Direct mesh or via relay
```

## Current Phase

> Update this section as work progresses

**Phase**: Planning / Documentation
**Focus**: Establishing project structure and technical specifications
**Next**: Begin implementation with relay server (small, self-contained, proves the stack)

## Implementation Order (Suggested)

1. **cvlan-relay** - Rust relay server, proves the stack
2. **cvlan-controller** - Basic control plane, node registration
3. **cvlan-agent** - WireGuard client agent
4. **cvlan-policy** - ACL engine
5. **IPsec integration** - Add IKEv2 support
6. **cvlan-vrouter** - Managed IPsec concentrator

## Code Conventions

- **Language**: Rust for all data plane and performance-critical code
- **Async runtime**: Tokio
- **Error handling**: `thiserror` for libraries, `anyhow` for binaries
- **CLI parsing**: `clap`
- **Serialization**: `serde` with JSON for configs, binary protocols for wire
- **No unsafe** unless absolutely necessary and well-documented

## Out of Scope

- User documentation (separate repo if needed)
- Mobile clients (future phase)
- GUI clients (CLI-first)
- Kubernetes operators (future phase)
- Terraform providers (future phase)

## Session Instructions for Claude

1. **Read this file first** in any new session
2. **Check WORK_TRACKING.md** for current status and open items
3. **Do not add Claude as co-author** in git commits
4. **Respect settled decisions** listed above—don't re-debate them
5. **Keep code simple**—no over-engineering, no premature abstraction
6. **Update WORK_TRACKING.md** when completing significant items
7. **Update JOURNAL.md** with interesting learnings or challenges

## Key Files

| File | Purpose |
|------|---------|
| `CLAUDE.md` | This file - project context |
| `WORK_TRACKING.md` | Current progress, backlog, decisions log |
| `JOURNAL.md` | Notes for eventual blog post |
| `technical/architecture.md` | Detailed system design |
| `technical/technology-choices.md` | Why Rust, why no GC, key deps |
