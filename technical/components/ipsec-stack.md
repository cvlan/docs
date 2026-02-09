# IPsec Stack Integration

## Overview

IPsec/IKEv2 support is a key differentiator for CloudVLAN. While WireGuard is the default for modern endpoints, IPsec provides:

- Compliance (FIPS 140-2, government requirements)
- Compatibility with legacy devices
- Native OS support (no additional software)
- Enterprise acceptance

## Implementation Strategy

### Phase 1: strongSwan Integration

Wrap strongSwan via its VICI control interface:

```
┌─────────────────────────────────────────────────────────┐
│                    cvlan-agent                          │
│                                                         │
│  ┌─────────────────┐      ┌───────────────────────┐    │
│  │  IPsec Manager  │      │   WireGuard Manager   │    │
│  │     (Rust)      │      │       (Rust)          │    │
│  └────────┬────────┘      └───────────────────────┘    │
│           │                                             │
│           │ VICI protocol                               │
│           ▼                                             │
│  ┌─────────────────┐                                   │
│  │   strongSwan    │                                   │
│  │   (charon)      │                                   │
│  └─────────────────┘                                   │
└─────────────────────────────────────────────────────────┘
```

**Pros:**
- Proven, mature implementation
- Excellent interoperability
- Faster time to market

**Cons:**
- C dependency
- Separate process to manage
- Less control over resource usage

### Phase 2: Native Rust (Future)

Pure Rust IKEv2/ESP implementation:

```
┌─────────────────────────────────────────────────────────┐
│                    cvlan-agent                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Unified Tunnel Engine               │   │
│  │                                                  │   │
│  │  ┌─────────────┐         ┌─────────────────┐    │   │
│  │  │  WireGuard  │         │  IPsec (native) │    │   │
│  │  │   Module    │         │     Module      │    │   │
│  │  └─────────────┘         └─────────────────┘    │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**Pros:**
- Single binary
- Full control over resources
- Better integration

**Cons:**
- Significant development effort
- Must prove interoperability
- Security audit required

## IKEv2 Protocol Support

### Authentication Methods

| Method | Use Case | Priority |
|--------|----------|----------|
| Certificates (X.509) | Enterprise, managed devices | High |
| Pre-Shared Keys | Simple deployments, testing | High |
| EAP-TLS | User + device auth | Medium |
| EAP-MSCHAPv2 | Windows native integration | Medium |

### Cipher Suites

Support common enterprise configurations:

```
# Encryption
AES-256-GCM (default)
AES-128-GCM
ChaCha20-Poly1305

# Integrity (non-AEAD)
HMAC-SHA256
HMAC-SHA384

# Key Exchange
ECP256 (default)
ECP384
Curve25519
MODP2048 (legacy compatibility)

# PRF
HMAC-SHA256
HMAC-SHA384
```

### Configuration Payloads

Support IKEv2 configuration payloads for:
- Internal IP address assignment
- DNS server configuration
- Split tunneling (traffic selectors)

## strongSwan Integration

### VICI Protocol

strongSwan's VICI (Versatile IKE Configuration Interface):

```rust
use vici::{Client, Message};

struct StrongswanManager {
    client: Client,
}

impl StrongswanManager {
    async fn connect() -> Result<Self> {
        let client = Client::connect("/var/run/charon.vici").await?;
        Ok(Self { client })
    }

    async fn load_connection(&self, config: &IpsecConfig) -> Result<()> {
        let msg = Message::new()
            .add("version", 2)
            .add("local_addrs", vec![config.local_addr])
            .add("remote_addrs", vec![config.remote_addr])
            // ... more config
            ;

        self.client.request("load-conn", msg).await?;
        Ok(())
    }

    async fn initiate(&self, name: &str) -> Result<()> {
        let msg = Message::new().add("child", name);
        self.client.request("initiate", msg).await?;
        Ok(())
    }

    async fn terminate(&self, name: &str) -> Result<()> {
        let msg = Message::new().add("ike", name);
        self.client.request("terminate", msg).await?;
        Ok(())
    }

    async fn list_sas(&self) -> Result<Vec<SecurityAssociation>> {
        let response = self.client.request("list-sas", Message::new()).await?;
        // Parse response
        Ok(sas)
    }
}
```

### strongSwan Configuration

Generate strongSwan config from CloudVLAN policy:

```
# /etc/strongswan.d/cvlan.conf

connections {
    cvlan-peer-1 {
        version = 2
        local_addrs = %any
        remote_addrs = 203.0.113.10

        local {
            auth = pubkey
            certs = /etc/cvlan/certs/node.crt
            id = node-abc123
        }

        remote {
            auth = pubkey
            id = peer-xyz789
        }

        children {
            cvlan-child-1 {
                local_ts = 100.64.1.0/24
                remote_ts = 100.64.2.0/24
                esp_proposals = aes256gcm128-ecp256
            }
        }
    }
}
```

### Process Management

cvlan-agent manages strongSwan lifecycle:

```rust
struct StrongswanProcess {
    child: Child,
    vici: StrongswanManager,
}

impl StrongswanProcess {
    async fn start() -> Result<Self> {
        // Start charon daemon
        let child = Command::new("charon")
            .arg("--debug-ike").arg("2")
            .spawn()?;

        // Wait for VICI socket
        tokio::time::sleep(Duration::from_secs(1)).await;

        let vici = StrongswanManager::connect().await?;
        Ok(Self { child, vici })
    }

    async fn stop(&mut self) -> Result<()> {
        self.child.kill().await?;
        Ok(())
    }
}
```

## Certificate Management

### PKI Integration

For enterprise deployments:

```
┌─────────────────┐
│  Enterprise CA  │
│  (or cvlan CA)  │
└────────┬────────┘
         │
         │ Issues certificates
         ▼
┌─────────────────┐     ┌─────────────────┐
│  cvlan-agent    │     │  cvlan-vrouter  │
│  cert: node.crt │     │  cert: vr.crt   │
└─────────────────┘     └─────────────────┘
```

### Certificate Distribution

Options:
1. **Manual**: Admin provisions certificates
2. **ACME-like**: Automated certificate issuance via controller
3. **SCEP/EST**: Enterprise certificate enrollment

### Certificate Validation

```rust
fn validate_peer_cert(cert: &X509, peer_id: &str) -> Result<()> {
    // Check chain to trusted CA
    verify_chain(cert, &TRUSTED_CAS)?;

    // Check not expired
    verify_validity(cert)?;

    // Check revocation (CRL or OCSP)
    verify_not_revoked(cert)?;

    // Check identity matches expected peer
    verify_identity(cert, peer_id)?;

    Ok(())
}
```

## Interoperability

### Tested Platforms

| Platform | Version | Auth Methods | Notes |
|----------|---------|--------------|-------|
| Cisco IOS | 15.x+ | PSK, Cert | Standard IKEv2 |
| Cisco ASA | 9.x+ | PSK, Cert | Crypto map or VTI |
| Palo Alto | PAN-OS 8+ | PSK, Cert | |
| Fortinet | FortiOS 6+ | PSK, Cert | |
| Windows | 10/11, Server 2016+ | Cert, EAP | Native IKEv2 |
| macOS | 10.15+ | Cert | Native IKEv2 |
| iOS | 13+ | Cert | Native IKEv2 |
| Android | 11+ | Cert | Native IKEv2 |
| strongSwan | 5.x | All | Reference impl |
| Libreswan | 4.x | All | |

### Compatibility Matrix

Test matrix for each release:

```
          | Cisco | PaloAlto | Windows | macOS | strongSwan
----------|-------|----------|---------|-------|------------
PSK       |  ✓    |    ✓     |   ✓     |   -   |     ✓
Cert      |  ✓    |    ✓     |   ✓     |   ✓   |     ✓
EAP-TLS   |  -    |    -     |   ✓     |   -   |     ✓
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Phase 1 timeout | Firewall blocking UDP 500/4500 | Check firewall rules |
| Auth failure | Certificate mismatch | Verify ID and certificate CN/SAN |
| No traffic | Traffic selectors mismatch | Check local_ts/remote_ts |
| Rekey failure | Lifetime mismatch | Align rekey timers |

### Debug Logging

```bash
# Enable verbose logging
cvlan-agent --log-level debug

# strongSwan specific
cvlan-agent --ipsec-debug 2
```

### Diagnostics

```bash
# Show IPsec status
cvlanctl ipsec status

# Show SAs
cvlanctl ipsec list-sas

# Test connectivity
cvlanctl ipsec test-peer <peer-id>
```
