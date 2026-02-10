# E2E Stack

The E2E test environment runs all CloudVLAN services in Docker Compose for integration testing.

## Topology

![E2E Topology](../images/e2e-topology.svg)

## Services

| Service | Image | Port (external) | Profile | Purpose |
|---------|-------|-----------------|---------|---------|
| **postgres** | postgres:16-alpine | 5434 | — | Database |
| **step-ca** | smallstep/step-ca | — | — | Certificate authority for mTLS |
| **controller** | cvlan-api | 9080 | — | Full API (controller mode) |
| **discovery** | cvlan-api | 9081 | — | Discovery endpoint |
| **ui** | cvlan-ui | 3000 | — | Management frontend |
| **vrouter** | vrouter | — | vrouter | VPP gateway (optional) |
| **client** | cvlan-client | — | client | Legacy client daemon (optional) |
| **client-1** | cvlan-client | — | cvlan-client | cvlan-ctrl node 1 (optional) |
| **client-2** | cvlan-client | — | cvlan-client | cvlan-ctrl node 2 (optional) |
| **test-runner** | ubuntu:24.04 | — | — | curl-based test scripts |

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
cd ~/ws/e2e && scripts/up --client           # Include legacy client daemon
cd ~/ws/e2e && scripts/up --cvlan-client     # Bootstrap 2 cvlan-ctrl nodes (register + poll)

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

## Client Nodes (`--cvlan-client`)

The `--cvlan-client` flag bootstraps a full client registration flow:

1. Starts base services (postgres, step-ca, controller)
2. Creates an e2e tenant with `identifier` and `public_key` (required for coordination)
3. Creates an `e2e-net` CVLAN (10.200.0.0/24)
4. Creates 2 single-use registration tokens
5. Starts `client-1` and `client-2` containers running `cvlan-ctrl`
6. Both clients register, receive real X.509 certs from step-ca, and begin polling
7. Each client discovers the other as a peer (`Peers updated added=1`)

On subsequent runs, if certs already exist on disk (`data/client-{1,2}/certs/client.crt`), the bootstrap is skipped and clients resume with existing credentials.

```bash
# Full bootstrap
./scripts/up --cvlan-client

# Clean client state (forces re-registration on next start)
make clean-cvlan-client

# Shell into containers
./scripts/shells/shell-client1
./scripts/shells/shell-client2

# Verify registration
docker logs e2e-client-1-1 2>&1 | grep "Registered\|Peers updated"
```

## Test Scripts

```
e2e/
├── scripts/
│   ├── up                     # Start services (with optional flags)
│   ├── down                   # Stop and remove
│   ├── test                   # Run E2E test suite
│   ├── token                  # Generate JWT for manual testing
│   ├── cleanup-cvlan-client   # Clean client-1/client-2 state
│   └── shells/
│       ├── shell-api          # Shell into controller
│       ├── shell-disc         # Shell into discovery
│       ├── shell-db           # Shell into postgres
│       ├── shell-client1      # Shell into client-1 (cvlan-ctrl)
│       └── shell-client2      # Shell into client-2 (cvlan-ctrl)
├── config/
│   ├── vrouter.yaml           # VRouter E2E config
│   └── cvlan-ctrl.yaml        # Client control plane E2E config
└── tests/
    └── run_e2e.sh             # E2E test suite (CLIENT_TESTS=1 for client tests)
```

## Refreshing a Single Service

To rebuild and restart one service without tearing down the whole stack:

```bash
~/ws/e2e/refresh <service-name>
```

This rebuilds the Docker image and recreates just that container, preserving the database and other services.
