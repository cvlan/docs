# CloudVLAN

**Next-generation overlay network with dual-protocol support, first-class self-hosting, and enterprise-grade compliance.**

## What is CloudVLAN?

CloudVLAN (cvlan) is a secure overlay network that connects devices, servers, and cloud resources across any network topology. Think of it as:

- **Tailscale** — but with IPsec support for compliance
- **SD-WAN** — but without the complexity and cost
- **Self-hosted first** — not a second-class open-source afterthought

## Key Features

| Feature | Description |
|---------|-------------|
| **Dual Protocol** | WireGuard for speed, IKEv2/IPsec for compliance |
| **Self-Hosted** | Run the entire stack on your infrastructure |
| **Air-Gapped** | Fully offline operation for secure environments |
| **Cloud vRouters** | Managed IPsec concentrators for hybrid connectivity |
| **Cloud Neutral** | Deploy on any cloud, colo, or on-premises |
| **Cost Efficient** | 100k nodes on <$2k/month infrastructure |

## Components

| Component | Description |
|-----------|-------------|
| `cvlan-controller` | Coordination server (auth, policy, node management) |
| `cvlan-agent` | Client agent for endpoints |
| `cvlan-relay` | Fallback relay for NAT traversal |
| `cvlan-vrouter` | IPsec concentrator / gateway |
| `cvlanctl` | CLI management tool |

## Documentation

- [Architecture](technical/architecture.md)
- [Technology Choices](technical/technology-choices.md)
- [Component Specifications](technical/components/)
- [Deployment Models](technical/deployment/)

## Project Status

This project is in active development. See [WORK_TRACKING.md](WORK_TRACKING.md) for current progress.

## License

TBD
