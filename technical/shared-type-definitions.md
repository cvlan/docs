# Shared Type Definitions

How Protocol Buffers provide a single source of truth for data types across Rust and TypeScript.

## The Problem

Without shared types, the API server, gateway, client, and UI each define their own versions of the same structures. This leads to:
- Field name drift (snake_case vs camelCase)
- Missing fields in some consumers
- Enum value mismatches
- Manual synchronization when adding fields

## The Solution

![Type Generation Pipeline](../images/type-generation-pipeline.svg)

All shared types are defined once in `.proto` files and generated into both Rust and TypeScript:

```
protodefs/cloudvlan/v1/*.proto
         │
    buf generate
    ├────────────────┐
    │                │
    ▼                ▼
Rust types       TypeScript types
(prost)          (ts-proto)
```

## Proto Structure

```
protodefs/
├── buf.yaml              # Module config (package: cloudvlan.v1)
├── buf.gen.yaml          # Code generation config
└── cloudvlan/v1/
    ├── common.proto      # Status, Capacity, Pagination
    ├── vrouter.proto     # VRouter, VRouterStats
    ├── region.proto      # Region, DerpNode
    └── ...               # More as needed
```

## Design Conventions

| Convention | Example |
|------------|---------|
| Package | `cloudvlan.v1` |
| Enums start at 0 | `STATUS_UNSPECIFIED = 0` |
| Optional fields | `optional string description = 3;` |
| UUIDs | Serialized as strings |
| Timestamps | `google.protobuf.Timestamp` |
| Lists | `repeated` keyword |

## Code Generation

### Tools

| Tool | Version | Purpose |
|------|---------|---------|
| **buf** | v1.28.1 | Proto management, lint, breaking change detection |
| **protoc-gen-prost** | — | Rust struct generation |
| **protoc-gen-prost-serde** | — | serde Serialize/Deserialize derives |
| **protoc-gen-ts_proto** | — | TypeScript interface generation |

### Outputs

**Rust** (api/src/gen/cloudvlan/v1/):
```rust
#[derive(Clone, PartialEq, ::prost::Message, serde::Serialize, serde::Deserialize)]
pub struct VRouter {
    #[prost(string, tag = "1")]
    pub id: String,
    #[prost(string, tag = "2")]
    pub name: String,
    // ...
}
```

**TypeScript** (protodefs/gen/ts/cloudvlan/v1/):
```typescript
export interface VRouter {
  id: string;
  name: string;
  // ...
}
```

### Running Generation

```bash
# Inside cvlan-build container (never on host)
cd /app/protodefs && buf generate
```

TypeScript output is synced from `protodefs/gen/ts/` to `ui/src/gen/` to keep the UI repo self-contained.

## Consumers

| Consumer | Language | Usage |
|----------|----------|-------|
| cvlan-api | Rust | API request/response types |
| vrouterd | Rust | Configuration types from controller |
| cvlan-ctrl | Rust | Registration/poll types |
| Management UI | TypeScript | API client types |

## Migration Status

The proto type system is being rolled out incrementally. Current REST handlers still use `models::` types with hand-written serde. The migration to `gen::` types (proto3 canonical JSON: camelCase, uppercase enums) is tracked as a future task.
