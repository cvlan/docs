# Air-Gapped Deployment

CloudVLAN supports fully offline operation for environments with no internet connectivity: defense, critical infrastructure, secure manufacturing, isolated research.

## Key Differences from Connected Deployment

| Aspect | Connected | Air-Gapped |
|--------|-----------|------------|
| Authentication | JWT + optional external IdP | JWT with local secrets |
| Certificate Authority | step-ca (local) | step-ca (same, local) |
| Software Updates | Docker pull | Manual image transfer |
| Discovery | Public endpoint | Internal DNS/IP |
| Time Sync | Internet NTP | Internal NTP or GPS |

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Air-Gapped Environment                   │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                  Management Zone                     │ │
│  │                                                      │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │ │
│  │  │cvlan-api │  │ step-ca  │  │PostgreSQL│          │ │
│  │  │(combined)│  │ (mTLS CA)│  │          │          │ │
│  │  └──────────┘  └──────────┘  └──────────┘          │ │
│  │                                                      │ │
│  │  ┌──────────┐  ┌──────────┐                         │ │
│  │  │   UI     │  │ Internal │                         │ │
│  │  │          │  │  NTP     │                         │ │
│  │  └──────────┘  └──────────┘                         │ │
│  └─────────────────────────────────────────────────────┘ │
│                          │                                │
│                  Internal Network                         │
│                          │                                │
│  ┌───────────────────────┼─────────────────────────────┐ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │ │
│  │  │ vrouterd │  │cvlan-ctrl│  │cvlan-ctrl│          │ │
│  │  │ (site GW)│  │ (node 1) │  │ (node 2) │          │ │
│  │  └──────────┘  └──────────┘  └──────────┘          │ │
│  └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

## What Works Offline

| Feature | Status | Notes |
|---------|--------|-------|
| Node registration | Works | Ed25519 tokens can be generated offline |
| mTLS certificates | Works | step-ca runs locally |
| Peer discovery | Works | Via poll from local controller |
| WireGuard tunnels | Works | Direct mesh between nodes |
| Policy enforcement | Works | Distributed via poll responses |
| Management UI | Works | Served locally |
| VRouter gateways | Works | Register with local controller |

## Software Updates

Without internet access, updates are transferred manually:

```bash
# On connected machine: save Docker images
docker save cvlan-api:latest | gzip > cvlan-api.tar.gz
docker save cvlan-ui:latest | gzip > cvlan-ui.tar.gz

# Transfer via sneakernet (USB, etc.)

# On air-gapped machine: load images
docker load < cvlan-api.tar.gz
docker load < cvlan-ui.tar.gz
docker compose up -d
```

## Registration Tokens

Ed25519 registration tokens can be generated on any machine with the tenant's private key — they don't require network access to the controller. This means tokens can be pre-generated in bulk and distributed to air-gapped nodes.

## Related

- [Self-Hosted Deployment](self-hosted.md)
- [Architecture: Node Registration](../../architecture/node-registration.md)
- [Technical: Authentication](../../technical/authentication-and-identity.md)
