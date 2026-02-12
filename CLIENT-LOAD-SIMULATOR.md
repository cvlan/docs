# Client Load Simulator

## Context

Change distribution is a core architectural element of any SDN system. We need to measure how quickly network state changes propagate from the controller to all affected clients via the poll protocol. Good systems achieve 1-2 second p99; some successful products take >2 hours for 6k nodes.

**Critical finding from code review**: The current poll handler has a 60-second "freshness" window (`coordination.rs:387`): if `last_seen_version >= version - 60`, it returns keepalive with NO change data. Since cvlan-ctrl polls every 30 seconds and sends `last_seen_version`, clients **always** hit this window and **never** see changes after initial registration. This is the #1 bottleneck the simulator will quantify.

## Change Taxonomy

Changes come from two directions: **server-side** (admin actions, infrastructure events) and **client-side** (node behavior, network conditions).

### Server-Side Changes (admin/infrastructure)

| Change Type | Description | Blast Radius | Detection Signal |
|-------------|-------------|--------------|-----------------|
| Node join | New node registers via /client/register | Peers in same CVLAN | New node_id in peer list |
| Node leave | Admin deletes/disables a node | Peers in same CVLAN | node_id disappears from peer list |
| Node move | Admin moves node to different CVLAN | Source + destination CVLAN peers | Removal in source, addition in dest |
| Endpoint update | Admin updates node endpoints/metadata | Peers in same CVLAN | PeerInfo.endpoints changed |
| Bulk enrollment | Fleet deployment: many nodes register at once | Target CVLAN peers | Multiple new node_ids appear |
| Policy change | ACL rules updated (future: packet_filter) | All nodes in affected CVLAN | packet_filter field in poll response |

### Client-Side Changes (node behavior)

| Change Type | Description | Blast Radius | Detection Signal |
|-------------|-------------|--------------|-----------------|
| Client reconnect | Node goes offline briefly, resumes polling | Peers see status change (offline→online) | PeerInfo.online toggling |
| Client reboot | Node restarts, re-registers or uses cached certs | Peers may see brief offline then online | Status toggle + potentially new endpoints |
| Client churn | Rapid connect/disconnect (flapping) | Peers in same CVLAN | Repeated status changes |
| Cert renewal | Auto-renewal at 50% cert validity | None (transparent to peers) | Measures controller cert-minting throughput |
| Stale node timeout | Node stops polling, background job marks offline | Peers in same CVLAN | PeerInfo.online → false |

### VRouter Events (high-impact, multi-tenant)

VRouters are fundamentally different from client nodes — they serve **multiple CVLANs across multiple tenants**. A single vrouter event has much higher blast radius.

| Change Type | Description | Blast Radius | Detection Signal |
|-------------|-------------|--------------|-----------------|
| VRouter restart | VRouter comes back online, must sync ALL assigned CVLANs | All nodes in all assigned CVLANs (potentially thousands) | Burst of peer change records consumed via sequence-based sync |
| VRouter failover | VRouter goes offline, another takes over | All CVLANs on both old and new vrouter | Mass CVLAN reassignment, peer updates for new gateway |
| VRouter scale-out | New vrouter added, CVLANs redistributed | CVLANs being moved to new vrouter | Gateway change for affected CVLANs |
| VRouter maintenance | Graceful drain of CVLANs before shutdown | CVLANs being migrated | Sequential CVLAN reassignment |

**Key difference**: Vrouter sync uses the sequence-based `vrouter_peer_changes` table (incremental, per-vrouter sequence numbers). Client poll uses the timestamp-based version check (60s window). These are different code paths with different performance characteristics.

## Scenario Profile

The simulator reads a YAML profile that defines topology, workload rates, and chaos controls. CLI flags can override individual values.

### Profile Structure

```yaml
# Profile: 2k-baseline
name: "2k-baseline"
description: "Baseline measurement at 2k nodes, no chaos"

# ─── Topology ───────────────────────────────────────────────
# Defines the static structure created during setup phase.
topology:
  tenants:
    - name: "large-vlans"
      cvlan_count: 5
      nodes_per_cvlan: 200
      cidr_prefix: 16          # /16 subnets (large IP space)
    - name: "small-vlans"
      cvlan_count: 100
      nodes_per_cvlan: 10
      cidr_prefix: 24          # /24 subnets (252 usable IPs)
  # Derived: 2 tenants, 105 CVLANs, 2000 nodes

# ─── Workload ───────────────────────────────────────────────
# Rates of operations during the injection phase.
# All rates are per minute. Duration controls how long injection runs.
workload:
  duration_secs: 300           # injection phase length

  # Server-side changes (via admin API with JWT)
  api:
    node_join_per_min: 20      # new registrations via /client/register
    node_leave_per_min: 16     # DELETE /tenants/:tid/nodes/:nid
    node_move_per_min: 6       # delete + re-register to different CVLAN
    endpoint_update_per_min: 12 # PATCH node endpoints/metadata
    bulk_enroll_per_min: 4     # burst events (each adds bulk_size nodes)
    bulk_enroll_size: [5, 10]  # uniform random range per event

  # Client-side changes (simulated behavior)
  client:
    reconnect_per_min: 16      # pause polling N seconds, then resume
    reconnect_duration: [5, 30] # seconds offline (uniform random)
    reboot_per_min: 8          # stop + re-register with new WG key
    cert_renewal_per_min: 10   # force cert renewal check
    disconnect_per_min: 5      # go offline permanently (stale node)

# ─── Chaos Controls ────────────────────────────────────────
# Fault injection knobs. All default to 0 (disabled).
chaos:
  # Network faults
  poll_timeout_pct: 0          # % of polls that timeout (simulate network loss)
  poll_latency_pct: 0          # % of polls with injected delay
  poll_latency_ms: [100, 500]  # delay range when injected

  # Infrastructure faults
  step_ca_failure_pct: 0       # % of cert minting attempts that fail
  db_contention_pct: 0         # % of admin API calls that get 503/retry

  # Client faults
  client_flap_per_min: 0       # rapid on/off cycles (register, poll once, disconnect)
  stale_node_pct: 0            # % of nodes that silently stop polling at start
  wrong_token_pct: 0           # % of registrations with invalid/expired token

# ─── Polling ───────────────────────────────────────────────
polling:
  interval_secs: 5             # how often each node polls
  mode: "forced-full"          # "forced-full" or "realistic"
  jitter_ms: 500               # random jitter added to interval (avoids thundering herd)

# ─── Measurement ───────────────────────────────────────────
measurement:
  probe_enabled: true          # fire immediate observer poll after each change
  detection_timeout_secs: 120  # max wait for organic convergence
  warmup_secs: 5               # polling warm-up before injection starts
```

### Preset Profiles

| Profile | Nodes | Chaos | Purpose |
|---------|-------|-------|---------|
| `2k-baseline.yaml` | 2,000 | None | Establish baseline metrics |
| `2k-chaos.yaml` | 2,000 | Network faults, flapping | Resilience under failure |
| `10k-steady.yaml` | 10,000 | None | First scale milestone |
| `100k-steady.yaml` | 100,000 | None | Target scale |
| `100k-burst.yaml` | 100,000 | Bulk enrollment spikes | Fleet deployment scenario |

### Example: 2k with chaos

```yaml
name: "2k-chaos"
topology:
  tenants:
    - { name: "large-vlans", cvlan_count: 5, nodes_per_cvlan: 200, cidr_prefix: 16 }
    - { name: "small-vlans", cvlan_count: 100, nodes_per_cvlan: 10, cidr_prefix: 24 }
workload:
  duration_secs: 300
  api:
    node_join_per_min: 20
    node_leave_per_min: 16
    node_move_per_min: 6
    endpoint_update_per_min: 12
    bulk_enroll_per_min: 4
    bulk_enroll_size: [5, 10]
  client:
    reconnect_per_min: 30      # higher reconnect rate
    reconnect_duration: [2, 60] # wider range
    reboot_per_min: 10
    cert_renewal_per_min: 15
    disconnect_per_min: 10
chaos:
  poll_timeout_pct: 5          # 5% of polls timeout
  poll_latency_pct: 10         # 10% get 100-500ms extra latency
  poll_latency_ms: [100, 500]
  client_flap_per_min: 20      # 20 rapid on/off cycles per minute
  stale_node_pct: 2            # 2% of nodes go stale at start
  wrong_token_pct: 1           # 1% bad registrations
polling:
  interval_secs: 5
  mode: "forced-full"
  jitter_ms: 1000
measurement:
  probe_enabled: true
  detection_timeout_secs: 180  # longer timeout for chaos
  warmup_secs: 10
```

### Distribution Controls

The parent-child distribution is defined in the topology section:

```
Tenant "large-vlans"  ──→  5 CVLANs  ──→  200 nodes each  = 1000 nodes
Tenant "small-vlans"  ──→  100 CVLANs ──→  10 nodes each   = 1000 nodes

Total: 2 tenants, 105 CVLANs, 2000 nodes
Avg nodes/CVLAN: 19 (skewed: 95% of CVLANs are small)
Largest CVLAN: 200 nodes (heaviest convergence target)
Smallest CVLAN: 10 nodes (fastest convergence, but most numerous)
```

At 100k scale:
```
Tenant "enterprise-1"  ──→  20 CVLANs  ──→  500 nodes each  = 10,000 nodes
Tenant "enterprise-2"  ──→  50 CVLANs  ──→  100 nodes each  = 5,000 nodes
...8 more tenants with varying distributions...
~20 tenants, ~2000 CVLANs, ~100k nodes, avg 50 nodes/CVLAN
```

## Sequences

Sequences are time-ordered event scripts in YAML files. They define specific scenarios that play out over the running simulation. Composable — run one or chain several.

### Sequence File Format

```yaml
# sequences/fleet-rollout.yaml
name: "Fleet Rollout"
description: "500 new devices deployed in waves to a large CVLAN"

steps:
  # Absolute timing (from sequence start)
  - at: 0s
    action: bulk_enroll
    target: "large-vlans/cvlan-1"    # tenant_name/cvlan_name
    params:
      count: 50

  - at: 10s
    action: bulk_enroll
    target: "large-vlans/cvlan-1"
    params:
      count: 100

  # Relative timing (after previous step completes)
  - after: 5s
    action: endpoint_update
    target: random                   # pick random existing node(s)
    params:
      count: 20

  # Repeated pattern
  - repeat: 5
    interval: 30s
    action: node_leave
    target: random
    params:
      count: 10

  # Chaos window — activate for a duration, then auto-revert
  - at: 120s
    action: chaos
    params:
      poll_timeout_pct: 10
      duration: 60s

  # Checkpoint — wait for all pending changes to be detected before continuing
  - action: checkpoint
    timeout: 30s
```

### Actions

| Action | Params | Description |
|--------|--------|-------------|
| `node_join` | `target`, `count` | Register new nodes via /client/register |
| `node_leave` | `target` or `random`, `count` | Delete nodes via admin API |
| `node_move` | `target`, `dest` | Move nodes between CVLANs |
| `endpoint_update` | `target` or `random`, `count` | Patch node endpoints |
| `bulk_enroll` | `target`, `count` | Burst registration to one CVLAN |
| `reconnect` | `target` or `random`, `count`, `duration` | Pause/resume polling |
| `reboot` | `target` or `random`, `count` | Stop + re-register with new key |
| `cert_renewal` | `target` or `random`, `count` | Force cert renewal check |
| `disconnect` | `target` or `random`, `count` | Permanent offline (stale) |
| `chaos` | chaos knobs + `duration` | Enable chaos controls temporarily |
| `checkpoint` | `timeout` | Wait for all pending detections |
| `sleep` | `duration` | Pause sequence execution |

### Target Resolution

- `"large-vlans/cvlan-1"` — specific CVLAN by tenant/name
- `"large-vlans/*"` — random CVLAN in tenant
- `"random"` — random node from entire topology
- `"random:large-vlans"` — random node from tenant
- `"random:large-vlans/cvlan-1"` — random node from specific CVLAN

### Sequence Library

```
sequences/
├── fleet-rollout.yaml         # Bulk enrollment in waves
├── datacenter-drain.yaml      # Progressive node removal (maintenance)
├── network-partition.yaml     # Chaos: timeout % of polls for N minutes
├── cert-rotation-storm.yaml   # Many certs renewing simultaneously
├── vlan-migration.yaml        # Move nodes between CVLANs in batches
├── client-churn.yaml          # Rapid connect/disconnect cycles
├── day-in-the-life.yaml       # Composite: startup, activity, maintenance window
└── steady-state.yaml          # Background noise (low-rate joins/leaves)
```

### How Sequences Relate to Profiles

- **Profile** = topology + background workload rates + chaos defaults. Static. Defines the world.
- **Sequence** = specific event script played over the running world. Dynamic. Defines what happens.
- **Multiple sequences** can be composed: `--sequence fleet-rollout.yaml --sequence network-partition.yaml`

In batch mode, sequences replace the `workload` section of the profile. In interactive mode, sequences can be triggered on demand.

## Intent Generator

Each simulated client has an autonomous behavior model driven by a per-node PRNG. This produces emergent, realistic network activity without hand-writing sequences for every node.

### Design

**Global seed** → per-node seed → deterministic event stream:
```
global_seed = 12345  (CLI: --seed 12345, or random if omitted)
node_seed = hash(global_seed, node_index)
node_rng = StdRng::seed_from_u64(node_seed)
```

Each poll cycle, the node's intent generator rolls against probabilities from the profile:

```yaml
# In profile YAML
intent:
  seed: null                         # null = random (printed at start for replay)
  # Per-poll-interval probabilities
  reconnect_pct: 0.5                 # 0.5% chance per interval to go offline briefly
  reconnect_duration: [5, 60]        # seconds offline (uniform random)
  reboot_pct: 0.1                    # 0.1% chance to full reboot (re-register)
  endpoint_change_pct: 1.0           # 1% chance to change endpoints
  disconnect_pct: 0.05               # 0.05% chance to go permanently offline
  # Timing jitter
  poll_jitter_pct: 20                # ±20% jitter on poll interval per node
```

**Per-node state machine:**
```
                    ┌──────────┐
        ┌──────────│  ONLINE   │──────────┐
        │          │ (polling) │          │
        │          └─────┬─────┘          │
  reconnect_pct     endpoint_     disconnect_pct
        │          change_pct         │
        ▼               │             ▼
  ┌───────────┐         │     ┌──────────────┐
  │  OFFLINE  │         │     │  PERMANENT   │
  │ (paused)  │         │     │  OFFLINE     │
  └─────┬─────┘         │     └──────────────┘
        │               │
   duration expires      │
        │               ▼
        │       ┌──────────────┐
        └──────►│   ONLINE     │
                │ (new endpoints)│
  reboot_pct    └──────────────┘
        │
        ▼
  ┌───────────┐
  │ REBOOTING │ delete + re-register
  │           │ new node_id, new key
  └─────┬─────┘
        │
        ▼
  ┌──────────┐
  │  ONLINE  │
  │(new identity)│
  └──────────┘
```

### Reproducibility

**Same seed = same simulation:**
```bash
# First run — seed printed at startup
$ client-load-simulator --profile 2k-baseline.yaml
[INFO] Intent seed: 847291036 (use --seed 847291036 to replay)
...

# Replay — exact same client behavior
$ client-load-simulator --profile 2k-baseline.yaml --seed 847291036
```

**Intent log** — every intent decision is logged with enough context to understand what happened:
```jsonl
{"t_ms":0,"node_idx":0,"node_id":"abc-123","intent":"online","seed":847291036}
{"t_ms":30250,"node_idx":0,"node_id":"abc-123","intent":"poll","result":"keepalive"}
{"t_ms":60100,"node_idx":0,"node_id":"abc-123","intent":"poll","result":"peers_changed","added":1}
{"t_ms":90050,"node_idx":0,"node_id":"abc-123","intent":"reconnect","duration_s":15,"roll":0.003}
{"t_ms":105200,"node_idx":0,"node_id":"abc-123","intent":"online","after":"reconnect"}
{"t_ms":120300,"node_idx":42,"node_id":"def-456","intent":"reboot","roll":0.0008}
```

**Replay from log** (alternative to seed replay — useful when profile changed but you want same events):
```bash
$ client-load-simulator --profile 2k-v2.yaml --replay intent-log.jsonl
```

### How Intent Relates to Other Event Sources

All four event sources run simultaneously during a simulation:

```
┌─────────────────────────────────────────────────────┐
│                   Running Simulation                 │
│                                                     │
│  ┌──────────┐  Per-node PRNG rolls each interval    │
│  │ Intent   │  → reconnect, reboot, endpoint change │
│  │Generator │  → emergent, autonomous, reproducible  │
│  └────┬─────┘                                       │
│       │  ┌──────────┐  Time-ordered event scripts   │
│       ├──│Sequences │  → fleet rollout, drain, chaos│
│       │  └──────────┘  → orchestrated, composable   │
│       │  ┌──────────┐  Profile workload rates       │
│       ├──│Background│  → steady-state API operations│
│       │  └──────────┘  → continuous noise            │
│       │  ┌──────────┐  Manual/scripted commands     │
│       └──│   CLI    │  → inject, chaos set, etc     │
│          └──────────┘  → on-demand, interactive      │
│                                                     │
│  All feed into the same PendingChanges tracker      │
│  and measurement pipeline                           │
└─────────────────────────────────────────────────────┘
```

- **Intent** = what each node decides to do autonomously (bottom-up)
- **Sequences** = what the operator scripts to happen (top-down, planned)
- **Background workload** = steady-state noise from profile rates (statistical)
- **CLI** = manual intervention (ad-hoc)

## Interactive CLI

Long-running mode (`--interactive`) where the simulator keeps pollers alive and accepts commands via a control interface.

### Commands

```
> status                                    # Live metrics snapshot
  Nodes: 2000 polling  |  Changes: 142/500 detected  |  Noise: 97.2%
  Probe p99: 45ms  |  Convergence p99: 5.2s  |  Poll p99: 12ms

> inject node-join large-vlans/cvlan-1      # Manual change injection
> inject node-leave <node-id>
> inject bulk-enroll small-vlans/cvlan-42 count=20
> inject endpoint-update random count=10

> chaos set poll-timeout 5%                 # Turn on a chaos knob
> chaos set client-flap 10/min
> chaos get                                 # Show active chaos settings
> chaos clear                               # Disable all chaos

> sequence run fleet-rollout.yaml           # Play a sequence file
> sequence stop                             # Abort running sequence
> sequence list                             # Show available sequences

> report                                    # Full metrics dump (same as batch output)
> report reset                              # Reset all counters/histograms

> topology                                  # Show current state
  Tenants: 2  CVLANs: 105  Nodes: 2000 (1987 online, 13 stale)

> pause                                     # Pause injection (pollers continue)
> resume
> quit                                      # Graceful shutdown + final report
```

### Interface Options

| Mode | Interface | Use Case |
|------|-----------|----------|
| `--interactive` | stdin/stdout (TTY) | Manual exploration, demos |
| `--control-socket /tmp/sim.sock` | Unix socket | Scripted/robotic control |
| `--control-port 9090` | TCP | Remote control from another machine |

Scriptable via socket:
```bash
# Robot script
echo "sequence run fleet-rollout.yaml" | nc -U /tmp/sim.sock
sleep 300
echo "report" | nc -U /tmp/sim.sock > results/fleet-rollout-report.txt
echo "report reset" | nc -U /tmp/sim.sock
echo "chaos set poll-timeout 10%" | nc -U /tmp/sim.sock
echo "sequence run steady-state.yaml" | nc -U /tmp/sim.sock
```

### Execution Modes

| Mode | Flag | Behavior |
|------|------|----------|
| **Batch** | (default) | Load profile + sequences, run to completion, report, exit |
| **Interactive** | `--interactive` | Load profile, start pollers, accept commands until `quit` |
| **Soak** | `--soak --duration 4h` | Run sequences on repeat, report every N minutes |

### Phase 2 Scope (follow-up)

- VRouter restart with multi-CVLAN burst sync
- VRouter failover with CVLAN reassignment
- Scale profiles (10k, 100k)
- Push model optimization + re-measurement

## Measurement

### The Noise Problem

At 2k nodes polling every 5s = 400 polls/sec. When a change affects a 10-node CVLAN, only 10 of those 400 polls carry useful data — **97.5% is noise**. At 100k nodes with 30s interval = 3.3k polls/sec, a 50-node CVLAN change = **1.5% signal**. The controller spends most of its capacity responding "nothing changed" to nodes that don't need anything. A future push model would eliminate this noise, but first we need to quantify it.

### Three Metrics

**1. Capability probe (system floor)**
After each injected change, immediately fire a dedicated observer poll from the affected CVLAN. Measures pure `DB_write → DB_read → network` round-trip. No poll-interval jitter. This answers: "how fast CAN the system propagate changes?"

**2. Organic convergence (real-world experience)**
Let the 2k normal pollers detect changes at their natural poll cadence. For each change, track when ALL affected CVLAN members have seen it. This answers: "how fast DO changes propagate given poll interval + noise?"

Expected: convergence ≈ poll_interval + (affected_nodes × poll_rtt / throughput). For a 200-node CVLAN at 5s interval, ~5-6 seconds. For 10-node CVLAN, ~5 seconds (dominated by interval).

**3. Noise ratio (waste quantification)**
Track every poll response: keepalive (no data) vs change-carrying (useful). Report:
- Keepalive ratio: % of polls that returned no useful data
- Effective throughput: useful polls / total polls
- Controller overhead: CPU/memory spent on noise

This answers: "how much capacity would a push model recover?"

### Modes

- **Forced-full mode** (default): Send `last_seen_version = 0` on every poll → server always returns full peer list → detect changes by diffing peer sets client-side. Both probe and organic metrics work.

- **Realistic mode** (`--realistic`): Send real `last_seen_version` like cvlan-ctrl does → demonstrates that changes are invisible due to the 60s window → quantifies the production bottleneck. Only probe metric works (organic detection will timeout).

### Latency Definitions

| Metric | Start | End | What it measures |
|--------|-------|-----|-----------------|
| Probe latency | `t_api_complete` | `t_probe_poll_returns` | System capability (floor) |
| First-detection latency | `t_api_complete` | `t_first_organic_poller_detects` | Fastest organic propagation |
| Convergence latency | `t_api_complete` | `t_last_affected_poller_detects` | Full network convergence |
| Poll round-trip | `t_poll_sent` | `t_poll_response` | Raw API performance under load |

## Connection Model

**Full mTLS per node.** Each simulated node gets its own certificate from registration and its own `reqwest::Client` with that identity. This is the real channel — a future push model (SSE/WebSocket) would run over the same long-lived mTLS connection. Skipping it would mean rebuilding the simulator later.

### Resource Budget

| Scale | mTLS Clients | TLS Memory | Steady-State Polls/sec | Hardware |
|-------|-------------|-----------|----------------------|----------|
| 2k | 2,000 | ~200MB | 400 (5s interval) | Laptop (14c) |
| 10k | 10,000 | ~1GB | 2,000 (5s interval) | Laptop or server |
| 100k | 100,000 | ~10GB | 3,333 (30s interval) | Server (24-64c, 100GB+) |

### Connection Management

- Each `SimNode` holds its own `reqwest::Client` created with `Identity::from_pem(cert + key)` + `add_root_certificate(ca_chain)`
- Connection keepalive enabled (HTTP/2 multiplexing where supported)
- Max idle connections per host: 1 per client (we only talk to one server)

### Distributed Execution (Partitioned Replicas)

At 2k: single process is sufficient. At 10k+: distribute across multiple containers.

**Why distribute:**
- **FD limits**: Each container has its own `ulimit -n`, no host-level FD exhaustion
- **NAT isolation**: Containers are NATed, so host limits don't apply; each gets its own source IP
- **Cleaner measurements**: Lower per-process load = less tokio jitter = more trustworthy numbers
- **Realistic traffic pattern**: Controller sees connections from multiple IPs, tests rate limiting / load balancing
- **Linear scaling**: 100k across 10 containers = 10k each, comfortable per-process

**Partitioning model:**
```
CLI:  --partition 1/3    (I am replica 1 of 3)

Replica 1: owns nodes     1 -   667, polls them, detects changes for its slice
Replica 2: owns nodes   668 - 1334, polls them, detects changes for its slice
Replica 3: owns nodes  1335 - 2000, polls them, detects changes for its slice
           └── also runs the injector (injects changes + probes)
```

The last replica is the **injector** by convention (runs setup, injects changes, fires probes). All other replicas are pure pollers.

**Coordination via shared volume (no inter-process protocol):**
```
/shared/                          # Docker volume mounted by all replicas
├── changes/                      # Injector writes change events here
│   ├── 000001.json              # { id, type, cvlan_id, node_id, t_injected }
│   ├── 000002.json
│   └── ...
├── results/                      # Each replica writes detection results
│   ├── replica-1.jsonl          # { change_id, t_detected, node_id }
│   ├── replica-2.jsonl
│   └── replica-3.jsonl
└── topology.json                 # Injector writes after setup; replicas read to know assignments
```

Post-run: the injector replica reads all `results/*.jsonl`, merges histograms, prints the unified report.

**Docker Compose for distributed mode** (added to test stack):
```yaml
services:
  simulator:
    image: cvlan-build:latest
    command: /app/target/release/client-load-simulator --partition ${PARTITION} --profile /app/profiles/10k-steady.yaml
    volumes:
      - simulator-shared:/shared
      - build_target-cache:/app/target:ro
    network_mode: "service:api"   # or cvlan-tests_default
    deploy:
      replicas: 3
```

**For 2k baseline**: just run single process (no `--partition` flag = own everything). Distribution is opt-in.

### Self-Instrumentation

The simulator measures its own overhead to ensure measurements reflect the controller, not the test harness:

| Metric | How | What it catches |
|--------|-----|----------------|
| **Poll scheduling jitter** | `actual_poll_time - intended_poll_time` per poll | Tokio task scheduling delays under load |
| **Diff computation time** | `Instant` around peer set diff logic | Simulator CPU bottleneck on large CVLANs |
| **Client creation time** | Time to build each `reqwest::Client` with TLS identity | TLS context overhead during setup |
| **Memory RSS** | `sysinfo` sampling every 10s | Memory pressure affecting performance |
| **CPU usage** | `sysinfo` sampling every 10s | Simulator saturating cores |
| **Task backlog** | Count of pollers that missed their interval | Simulator can't keep up with poll rate |

Reported as a separate section:
```
Simulator Self-Check:
  Poll scheduling jitter:  p99 XX ms (target < 100ms)
  Diff computation:        p99 XX ms (target < 1ms)
  Missed poll intervals:   XX / XXXXX (target < 0.1%)
  Peak memory:             XXX MB
  Peak CPU:                XX%
  ⚠ WARNING: jitter > 100ms — measurements may include simulator overhead
```

If self-check shows high jitter or missed intervals, the report flags a warning so we know the numbers are suspect.

## Implementation

### Crate: `tests/client-load-simulator/`

```
tests/client-load-simulator/
├── Cargo.toml
├── profiles/
│   ├── 2k-baseline.yaml
│   └── 2k-chaos.yaml
├── sequences/
│   ├── fleet-rollout.yaml
│   ├── datacenter-drain.yaml
│   ├── client-churn.yaml
│   └── steady-state.yaml
└── src/
    ├── main.rs           # CLI args, mode selection, phase orchestration
    ├── profile.rs        # YAML profile deserialization + validation
    ├── sequence.rs       # Sequence file parser + step executor + timing engine
    ├── models.rs         # SimNode, ChangeEvent, PendingChanges, PeerSnapshot
    ├── api_client.rs     # Admin API (JWT) + coordination mTLS client per node
    ├── topology.rs       # Build topology from profile, create tenants/CVLANs
    ├── setup.rs          # Register nodes, create tokens, build mTLS clients
    ├── poller.rs         # Async poll loop per node, peer diff detection
    ├── intent.rs         # Per-node PRNG intent generator + state machine
    ├── injector.rs       # Executes actions (join/leave/move/etc) against API
    ├── chaos.rs          # Fault injection (timeouts, latency, flapping)
    ├── probe.rs          # Immediate observer polls after changes
    ├── control.rs        # Interactive CLI + Unix socket + TCP control interface
    ├── partition.rs      # Node slice assignment for distributed mode
    ├── metrics.rs        # HDR histograms, self-instrumentation, comfy_table report
```

### Phase Execution

```
Phase 1: Setup (sequential)
  ├── Create tenants (set identifier + public_key via PATCH)
  ├── Create CVLANs per profile topology
  ├── Create registration tokens (1 per CVLAN, max_uses = node_count + buffer)
  ├── Register nodes via /client/register (concurrency 50, step-ca bottleneck)
  │   └── Each registration returns cert + key + ca_chain → stored in SimNode
  └── Create per-node mTLS reqwest::Client from stored certs

Phase 2: Warm-up
  ├── All pollers start using their mTLS clients
  ├── Each completes one full poll cycle (baseline peer snapshot)
  └── Self-check: verify scheduling jitter is acceptable

Phase 3+4: Injection + Polling (concurrent)
  ├── Pollers: each node polls on its mTLS client at profile interval + jitter
  ├── Injector: rate-controlled changes per workload profile
  ├── Probes: immediate mTLS poll after each change from affected CVLAN
  └── Self-instrumentation: continuous jitter/memory/CPU sampling

Phase 5: Drain + Report
  ├── Wait for all changes detected or timeout
  ├── Stop pollers, collect metrics
  ├── Print capability, convergence, noise, and self-check reports
  └── Cleanup (delete tenants, cascade)
```

### Client-Side Change Simulation

**Client reconnect**: Poller task pauses for N seconds (simulating network loss), then resumes. Meanwhile, the node's `last_poll_at` ages, background job may mark it offline. On resume, peers should see the node come back online.

**Client reboot**: Poller task stops, node is deleted via admin API, then re-registered via /client/register with new WG key. Gets new node_id and IP. Peers see old node disappear, new node appear.

**Cert renewal**: Set `CVLAN_CLIENT_CERT_VALIDITY_HOURS=1` in test, so certs expire quickly. Poll triggers auto-renewal (server checks `cert_expires_at > 50% elapsed`). Measures cert-minting throughput under load.

**Stale timeout**: Stop a poller, wait for the background stale-node-marker job to flip status to offline. Measures how long until peers see the node go offline.

### Change Detection

Each poller maintains `HashMap<Uuid, PeerSnapshot>` (node_id → last-known peer state).

After each poll response:
1. Build current peer map from response
2. Compute `added = current_keys - previous_keys`
3. Compute `removed = previous_keys - current_keys`
4. For each added/removed, check `PendingChanges` registry (shared `DashMap`)
5. On match: record `detected_at`, compute latency, move to completed set
6. For endpoint updates: compare `PeerSnapshot.endpoints` for known peers

`PendingChanges` keyed by `(cvlan_id, expected_node_id, change_type)` for O(1) lookup.

### CLI Arguments

```
# Required
--api-url             http://localhost:8080       # API base URL
--jwt-token           (or $JWT_TOKEN)             # admin auth
--profile             profiles/2k-baseline.yaml   # topology + defaults

# Sequences (batch mode)
--sequence            sequences/fleet-rollout.yaml  # one or more sequence files
--sequence            sequences/steady-state.yaml   # composed in order

# Execution mode
--interactive                                     # interactive CLI mode
--control-socket      /tmp/sim.sock               # Unix socket for scripted control
--control-port        9090                         # TCP for remote control
--soak --duration 4h                              # repeat sequences, report periodically

# Intent
--seed                12345                         # deterministic replay (omit = random)
--replay              intent-log.jsonl              # replay from log file
--intent-log          intent-log.jsonl              # write intent decisions to file

# Distribution
--partition           1/3                          # this replica / total replicas
--shared-dir          /shared                      # shared volume for coordination

# Options
--skip-setup                                      # reuse previous topology
--cleanup                                         # delete everything after
--output              table|json                   # report format
--setup-concurrency   50                           # parallel registrations

# Profile overrides
--poll-interval       5
--poll-mode           forced-full|realistic
--probe               true|false
```

### Report Format

```
======================================================================
CLIENT LOAD SIMULATOR RESULTS
======================================================================

Topology: 2 tenants, 105 CVLANs, 2000 nodes
Duration: XXs  Poll interval: 5s  Mode: forced-full

Capability (probe latency — immediate poll after change):
+-------------------+-------+--------+--------+--------+
| Change Type       | Count | p50 ms | p95 ms | p99 ms |
+-------------------+-------+--------+--------+--------+
| Node Join         |   100 |    XX  |    XX  |    XX  |
| Node Leave        |    80 |    XX  |    XX  |    XX  |
| Client Reconnect  |    80 |    XX  |    XX  |    XX  |
| Endpoint Update   |    60 |    XX  |    XX  |    XX  |
| Client Reboot     |    40 |    XX  |    XX  |    XX  |
| Bulk Enroll       |    60 |    XX  |    XX  |    XX  |
| Node Move         |    30 |    XX  |    XX  |    XX  |
| Cert Renewal      |    50 |    XX  |    XX  |    XX  |
+-------------------+-------+--------+--------+--------+
| ALL               |   500 |    XX  |    XX  |    XX  |
+-------------------+-------+--------+--------+--------+

Convergence (time until ALL affected CVLAN members detect change):
+-------------------+-------+---------+--------+--------+--------+
| Change Type       | Count | Success | p50 ms | p95 ms | p99 ms |
+-------------------+-------+---------+--------+--------+--------+
| Node Join         |   100 |   100%  |    XX  |    XX  |    XX  |
| ...               |       |         |        |        |        |
+-------------------+-------+---------+--------+--------+--------+

Noise Ratio:
  Total polls:           XXXXX
  Keepalive (no data):   XXXXX (XX.X%)
  Change-carrying:       XXXXX (X.X%)
  Signal efficiency:     X.X%

Poll Performance:
  Throughput:        XXX polls/sec
  p50 round-trip:    XX ms
  p99 round-trip:    XX ms

CLIENT_LOAD_METRICS: probe_p99_ms=XX converge_p99_ms=XX noise_pct=XX.X polls_sec=XXX changes=500/500 nodes=2000
```

### Integration

- Workspace: `Cargo.toml` members += `"tests/client-load-simulator"`
- Script: `./scripts/test --client-load` (extends existing wrapper)
- Test stack: `docker compose -p cvlan-tests` with **step-ca** service
- `CVLAN_DISABLE_HARDENING=1` to bypass rate limits

### Key Files to Reference

| File | Purpose |
|------|---------|
| `client/cvlan-ctrl/src/api/client.rs` | Real client HTTP calls to mirror |
| `client/cvlan-ctrl/src/api/mod.rs` | Request/response models |
| `client/cvlan-ctrl/src/main.rs:97-151` | `do_poll` to simulate |
| `api/src/rest/handlers/coordination.rs:325-417` | Server poll handler (60s window) |
| `api/src/rest/handlers/coordination.rs:434-518` | Token creation endpoint |
| `api/src/models/client_protocol.rs` | Server-side serde models |
| `tests/load/src/main.rs` | Existing load test (metrics, Docker patterns) |
| `tests/docker-compose.test.yml` | Test stack with step-ca |
| `scripts/test` | Wrapper script to extend |

## Linear Ticket Structure

**Parent**: `Client Load Simulator` (Improvement, label: Improvement)

Sub-tasks:
1. Create crate skeleton, workspace integration, profile/sequence YAML parsing
2. Implement API client (admin JWT + per-node mTLS clients)
3. Implement setup phase (topology creation, bulk registration, mTLS client construction)
4. Implement poller with peer-diff change detection + self-instrumentation
5. Implement injector actions (all change types) + probe observer
6. Implement sequence engine (step executor, timing, checkpoints, repeat)
7. Implement chaos module (fault injection knobs)
8. Implement metrics + reporting (capability, convergence, noise, self-check)
9. Implement interactive CLI + control socket
10. Implement partition mode for distributed execution
11. Integrate with scripts/test wrapper, write 2k-baseline profile + initial sequences
12. Run baseline measurement at 2k, document findings
