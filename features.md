# Features

## Current

### Network Management
- **Multi-tenancy** — Fully isolated tenants with separate CVLANs, users, policies, and audit trails
- **CVLAN creation** — Define virtual networks with IPv4 and IPv6 CIDR ranges
- **Automatic IP allocation** — Nodes receive IPs automatically on registration, no manual assignment
- **CIDR auto-assignment** — Create CVLANs without specifying a range, the system picks one from RFC 1918 space

### Node Onboarding
- **Token-based registration** — Generate a registration token, hand it to a node, it joins the right network automatically
- **Automatic certificate issuance** — Nodes receive mTLS certificates on registration, no manual PKI
- **Zero-touch discovery** — Nodes can discover the controller URL automatically via a discovery endpoint
- **Persistent identity** — Nodes survive restarts without re-registering

### Access Control
- **Role-based access** — Four roles: superadmin, tenant admin, member, read-only
- **Group-based policies** — Define allow/deny rules between groups of nodes
- **API key authentication** — Scoped keys for CLI and automation workflows
- **Audit logging** — Every state change is logged with 90-day retention and export

### Managed Gateways
- **VPP-powered vrouters** — Managed gateway nodes with kernel-bypass performance
- **Automatic configuration** — Gateways poll the controller and self-configure
- **Regional deployment** — Assign gateways to regions for geographic distribution
- **Bootstrap mode** — Run gateways in software mode for testing without specialized hardware

### Management
- **Web dashboard** — Full CRUD for tenants, CVLANs, nodes, users, groups, policies, gateways, and audit logs
- **Admin CLI** — `cvlanctl` for scripting and automation
- **Gateway CLI** — `vrouterctl` for gateway inspection and debugging
- **Background scheduler** — Automated housekeeping (audit log cleanup, node health checks, cert expiry warnings)

### Deployment Flexibility
- **Self-hosted** — Run the entire stack on your own infrastructure
- **Air-gapped** — Fully offline operation with no internet dependency
- **Single-box mode** — SQLite backend for simple deployments without a database server
- **Scaled mode** — PostgreSQL backend for multi-node, high-concurrency deployments
- **Tiered server modes** — Run discovery, registration, and session endpoints on separate infrastructure

### Security
- **mTLS everywhere** — All node-to-controller communication uses mutual TLS
- **Private keys stay local** — Node keys are generated on-device and never transmitted
- **Certificate rotation** — Short-lived certificates for gateways (8 hours), longer for clients (1 year)
- **Per-IP rate limiting** — Configurable rate limits on all public endpoints
- **No privileged containers** — All components run unprivileged

## Planned

### Connectivity
- **End-to-end WireGuard tunnels** — Encrypted mesh between all nodes in a CVLAN
- **NAT traversal** — Relay servers for nodes behind firewalls and NAT
- **Direct peer-to-peer** — STUN-based hole punching for direct connections when possible
- **Hub-and-spoke via gateways** — Route traffic through managed vrouters for site-to-site connectivity

### Protocols
- **IPsec (IKEv2)** — Compliance-ready tunnels for regulated environments (VPP infrastructure exists, not yet wired end-to-end)
- **Dual protocol** — Choose WireGuard or IPsec per connection based on requirements

### DNS & Discovery
- **Name resolution** — Resolve peer hostnames to their tunnel IP addresses
- **Split DNS** — Route DNS queries for CVLAN domains through the tunnel, everything else normally

### Identity
- **SSO / OIDC** — Federated authentication with external identity providers
- **Device approval workflows** — Admin approval before new nodes can join

### Client
- **Client debug CLI** — `cvlancli` for inspecting tunnel status, peer list, and connection health
- **Cross-platform packages** — Debian, RPM, and container images for Linux; macOS and Windows later

### Operations
- **Kubernetes operator** — Deploy and manage CloudVLAN components via custom resources
- **Terraform provider** — Infrastructure-as-code for tenant, CVLAN, and policy management
- **Prometheus metrics** — Application-level metrics export for monitoring
- **Webhook notifications** — Notify external systems on node join, policy change, and other events
