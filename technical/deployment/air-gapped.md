# Air-Gapped Deployment

## Overview

Air-gapped deployment enables CloudVLAN in environments with no internet connectivity:
- Defense and intelligence
- Critical infrastructure (power, water, transportation)
- Secure manufacturing
- Isolated research environments

## Key Differences from Connected Deployment

| Aspect | Connected | Air-Gapped |
|--------|-----------|------------|
| Authentication | SSO/OIDC via internet IdP | Local auth or internal IdP |
| Certificate Authority | Public CA (Let's Encrypt) | Internal CA |
| Software Updates | Automatic or manual download | Manual transfer via sneakernet |
| Relay Servers | Cloud-hosted option | Must be internal |
| Time Sync | Internet NTP | Internal NTP or GPS |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Air-Gapped Environment                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   Management Zone                         │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │   │
│  │  │ Controller  │  │  Internal   │  │   Internal      │   │   │
│  │  │             │  │     CA      │  │   NTP Server    │   │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘   │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌─────────────┐                        │   │
│  │  │  Database   │  │   Local     │                        │   │
│  │  │ (PostgreSQL)│  │   IdP       │                        │   │
│  │  └─────────────┘  └─────────────┘                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                    Internal Network                              │
│                              │                                   │
│  ┌───────────────────────────┼───────────────────────────────┐  │
│  │                           │                                │  │
│  │  ┌──────────┐  ┌──────────┴───┐  ┌──────────┐             │  │
│  │  │  Relay   │  │    Relay     │  │  Relay   │             │  │
│  │  │  (site1) │  │   (site2)    │  │  (site3) │             │  │
│  │  └──────────┘  └──────────────┘  └──────────┘             │  │
│  │                                                            │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │  │  Agent   │  │  Agent   │  │  Agent   │  │ vRouter  │  │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│              ════════════════════════════════════                │
│                        AIR GAP                                   │
│              ════════════════════════════════════                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                    Sneakernet for updates
                                │
                                ▼
                    ┌───────────────────┐
                    │   Outside World   │
                    └───────────────────┘
```

## Authentication

### Option 1: Local Users

Simple user database in the controller:

```toml
# /etc/cvlan/controller.toml

[auth]
method = "local"

[[auth.users]]
username = "admin"
password_hash = "$argon2id$..."  # Use cvlanctl hash-password
role = "admin"

[[auth.users]]
username = "operator"
password_hash = "$argon2id$..."
role = "operator"
```

### Option 2: Internal OIDC/LDAP

Connect to internal identity provider:

```toml
[auth]
method = "oidc"
oidc_issuer = "https://internal-idp.local"  # Internal URL
oidc_client_id = "cvlan"
oidc_client_secret = "secret"

# Or LDAP
# method = "ldap"
# ldap_url = "ldaps://ldap.internal.local"
# ldap_base_dn = "dc=internal,dc=local"
```

### Option 3: Certificate-Based

Node authentication via certificates only:

```toml
[auth]
method = "certificate"
ca_cert = "/etc/cvlan/ca.crt"

# Nodes authenticate with client certificates
# No user login required for node-to-node communication
```

## Certificate Management

### Internal CA Setup

```bash
# Create CA (do once)
cvlanctl ca init --output /etc/cvlan/ca

# Generates:
# - /etc/cvlan/ca/ca.crt (distribute to all nodes)
# - /etc/cvlan/ca/ca.key (keep secure!)

# Issue controller certificate
cvlanctl ca issue \
  --ca-dir /etc/cvlan/ca \
  --cn "controller.internal" \
  --san "controller.internal,10.0.0.1" \
  --output /etc/cvlan/certs/controller

# Issue node certificate
cvlanctl ca issue \
  --ca-dir /etc/cvlan/ca \
  --cn "node-001" \
  --output /etc/cvlan/certs/node-001
```

### Certificate Distribution

For air-gapped environments, certificates must be distributed manually:
1. Generate certificates on secure workstation
2. Transfer to nodes via removable media
3. Verify fingerprints

### Certificate Rotation

Plan for certificate rotation before expiry:
- Default validity: 1 year
- Set calendar reminder for renewal
- Document rotation procedure

## Software Updates

### Update Process

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Release Server │     │  Secure         │     │  Air-Gapped     │
│  (internet)     │────►│  Workstation    │────►│  Environment    │
│                 │     │  (transfer)     │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                      │                       │
    Download              Verify & copy           Install
    packages              to media                updates
```

### Verification

Always verify package signatures:

```bash
# On secure workstation (with internet)
curl -LO https://releases.cloudvlan.io/v1.2.0/cvlan-v1.2.0.tar.gz
curl -LO https://releases.cloudvlan.io/v1.2.0/cvlan-v1.2.0.tar.gz.sig

# Verify signature
gpg --verify cvlan-v1.2.0.tar.gz.sig cvlan-v1.2.0.tar.gz

# Copy to removable media
cp cvlan-v1.2.0.tar.gz /media/usb/

# In air-gapped environment
gpg --verify cvlan-v1.2.0.tar.gz.sig cvlan-v1.2.0.tar.gz
tar xzf cvlan-v1.2.0.tar.gz
./install.sh
```

### Internal Package Mirror

For larger deployments, maintain internal package repository:

```bash
# Sync to internal mirror (on transfer workstation)
rsync -av releases.cloudvlan.io::packages /media/transfer/

# In air-gapped environment
# Set up internal mirror
cp -r /media/transfer/packages /var/www/cvlan-releases/

# Configure nodes to use internal mirror
cvlan-agent config set update_url "https://releases.internal/cvlan"
```

## Time Synchronization

Accurate time is critical for:
- Certificate validation
- Log correlation
- Tunnel keepalives

### Options

| Method | Use Case |
|--------|----------|
| Internal NTP server | Preferred, sync to GPS or atomic clock |
| GPS time receiver | Highly accurate, independent |
| Manual sync | Last resort, prone to drift |

### Configuration

```toml
# /etc/cvlan/controller.toml

[time]
# Warn if clock drift exceeds this
max_drift_seconds = 60

# Don't require external NTP
require_ntp = false
```

## Network Configuration

### No Internet Dependency

Ensure no component attempts internet access:

```toml
# /etc/cvlan/controller.toml

[network]
# Disable any internet features
telemetry = false
update_check = false
external_dns = false

[relay]
# Use only internal relays
external_relays = false
```

### Internal DNS

Set up internal DNS for CloudVLAN:

```
; Internal DNS zone
controller.cvlan.internal.    A     10.0.0.10
relay-1.cvlan.internal.       A     10.0.1.10
relay-2.cvlan.internal.       A     10.0.2.10
```

## Operational Procedures

### Initial Deployment Checklist

- [ ] Set up internal CA
- [ ] Issue certificates for all components
- [ ] Deploy controller
- [ ] Deploy relay servers
- [ ] Configure internal DNS
- [ ] Test node enrollment
- [ ] Document all IPs and certificates
- [ ] Create backup procedures
- [ ] Train operators

### Regular Maintenance

| Task | Frequency |
|------|-----------|
| Certificate rotation check | Monthly |
| Backup verification | Weekly |
| Log review | Daily |
| Security updates | As released (via sneakernet) |
| Time sync verification | Weekly |

### Incident Response

Without internet access, must have:
- Offline documentation
- Local expertise
- Pre-staged replacement hardware
- Defined escalation procedures

## Compliance Considerations

Air-gapped deployments often have compliance requirements:

| Framework | Relevance |
|-----------|-----------|
| NIST 800-171 | CUI protection |
| ICS-CERT | Industrial control systems |
| NERC CIP | Power grid |
| FedRAMP High | Government |

CloudVLAN supports:
- Full audit logging
- Certificate-based authentication
- Encryption in transit (WireGuard/IPsec)
- Role-based access control
