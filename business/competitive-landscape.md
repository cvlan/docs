# Competitive Landscape

## Market Positioning

CloudVLAN sits at the intersection of:
- **Modern mesh VPN** (Tailscale simplicity)
- **Traditional SD-WAN** (enterprise features, IPsec)
- **Self-hosted infrastructure** (full control)

```
                    Enterprise Features
                           ▲
                           │
         Cisco/VMware      │     CloudVLAN
         SD-WAN            │     (target position)
                           │
    ◄──────────────────────┼──────────────────────►
    SaaS-only              │              Self-hosted
                           │
         Tailscale         │     Headscale
         ZeroTier          │     Netmaker
                           │
                           ▼
                    Consumer/Simple
```

## Traditional SD-WAN Vendors

| Vendor | Strengths | Weaknesses | Our Advantage |
|--------|-----------|------------|---------------|
| **Cisco Viptela/Meraki** | Brand trust, integration | $$$$, complexity, lock-in | 10-50x cost reduction, no hardware |
| **VMware SD-WAN** | vSphere integration | Acquisition bloat, cloud-centric | Cloud-neutral, simpler |
| **Palo Alto Prisma** | Security integration | Enterprise pricing, sales-driven | Transparent pricing, self-serve |
| **Fortinet** | Firewall integration | Hardware-centric | Software-only, portable |
| **Aruba/Silver Peak** | Performance | HPE acquisition uncertainty | Independent, focused |

**Common SD-WAN problems we solve:**
- $500-2000/site/month → $0.02/node/month infrastructure
- 6-12 month deployments → Hours
- Proprietary appliances → Software on any infrastructure
- Vendor lock-in → Cloud-neutral, standards-based

## Modern Mesh VPN

| Vendor | Strengths | Weaknesses | Our Advantage |
|--------|-----------|------------|---------------|
| **Tailscale** | UX, developer adoption | WireGuard-only, SaaS-first | IPsec compliance, self-hosted first |
| **ZeroTier** | Flexibility | Similar limitations to Tailscale | Dual protocol, enterprise focus |
| **Netmaker** | Open source | WireGuard-only, less mature | Production-ready, IPsec |
| **Nebula (Slack)** | Simple, proven | No IPsec, limited UI | Full management, compliance |
| **Headscale** | Free, self-hosted | Community project, single tailnet | First-class product, multi-tenant |

**Why WireGuard-only is a limitation:**
- Not FIPS 140-2 compliant (US government, finance, healthcare)
- Blocked by many enterprise security policies
- Regulatory issues in some jurisdictions
- No support for legacy devices that only speak IPsec
- IPsec is the universal enterprise fallback

## Enterprise VPN/ZTNA

| Vendor | Strengths | Weaknesses | Our Advantage |
|--------|-----------|------------|---------------|
| **Zscaler** | Scale, security | SaaS-only, expensive, latency | Self-hosted option, direct mesh |
| **Cloudflare Access** | Edge network | Mostly HTTP/S, SaaS-only | Full network layer, self-hosted |
| **Palo Alto GlobalProtect** | Security stack | Requires Palo Alto hardware | Standalone, any infrastructure |

## The Gap We Fill

| Need | Traditional SD-WAN | Tailscale | CloudVLAN |
|------|-------------------|-----------|-----------|
| IPsec compliance | ✅ | ❌ | ✅ |
| WireGuard simplicity | ❌ | ✅ | ✅ |
| Self-hosted first | ❌ | ❌ | ✅ |
| Air-gapped support | Partial | ❌ | ✅ |
| Low cost | ❌ | ✅ | ✅ |
| No hardware required | ❌ | ✅ | ✅ |
| Cloud-neutral | ❌ | ✅ | ✅ |

## Target Customer Segments

| Segment | Pain Point | Why CloudVLAN |
|---------|------------|---------------|
| **Regulated enterprises** | Can't use WireGuard, need compliance | IPsec + audit-ready |
| **Multi-cloud orgs** | Vendor lock-in, complexity | Cloud-neutral, portable |
| **Cost-conscious large orgs** | $50k+/month SD-WAN bills | 10-50x reduction |
| **Security-first orgs** | Won't trust SaaS control plane | Self-hosted primary |
| **Air-gapped environments** | No internet connectivity | Fully offline operation |
| **SD-WAN refugees** | Burned by broken promises | Simplicity, honest pricing |

## Competitive Moats

1. **Dual protocol**: Few competitors offer both WireGuard and IPsec in one product
2. **Self-hosted parity**: Not a crippled open-source version
3. **Cost structure**: Rust + no GC enables genuinely lower infrastructure costs
4. **No VC pressure**: Can price fairly, won't pivot or enshittify
5. **Air-gapped**: Genuine offline support is rare and hard to add later
