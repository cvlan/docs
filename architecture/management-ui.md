# Management UI

The management UI is a React single-page application that provides a web interface for all CloudVLAN management operations. It talks to the cvlan-api REST API.

## Technology Stack

| Layer | Choice |
|-------|--------|
| Framework | React 18 |
| Language | TypeScript 5 |
| Build | Vite 5 |
| State (server) | TanStack Query (React Query) |
| State (client) | Zustand |
| Styling | Tailwind CSS |
| Linting | ESLint |
| Testing | Vitest |

## Pages

The UI covers all entity types in the system:

| Page | Operations | Notes |
|------|-----------|-------|
| **Tenants** | List, create, edit, delete | Superadmin only |
| **CVLANs** | List, create, edit, delete | CIDR display, node count |
| **Nodes** | List, create, edit, delete | Status indicator, last seen |
| **Users** | List, create, edit, delete | Role assignment |
| **Groups** | List, create, edit, members | Group membership management |
| **Policies** | List, create, edit, delete | Source/destination groups, allow/deny |
| **VRouters** | List, create, edit, delete | Region, status, stats |
| **Regions** | List, create, edit, delete | Provider, location |
| **Relays** | List, create, edit, delete | Future |
| **Audit Logs** | List, filter, export | 90-day retention |

## Architecture

```
Browser
  └── React SPA (:3000)
        ├── TanStack Query → cvlan-api REST (:8080/9080)
        ├── Zustand (local UI state: selected tenant, theme, etc.)
        └── Tailwind CSS (utility-first styling)
```

- **TanStack Query** handles all server state: caching, background refetching, optimistic updates, pagination
- **Zustand** handles client-only state: currently selected tenant, UI preferences, sidebar state
- **No Redux** — intentionally avoided for simplicity

## Type Safety

TypeScript types are generated from Protocol Buffers via ts-proto. The UI imports generated types from `src/gen/cloudvlan/v1/`, ensuring the UI and API always agree on data shapes.

## Build & Development

```bash
# Development (hot reload)
cd ~/ws/ui && ./build/scripts/dev.sh     # :3000

# Production build
cd ~/ws/ui && ./build/scripts/build-oci   # Docker image

# Lint
cd ~/ws/ui && ./build/scripts/lint.sh

# Test
cd ~/ws/ui && ./build/scripts/test.sh -- --run
```

All builds run inside Docker containers (never npm on host).

## Repository Structure

```
ui/
├── src/
│   ├── components/    # Reusable UI components
│   ├── pages/         # Route-level pages
│   ├── hooks/         # Custom React hooks
│   ├── gen/           # Generated TypeScript types (from protobuf)
│   ├── api/           # API client functions
│   └── stores/        # Zustand stores
├── build/
│   └── scripts/       # Docker-based build scripts
└── vite.config.ts
```
