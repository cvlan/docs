# Security Model

## Trust Boundaries

![Trust Boundaries](../images/trust-boundaries.svg)

The system has four trust tiers, each with increasing access requirements:

| Tier | Access | Auth Required | Exposure |
|------|--------|---------------|----------|
| **Internet** | Discovery endpoint only | None (rate-limited) | Public port 443 |
| **Registration** | Register new nodes | Ed25519 signed token | Semi-public |
| **Session** | Poll for updates | mTLS X.509 certificate | Internal only |
| **Data Plane** | Peer-to-peer traffic | WireGuard tunnel keys | End-to-end encrypted |

## Key Management

### Private Keys Never Leave Nodes

- WireGuard private keys are generated locally (x25519-dalek)
- mTLS private keys are received during registration and stored locally (0o600)
- The controller never stores node private keys
- CA chain is distributed, but CA private key stays on step-ca

### Key Types

| Key | Algorithm | Where Stored | Purpose |
|-----|-----------|-------------|---------|
| WG keypair | Curve25519 (x25519-dalek) | Node local storage | WireGuard tunnels |
| mTLS cert + key | X.509 (step-ca issued) | Node /var/lib/*/certs/ | Control plane auth |
| Tenant signing key | Ed25519 | Admin custody (offline OK) | Token signing |
| JWT secret | HS256 | Server environment variable | Management API auth |
| API key hash | Argon2 | PostgreSQL | CLI/automation auth |
| Password hash | Argon2 | PostgreSQL | User login |

## Certificate Pinning

Nodes pin the CA chain received during registration. Subsequent mTLS connections verify the controller's certificate against the pinned CA, preventing MITM attacks even if a public CA is compromised.

## Replay Protection

Every registration and poll request includes:
- **MAC address** — Binds the request to a physical device
- **Nonce** — Fresh 32-byte random hex on every request
- **Binding** — SHA256(mac + nonce) in certificate extension

This prevents:
- Captured requests being replayed
- Certificates being reused across devices
- Token reuse after initial registration

## Container Security

### No Privileged Containers

All CloudVLAN containers run unprivileged:
- No `--privileged` flag
- No host filesystem mounts (except source code volumes in build containers)
- No `CAP_SYS_ADMIN` or other dangerous capabilities

The only exception is VPP in vrouter, which needs DPDK access (hugepages, NIC passthrough). Bootstrap mode avoids this for testing.

### No Third-Party Monitoring Images

No cAdvisor, Prometheus node-exporter, or similar images with broad host access:
- Even read-only mounts expose: SSH keys, Docker socket, /etc/shadow, credentials
- Monitoring is done via application-level metrics (exported via API)

### Build Container Isolation

Build containers (cvlan-build, vrouter-build, etc.) mount only source directories. They never mount:
- `~/.ssh/`
- `~/.aws/`
- Docker socket
- Host /etc or /var

## Multi-Tenant Isolation

### Query-Level Isolation

Every database query includes `WHERE tenant_id = $1`. There is no query path that returns data across tenants (except for superadmin global operations).

### CIDR Isolation

Two tenants can use the same CIDR range (e.g., both use 10.0.0.0/24) without conflict. IP allocation is per-CVLAN, and CVLANs are tenant-scoped.

### Audit Trail

All state-changing operations are logged with tenant_id, user_id, action, and details. Logs are retained for 90 days and can be exported for compliance.

## Network Security

### TLS Everywhere

All control plane communication uses TLS (rustls, no OpenSSL):
- Management API: HTTPS with JWT
- Node registration: HTTPS
- Node polling: mTLS (mutual TLS)
- step-ca communication: HTTPS

### Data Plane Encryption

All data plane traffic is encrypted:
- **WireGuard**: ChaCha20-Poly1305 (default)
- **IPsec**: Configurable cipher suites (AES-GCM, etc.)

No plaintext traffic crosses network boundaries.

## Threat Model

| Threat | Mitigation |
|--------|------------|
| DDoS on discovery | Tiered architecture, separate server modes |
| Token theft | Time-limited, one-time use, server-authoritative CVLAN |
| Certificate theft | Short validity (8hr for vrouters), MAC binding |
| Insider threat | RBAC with least privilege, audit logging |
| Cross-tenant data leak | Query-level tenant isolation, no shared queries |
| Man-in-the-middle | mTLS, certificate pinning, WireGuard encryption |
| Replay attack | Fresh nonce per request, MAC binding |
| Key compromise | Keys don't leave nodes, certificate rotation |
