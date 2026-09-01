# CellaFlow Engine

[![Docker Image](https://img.shields.io/badge/ghcr.io-cellaflow%2Fcellaflow-blue?logo=docker)](https://github.com/orgs/cellaflow/packages/container/package/cellaflow)
[![Platforms](https://img.shields.io/badge/platform-linux%2Famd64%20%7C%20linux%2Farm64-lightgrey)](#supported-architectures)
[![gRPC](https://img.shields.io/badge/protocol-gRPC%20(HTTP%2F2)-green)](https://docs.cellaflow.com/api-reference)
[![Docs](https://img.shields.io/badge/docs-docs.cellaflow.com-blue)](https://docs.cellaflow.com)
[![PyPI SDK](https://img.shields.io/pypi/v/cellaflow?logo=pypi&label=cellaflow-sdk)](https://pypi.org/project/cellaflow/)

Official Docker image for **CellaFlow Engine** — a systems-grade, stateful execution runtime designed for autonomous AI agents and complex distributed workflows. Built in Rust for extreme performance, memory safety, and sub-millisecond local commit latency.

CellaFlow manages state checkpoints using an immutable **Cognitive Graph**, provides **deterministic replay** for crash recovery, enforces **singleflight idempotency** across agent swarms, and guarantees strict execution semantics with **fencing tokens** and durable leases.

---

## Quick Start

### 1. Pull the Image

```bash
docker pull ghcr.io/cellaflow/cellaflow:latest
```

### 2. Run the Container

Start the engine with persistent storage mounted to `/data`:

```bash
docker run -d \
  --name cellaflow \
  -p 50051:50051 \
  -p 9090:9090 \
  -v cellaflow-data:/data \
  ghcr.io/cellaflow/cellaflow:latest
```

### 3. Verify Health & Readiness

```bash
curl http://localhost:9090/health/ready
# Output: {"status":"ready"} (HTTP 200)
```

### 4. Inspect with gRPC Reflection (Optional)

gRPC Server Reflection is enabled by default. Inspect the exposed services using [`grpcurl`](https://github.com/fullstorydev/grpcurl):

```bash
grpcurl -plaintext localhost:50051 list
# Output:
# cellaflow.v1.WorkflowEngineService
# grpc.reflection.v1.ServerReflection
# grpc.reflection.v1alpha.ServerReflection
```

---

## Exposed Ports

| Port | Protocol | Purpose | Description |
|:---|:---|:---|:---|
| `50051` | gRPC (HTTP/2) | Core Engine API | Primary endpoint for SDKs, workers, and orchestration traffic. |
| `9090` | HTTP/1.1 | Management & Health | HTTP endpoint for Kubernetes/container liveness and readiness probes (`/health/ready`). |

---

## Storage & Persistence

CellaFlow uses **RocksDB** by default for high-throughput, low-latency state persistence.

* **Container Data Directory**: `/data/cellaflow` (configured via `CELLAFLOW_DB_PATH`).
* **Volume Mount**: Mount a host path or Docker named volume to `/data`.
* **Security & Non-Root User**: The container runs under non-root user `appuser` (`UID: 10001`, `GID: 10001`). If you mount a host directory, ensure it has write permissions for UID `10001`:

```bash
mkdir -p ./cellaflow-data
chown -R 10001:10001 ./cellaflow-data
docker run -d \
  --name cellaflow \
  -p 50051:50051 \
  -p 9090:9090 \
  -v $(pwd)/cellaflow-data:/data \
  ghcr.io/cellaflow/cellaflow:latest
```

> **Important**: RocksDB requires single-process exclusive file locks. Do not attach multiple running engine containers simultaneously to the same storage path.

---

## Configuration Reference

All settings can be customized via environment variables:

| Environment Variable | Binary Default | Image Default | Description |
|:---|:---|:---|:---|
| `CELLAFLOW_HOST` | `[::1]` (loopback) | `::` (all interfaces) | Host interface address to bind gRPC server. |
| `CELLAFLOW_PORT` | `50051` | `50051` | gRPC service port. |
| `CELLAFLOW_MANAGEMENT_PORT` | `9090` | `9090` | HTTP health and management probe port. |
| `CELLAFLOW_DB_PATH` | `data/cellaflow` | `/data/cellaflow` | Path to the RocksDB state store directory. |
| `CELLAFLOW_AUTH_TOKEN` | *None* (disabled) | *None* (disabled) | Pre-shared secret key for gRPC API authentication. |
| `CELLAFLOW_TLS_CERT` | *None* (cleartext) | *None* (cleartext) | Path to TLS certificate PEM file inside container. |
| `CELLAFLOW_TLS_KEY` | *None* (cleartext) | *None* (cleartext) | Path to TLS private key PEM file inside container. |
| `CELLAFLOW_LOG_LEVEL` | `info` | `info` | Structured JSON log filter (`trace`, `debug`, `info`, `warn`, `error`). |
| `CELLAFLOW_BACKEND` | `rocksdb` | `rocksdb` | Storage backend engine (`rocksdb`, `postgres`, `dynamodb`). |
| `CELLAFLOW_SYNC_WRITES` | `true` | `true` | When `true`, commits are `fsync`'d before returning (guarantees durability across host power failures). Set `false` for higher throughput if relying on OS cache. |
| `CELLAFLOW_SESSION_IDLE_TIMEOUT_SECS` | `1800` (30m) | `1800` (30m) | Idle duration before an in-memory session actor is safely evicted from RAM. |
| `CELLAFLOW_ACTOR_SEND_TIMEOUT_MS` | `5000` (5s) | `5000` (5s) | Mailbox backpressure timeout before returning `RESOURCE_EXHAUSTED`. |
| `CELLAFLOW_MAX_LEASE_LIFETIME_MS` | `3600000` (1h) | `3600000` (1h) | Maximum ceiling on tool execution lease lifetime. |
| `CELLAFLOW_OTLP_ENDPOINT` | `http://localhost:4317` | `http://localhost:4317` | OpenTelemetry OTLP/gRPC collector endpoint for metrics and traces. |
| `CELLAFLOW_STORAGE_METRICS_INTERVAL_SECS` | `15` | `15` | Interval in seconds between internal storage telemetry snapshots. |

*Note: Common variables also accept unprefixed aliases (e.g. `HOST`, `PORT`, `DB_PATH`, `AUTH_TOKEN`, `API_KEY`, `RUST_LOG`).*

---

## Docker Compose Examples

### 1. Standard Persistent Setup

```yaml
# docker-compose.yml
services:
  cellaflow:
    image: ghcr.io/cellaflow/cellaflow:latest
    container_name: cellaflow
    restart: unless-stopped
    ports:
      - "50051:50051"
      - "9090:9090"
    volumes:
      - cellaflow-data:/data
    environment:
      - CELLAFLOW_LOG_LEVEL=info
      - CELLAFLOW_SYNC_WRITES=true
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9090/health/ready || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 5s

volumes:
  cellaflow-data:
```

### 2. Authenticated Production Setup with Observability

```yaml
# docker-compose.prod.yml
services:
  cellaflow:
    image: ghcr.io/cellaflow/cellaflow:latest
    container_name: cellaflow
    restart: unless-stopped
    ports:
      - "50051:50051"
      - "9090:9090"
    volumes:
      - ./data:/data
      - ./certs:/certs:ro
    environment:
      - CELLAFLOW_AUTH_TOKEN=your-production-secret-token
      - CELLAFLOW_TLS_CERT=/certs/server.crt
      - CELLAFLOW_TLS_KEY=/certs/server.key
      - CELLAFLOW_OTLP_ENDPOINT=http://jaeger:4317
      - CELLAFLOW_LOG_LEVEL=info
    depends_on:
      - jaeger

  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    ports:
      - "16686:16686" # Web UI
      - "4317:4317"   # OTLP gRPC receiver
```

---

## Authentication & Security

When `CELLAFLOW_AUTH_TOKEN` is configured, all gRPC RPC calls must include authentication credentials. The engine validates both:

* **Authorization Header**: `Authorization: Bearer <token>`
* **API Key Header**: `x-api-key: <token>`

### Example Authenticated Call via `grpcurl`

```bash
grpcurl -plaintext \
  -H "x-api-key: your-production-secret-token" \
  -d '{"workflow_id": "research-agent", "version": "1.0.0"}' \
  localhost:50051 \
  cellaflow.v1.WorkflowEngineService/StartSession
```

---

## Health Checks & Readiness Lifecycle

The HTTP health server on port `9090` exposes readiness probes:

```
GET http://<container-host>:9090/health/ready
```

* **`503 Service Unavailable`**: Returned during container startup while boot recovery is in progress (reading RocksDB journals, reconstructing state graphs, respawning durable timers).
* **`200 OK` (`{"status":"ready"}` or `OK`)**: Returned once recovery completes and the gRPC server is actively accepting connections.

> **Best Practice**: Always configure orchestrators (Kubernetes, Docker Compose, Nomad) to wait for `/health/ready` before routing live agent traffic to the engine.

### Graceful Shutdown

The engine natively traps `SIGINT` and `SIGTERM` OS signals (which Kubernetes and Docker send during pod evictions, scaling down, or restarts). Upon receiving a shutdown signal, the engine will:
1. Stop accepting new gRPC connections.
2. Wait for active, in-flight operations and leases to conclude cleanly.
3. Synchronously `fsync` and safely close the RocksDB state store before exiting.

This ensures **zero data loss or corruption** during standard orchestration lifecycle events, meaning it is perfectly safe to rely on Kubernetes rolling updates (`strategy: Recreate`) and standard container restart policies.

---

## Connecting with Client SDKs

### Python SDK Example

Install the SDK:

```bash
pip install cellaflow
```

Create a durable, crash-proof workflow (`agent.py`):

```python
import sys
from cellaflow import workflow, step, tool, IdempotencyScope

# 1. Non-deterministic tool call: automatically cached with fencing tokens
@tool(scope=IdempotencyScope.SCOPE_SESSION_WIDE)
def fetch_data(query: str) -> dict:
    print(f"Executing live search for: {query}")
    return {"query": query, "result": "Extracted insights"}

# 2. Sequential workflow steps
@step
def transform_summary(data: dict) -> str:
    return f"Summary: {data['result']}"

# 3. Deterministic workflow definition
@workflow(version="1.0.0", target="localhost:50051")
def run_agent(query: str) -> str:
    raw = fetch_data(query)
    summary = transform_summary(raw)
    return summary

if __name__ == "__main__":
    session_id = sys.argv[1] if len(sys.argv) > 1 else None
    
    # Passing an existing session_id instantly replays completed steps in 0ms!
    result = run_agent("Autonomous Systems", _session_id=session_id)
    print(f"Result: {result}")
```

---

## Kubernetes & Helm Deployment

### 1. Using the Included Helm Chart

The production Helm chart is included directly in this repository under [`charts/cellaflow-engine`](charts/cellaflow-engine):

```bash
# Install from the root of this repository
helm install cellaflow-engine ./charts/cellaflow-engine \
  --set persistence.size=20Gi
```

For advanced chart configuration (OpenTelemetry APM routing, Ingress, TLS, Secret references), see the [Helm Chart Documentation](charts/cellaflow-engine/README.md).

---

### 2. Using Pure Kubernetes Manifests (`kubectl`)

You can also deploy directly using the standalone manifest provided in [`deploy/kubernetes.yaml`](deploy/kubernetes.yaml):

```bash
kubectl apply -f deploy/kubernetes.yaml
```

> **Key Kubernetes Architectural Rules**:
> * **`replicaCount: 1` & `strategy: Recreate`**: Because RocksDB maintains an exclusive process lock on `/data`, deployments must run with single replicas and `strategy: Recreate` to ensure clean lock handoff during upgrades.
> * **Non-Root Execution**: The container runs under UID `10001`. The provided manifests and Helm chart include a volume permissions `initContainer` to automatically ensure appropriate ownership on the mounted PVC.

---

## Supported Architectures

Images are built natively as multi-architecture container manifests:

* `linux/amd64` (x86_64 servers & cloud instances)
* `linux/arm64` (AWS Graviton, Apple Silicon, Ampere)

---

## Resources & Community

* **Documentation**: [https://docs.cellaflow.com](https://docs.cellaflow.com)
* **Website**: [https://cellaflow.com](https://cellaflow.com)
* **Python SDK (PyPI)**: [https://pypi.org/project/cellaflow/](https://pypi.org/project/cellaflow/)
* **GitHub Organization**: [https://github.com/cellaflow](https://github.com/cellaflow)
* **Issue Tracker & Support**: [support@cellaflow.com](mailto:support@cellaflow.com)