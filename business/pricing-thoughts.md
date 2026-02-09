# Pricing Thoughts

> Early thinking on pricing model. Will evolve based on market feedback.

## Guiding Principles

1. **Fair pricing**: Don't exploit lock-in
2. **Sustainable**: Must cover costs + margin at break-even point
3. **Simple**: Easy to understand and predict
4. **Self-hosted friendly**: Don't punish people for running their own infra

## Cost Structure Reality

### Infrastructure Costs (at scale)

| Component | 100k nodes | Cost/month |
|-----------|------------|------------|
| Control plane (3x 4GB nodes) | Shared | ~$100 |
| Database (self-managed Postgres) | Shared | ~$100 |
| Relay servers (50x small VPS) | ~2k nodes each | ~$250 |
| Bandwidth (relay traffic) | Variable | ~$100-500 |
| Monitoring | Shared | ~$50 |
| **Total** | | **~$600-1000** |

At 100k nodes and ~$800 infrastructure cost:
- **Cost per node: ~$0.008/month**
- Target infrastructure budget ($2k) gives 60% headroom

### Break-Even Analysis

If break-even target is "few hundred nodes" with minimal fixed costs:
- 500 nodes × $5/node = $2,500/month revenue
- Fixed costs (control plane, etc.): ~$200-300/month
- Leaves healthy margin for continued development

## Pricing Model Options

### Option A: Open Core

| Tier | Price | Includes |
|------|-------|----------|
| **Community** | Free | Core networking, WireGuard, basic ACLs, self-hosted only |
| **Pro** | $5/node/month | IPsec, advanced ACLs, priority support |
| **Enterprise** | Custom | Air-gapped, vRouters, SLAs, dedicated support |

**Pros**: Builds community, low barrier to adoption
**Cons**: Free tier might be "good enough" for most

### Option B: Self-Hosted License

| Tier | Price | Includes |
|------|-------|----------|
| **Starter** | $500/year | Up to 100 nodes, all features |
| **Business** | $2,000/year | Up to 1,000 nodes, all features |
| **Enterprise** | $10,000/year | Unlimited nodes, support, SLAs |

**Pros**: Predictable revenue, simple
**Cons**: Per-seat resistance, harder to start small

### Option C: Hybrid (Leaning Toward)

| Offering | Model |
|----------|-------|
| **Self-hosted core** | Open source (Apache 2.0 or similar) |
| **SaaS** | Per-node pricing, managed everything |
| **Enterprise add-ons** | License for: vRouter, air-gapped, advanced features |
| **Support** | Paid support tiers |

**Rationale**:
- Open source core builds trust and adoption
- SaaS for those who don't want to operate
- Enterprise add-ons for differentiated value
- Support as natural upsell

## vRouter Pricing

vRouters are a natural premium offering:

| Size | Capacity | Price |
|------|----------|-------|
| Small | 100 tunnels | $50/month |
| Medium | 500 tunnels | $150/month |
| Large | 2,000 tunnels | $400/month |

Managed vRouters include:
- Deployment in customer's cloud or ours
- Monitoring and alerting
- Automatic updates
- HA configuration

## What NOT to Do

- **Per-feature nickel-and-diming**: Breeds resentment
- **Usage-based pricing that's unpredictable**: Enterprises hate surprises
- **Artificially limiting self-hosted**: Defeats the value prop
- **Requiring phone calls for pricing**: Just publish it

## Open Questions

- [ ] What's the right balance of open source vs. paid?
- [ ] Should IPsec be free or premium?
- [ ] How to price air-gapped (higher support burden)?
- [ ] Geographic pricing differences?

## Comparison to Market

| Vendor | Typical Cost (1000 nodes) | CloudVLAN Target |
|--------|---------------------------|------------------|
| Cisco SD-WAN | $50,000+/month | |
| Tailscale | ~$6,000/month (business) | |
| CloudVLAN SaaS | | ~$5,000/month |
| CloudVLAN Self-hosted | | ~$2,000/year license |
