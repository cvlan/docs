# Technology Choices

## Language: Rust

### Why Rust for Everything

| Reason | Detail |
|--------|--------|
| **No GC** | Predictable memory usage, no GC pauses |
| **Memory efficiency** | Use exactly what you allocate |
| **Performance** | Zero-cost abstractions, C-level speed |
| **Safety** | Memory safety without runtime overhead |
| **Async ecosystem** | Tokio is mature, handles thousands of connections |

### The GC Problem at Scale

Go and Java use garbage collection. At scale, this creates unpredictable costs:

| Issue | Impact |
|-------|--------|
| **Heap bloat** | GC needs headroom, 2-3x actual usage |
| **GC pauses** | Latency spikes during collection |
| **Unpredictable sizing** | Can't accurately forecast memory needs |
| **Over-provisioning** | Must allocate extra = wasted spend |

### Where Rust Is Used

| Component | Binary | Rationale |
|-----------|--------|-----------|
| cvlan-api | cvlan-api | Consistency, performance at scale (21K ops/s) |
| cvlanctl | cvlanctl | CLI, same crate dependencies as API |
| vrouterd | vrouterd | VPP binary API binding, FFI safety |
| vrouterctl | vrouterctl | CLI, consistency |
| cvlan-ctrl | cvlan-ctrl | Tiny footprint control plane daemon |
| cvland | cvland | WireGuard data path, minimal overhead |
| cvlancli | cvlancli | CLI, consistency |

### UI Exception

The management UI is TypeScript/React — the right tool for web UIs. Types are generated from Protocol Buffers to stay in sync with Rust.

## Key Dependencies

### Core Stack

| Dependency | Version | Purpose |
|------------|---------|---------|
| **tokio** | 1.x | Async runtime |
| **axum** | 0.7 | HTTP framework (cvlan-api) |
| **reqwest** | 0.12 | HTTP client (vrouterd, cvlan-ctrl) |
| **rustls** | 0.23 | TLS (no OpenSSL dependency) |
| **serde** | 1.x | Serialization (JSON, YAML) |
| **clap** | 4.x | CLI argument parsing |
| **tracing** | 0.1 | Structured logging |
| **thiserror** | 2.x | Library error types |
| **anyhow** | 1.x | Binary error handling |

### Database

| Dependency | Version | Purpose |
|------------|---------|---------|
| **sqlx** | 0.8 | PostgreSQL async driver (compile-time verified queries) |
| **rusqlite** | 0.32 | SQLite for local state (bundled, zero system deps) |

SQLx 0.8 provides compile-time query verification against a real database. rusqlite is used with the `bundled` feature flag — it compiles SQLite from source, so there's no system dependency.

### Cryptography

| Dependency | Version | Purpose |
|------------|---------|---------|
| **x25519-dalek** | 2.x | Curve25519 WireGuard key generation |
| **ed25519-dalek** | 2.x | Ed25519 token signing/verification |
| **sha2** | 0.10 | SHA-256 for MAC/nonce hashing |
| **argon2** | 0.5 | Password and API key hashing |
| **rand** | 0.8 | Cryptographic random number generation |
| **base64** | 0.22 | Key encoding |

### Infrastructure

| Dependency | Purpose |
|------------|---------|
| **step-ca** | Lightweight ACME CA for mTLS certificates |
| **VPP 24.10** | Kernel-bypass packet processing (vrouter) |
| **PostgreSQL 16** | Primary database for multi-node deployments |
| **Docker** | Container runtime for all builds and deployments |

### Type System

| Dependency | Purpose |
|------------|---------|
| **Protocol Buffers** | Single source of truth for shared types |
| **buf** (v1.28.1) | Proto management and code generation |
| **prost** | Rust protobuf codegen |
| **prost-serde** | Rust serde derives for protobuf types |
| **ts-proto** | TypeScript protobuf codegen |

## Protocol Choices

### Control Plane

| Layer | Choice | Status |
|-------|--------|--------|
| Transport | HTTPS | Working |
| Auth (management) | JWT (HS256) + API keys | Working |
| Auth (nodes) | Ed25519 tokens → mTLS | Working |
| Auth (future) | Noise_IK | Deferred |
| Encoding | JSON (REST API) | Working |
| Types | Protocol Buffers | Working |

**Noise protocol** was originally planned for control plane auth but deferred. The current mTLS + JWT approach works well and is simpler to debug.

### Data Plane

| Protocol | Use Case | Status |
|----------|----------|--------|
| WireGuard | Default tunnel protocol | VPP: working, Client: stub |
| IKEv2/IPsec | Compliance (enterprise) | VPP supports it, not wired end-to-end |

### IPsec Approach (Updated)

The original plan was strongSwan FFI. The actual implementation uses **VPP's built-in IKEv2/IPsec** — no external dependency, kernel-bypass performance, and VPP handles all the protocol negotiation internally.

## Database Architecture

### Dual-Backend

| Backend | Use Case | Driver |
|---------|----------|--------|
| **PostgreSQL** | Multi-node, SaaS, high concurrency | SQLx 0.8 (async, compile-time verified) |
| **SQLite** | Single-box self-hosted, embedded | rusqlite (bundled) |

Both backends share the same schema. SQLite-specific adaptations:
- UUIDs stored as 16-byte BLOBs (not TEXT)
- Timestamps as INTEGER (Unix epoch)
- `busy_timeout(5000)` for write contention

### Local State (rusqlite)

vrouterd and cvlan-ctrl each maintain a local SQLite database (WAL mode) for:
- Node identity persistence (survives restart)
- Peer cache (operates with stale data if controller unreachable)
- Sync state (delta polling checkpoint)

## Infrastructure

### Cloud Neutral

No cloud-specific services. Runs on any Linux box:
- Docker containers for all components
- PostgreSQL (managed or self-hosted)
- nginx/HAProxy for load balancing
- Standard Linux VMs or bare metal

### Build System

All builds happen inside Docker singleton containers (never cargo/npm on host):
- `cvlan-build`: Rust + SQLx + protoc + buf + clippy + rustfmt
- `vrouter-build`: Rust + VPP 24.10 libs + DPDK
- `cvlan-ui-build`: Node 20 + npm + Vite + React
- `client-build`: Rust + clang + lld (cross-compilation)

### Deployment Artifacts

| Artifact | Format | Status |
|----------|--------|--------|
| Docker images | OCI (multi-stage) | Working |
| Debian packages | .deb | Client only (in progress) |
| Static binaries | Linux (musl) | Future |
| Helm charts | K8s | Future |
