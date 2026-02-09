# Work Tracking

## Current Focus

- [ ] Complete documentation structure
- [ ] Begin cvlan-relay implementation (first component)

## Backlog (Prioritized)

### Phase 1: Foundation
- [ ] cvlan-relay: Basic Rust relay server
- [ ] cvlan-relay: Connection handling
- [ ] cvlan-relay: Packet forwarding

### Phase 2: Control Plane
- [ ] cvlan-controller: Node registration
- [ ] cvlan-controller: Key distribution
- [ ] cvlan-controller: Network map generation
- [ ] cvlan-controller: Basic API

### Phase 3: Client Agent
- [ ] cvlan-agent: WireGuard integration
- [ ] cvlan-agent: Control plane communication
- [ ] cvlan-agent: Relay fallback

### Phase 4: Policy
- [ ] cvlan-policy: ACL definition format
- [ ] cvlan-policy: Policy evaluation engine
- [ ] cvlan-policy: Policy distribution

### Phase 5: IPsec
- [ ] IPsec stack integration (strongSwan FFI or native)
- [ ] IKEv2 support in cvlan-agent
- [ ] Certificate management

### Phase 6: vRouter
- [ ] cvlan-vrouter: IPsec concentrator
- [ ] cvlan-vrouter: Multi-tenant support
- [ ] cvlan-vrouter: Cloud deployment templates

## Done

- [x] 2026-01-19: Initial project planning and context establishment
- [x] 2026-01-19: Documentation structure created

## Decisions Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-01-19 | Rust for all data plane components | No GC overhead, predictable memory at scale |
| 2026-01-19 | Dual protocol (WireGuard + IPsec) | WireGuard for simplicity, IPsec for compliance |
| 2026-01-19 | Self-hosted as primary deployment | Differentiation from Tailscale/Headscale model |
| 2026-01-19 | Cloud-neutral infrastructure | No vendor lock-in, any cloud/colo works |
| 2026-01-19 | Target: 100k clients < $2k/month | Proves cost efficiency at scale |
| 2026-01-19 | Start with relay server | Smallest component, proves Rust stack |

## Open Questions

- [ ] IPsec implementation: FFI to strongSwan vs native Rust?
- [ ] Database choice for controller: PostgreSQL vs SQLite vs embedded?
- [ ] Wire protocol: Custom binary vs gRPC vs REST+WebSocket?
- [ ] License model: Open core? AGPLv3? Apache 2.0?

## Notes

Use this file to track what's in progress, what's done, and decisions made. Keep it updated so any Claude Code session can understand current state.
