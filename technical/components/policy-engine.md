# cvlan-policy: Policy Engine

## Overview

The policy engine manages access control across the CloudVLAN network. It defines who can communicate with whom, using what protocols, on which ports.

**Key principle**: Deny by default. All traffic is blocked unless explicitly allowed.

## Design Goals

1. **Simple for common cases**: Easy to express "devs can access dev servers"
2. **Powerful for complex needs**: Support for tags, groups, conditions
3. **Performant**: Evaluated on every packet, must be fast
4. **Distributed**: Policies enforced locally on each node
5. **Protocol-agnostic**: Same policies for WireGuard and IPsec

## Policy Model

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Subject** | Who is making the request (user, group, tag, IP) |
| **Object** | What they're accessing (node, tag, IP, subnet) |
| **Action** | Allow or Deny |
| **Conditions** | Port, protocol, time, etc. |

### Policy Structure

```yaml
policies:
  - name: "developers-to-dev-servers"
    subjects:
      - group: developers
    objects:
      - tag: env:development
    ports:
      - 22    # SSH
      - 443   # HTTPS
    action: allow

  - name: "block-prod-from-contractors"
    subjects:
      - group: contractors
    objects:
      - tag: env:production
    action: deny
    priority: 100  # Higher = evaluated first
```

## Matching

### Subject Matchers

| Type | Syntax | Example |
|------|--------|---------|
| User | `user:<email>` | `user:alice@example.com` |
| Group | `group:<name>` | `group:developers` |
| Tag | `tag:<key>:<value>` | `tag:team:platform` |
| Node | `node:<name>` | `node:server-1` |
| IP/CIDR | `ip:<cidr>` | `ip:10.0.0.0/8` |
| Any | `*` | `*` |

### Object Matchers

Same syntax as subjects, plus:

| Type | Syntax | Example |
|------|--------|---------|
| Port | `port:<number>` | `port:443` |
| Port range | `port:<start>-<end>` | `port:8000-9000` |
| Protocol | `proto:<name>` | `proto:tcp` |

## Policy File Format

### HuJSON (Human JSON)

Like Tailscale, support HuJSON for readability:

```hjson
{
  // Network policies for CloudVLAN

  groups: {
    "developers": ["alice@example.com", "bob@example.com"],
    "ops": ["carol@example.com"],
    "admins": ["group:developers", "group:ops"],
  },

  tagOwners: {
    "tag:production": ["group:ops"],
    "tag:development": ["group:developers"],
  },

  acls: [
    // Admins can access everything
    {
      action: "accept",
      src: ["group:admins"],
      dst: ["*:*"],
    },

    // Developers can access dev environment
    {
      action: "accept",
      src: ["group:developers"],
      dst: ["tag:development:*"],
    },

    // Everyone can access DNS
    {
      action: "accept",
      src: ["*"],
      dst: ["*:53"],
    },
  ],
}
```

## Evaluation

### Order of Evaluation

1. Check policies in priority order (highest first)
2. First match wins
3. If no match, default deny

### Evaluation Algorithm

```rust
fn evaluate(packet: &Packet, policies: &[Policy]) -> Action {
    // Sort by priority (cached)
    for policy in policies {
        if matches_subject(&packet.source, &policy.subjects) &&
           matches_object(&packet.dest, &policy.objects) &&
           matches_conditions(packet, &policy.conditions) {
            return policy.action;
        }
    }
    Action::Deny  // Default
}

fn matches_subject(source: &Source, matchers: &[Matcher]) -> bool {
    matchers.iter().any(|m| m.matches(source))
}
```

### Performance

Policies are evaluated on every packet. Optimization strategies:

| Strategy | Description |
|----------|-------------|
| **Compile to rules** | Convert policies to iptables/nftables rules |
| **BPF** | Use eBPF for kernel-level enforcement |
| **Caching** | Cache (src, dst, port) → decision |
| **Indexing** | Index policies by subject/object for fast lookup |

## Distribution

### Flow

```
┌────────────────┐
│  Admin edits   │
│  policy file   │
└───────┬────────┘
        │
        ▼
┌────────────────┐
│  Controller    │
│  validates &   │
│  stores        │
└───────┬────────┘
        │
        ▼
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│  cvlan-agent   │     │  cvlan-agent   │     │  cvlan-vrouter │
│  (node 1)      │     │  (node 2)      │     │                │
│                │     │                │     │                │
│  ┌──────────┐  │     │  ┌──────────┐  │     │  ┌──────────┐  │
│  │ compiled │  │     │  │ compiled │  │     │  │ compiled │  │
│  │ policies │  │     │  │ policies │  │     │  │ policies │  │
│  └──────────┘  │     │  └──────────┘  │     │  └──────────┘  │
└────────────────┘     └────────────────┘     └────────────────┘
```

### Update Mechanism

1. Admin updates policy via API or file
2. Controller validates syntax and semantics
3. Controller computes per-node policy view
4. Push to nodes via existing control channel
5. Nodes compile and apply atomically

### Versioning

```rust
struct PolicyUpdate {
    version: u64,
    policies: Vec<CompiledPolicy>,
    checksum: [u8; 32],
}
```

Nodes track version, request full sync if gap detected.

## Local Enforcement

### Linux Implementation

Convert policies to nftables rules:

```nft
table inet cvlan {
    chain input {
        type filter hook input priority 0;

        # Allow established
        ct state established,related accept

        # Policy: devs → dev-servers:22,443
        ip saddr @developers ip daddr @dev_servers tcp dport {22, 443} accept

        # Default deny
        drop
    }
}
```

### macOS/Windows

Userspace enforcement in cvlan-agent:
- Inspect packets from TUN device
- Apply policy before forwarding
- Drop denied packets

## Advanced Features

### Time-Based Policies

```yaml
- name: "maintenance-window"
  subjects:
    - group: ops
  objects:
    - tag: production
  action: allow
  conditions:
    time:
      days: [saturday, sunday]
      hours: "02:00-06:00"
      timezone: "America/New_York"
```

### Approval Workflows (Future)

```yaml
- name: "prod-access-requires-approval"
  subjects:
    - group: developers
  objects:
    - tag: production
  action: allow
  conditions:
    requires_approval:
      approvers: ["group:ops"]
      duration: "4h"
```

### Posture Checks (Future)

```yaml
- name: "require-updated-agent"
  subjects:
    - "*"
  objects:
    - tag: sensitive
  action: allow
  conditions:
    posture:
      agent_version: ">=1.5.0"
      os_updated_within: "7d"
```

## API

### Policy Management

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/policies` | GET | List all policies |
| `/api/v1/policies` | PUT | Replace all policies |
| `/api/v1/policies/validate` | POST | Validate without applying |
| `/api/v1/policies/test` | POST | Test a hypothetical packet |

### Testing

```bash
# Test if alice can reach server-1:443
curl -X POST https://controller/api/v1/policies/test \
  -d '{
    "source": {"user": "alice@example.com", "node": "laptop-1"},
    "destination": {"node": "server-1", "port": 443, "proto": "tcp"}
  }'

# Response
{
  "action": "allow",
  "matched_policy": "developers-to-dev-servers",
  "evaluation_path": [...]
}
```

## Audit Logging

All policy decisions logged:

```json
{
  "timestamp": "2026-01-19T10:30:00Z",
  "source_node": "laptop-1",
  "source_user": "alice@example.com",
  "dest_node": "server-1",
  "dest_port": 443,
  "proto": "tcp",
  "action": "allow",
  "policy": "developers-to-dev-servers"
}
```

For compliance, ship logs to SIEM.
