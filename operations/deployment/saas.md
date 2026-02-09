# SaaS Deployment

CloudVLAN SaaS is the managed offering where we operate the control plane infrastructure. Customers get the simplicity of Tailscale with CloudVLAN's protocol flexibility and multi-tenancy.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 CloudVLAN SaaS Infrastructure                 │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    Control Plane                         │ │
│  │                                                          │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │ │
│  │  │cvlan-api │  │cvlan-api │  │cvlan-api │              │ │
│  │  │(discover)│  │(register)│  │(session) │              │ │
│  │  └──────────┘  └──────────┘  └──────────┘              │ │
│  │        │              │             │                    │ │
│  │        └──────────────┼─────────────┘                   │ │
│  │                       ▼                                  │ │
│  │              ┌──────────────┐    ┌──────────┐           │ │
│  │              │ PostgreSQL   │    │  step-ca │           │ │
│  │              │ (Primary+RR) │    │          │           │ │
│  │              └──────────────┘    └──────────┘           │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Managed vRouter Network                      │ │
│  │                                                          │ │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐        │ │
│  │  │vrouterd│  │vrouterd│  │vrouterd│  │vrouterd│        │ │
│  │  │ US-E   │  │ US-W   │  │  EU    │  │ APAC   │        │ │
│  │  └────────┘  └────────┘  └────────┘  └────────┘        │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Relay Network (future)                       │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                    Internet  │
                              │
         ┌────────────────────┼─────────────────────┐
         │                    │                      │
    ┌────┴─────┐       ┌─────┴────┐          ┌─────┴────┐
    │cvlan-ctrl│       │cvlan-ctrl│          │cvlan-ctrl│
    │(customer)│       │(customer)│          │(customer)│
    └──────────┘       └──────────┘          └──────────┘
```

## Tiered Server Deployment

In SaaS, each server mode runs as a separate service:

| Mode | Exposure | Scaling |
|------|----------|---------|
| Discovery | Public CDN/Anycast | High (handle DDoS) |
| Registration | Behind WAF | Medium (burst on onboarding) |
| Session | Internal LB | High (all registered nodes poll) |
| Controller | Admin VPN | Low (admin-only) |

## Multi-Tenancy at Scale

SaaS relies heavily on the built-in multi-tenancy:
- Full tenant isolation at the query level
- Per-tenant RBAC (tenantadmin manages their own org)
- Per-tenant CIDR allocation (no overlap conflicts)
- Per-tenant audit logging (compliance-ready)

Target: **100,000 nodes across all tenants at < $2,000/month infrastructure cost**.

## Managed vRouters

Regional VPP gateways managed by CloudVLAN:
- VPP kernel-bypass for line-rate performance
- Automatic registration + config via poll loop
- 8-hour certificate rotation (security)
- Stats reporting to control plane

Customers don't manage gateways — they just connect nodes.

## Pricing Model (Planned)

| Tier | Nodes | Features |
|------|-------|----------|
| Free | 5 | WireGuard, 1 CVLAN |
| Team | 100 | + policies, groups, audit logs |
| Business | 1,000 | + managed vRouters, IPsec, SSO |
| Enterprise | Unlimited | + dedicated infra, SLA, air-gapped option |

No VC extraction — sustainable pricing that covers infrastructure costs.

## Related

- [Self-Hosted Deployment](self-hosted.md) — Same product, self-managed
- [Air-Gapped Deployment](air-gapped.md) — Fully offline variant
- [Architecture: Control Plane API](../../architecture/control-plane-api.md) — Server modes
