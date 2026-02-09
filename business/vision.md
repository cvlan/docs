# CloudVLAN Vision

## The Problem

Enterprises need secure, compliant network connectivity across clouds, data centers, and edge locations. Current options are broken:

**Traditional SD-WAN:**
- Expensive ($500-2000/site/month)
- Complex deployments (6-12 months)
- Proprietary hardware lock-in
- Over-engineered for actual needs

**Modern Mesh VPN (Tailscale, ZeroTier):**
- WireGuard-only (not compliant for many enterprises)
- SaaS-centric (self-hosted is afterthought)
- No IPsec for legacy device support
- Limited enterprise features

**Enterprise VPN (Zscaler, GlobalProtect):**
- Expensive licensing
- SaaS-only or requires specific hardware
- Complex to operate

## The Solution

CloudVLAN provides:

1. **Dual Protocol**: WireGuard when you want speed, IPsec when you need compliance
2. **First-Class Self-Hosting**: Same product, same features, your infrastructure
3. **Air-Gapped Support**: Fully offline operation for secure environments
4. **Cloud vRouters**: Managed IPsec concentrators without the hardware
5. **Radical Simplicity**: SD-WAN outcomes without SD-WAN complexity

## Core Beliefs

### Self-Hosted Should Be First-Class
Headscale exists because Tailscale didn't prioritize self-hosting. We believe self-hosted customers deserve the same product, not a community-maintained alternative.

### Compliance Shouldn't Require Complexity
IPsec is the compliant choice. It shouldn't require Cisco pricing or complexity to use it.

### Cost Efficiency is a Feature
If your infrastructure costs are 10-50x lower than competitors, you can price sustainably without VC pressure.

### No VC Extraction
We're building a sustainable business, not chasing ARR metrics. This means:
- Fair pricing that doesn't exploit customer lock-in
- Features driven by user needs, not investor pressure
- Long-term stability over hyper-growth

## Target Outcomes

| Metric | Target |
|--------|--------|
| Infrastructure cost at 100k nodes | < $2,000/month |
| Deployment time | Hours, not months |
| Protocol support | WireGuard + IKEv2/IPsec |
| Deployment options | SaaS, self-hosted, air-gapped |

## What Success Looks Like

**For Customers:**
- Connect any device, anywhere, securely
- Choose protocol based on needs, not vendor limitations
- Run on their infrastructure if they want
- Pay fair prices for real value

**For the Project:**
- Sustainable business that sells itself
- Or: valuable open-source contribution with documented learnings
