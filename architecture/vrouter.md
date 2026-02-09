# VRouter (vrouterd)

vrouterd is a VPP-based virtual router that acts as a managed gateway. It provides WireGuard and IPsec termination using VPP's kernel-bypass data plane, achieving line-rate performance (400Gbps capable) without touching the Linux kernel networking stack.

## Architecture

vrouterd runs as a long-running daemon alongside a VPP instance. It:

1. **Registers** with the controller using a join token
2. **Receives** its configuration (WireGuard peers, routes, interfaces) via mTLS polling
3. **Reconciles** the desired state from the controller with VPP's actual state using DiffSync
4. **Reports** statistics back to the controller on each poll

## Registration Flow

![VRouter Registration Flow](../images/vrouter-registration-flow.svg)

1. Admin creates a VRouter entity in the API and gets a join token
2. vrouterd starts with the join token in its config
3. vrouterd calls `POST /vrouter/register` with the token + WireGuard public key
4. Controller verifies the token, creates a CSR, and requests a certificate from step-ca
5. Controller returns: mTLS certificate, client key, CA chain, assigned config
6. vrouterd saves certs to disk (key file with 0o600 permissions)
7. vrouterd creates an mTLS HTTP client and begins polling

Certificate validity: 8 hours (short-lived, forces regular re-registration for security).

## Poll Loop + DiffSync

After registration, vrouterd polls the controller every 5 seconds:

```
Poll Request:
  - MAC address + nonce (replay protection)
  - Current endpoint list
  - Last seen config version

Poll Response:
  - Delta peer changes (added/removed)
  - Route updates
  - Interface configuration
  - VPP-specific parameters
```

**DiffSync reconciliation**: On each poll response, vrouterd compares the desired state (from controller) with VPP's actual state (queried via binary API). It then applies only the differences:
- Add missing WireGuard peers
- Remove stale peers
- Update routes
- Adjust interface configuration

This ensures VPP's state converges to the controller's intent without full resets.

## VPP Integration

vrouterd communicates with VPP via the binary API over a Unix socket:

| VPP Feature | Usage |
|-------------|-------|
| **WireGuard** | Tunnel termination for client nodes |
| **IPsec (IKEv2)** | Compliance tunnels for enterprise sites |
| **DPDK** | Kernel-bypass NIC drivers for line-rate performance |
| **VRF** | Traffic isolation between CVLANs |

**VPP 24.10** socket API protocol: vrouterd sends binary-encoded messages and receives responses. No shared memory needed (unlike older VPP API bindings).

## Bootstrap Mode

For testing and development without VPP hardware:

```bash
vrouterd --bootstrap
```

In bootstrap mode:
- Software WireGuard (no VPP, no DPDK)
- No privileged container needed
- Suitable for E2E tests and local development
- Same registration + poll loop, just different tunnel backend

## Local State

vrouterd maintains state in SQLite (WAL mode):

| Table | Purpose |
|-------|---------|
| `controller_state` | Node ID, assigned IP, WG pubkey, controller URL |
| `interfaces` | VPP interfaces with sw_if_index, type, endpoints |
| `routes` | Routing table (prefix, next_hop, VRF) |
| `sync_state` | Last seen version, last poll timestamp |

On restart, vrouterd checks if certificates exist on disk. If yes, it resumes polling with the mTLS client. If not, it re-registers.

## vrouterctl

Debug CLI for inspecting VPP state:

```bash
vrouterctl show interfaces    # List VPP interfaces
vrouterctl show peers          # WireGuard peer table
vrouterctl show routes         # Routing table
vrouterctl show stats          # VPP counters
```

Connects to vrouterd via Unix socket IPC.

## Repository Structure

```
vrouter/
├── vrouterd/           # Gateway daemon
│   └── src/
│       ├── main.rs     # Bootstrap → register → poll loop
│       ├── config.rs   # Configuration (YAML)
│       ├── state.rs    # SQLite state
│       └── api/        # HTTP client (discover, register, poll)
├── vrouterctl/         # Debug CLI
├── vrouter-gen/        # VPP API bindings generator
└── build/              # Docker build infrastructure
```

**Tech stack**: Rust + VPP 24.10 + rusqlite + reqwest + rustls
