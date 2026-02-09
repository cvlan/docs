# Sprint Plan

> Sprint 1 is 4 weeks (project inception → Feb 18). All subsequent sprints are 2 weeks.

## Milestone: 100k Simulated Clients

Demonstrate 100,000 simulated clients on a home lab:
- 100 tenants
- 100 CVLANs per tenant (10,000 CVLANs total)
- 10 nodes per CVLAN (100,000 nodes total)
- 2 regions with vrouters
- ~20% traffic via vrouter (hub-and-spoke), 80% direct mesh
- Full control plane: registration, polling, peer distribution, policy enforcement

---

## Sprint 1: First Connection (Jan 19 – Feb 18)

**Goal**: Two clients register, discover each other, establish a WireGuard tunnel, and ping.

```
laptop-A → cvlan-ctrl → cvland (boringtun) → WireGuard tunnel → cvland → cvlan-ctrl → laptop-B
```

### Done (Weeks 1–3)
- [x] Control plane API (multi-tenancy, RBAC, audit, IP allocation)
- [x] Management UI
- [x] VRouter integration (VPP, mTLS, DiffSync, bootstrap)
- [x] Protocol Buffers type pipeline
- [x] Client control plane daemon (cvlan-ctrl: discover → register → poll)
- [x] Regression suite (7/7 stages)
- [x] Documentation rewrite (23 SVGs, 27 PDFs)

### Remaining (Week 4: Feb 10–18)

| # | Task | Details |
|---|------|---------|
| 1 | **cvland data plane** | boringtun WireGuard tunnel creation/teardown, listen on assigned IP |
| 2 | **cvlan-ctrl → cvland IPC** | Unix socket: push peer config (pubkey, endpoint, allowed IPs) on poll update |
| 3 | **Peer list includes vrouters** | `get_peers_for_cvlan` returns vrouters assigned to the CVLAN |
| 4 | **Node endpoint reporting** | cvlan-ctrl detects and reports its public endpoint (IP:port) |
| 5 | **E2E: two-node connectivity** | Two cvlan-ctrl + cvland instances register, tunnel forms, ping works |

### Success Criteria
- `ping 10.100.0.2` from node A (10.100.0.1) succeeds over WireGuard tunnel
- Both nodes registered via E2E stack, certs issued, peers exchanged via poll
- Tunnel survives node restart (persistent identity from state.db)

---

## Sprint 2: Stable Mesh + Relay (Feb 18 – Mar 4)

**Goal**: Multiple nodes form a stable mesh. Nodes behind NAT can connect via relay.

| # | Task | Details |
|---|------|---------|
| 1 | **Multi-node mesh** | 5+ nodes in a CVLAN, all can reach each other |
| 2 | **Relay server** | DERP-style relay binary for NAT traversal |
| 3 | **STUN endpoint discovery** | Nodes discover their public IP:port via STUN |
| 4 | **Relay fallback** | Direct connection attempted first, relay used when direct fails |
| 5 | **Hub-and-spoke via vrouter** | Nodes route through vrouter when configured |
| 6 | **Connection health monitoring** | cvlan-ctrl tracks peer reachability, reports to controller |

### Success Criteria
- 5 nodes mesh, all pairs can ping
- Node behind NAT connects via relay
- Traffic routes through vrouter for cross-region peers

---

## Sprint 3: Client Polish + Simulation Prep (Mar 4 – Mar 18)

**Goal**: Client is installable and handles cert renewal. Start building simulation tooling.

| # | Task | Details |
|---|------|---------|
| 1 | **Client packaging polish** | Fix existing .deb: systemd units, maintainer scripts, arch detection |
| 2 | **Certificate renewal** | Auto-renew before expiry without full re-registration |
| 3 | **Simulated client agent** | Lightweight process that registers + polls but skips WireGuard |
| 4 | **Bulk provisioning** | Script to create 100 tenants × 100 CVLANs × 10 nodes via API |

### Success Criteria
- `apt install cvlan-client && cvlan-ctrl --token=...` joins the network
- Cert renewal works without operator intervention
- Simulated agent can register and poll at high rate

---

## Sprint 4: Load Simulation (Mar 18 – Apr 1)

**Goal**: Run 100k simulated clients against the control plane on home lab.

| # | Task | Details |
|---|------|---------|
| 1 | **100k registration run** | All 100 tenants × 100 CVLANs × 10 nodes registered |
| 2 | **Sustained polling** | 100k nodes × 30s interval = ~3,300 polls/sec sustained |
| 3 | **Multi-region setup** | 2 vrouter regions on home lab (separate VMs or containers) |
| 4 | **Metrics collection** | Registration latency, poll latency, memory/CPU per component |
| 5 | **Database at scale** | PostgreSQL tuning for 100k nodes, 10k CVLANs |
| 6 | **Bottleneck identification** | Profile and fix whatever breaks first |

### Success Criteria
- 100k simulated clients registered across 100 tenants
- Sustained 3,300 polls/sec with p99 < 50ms
- Controller memory < 2GB, CPU < 50% on home lab hardware

---

## Sprint 5: 100k Milestone (Apr 1 – Apr 15)

**Goal**: Full 100k demonstration on home lab.

| # | Task | Details |
|---|------|---------|
| 1 | **100k simulated clients** | 100 tenants × 100 CVLANs × 10 nodes, all registered and polling |
| 2 | **2 regions with vrouters** | Cross-region traffic via vrouter, intra-region direct mesh |
| 3 | **80/20 traffic split** | 80% direct mesh, 20% via vrouter — verified with traffic counters |
| 4 | **Policy enforcement at scale** | Policies applied, denied traffic logged |
| 5 | **Steady-state metrics** | Memory, CPU, latency, throughput documented over 1-hour run |
| 6 | **Infrastructure cost** | Measured: total resources used, projected monthly cost at scale |

### Success Criteria
- 100,000 nodes registered and actively polling
- Cross-region connectivity via vrouter demonstrated
- Infrastructure cost projection < $2k/month at scale
- System stable for 1+ hours under sustained load
- Results documented with graphs and metrics

---

## Sprint Cadence

| Sprint | Dates | Theme |
|--------|-------|-------|
| **1** | Jan 19 – Feb 18 | First connection (2 clients talk) |
| **2** | Feb 18 – Mar 4 | Stable mesh + relay |
| **3** | Mar 4 – Mar 18 | Client polish + simulation prep |
| **4** | Mar 18 – Apr 1 | 100k load simulation |
| **5** | Apr 1 – Apr 15 | 100k milestone demo |

## After the Milestone

Deferred until after 100k demo (not yet planned):
- DNS resolver + split DNS
- cvlancli debug CLI
- REST API proto3 JSON compliance
- SSO / OIDC integration
- IPsec end-to-end wiring
- Kubernetes operator
- Terraform provider
- macOS / Windows clients
- Public beta
