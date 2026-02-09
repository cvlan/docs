# cvlan-vrouter: Virtual Router

## Overview

cvlan-vrouter is a managed IPsec concentrator that acts as a gateway between:
- Legacy IPsec-only devices and the CloudVLAN mesh
- Different tenant networks
- Cloud VPCs and on-premises networks

Think of it as a software-defined VPN gateway, deployable in any cloud.

## Use Cases

### 1. Legacy Device Integration

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  Legacy Router  │         │  cvlan-vrouter  │         │  cvlan-agent    │
│  (IPsec only)   │◄═══════►│                 │◄═══════►│  (WireGuard)    │
│                 │  IPsec  │                 │   WG    │                 │
└─────────────────┘         └─────────────────┘         └─────────────────┘
```

### 2. Site-to-Site Gateway

```
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│         Site A                   │         │         Site B                   │
│  ┌───────────┐  ┌─────────────┐ │         │ ┌─────────────┐  ┌───────────┐  │
│  │  Servers  │──│cvlan-vrouter│◄┼─────────┼►│cvlan-vrouter│──│  Servers  │  │
│  └───────────┘  └─────────────┘ │  IPsec  │ └─────────────┘  └───────────┘  │
└─────────────────────────────────┘         └─────────────────────────────────┘
```

### 3. Cloud VPC Gateway

```
┌─────────────────────────────────┐
│          AWS VPC                 │
│  ┌─────────────┐                │
│  │cvlan-vrouter│────────────────┼──────► CloudVLAN mesh
│  └──────┬──────┘                │
│         │                       │
│  ┌──────┴──────────────────┐   │
│  │  Private Subnet          │   │
│  │  10.0.0.0/16            │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

## Responsibilities

| Function | Description |
|----------|-------------|
| IPsec Termination | Accept IKEv2 connections from clients/routers |
| Protocol Translation | Bridge IPsec ↔ WireGuard traffic |
| Subnet Routing | Advertise and route local subnets |
| Multi-tenant | Isolate tunnels and routes per tenant |
| High Availability | Active-passive or active-active clustering |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                       cvlan-vrouter                         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Control Plane                       │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │   │
│  │  │Controller│  │  Config  │  │   Health/Metrics │   │   │
│  │  │  Client  │  │  Manager │  │                  │   │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Data Plane                         │   │
│  │                                                      │   │
│  │  ┌──────────────┐          ┌──────────────┐         │   │
│  │  │ IPsec Engine │          │  WireGuard   │         │   │
│  │  │              │          │   Engine     │         │   │
│  │  │ - IKEv2      │          │              │         │   │
│  │  │ - ESP        │          │              │         │   │
│  │  └──────┬───────┘          └──────┬───────┘         │   │
│  │         │                         │                  │   │
│  │         ▼                         ▼                  │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │              Routing Engine                   │   │   │
│  │  │  - Policy routing                            │   │   │
│  │  │  - NAT (if needed)                           │   │   │
│  │  │  - Tenant isolation                          │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                            ▼                                │
│                    Network Interfaces                       │
│              (public + private/VPC)                        │
└─────────────────────────────────────────────────────────────┘
```

## IPsec Implementation

### IKEv2 Support

Standard IKEv2 with:
- Certificate authentication (enterprise)
- PSK authentication (simpler setups)
- EAP authentication (user + device)

### Cipher Suites

Support compliant cipher suites:

| Component | Algorithms |
|-----------|------------|
| Encryption | AES-256-GCM, AES-128-GCM, ChaCha20-Poly1305 |
| Integrity | SHA-256, SHA-384, SHA-512 |
| DH Groups | ECP256, ECP384, Curve25519 |
| PRF | HMAC-SHA256, HMAC-SHA384 |

### Implementation Options

1. **strongSwan integration** (recommended initially)
   - Mature, proven
   - Excellent interoperability
   - FFI from Rust control plane

2. **Native Rust** (future)
   - Cleaner integration
   - Better resource control
   - Significant development effort

## Multi-Tenancy

Each vRouter instance can serve multiple tenants:

```
┌─────────────────────────────────────────────────────┐
│                   cvlan-vrouter                     │
│                                                     │
│  ┌─────────────────┐    ┌─────────────────┐       │
│  │   Tenant A      │    │   Tenant B      │       │
│  │                 │    │                 │       │
│  │  VRF: tenant-a  │    │  VRF: tenant-b  │       │
│  │  Tunnels: 5     │    │  Tunnels: 3     │       │
│  │  Subnets:       │    │  Subnets:       │       │
│  │   10.1.0.0/16   │    │   10.2.0.0/16   │       │
│  └─────────────────┘    └─────────────────┘       │
└─────────────────────────────────────────────────────┘
```

Isolation via:
- Linux VRFs or network namespaces
- Separate routing tables
- Firewall rules between tenants

## Configuration

### Controller-Managed

vRouters receive configuration from controller:

```json
{
  "vrouter_id": "vr-abc123",
  "tenant_id": "tenant-xyz",
  "tunnels": [
    {
      "id": "tun-001",
      "type": "ipsec",
      "remote_id": "vpn.customer.com",
      "auth": {
        "method": "psk",
        "psk": "encrypted:..."
      },
      "local_subnets": ["10.0.0.0/8"],
      "remote_subnets": ["192.168.0.0/16"]
    }
  ],
  "wireguard_peers": [
    {
      "public_key": "...",
      "allowed_ips": ["100.64.1.0/24"]
    }
  ]
}
```

### Local Overrides

```toml
# /etc/cvlan/vrouter.toml

[controller]
url = "https://controller.example.com"

[network]
public_interface = "eth0"
private_interface = "eth1"

[ipsec]
# Use strongSwan
backend = "strongswan"
strongswan_socket = "/var/run/charon.vici"

[performance]
# Tune for throughput
worker_threads = 4
```

## High Availability

### Active-Passive

```
┌─────────────────┐         ┌─────────────────┐
│  vRouter Primary│         │ vRouter Standby │
│                 │◄───────►│                 │
│  VIP: 1.2.3.4   │  sync   │                 │
│  (active)       │         │  (passive)      │
└─────────────────┘         └─────────────────┘
```

- Floating VIP
- State sync for IKE SAs
- Automatic failover

### Active-Active

```
           ┌─────────────────┐
           │  Load Balancer  │
           │  (DNS or ECMP)  │
           └────────┬────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│   vRouter 1     │     │   vRouter 2     │
└─────────────────┘     └─────────────────┘
```

- Stateless design where possible
- Session affinity via IKE cookie
- No shared state required for WireGuard

## Cloud Deployment

### Cloud-Neutral Design

No cloud-specific APIs. Works anywhere with:
- Public IP address
- Standard Linux networking
- Optionally: private network interface for VPC

### Deployment Templates

Provide templates for common clouds:
- AWS: CloudFormation / Terraform
- GCP: Deployment Manager / Terraform
- Azure: ARM / Terraform
- Generic: cloud-init script

### Example (AWS)

```hcl
resource "aws_instance" "vrouter" {
  ami           = data.aws_ami.cvlan_vrouter.id
  instance_type = "t3.small"

  network_interface {
    network_interface_id = aws_network_interface.public.id
    device_index         = 0
  }

  network_interface {
    network_interface_id = aws_network_interface.private.id
    device_index         = 1
  }

  user_data = <<-EOF
    #!/bin/bash
    cvlan-vrouter init \
      --controller https://controller.example.com \
      --token ${var.vrouter_token}
  EOF
}
```

## Sizing

| Size | Tunnels | Throughput | Instance |
|------|---------|------------|----------|
| Small | 100 | 500 Mbps | 2 vCPU, 2 GB |
| Medium | 500 | 2 Gbps | 4 vCPU, 4 GB |
| Large | 2000 | 10 Gbps | 8 vCPU, 8 GB |

## Monitoring

### Metrics

```
# Tunnel status
cvlan_vrouter_tunnels_active{tenant="...",type="ipsec"} 5
cvlan_vrouter_tunnels_active{tenant="...",type="wireguard"} 10

# Traffic
cvlan_vrouter_bytes_in_total{tunnel="..."} 123456789
cvlan_vrouter_bytes_out_total{tunnel="..."} 987654321

# IPsec specific
cvlan_vrouter_ike_sa_active 15
cvlan_vrouter_child_sa_active 20
cvlan_vrouter_ike_rekeys_total 100
```

### Logging

Structured logs for:
- Tunnel up/down events
- Authentication failures
- Routing changes
- HA failover events

## Security Considerations

| Concern | Mitigation |
|---------|------------|
| Key storage | Keys in memory only, or secure enclave |
| Tenant isolation | VRF/namespace separation, strict firewall |
| DDoS | Rate limiting, cloud DDoS protection |
| Compromise | Each vRouter isolated, limited blast radius |
