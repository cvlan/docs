# Client Load Simulator — Implementation Guide

Design document: [CLIENT-LOAD-SIMULATOR.md](CLIENT-LOAD-SIMULATOR.md)

## Location

```
cvlan/tests/client-load-simulator/
├── Cargo.toml
├── profiles/
│   ├── 2k-baseline.yaml        # 2k nodes, no chaos, 5min workload
│   └── 2k-chaos.yaml           # 2k nodes, 5% timeouts, 10% latency, flapping
├── sequences/
│   ├── fleet-rollout.yaml      # 500 nodes deployed in waves
│   ├── steady-state.yaml       # Low-rate background noise (joins/leaves/updates)
│   ├── client-churn.yaml       # Rapid connect/disconnect cycles
│   └── datacenter-drain.yaml   # Progressive removal + re-enrollment
└── src/
    ├── main.rs                 # CLI args, phase orchestration, workload runner
    ├── profile.rs              # YAML profile deserialization + validation
    ├── sequence.rs             # Sequence file parser + duration parsing
    ├── models.rs               # SimNode, ChangeEvent, PendingChanges, PollStats
    ├── api_client.rs           # AdminClient (JWT) + per-node mTLS client
    ├── topology.rs             # Build topology from profile, CIDR generation
    ├── setup.rs                # Create tenants/CVLANs/tokens, bulk register nodes
    ├── poller.rs               # Async poll loop, peer diff detection, self-instrumentation
    ├── injector.rs             # Change injection (join, leave, update, bulk, reconnect, disconnect)
    ├── probe.rs                # Immediate observer poll (capability metric)
    ├── chaos.rs                # Fault injection (poll timeout, latency, flapping)
    ├── metrics.rs              # HDR histograms, report tables, CLIENT_LOAD_METRICS line
    ├── intent.rs               # Per-node PRNG state machine (seeded, reproducible)
    ├── control.rs              # Interactive CLI + Unix socket control interface
    └── partition.rs            # Node slice assignment for distributed execution
```

## Running

```bash
# Via wrapper script (handles build, image rebuild, stack, JWT, everything)
cd ~/ws/cvlan
./scripts/test --client-load

# Manual (from inside Docker, after API is running)
client-load-simulator \
  --api-url http://cvlan-tests-api-1:8080 \
  --jwt-token "$JWT_TOKEN" \
  --profile profiles/2k-baseline.yaml \
  --cleanup
```

### CLI flags

| Flag | Description |
|------|-------------|
| `--api-url` | API base URL (default: `http://localhost:8080`) |
| `--jwt-token` | Admin JWT for tenant/CVLAN/node CRUD |
| `--profile` | Path to YAML profile (required) |
| `--sequence` | Sequence file(s), can repeat |
| `--seed N` | Deterministic replay seed |
| `--interactive` | Keep running, accept commands on stdin |
| `--control-socket /path` | Unix socket for scripted control |
| `--partition 1/3` | Distributed mode: this replica / total |
| `--skip-setup` | Reuse previous topology |
| `--cleanup` | Delete all created resources after |
| `--setup-concurrency N` | Parallel registrations (default: 50) |
| `--poll-interval N` | Override poll interval from profile |
| `--poll-mode forced-full\|realistic` | Override poll mode |

## Architecture

### Phase execution

```
Phase 1: Setup (sequential)
  ├── Create tenants via admin API (JWT auth)
  ├── Create CVLANs with CIDR from profile topology
  ├── Create registration tokens (1 per CVLAN, max_uses = node_count + 100)
  └── Register nodes via POST /client/register (concurrency 50)
      └── Each gets cert + key + ca_chain → stored in SimNode → builds mTLS reqwest::Client

Phase 2: Warm-up
  ├── Spawn one tokio task per node (poll_loop)
  ├── Each node polls at profile interval + jitter using its mTLS client
  └── Wait warmup_secs, check scheduling jitter is acceptable

Phase 3: Injection + Polling (concurrent)
  ├── Pollers: each node polls on its mTLS client, diffs peer sets
  ├── Injector: rate-controlled changes per workload profile
  │   └── node_join, node_leave, endpoint_update, bulk_enroll, reconnect, disconnect
  ├── Probes: immediate mTLS poll from affected CVLAN after each change
  └── Self-instrumentation: jitter, diff time, missed intervals

Phase 5: Drain + Report
  ├── Wait for all changes detected (or timeout)
  ├── Shutdown pollers via watch channel
  ├── Print capability, convergence, noise, self-check reports
  └── Cleanup (delete tenants, cascade) if --cleanup
```

### Connection model

Each simulated node gets its own `reqwest::Client` built from the PEM cert/key/ca returned by `/client/register`. This is the real mTLS channel — identical to what cvlan-ctrl uses. At 2k nodes = 2000 mTLS clients.

### Change detection

Each poller maintains `HashMap<Uuid, PeerSnapshot>`. After each poll:
1. Build current peer map from response
2. Diff against previous: `added`, `removed`, `changed` peers
3. Check `PendingChanges` registry (shared `DashMap`) for this CVLAN
4. On match: record detection time, compute latency

`PendingChanges` is the shared coordination point between injectors and pollers.

### Three metrics

| Metric | What it measures | How |
|--------|-----------------|-----|
| **Capability probe** | System floor (DB write → DB read → network) | Immediate poll after each change, `last_seen_version=0` |
| **Organic convergence** | Real-world propagation to ALL CVLAN members | Normal pollers detect changes at their natural cadence |
| **Noise ratio** | Waste quantification (keepalive vs useful polls) | Track every poll response type |

### Poll modes

- **forced-full** (default): Send `last_seen_version=0` every poll → server always returns full peer list → detect changes by diffing. Both probe and organic metrics work.
- **realistic**: Send real `last_seen_version` like cvlan-ctrl → demonstrates the 60s window bug where changes are invisible. Only probe metric works.

### Intent generator

Per-node seeded PRNG drives autonomous client behavior:
```
global_seed → hash(global_seed, node_index) → StdRng per node
```
Each poll interval, the node rolls against probabilities from the profile (reconnect, reboot, endpoint change, disconnect). Same seed = same simulation.

### Distributed mode

`--partition 1/3` splits nodes across containers. Last replica is the injector (runs setup, injects changes, fires probes). All others are pure pollers. Coordination via shared Docker volume (`/shared/changes/`, `/shared/results/`).

## Report output

```
======================================================================
CLIENT LOAD SIMULATOR RESULTS
======================================================================

Topology: 2 tenants, 105 CVLANs, 2000 nodes
Duration: 300s  Poll interval: 5s  Mode: forced-full

Capability (probe latency -- immediate poll after change):
┌───────────────────┬───────┬────────┬────────┬────────┐
│ Change Type       │ Count │ p50 ms │ p95 ms │ p99 ms │
├───────────────────┼───────┼────────┼────────┼────────┤
│ Node Join         │   100 │    XX  │    XX  │    XX  │
│ ...               │       │        │        │        │
│ ALL               │   500 │    XX  │    XX  │    XX  │
└───────────────────┴───────┴────────┴────────┴────────┘

Convergence (time until ALL affected CVLAN members detect change):
  [similar table with Success % column]

Noise Ratio:
  Total polls:           XXXXX
  Keepalive (no data):   XXXXX (XX.X%)
  Change-carrying:       XXXXX (X.X%)

Simulator Self-Check:
  Poll scheduling jitter:  p99 XX ms (target < 100ms)
  Diff computation:        p99 XX ms (target < 1ms)
  Missed poll intervals:   XX / XXXXX (target < 0.1%)

CLIENT_LOAD_METRICS: probe_p99_ms=XX converge_p99_ms=XX noise_pct=XX.X polls_sec=XXX changes=500/500 nodes=2000
```

The `CLIENT_LOAD_METRICS` line is machine-readable for the regression suite.

## Key dependencies

| Crate | Purpose |
|-------|---------|
| `x25519-dalek` | Proper Curve25519 WG keypairs (not random bytes) |
| `dashmap` | Lock-free concurrent map for PendingChanges |
| `hdrhistogram` | HDR histograms for p50/p95/p99 latency |
| `comfy-table` | Pretty report tables |
| `sysinfo` | CPU/memory self-instrumentation |
| `reqwest` (rustls-tls) | HTTP client with mTLS identity support |

## Interactive mode

```bash
client-load-simulator --profile 2k-baseline.yaml --interactive
```

Commands: `status`, `topology`, `chaos get/set/clear`, `help`, `quit`

Scriptable via Unix socket:
```bash
echo "status" | nc -U /tmp/sim.sock
echo "chaos set poll-timeout 5%" | nc -U /tmp/sim.sock
```

## Profile format

Profiles define topology (tenants, CVLANs, nodes per CVLAN, CIDR prefix), workload rates (per-minute for each change type), chaos controls (timeout/latency percentages, flapping), polling config (interval, mode, jitter), measurement config (probe enabled, detection timeout, warmup), and intent probabilities (reconnect, reboot, endpoint change, disconnect percentages).

See `profiles/2k-baseline.yaml` for the full annotated example.

## Sequence format

Sequences are time-ordered event scripts. Steps can use absolute timing (`at: "10s"`), relative timing (`after: "5s"`), or repeat (`repeat: 5, interval: "30s"`). Actions: `node_join`, `node_leave`, `endpoint_update`, `bulk_enroll`, `reconnect`, `disconnect`, `chaos`, `checkpoint`, `sleep`.

Target resolution: `"tenant_name/cvlan_name"`, `"tenant_name/*"` (random CVLAN in tenant), `"random"`.

See `sequences/fleet-rollout.yaml` for an example.

## What's not yet implemented

These are wired up structurally but need real server support or more complex logic:

- **Node move** (delete + re-register to different CVLAN) — injector stub exists
- **Cert renewal** tracking — server auto-renews on poll, but measurement not wired
- **Reboot** via intent generator — state machine exists, needs injector integration
- **Soak mode** (`--soak --duration 4h`) — flag parsed but loop not implemented
- **TCP control port** (`--control-port 9090`) — flag parsed, only Unix socket implemented
- **Shared volume coordination** for distributed mode — partition logic works, file I/O not implemented
- **Intent log** (`--intent-log`, `--replay`) — flags not yet added
- **VRouter events** — Phase 2 scope per design doc

## Workspace integration

- Added `"tests/client-load-simulator"` to `cvlan/Cargo.toml` workspace members
- Added `--client-load` flag to `cvlan/scripts/test` (builds API + simulator, starts test stack with step-ca, runs with 2k-baseline profile)
