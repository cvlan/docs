# CloudVLAN Positioning

## Tagline Options

**Primary:**
> "Enterprise overlay networking. Simple. Compliant. Yours."

**Alternatives:**
> "The network you control. WireGuard speed. IPsec compliance."

> "SD-WAN outcomes without SD-WAN complexity."

> "Tailscale for enterprises that need IPsec."

## Elevator Pitch

**30 seconds:**
> CloudVLAN connects your devices, servers, and clouds into one secure network. Unlike Tailscale, we support IPsec for compliance. Unlike SD-WAN, we're simple and affordable. Unlike both, self-hosting is a first-class option—same product, same features, your infrastructure.

**10 seconds:**
> Secure overlay networking with WireGuard and IPsec. Self-hosted or SaaS. 10x cheaper than SD-WAN.

## Key Messages

### For IT Leaders

| Message | Supporting Point |
|---------|------------------|
| **Reduce complexity** | One solution for remote access, site-to-site, and cloud connectivity |
| **Cut costs** | 10-50x cheaper than traditional SD-WAN |
| **Stay compliant** | IPsec when auditors require it, WireGuard when they don't |
| **Keep control** | Self-host the entire stack if you need to |

### For Security Teams

| Message | Supporting Point |
|---------|------------------|
| **Zero trust ready** | Deny-by-default policies, identity-based access |
| **Compliant protocols** | IKEv2/IPsec with FIPS-capable cipher suites |
| **No SaaS dependency** | Full air-gapped deployment option |
| **Audit everything** | Complete logging of all connections and policy decisions |

### For Network Engineers

| Message | Supporting Point |
|---------|------------------|
| **Modern and legacy** | WireGuard for new deployments, IPsec for existing gear |
| **Cloud neutral** | Works the same on AWS, GCP, Azure, or bare metal |
| **Easy integration** | Standard protocols, no proprietary lock-in |
| **vRouters** | Software gateways replace expensive hardware |

### For Developers

| Message | Supporting Point |
|---------|------------------|
| **Just works** | Install agent, authenticate, connected |
| **No VPN headaches** | Mesh networking, no hub-and-spoke bottleneck |
| **CLI first** | Everything scriptable, GitOps-ready |
| **Self-hostable** | Run it in your homelab or your datacenter |

## Differentiation

### vs. Tailscale

| Aspect | Tailscale | CloudVLAN |
|--------|-----------|-----------|
| Protocols | WireGuard only | WireGuard + IPsec |
| Compliance | Limited (WG not FIPS) | IPsec with compliant ciphers |
| Self-hosted | Headscale (community) | First-class, same product |
| Air-gapped | Not supported | Full support |
| vRouter/gateway | Limited | Full IPsec concentrator |

**When to choose CloudVLAN over Tailscale:**
- Need IPsec for compliance (FIPS, government, finance)
- Want self-hosted without compromise
- Have air-gapped environments
- Need to integrate legacy IPsec devices

### vs. Traditional SD-WAN

| Aspect | SD-WAN (Cisco, etc.) | CloudVLAN |
|--------|---------------------|-----------|
| Cost | $500-2000/site/month | ~$5/node/month |
| Hardware | Required (appliances) | Software only |
| Deployment | Months | Hours |
| Complexity | High | Low |
| Lock-in | Proprietary | Open standards |

**When to choose CloudVLAN over SD-WAN:**
- Don't need WAN optimization (have good internet)
- Want to avoid hardware procurement
- Need faster deployment
- Budget conscious
- Value simplicity

### vs. Enterprise VPN (Zscaler, etc.)

| Aspect | Enterprise VPN | CloudVLAN |
|--------|----------------|-----------|
| Model | SaaS only | SaaS or self-hosted |
| Networking | Hub-and-spoke | Mesh |
| Protocols | Proprietary + IPsec | WireGuard + IPsec |
| Cost | High | Lower |

## Use Cases

### Remote Access

> "Give employees secure access to internal resources without a traditional VPN"

- Works from anywhere (home, coffee shop, mobile)
- No VPN client configuration
- Identity-based access control
- Split tunneling by default

### Multi-Cloud Connectivity

> "Connect workloads across AWS, GCP, Azure, and on-prem"

- Cloud-neutral (no vendor lock-in)
- Direct mesh (no traffic tromboning)
- Consistent policy across clouds
- vRouters for VPC integration

### Site-to-Site

> "Replace expensive MPLS and SD-WAN with simple overlay"

- vRouters as site gateways
- IPsec for legacy router compatibility
- Fraction of the cost
- Deploy in hours, not months

### Air-Gapped / Secure Environments

> "Secure networking for disconnected environments"

- Fully offline operation
- Internal CA and authentication
- No cloud dependencies
- Compliance-ready

## Objection Handling

| Objection | Response |
|-----------|----------|
| "We already use Tailscale" | Great product! But if you need IPsec compliance or first-class self-hosting, CloudVLAN fills that gap. |
| "We have SD-WAN" | SD-WAN is powerful but complex and expensive. CloudVLAN handles 80% of use cases at 10% of the cost. |
| "Is WireGuard enterprise-ready?" | Yes, and when it isn't (FIPS requirements), we support IPsec too. Best of both worlds. |
| "One-person company?" | Built by an experienced SDN architect. Product quality speaks for itself. |
| "Why not just use VPN?" | Traditional VPNs are hub-and-spoke. CloudVLAN is mesh—direct connections, better performance. |

## Proof Points (Build Over Time)

- [ ] Customer case studies
- [ ] Performance benchmarks
- [ ] Security audit results
- [ ] Uptime statistics
- [ ] Cost comparison calculator

## Website Structure (Future)

```
cloudvlan.io/
├── / (home)              → Hero, key benefits, CTA
├── /product              → Features, architecture
├── /pricing              → Transparent pricing
├── /docs                 → Documentation
├── /blog                 → Updates, tutorials
├── /compare
│   ├── /tailscale        → Detailed comparison
│   ├── /sdwan            → Detailed comparison
│   └── /zscaler          → Detailed comparison
├── /use-cases
│   ├── /remote-access
│   ├── /multi-cloud
│   └── /air-gapped
└── /contact              → Sales, support
```
