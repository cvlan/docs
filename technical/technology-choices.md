# Technology Choices

## Language: Rust

### Why Rust for Data Plane

| Reason | Detail |
|--------|--------|
| **No GC** | Predictable memory usage, no GC pauses |
| **Memory efficiency** | Use exactly what you allocate |
| **Performance** | Zero-cost abstractions, C-level speed |
| **Safety** | Memory safety without runtime overhead |
| **Async ecosystem** | Tokio is mature, handles thousands of connections |

### The GC Problem at Scale

Go and Java use garbage collection. At scale, this causes:

| Issue | Impact |
|-------|--------|
| **Heap bloat** | GC needs headroom, 2-3x actual usage |
| **GC pauses** | Latency spikes during collection |
| **Unpredictable sizing** | Can't accurately forecast memory needs |
| **Over-provisioning** | Must allocate extra = wasted spend |

**Example:**
| Scenario | Go relay | Rust relay |
|----------|----------|------------|
| Memory per 1k connections | ~500MB | ~100MB |
| Instance size needed | 2GB RAM | 512MB RAM |
| Monthly cost (VPS) | ~$20 | ~$5 |
| At 50 relays | $1,000/month | $250/month |

### Where Rust Is Used

| Component | Language | Rationale |
|-----------|----------|-----------|
| cvlan-relay | Rust | High throughput packet forwarding |
| cvlan-agent (data path) | Rust | Tunnel handling, packet routing |
| cvlan-vrouter | Rust | IPsec termination, high volume |
| cvlan-controller | Rust (or Go acceptable) | Lower volume, but consistency |
| cvlanctl | Rust | CLI, consistency with stack |

## Key Dependencies

### Async Runtime

**Tokio** - The standard for async Rust
- Mature, well-documented
- Excellent networking primitives
- Used by most Rust networking projects

### Cryptography

**ring** or **rustls** for TLS
- ring: Fast, audited primitives
- rustls: Pure Rust TLS, no OpenSSL dependency

**snow** for Noise protocol (control plane auth)

### WireGuard

**boringtun** (Cloudflare) or **wireguard-rs**
- boringtun: Production-tested at Cloudflare
- Pure Rust userspace WireGuard

### IPsec

Options under evaluation:
1. **FFI to strongSwan** - Mature, proven, but C dependency
2. **Native Rust** - Cleaner, but significant effort
3. **Wrap existing kernel IPsec** - Linux only

Likely approach: Start with strongSwan FFI, consider native later

### Serialization

**serde** - Universal Rust serialization
- JSON for configs and APIs
- Binary format (bincode/postcard) for wire protocol

### CLI

**clap** - Standard Rust CLI parser

### Error Handling

- **thiserror** for library error types
- **anyhow** for application error handling

### Logging/Tracing

**tracing** - Structured logging and distributed tracing

## Protocol Choices

### Control Plane Protocol

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Transport | HTTPS | Universal, firewall-friendly |
| Auth | Noise_IK | Mutual auth, forward secrecy |
| Encoding | JSON (API), binary (streaming) | JSON for debug, binary for efficiency |

### Data Plane Protocols

| Protocol | Use Case |
|----------|----------|
| WireGuard | Default, modern endpoints |
| IKEv2/IPsec | Compliance, legacy devices |

### Relay Protocol

Custom lightweight protocol:
- Frame-based over TCP/WebSocket
- Node addressing by public key
- Encrypted payload pass-through

## Database

### Options

| Option | Pros | Cons |
|--------|------|------|
| **PostgreSQL** | Robust, scalable, familiar | Requires separate deployment |
| **SQLite** | Embedded, simple, no ops | Limited concurrency |
| **etcd** | Distributed, K8s native | Overkill for most deployments |

### Recommendation

- **Small deployments**: SQLite (embedded, zero ops)
- **Large/SaaS**: PostgreSQL (managed or self-hosted)
- Abstract database layer to support both

## Infrastructure

### Cloud Neutral

No cloud-specific services:
- ❌ AWS ALB, RDS, Lambda
- ❌ GCP Cloud SQL, Cloud Run
- ✅ Containers (Docker/Podman)
- ✅ Kubernetes (optional)
- ✅ Standard Linux VMs
- ✅ nginx/HAProxy for load balancing
- ✅ Self-managed Postgres

### Deployment Artifacts

| Artifact | Format |
|----------|--------|
| Binaries | Static Linux binaries (musl) |
| Containers | OCI images (Docker compatible) |
| Packages | deb, rpm for Linux distros |
| Helm charts | For Kubernetes deployments |

## What We DON'T Build

Leverage existing, proven technology:

| Component | Use Instead of Building |
|-----------|------------------------|
| WireGuard protocol | boringtun / wireguard-rs |
| IPsec/IKEv2 | strongSwan (initially) |
| NAT traversal | STUN/ICE libraries |
| TLS | rustls |
| Cryptographic primitives | ring |

Focus engineering effort on:
- Control plane logic
- Policy engine
- Multi-tenancy
- Integration and UX
