---
marp: true
theme: default
paginate: true
backgroundColor: #1a1a2e
color: #eaeaea
style: |
  section {
    font-family: 'Inter', 'Helvetica Neue', sans-serif;
  }
  h1 {
    color: #00d4ff;
  }
  h2 {
    color: #00d4ff;
  }
  strong {
    color: #00d4ff;
  }
  table {
    font-size: 0.8em;
  }
  .columns {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1rem;
  }
---

<!-- _class: lead -->
<!-- _backgroundColor: #0f0f1a -->

# **CloudVLAN**

## Enterprise Overlay Networking
### WireGuard Speed. IPsec Compliance. Your Infrastructure.

![bg right:40% 80%](https://api.dicebear.com/7.x/shapes/svg?seed=cloudvlan&backgroundColor=0f0f1a)

---

# The Problem

## Enterprise networking is broken

<div class="columns">
<div>

### SD-WAN Reality
- **$500-2,000**/site/month
- 6-12 month deployments
- Proprietary hardware lock-in
- Overly complex for 80% of use cases

</div>
<div>

### Modern VPN Gap
- Tailscale: No IPsec (compliance gap)
- Self-hosted = second-class
- No air-gapped support
- Can't integrate legacy devices

</div>
</div>

![bg right:30% 90%](https://api.dicebear.com/7.x/shapes/svg?seed=problem&backgroundColor=1a1a2e)

---

# The Opportunity

## $15B+ Market, Ripe for Disruption

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   SD-WAN Market: $8.4B (2025) → $19B (2030)            │
│   ████████████████████████████████████░░░░░░░  +15% CAGR│
│                                                         │
│   VPN/Zero Trust: $6.2B (2025) → $12B (2030)           │
│   █████████████████████████████░░░░░░░░░░░░░░  +14% CAGR│
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Target Segments:**
- Regulated enterprises (finance, healthcare, gov)
- Multi-cloud organizations
- Cost-conscious large enterprises
- Air-gapped / secure environments

---

# The Solution

## CloudVLAN: Best of All Worlds

```
                    ┌─────────────────────────────┐
                    │     CloudVLAN Controller    │
                    │   (Coordination & Policy)   │
                    └──────────────┬──────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
           ▼                       ▼                       ▼
    ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
    │   Laptop    │◄═══════►│   Server    │◄═══════►│  vRouter    │
    │ (WireGuard) │  Mesh   │ (WireGuard) │         │  (IPsec)    │
    └─────────────┘         └─────────────┘         └──────┬──────┘
                                                           │
                                                    ┌──────┴──────┐
                                                    │Legacy Router│
                                                    │  (IPsec)    │
                                                    └─────────────┘
```

**Dual Protocol** | **Self-Hosted First** | **Air-Gapped Ready** | **10x Cheaper**

---

# Why We Win

## Head-to-Head Comparison

| Capability | Tailscale | SD-WAN | CloudVLAN |
|------------|:---------:|:------:|:---------:|
| WireGuard (fast, modern) | ✅ | ❌ | ✅ |
| IPsec (compliant) | ❌ | ✅ | ✅ |
| First-class self-hosted | ❌ | ❌ | ✅ |
| Air-gapped support | ❌ | ⚠️ | ✅ |
| Cloud-neutral | ✅ | ❌ | ✅ |
| Cost (1000 nodes) | $6K/mo | $50K+/mo | **$5K/mo** |
| Deployment time | Hours | Months | **Hours** |

![bg right:25% 80%](https://api.dicebear.com/7.x/shapes/svg?seed=winning&backgroundColor=1a1a2e)

---

# Product Architecture

## Built for Scale & Efficiency

```
┌────────────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   Auth   │  │  Policy  │  │  Node    │  │ Network  │       │
│  │  Engine  │  │  Engine  │  │ Registry │  │   Maps   │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────┼──────────────────────────────────┐
│                    DATA PLANE (Rust, No GC)                     │
│                             │                                   │
│    ┌──────────┐      ┌──────┴─────┐      ┌──────────┐         │
│    │  Agents  │◄────►│   Relays   │◄────►│ vRouters │         │
│    │(endpoints)│      │ (fallback) │      │(gateways)│         │
│    └──────────┘      └────────────┘      └──────────┘         │
└────────────────────────────────────────────────────────────────┘
```

**100K nodes on <$2K/month infrastructure** — Rust data plane, no GC overhead

---

# Business Model

## Land & Expand with Multiple Revenue Streams

<div class="columns">
<div>

### SaaS
| Tier | Price | Target |
|------|-------|--------|
| Free | $0 | 5 nodes |
| Pro | $5/node | Teams |
| Business | $8/node | SMB |
| Enterprise | Custom | Large |

</div>
<div>

### Self-Hosted Licenses
| Tier | Price | Nodes |
|------|-------|-------|
| Starter | $500/yr | 100 |
| Business | $2K/yr | 1,000 |
| Enterprise | $10K/yr | Unlimited |

</div>
</div>

### Premium Add-ons
- **Managed vRouters**: $50-400/month per gateway
- **Air-gapped support**: Enterprise tier
- **Professional services**: Implementation, training

---

# Traction & Roadmap

## Execution Timeline

```
        Q1 2026          Q2 2026          Q3 2026          Q4 2026
           │                │                │                │
    ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
    │   CURRENT   │  │    NEXT     │  │   GROWTH    │  │    SCALE    │
    │             │  │             │  │             │  │             │
    │ • Docs &    │  │ • Relay     │  │ • Full      │  │ • vRouter   │
    │   Planning  │  │   Server    │  │   Control   │  │   Launch    │
    │ • Technical │  │ • Basic     │  │   Plane     │  │ • IPsec     │
    │   Design    │  │   Agent     │  │ • WireGuard │  │   Support   │
    │             │  │             │  │   Mesh      │  │ • First $   │
    └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
                                                              │
                                                              ▼
                                                     Target: 10 paying
                                                     customers by EOY
```

**Unfair Advantage**: Founder built multiple SDN solutions previously

---

# Team & Ask

<div class="columns">
<div>

## Founder

**[Name]**
- Multiple SDN products shipped
- Deep networking & security expertise
- AI-augmented development (Claude Code)

*"One founder + AI can now build what took teams of 10"*

</div>
<div>

## The Ask

### Seed Round: $1.5M

**Use of Funds:**
- Engineering (60%)
- Go-to-market (25%)
- Infrastructure (10%)
- Legal/Ops (5%)

**Milestones:**
- Production-ready product
- 50 paying customers
- $500K ARR

</div>
</div>

![bg right:20% 80%](https://api.dicebear.com/7.x/shapes/svg?seed=team&backgroundColor=1a1a2e)

---

<!-- _class: lead -->
<!-- _backgroundColor: #0f0f1a -->

# **CloudVLAN**

## The network enterprises actually need.
### Simple. Compliant. Theirs.

<br>

**contact@cloudvlan.io** | **cloudvlan.io**

![bg right:40% 80%](https://api.dicebear.com/7.x/shapes/svg?seed=cloudvlan&backgroundColor=0f0f1a)
