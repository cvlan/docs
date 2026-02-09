# Docs Rewrite Plan

> This file is the source of truth for the docs reorganization. Re-read on compaction.

## Goal

Rewrite ~/ws/docs/ to reflect the **actual state** of the CloudVLAN project as of Feb 2026.
The current docs are from Jan 19 (project inception) and use wrong component names,
describe unbuilt features, and miss everything that was actually built.

All .md files will be rendered as PDFs. Diagrams are SVG files in `images/`.
The user is a visual thinker — every document should have multiple diagrams.

## Key Constraints

- **Diagrams = SVG or PNG files** (NOT Mermaid code blocks). Referenced as `![alt](images/foo.svg)`
- **PDF-friendly** — all markdown will be converted to PDF
- **Lots of pictures** — 10+ key diagrams minimum, ~35-40 total across all files
- **Intuitive names** — no generic names like "coordination protocol", use "node registration"
- **Honest status** — include Headscale comparison, acknowledge gaps

## Color Palette for SVGs

- Control plane (cvlan-api): `#4A90D9` (blue)
- Data plane (vrouter/cvland): `#5CB85C` (green)
- Client: `#F0AD4E` (amber)
- UI: `#5BC0DE` (light blue)
- Infrastructure: `#95A5A6` (gray)
- Borders/text: `#333333`
- Background: white
- Font: sans-serif (Arial), 14px labels, 12px details

## Target Directory Structure

```
docs/
├── README.md                              # Landing page + component table
├── CLAUDE.md                              # Claude context (updated)
├── STATUS.md                              # What's built, what's planned, vs Headscale
├── DOCS-REWRITE-PLAN.md                   # THIS FILE
│
├── architecture/
│   ├── system-overview.md                 # The big picture
│   ├── control-plane-api.md               # cvlan-api deep dive
│   ├── vrouter.md                         # vrouterd + vrouterctl
│   ├── client.md                          # cvlan-ctrl + cvland + cvlancli
│   ├── management-ui.md                   # React frontend
│   └── node-registration.md              # How nodes join the network
│
├── technical/
│   ├── technology-choices.md              # Why Rust, key deps (updated)
│   ├── api-request-lifecycle.md           # HTTP request flow (from cvlan/api/API_HANDLING.md)
│   ├── ip-allocation.md                   # Address management (from cvlan/api/IP_ALLOCATION.md)
│   ├── shared-type-definitions.md         # Protobuf pipeline (from cvlan/protodefs/)
│   ├── authentication-and-identity.md     # All auth mechanisms
│   ├── database-design.md                 # Schema and storage
│   └── security-model.md                  # Trust boundaries, threat model
│
├── operations/
│   ├── build-system.md                    # Docker singletons, repo scripts
│   ├── testing.md                         # Full test strategy
│   ├── e2e-stack.md                       # Docker compose topology
│   └── deployment/
│       ├── self-hosted.md                 # (updated from existing)
│       ├── air-gapped.md                  # (updated from existing)
│       └── saas.md                        # (updated from existing)
│
├── business/                              # KEEP AS-IS
│   ├── vision.md
│   ├── competitive-landscape.md
│   └── pricing-thoughts.md
│
├── presentations/                         # KEEP AS-IS
│   └── vc-pitch.md
│
├── user-facing/                           # KEEP AS-IS
│   └── positioning.md
│
└── images/                                # ALL SVG diagrams
    ├── system-topology.svg
    ├── repo-map.svg
    ├── control-data-flow.svg
    ├── node-registration-flow.svg
    ├── vrouter-registration-flow.svg
    ├── client-three-crate.svg
    ├── client-lifecycle.svg
    ├── request-pipeline.svg
    ├── tiered-architecture.svg
    ├── tenant-hierarchy.svg
    ├── ip-allocation-flow.svg
    ├── type-generation-pipeline.svg
    ├── auth-decision-tree.svg
    ├── cert-lifecycle.svg
    ├── er-diagram.svg
    ├── sqlite-state-dbs.svg
    ├── trust-boundaries.svg
    ├── build-system.svg
    ├── test-pyramid.svg
    ├── regression-pipeline.svg
    ├── e2e-topology.svg
    ├── timeline.svg
    └── feature-comparison.svg
```

## Files to DELETE

| File | Reason |
|------|--------|
| `technical/components/client-agent.md` | Speculative, replaced by `architecture/client.md` |
| `technical/components/control-plane.md` | Speculative, replaced by `architecture/control-plane-api.md` |
| `technical/components/vrouter.md` | Speculative, replaced by `architecture/vrouter.md` |
| `technical/components/relay-server.md` | Not built yet, can revisit when implemented |
| `technical/components/policy-engine.md` | Not built yet, can revisit when implemented |
| `technical/components/ipsec-stack.md` | Not built yet, can revisit when implemented |
| `technical/architecture.md` | Replaced by `architecture/system-overview.md` |
| `technical/architecture.md.v1` | Old backup |
| `WORK_TRACKING.md` | Replaced by `STATUS.md` |

After deletion, remove empty `technical/components/` directory.

## Files MOVED IN from other repos

Source files stay in their repos (useful as local reference). Docs/ gets the canonical, visual version.

| Source | Destination | Edit Level |
|--------|-------------|------------|
| `cvlan/api/COORDINATION.md` | `architecture/node-registration.md` | Rewrite with diagrams |
| `cvlan/api/API_HANDLING.md` | `technical/api-request-lifecycle.md` | Light edit + diagrams |
| `cvlan/api/IP_ALLOCATION.md` | `technical/ip-allocation.md` | Light edit + diagrams |
| `cvlan/protodefs/TYPE_DEFINITIONS.md` | `technical/shared-type-definitions.md` | Light edit + diagrams |

## Diagram Inventory (~23 SVGs)

### System-level (system-overview.md)
1. **system-topology.svg** — All components, ports, protocols, how they connect
2. **repo-map.svg** — Repository → crate → binary tree diagram
3. **control-data-flow.svg** — What data flows on control plane vs data plane

### Registration & Auth (node-registration.md + authentication-and-identity.md)
4. **node-registration-flow.svg** — Sequence: admin creates token → node discovers → registers → gets certs
5. **vrouter-registration-flow.svg** — Sequence: VRouter join token → register → step-ca cert → poll
6. **auth-decision-tree.svg** — Which auth mechanism for which endpoint type
7. **cert-lifecycle.svg** — mTLS certificate: mint → use → renew/expire

### Control Plane API (control-plane-api.md)
8. **request-pipeline.svg** — HTTP → rate limit → auth → handler → DB → response
9. **tenant-hierarchy.svg** — Tenant → CVLAN → Node entity relationships
10. **tiered-architecture.svg** — Discovery (443) / Registration / Session (mTLS)

### VRouter (vrouter.md)
11. **vrouter-registration-flow.svg** — (shared with #5 above)
    VRouter has its own registration sequence details inside vrouter.md

### Client (client.md)
12. **client-three-crate.svg** — ctrl (control plane) / daemon (data plane) / cli (debug)
13. **client-lifecycle.svg** — State machine: unregistered → discover → register → poll → error → retry

### Technical Deep Dives
14. **ip-allocation-flow.svg** — Flowchart: check first_free → allocate → advance watermark → gap scan
15. **type-generation-pipeline.svg** — Proto files → buf generate → prost/ts-proto → Rust/TypeScript code
16. **er-diagram.svg** — Entity-relationship: tenants, cvlans, nodes, users, groups, policies, vrouters
17. **sqlite-state-dbs.svg** — Side-by-side: vrouter state.db vs client state.db schemas
18. **trust-boundaries.svg** — Internet → discovery → registration → session → data plane tiers

### Operations
19. **build-system.svg** — Singleton container: host → docker exec → cargo inside persistent container
20. **test-pyramid.svg** — Pyramid with actual numbers: 199 unit, 328 integration, 21K ops/s load, 6 e2e
21. **regression-pipeline.svg** — 7 stages: build → static → unit → integration → load → e2e → coverage
22. **e2e-topology.svg** — Docker compose: controller:9080, discovery:9081, postgres, step-ca, vrouter
23. **timeline.svg** — Project timeline: API(Jan 19) → UI(Jan 21) → vrouter(Jan 31) → proto(Feb 3) → client(Feb 9)

### Status
24. **feature-comparison.svg** — CloudVLAN vs Headscale feature heatmap

## STATUS.md Content Outline

```markdown
# Project Status

## State of the Project (Feb 2026)
- 3 weeks of development, ~60 hrs of AI-assisted building
- 5 repos, 100+ commits, 60+ architecture decisions
- Control plane is deep (multi-tenancy, policy, audit, scheduler, IP allocation)
- Data plane gateway works (VPP, mTLS, DiffSync poll loop)
- Client control plane daemon just landed (discover → register → poll)
- No working end-to-end node connectivity yet — that's the critical path

## vs Headscale
[feature-comparison.svg]
Content from ~/ws/compare-headscale-feb-6-2026.md
Bottom line: control plane ahead, no working client = can't connect devices

## Implementation Matrix
| Component | Repo | Status | Tests | Coverage |
|-----------|------|--------|-------|----------|
| cvlan-api | cvlan | ✅ Working | 199u + 328i | 47.5% |
| cvlanctl | cvlan | ✅ Working | via integration | — |
| Management UI | ui | ✅ Working | eslint clean | — |
| vrouterd | vrouter | ✅ Working | VPP integration | — |
| vrouterctl | vrouter | ✅ Working | — | — |
| cvlan-ctrl | client | ✅ Built | 11 unit | — |
| cvland | client | 🔲 Stub | 1 test | — |
| cvlancli | client | 🔲 Stub | 0 | — |
| Relay | — | 🔲 Not started | — | — |
| E2E suite | e2e | ✅ Working | 6 scenarios | — |
| Regression | e2e | ✅ 7/7 pass | — | 25.5% agg |

## Timeline
[timeline.svg]

## What's Next (critical path to first connection)
1. Client data plane (cvland: boringtun WireGuard tunnels)
2. Node-VRouter connectivity (nodes see vrouters as peers)
3. End-to-end: laptop → cvlan-ctrl → tunnel → vrouter → other laptop
4. NAT traversal (DERP relay)
5. DNS resolver
6. REST API proto3 JSON compliance

## Decisions Log
(60+ decisions migrated from journal.md, grouped by topic)
```

## Markdown File Content Notes

### README.md
- Update component table to actual names (cvlan-api, vrouterd, cvlan-ctrl, cvland, cvlancli)
- Update links to new file locations
- Keep the elevator pitch

### CLAUDE.md
- Update component names
- Update "Current Phase" to reality
- Update "Implementation Order" to actual path taken
- Reference STATUS.md instead of WORK_TRACKING.md

### architecture/system-overview.md
- New system topology diagram with ACTUAL components
- Server-authoritative CVLAN model (not Tailscale coordinator)
- mTLS cert-based auth (Noise protocol deferred)
- REST + poll pattern (not WebSocket/streaming)
- Protocol Buffers type system

### architecture/control-plane-api.md
- Content from cvlan/api/API_HANDLING.md + CLAUDE.md context
- Multi-tenant RBAC (superadmin/tenantadmin/member/readonly)
- Background job scheduler
- Server modes (controller/discovery/registration/session/combined)
- Rate limiting and load shedding

### architecture/vrouter.md
- VPP 24.10 integration (binary API, socket protocol, DPDK)
- Registration flow with step-ca
- DiffSync reconciliation
- Bootstrap mode for testing
- Poll loop with stats reporting

### architecture/client.md
- Three-crate architecture rationale
- cvlan-ctrl: config → discover → register → poll loop
- cvland: future data plane (boringtun, WireGuard tunnels)
- cvlancli: debug/introspection CLI
- IPC via unix sockets (stub, future)
- SQLite state with WAL mode

### architecture/node-registration.md
- Content from cvlan/api/COORDINATION.md rewritten
- Ed25519 token signing model
- Server-authoritative CVLAN assignment
- Discovery → register → poll sequence diagrams
- Certificate issuance
- Delta-based poll updates
- Tailscale compatibility aliases

### architecture/management-ui.md
- React 18 + TypeScript 5 + Vite 5
- Tanstack Query + Zustand state management
- Tailwind CSS
- Pages: tenants, cvlans, nodes, users, groups, policies, vrouters, regions, relays, audit logs
- Separate repo, Docker-based builds

### technical/technology-choices.md
- UPDATE existing file with actual deps used
- Add: sqlx 0.8, rusqlite, x25519-dalek, step-ca, VPP 24.10
- Update IPsec section (VPP handles it, not strongSwan FFI)
- Update database section (Postgres + SQLite dual-backend is reality)

### technical/api-request-lifecycle.md
- Content from cvlan/api/API_HANDLING.md
- Add request-pipeline.svg diagram
- Middleware stack details
- Auth flow details

### technical/ip-allocation.md
- Content from cvlan/api/IP_ALLOCATION.md
- Add ip-allocation-flow.svg diagram
- Dual-pointer algorithm
- Per-CVLAN mutex allocator

### technical/shared-type-definitions.md
- Content from cvlan/protodefs/TYPE_DEFINITIONS.md
- Add type-generation-pipeline.svg
- Proto → Rust/TypeScript pipeline

### technical/authentication-and-identity.md
- JWT tokens (HS256, role claims, scopes)
- API keys (Argon2 hashing, scoped permissions)
- mTLS certificates (X.509, step-ca, 8hr validity for vrouters)
- Ed25519 registration tokens (tenant-signed, server-authoritative CVLAN)
- Add auth-decision-tree.svg and cert-lifecycle.svg

### technical/database-design.md
- PostgreSQL schema (all tables)
- SQLite state databases (vrouter state.db, client state.db)
- Dual-backend architecture
- Add er-diagram.svg and sqlite-state-dbs.svg

### technical/security-model.md
- Trust boundaries
- Key management (private keys never leave nodes)
- No privileged containers policy
- Certificate pinning
- Add trust-boundaries.svg

### operations/build-system.md
- Singleton container pattern
- Per-repo build scripts (build/scripts/ vs scripts/)
- Docker-first development (never cargo on host)
- Cross-compilation (clang + lld for client)
- Add build-system.svg

### operations/testing.md
- Test pyramid with actual numbers
- Unit tests (199 cvlan + 11 client)
- Integration tests (328 cvlan)
- Load tests (21K ops/sec, 200 concurrency)
- E2E tests (6 scenarios)
- Regression suite (7 stages, accept/reject voting)
- Coverage tracking (47.5% line coverage)
- Add test-pyramid.svg and regression-pipeline.svg

### operations/e2e-stack.md
- Docker compose topology
- Ports: controller 9080, discovery 9081
- JWT secret: e2e-test-secret
- step-ca for mTLS
- VRouter bootstrap mode
- Add e2e-topology.svg

### operations/deployment/ (self-hosted, air-gapped, saas)
- Light updates to existing files
- Fix component names
- Add cross-references to new architecture docs

## Execution Order

1. Create directory structure ✅ DONE
2. Write all SVG diagrams to images/
3. Write STATUS.md (includes Headscale comparison)
4. Write README.md and CLAUDE.md updates
5. Write architecture/ files (system-overview first, then components)
6. Write technical/ files
7. Write operations/ files
8. Update deployment/ files
9. Delete old files (components/, WORK_TRACKING.md, architecture.md.v1)
10. Verify all image references resolve

## Reference Files (read these for content)

- ~/ws/journal.md — Timeline, decisions log, milestone list
- ~/ws/todo.md — Open items
- ~/ws/compare-headscale-feb-6-2026.md — Competitive comparison
- ~/ws/cvlan/CLAUDE.md — API details, JWT format, server modes, rate limiting
- ~/ws/cvlan/api/COORDINATION.md — Registration protocol design
- ~/ws/cvlan/api/API_HANDLING.md — Request handling deep dive
- ~/ws/cvlan/api/IP_ALLOCATION.md — IP allocation algorithm
- ~/ws/cvlan/protodefs/TYPE_DEFINITIONS.md — Proto definitions
- ~/ws/vrouter/CLAUDE.md — VRouter build, VPP integration, vrouterctl
- ~/ws/client/CLAUDE.md — Client build, cross-compilation, packaging
- ~/ws/ui/CLAUDE.md — UI build, React stack
- ~/ws/e2e/CLAUDE.md — E2E ports, JWT secret
- ~/ws/e2e/regression/README.md — Regression suite architecture
- ~/ws/e2e/regression/COVERAGE-PLAN.md — Singleton container pattern

## Actual Component Names (CRITICAL — use these, not the old names)

| Old Name (docs) | Actual Name | Binary | Repo |
|------------------|-------------|--------|------|
| cvlan-controller | cvlan-api | cvlan-api | cvlan |
| cvlanctl | cvlanctl | cvlanctl | cvlan |
| cvlan-agent | cvland + cvlan-ctrl + cvlancli | cvland, cvlan-ctrl, cvlancli | client |
| cvlan-vrouter | vrouterd + vrouterctl | vrouterd, vrouterctl | vrouter |
| cvlan-relay | — | — | not started |
| cvlan-policy | — | — | built into cvlan-api |
| — | Management UI | — | ui |
