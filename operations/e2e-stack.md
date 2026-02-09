# E2E Stack

The E2E test environment runs all CloudVLAN services in Docker Compose for integration testing.

## Topology

![E2E Topology](../images/e2e-topology.svg)

## Services

| Service | Image | Port (external) | Purpose |
|---------|-------|-----------------|---------|
| **postgres** | postgres:16-alpine | 5434 | Database |
| **controller** | cvlan-api | 9080 | Full API (controller mode) |
| **discovery** | cvlan-api | 9081 | Discovery endpoint |
| **ui** | cvlan-ui | 3000 | Management frontend |
| **vrouter** | vrouter | — | VPP gateway (optional) |
| **client** | cvlan-ctrl | — | Client daemon (optional) |
| **test-runner** | ubuntu:24.04 | — | curl-based test scripts |

## Configuration

### Ports

| Service | Internal | External |
|---------|----------|----------|
| Controller | 8080 | 9080 |
| Discovery | 8080 | 9081 |
| UI | 80 | 3000 |
| PostgreSQL | 5432 | 5434 |
| step-ca | 9000 | 9000 |

### Environment

| Variable | Value |
|----------|-------|
| JWT Secret | `e2e-test-secret` |
| Database URL | `postgres://cvlan:cvlan_e2e@postgres:5432/cvlan_e2e` |
| Discovery URL | `http://discovery:8080` |
| Controller URL | `http://controller:8080` |

### Network

All services share a Docker network (`e2e_default`). This is isolated from the development and test stacks.

## Usage

```bash
# Start all services
cd ~/ws/e2e && make build-all && make up

# Start with optional services
cd ~/ws/e2e && scripts/up --vrouter          # Include VPP gateway
cd ~/ws/e2e && scripts/up --bootstrap        # VRouter in bootstrap mode (no VPP)
cd ~/ws/e2e && scripts/up --client           # Include client daemon

# Run tests
cd ~/ws/e2e && make test

# Tear down
cd ~/ws/e2e && make down

# Generate JWT token for manual testing
cd ~/ws/e2e && scripts/token
```

## VRouter Modes

The VRouter can run in two modes:

| Mode | Flag | Requirements | Use Case |
|------|------|-------------|----------|
| **Full** | `--vrouter` | Privileged container, DPDK | VPP testing |
| **Bootstrap** | `--bootstrap` | Unprivileged | Software crypto, no VPP |

Bootstrap mode is the default for CI/CD — it doesn't need DPDK or hugepages.

## Test Scripts

```
e2e/
├── scripts/
│   ├── up              # Start services (with optional flags)
│   ├── down            # Stop and remove
│   ├── test            # Run E2E test suite
│   ├── token           # Generate JWT for manual testing
│   └── shells/
│       ├── shell-controller   # SSH into controller container
│       ├── shell-discovery    # SSH into discovery container
│       └── shell-postgres     # SSH into postgres container
└── tests/
    └── *.sh            # Individual test scenarios (curl-based)
```

## Refreshing a Single Service

To rebuild and restart one service without tearing down the whole stack:

```bash
~/ws/e2e/refresh <service-name>
```

This rebuilds the Docker image and recreates just that container, preserving the database and other services.
