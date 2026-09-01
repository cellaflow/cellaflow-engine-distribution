# CellaFlow Engine — Helm Chart

Deploy the **CellaFlow Engine** to your Kubernetes cluster in minutes. This chart installs the CellaFlow gRPC engine alongside an optional OpenTelemetry Collector sidecar for routing observability data to your APM provider.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration Reference](#configuration-reference)
- [Telemetry & Observability](#telemetry--observability)
  - [Sending to Your APM](#sending-to-your-apm)
  - [Opting Out of Fleet Telemetry](#opting-out-of-fleet-telemetry)
  - [Disabling Telemetry Entirely](#disabling-telemetry-entirely)
- [Persistence](#persistence)
- [Ingress (gRPC over TLS)](#ingress-grpc-over-tls)
- [Resource Tuning](#resource-tuning)
- [Health & Readiness](#health--readiness)
- [Upgrade Notes](#upgrade-notes)
- [Uninstalling](#uninstalling)
- [Values Reference](#values-reference)

---

## Prerequisites

| Requirement | Minimum Version |
|---|---|
| Kubernetes | 1.25+ |
| Helm | 3.12+ |
| Persistent storage | `ReadWriteOnce` storage class available |

---

## Quick Start

### Install the chart

From the root of this repository:

```bash
helm install cellaflow-engine ./charts/cellaflow-engine
```

Or from within the `charts/cellaflow-engine` directory:

```bash
helm install cellaflow-engine .
```

The engine will be running within ~30 seconds. Verify:

```bash
kubectl get pods -l app.kubernetes.io/name=cellaflow-engine
```

Expected output:

```
NAME                               READY   STATUS    RESTARTS   AGE
cellaflow-engine-xxxx-yyyy         2/2     Running   0          30s
```

---

## Configuration Reference

All configuration is done via `values.yaml` or `--set` flags. To see the full defaults:

```bash
helm show values ./charts/cellaflow-engine
```

To use a custom values file:

```bash
helm install cellaflow-engine ./charts/cellaflow-engine \
  -f my-values.yaml
```

---

## Telemetry & Observability

CellaFlow ships with an embedded OpenTelemetry Collector sidecar. It supports two independent telemetry routing paths:

| Route | Description |
|---|---|
| **Customer APM** | Your own observability stack (Datadog, Grafana Cloud, New Relic, etc.) |
| **Fleet telemetry** | Anonymous aggregate metrics sent to CellaFlow for reliability improvements |

The sidecar is only deployed when at least one telemetry path is active. If both are disabled, no sidecar is injected and no observability data leaves the pod.

---

### Sending to Your APM

Set your APM's OTLP/gRPC endpoint in `values.yaml`:

```yaml
telemetry:
  customerApm:
    endpoint: "https://otlp.datadoghq.com:4317"  # Your APM's OTLP endpoint
    apiKey: "dd-api-key-here"                      # Provide key directly
```

**Preferred — reference a Kubernetes Secret instead of embedding the key:**

```bash
kubectl create secret generic my-apm-secret --from-literal=apiKey=<your-api-key>
```

```yaml
telemetry:
  customerApm:
    endpoint: "https://otlp.datadoghq.com:4317"
    apiKeySecretRef: "my-apm-secret"  # Name of the Secret
```

#### Common APM Endpoints

| Provider | OTLP Endpoint |
|---|---|
| Datadog | `https://otlp.datadoghq.com:4317` |
| Grafana Cloud | `https://<your-instance>.grafana.net:443` |
| New Relic | `https://otlp.nr-data.net:4317` |
| Honeycomb | `https://api.honeycomb.io:443` |
| Self-hosted (Jaeger, Tempo) | `http://<your-collector>:4317` |

---

### Opting Out of Fleet Telemetry

Fleet telemetry sends only anonymous aggregate metrics (no traces, no logs, no customer data) to CellaFlow's reliability pipeline. To disable it:

```yaml
telemetry:
  optOut: true
```

> Even with `optOut: false` (the default), the fleet pipeline strips all customer-identifying attributes before forwarding. Only metrics are forwarded — never traces or logs.

---

### Disabling Telemetry Entirely

To run in a fully air-gapped environment with zero telemetry:

```yaml
telemetry:
  optOut: true          # Disable fleet telemetry
  customerApm:
    endpoint: ""        # Empty = no customer APM pipeline
```

When both are disabled, the OTel Collector sidecar is not injected at all.

---

## Persistence

CellaFlow uses RocksDB for durable workflow state. A `PersistentVolumeClaim` is created automatically.

```yaml
persistence:
  enabled: true          # Set to false to use ephemeral storage (not recommended)
  storageClass: ""       # Leave empty to use the cluster default storage class
  accessMode: ReadWriteOnce
  size: 10Gi             # Adjust for your expected workflow volume
```

> **Important:** `ReadWriteOnce` means the PVC can only be mounted by a single node at a time. This is correct for CellaFlow — the engine uses `strategy: Recreate` (not `RollingUpdate`) to ensure clean lock handoff on upgrades. Expect a brief downtime (~10–30s) during chart upgrades.

### Storage Class Examples

| Platform | Storage Class |
|---|---|
| GKE | `standard-rwo` |
| EKS | `gp3` |
| AKS | `managed-csi` |
| Minikube (dev) | `standard` (default) |

---

## Ingress (gRPC over TLS)

The engine exposes a gRPC server on port `50051`. To route external traffic, enable the Ingress with an nginx-grpc backend:

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # Optional: TLS via cert-manager
  hosts:
    - host: cellaflow.yourdomain.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: cellaflow-tls
      hosts:
        - cellaflow.yourdomain.com
```

> gRPC over TLS requires HTTP/2. Ensure your Ingress controller supports gRPC passthrough.

---

## Resource Tuning

No resource limits are set on the engine by default. For production, configure them explicitly:

```yaml
resources:
  cellaflow:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  otelCollector:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
```

---

## Health & Readiness

The engine exposes a health endpoint on port `9090`:

```
GET http://<pod-ip>:9090/health/ready
```

Returns `200 OK` with body `OK` when the engine has completed boot recovery and is ready to serve gRPC traffic.

Kubernetes liveness and readiness probes are pre-configured in the chart and will automatically restart the pod if it becomes unhealthy.

---

## Upgrade Notes

### Breaking Change: Non-Root User (v0.1.0+)

Starting from chart version `0.1.0`, the engine runs as a non-root user (UID `10001`). An `initContainer` automatically fixes permissions on the PVC during each upgrade, so no manual action is required.

### Upgrade Command

```bash
helm upgrade cellaflow-engine ./charts/cellaflow-engine \
  -f my-values.yaml
```

> Upgrades use `strategy: Recreate`. The old pod will terminate before the new one starts. Plan for ~10–30 seconds of downtime during the upgrade.

---

## Uninstalling

```bash
helm uninstall cellaflow-engine
```

> The `PersistentVolumeClaim` is **not** deleted automatically on uninstall to prevent accidental data loss. To delete it manually:
>
> ```bash
> kubectl delete pvc cellaflow-engine-data
> ```

---

## Values Reference

| Key | Default | Description |
|---|---|---|
| `replicaCount` | `1` | Number of engine replicas. Must be `1` (RocksDB is single-writer). |
| `image.repository` | `ghcr.io/cellaflow/cellaflow` | Engine container image repository. |
| `image.tag` | `""` | Image tag. Defaults to the chart `appVersion`. |
| `image.pullPolicy` | `IfNotPresent` | Kubernetes image pull policy. |
| `imagePullSecrets` | `[]` | List of pull secret names for GHCR authentication. |
| `nameOverride` | `""` | Override the chart name. |
| `fullnameOverride` | `""` | Override the fully-qualified resource name. |
| `serviceAccount.create` | `true` | Create a Kubernetes ServiceAccount for the pod. |
| `podSecurityContext.fsGroup` | `10001` | Group ID for volume ownership. |
| `securityContext.runAsNonRoot` | `true` | Enforce non-root execution. |
| `securityContext.runAsUser` | `10001` | UID to run the engine as. |
| `service.type` | `ClusterIP` | Kubernetes Service type. |
| `service.port` | `50051` | gRPC service port. |
| `ingress.enabled` | `false` | Enable Ingress for external gRPC access. |
| `resources.cellaflow` | `{}` | Resource requests/limits for the engine container. |
| `resources.otelCollector.limits.cpu` | `200m` | CPU limit for OTel sidecar. |
| `resources.otelCollector.limits.memory` | `256Mi` | Memory limit for OTel sidecar. |
| `persistence.enabled` | `true` | Enable PersistentVolumeClaim for RocksDB state. |
| `persistence.storageClass` | `""` | Storage class. Empty uses cluster default. |
| `persistence.size` | `10Gi` | PVC size. |
| `telemetry.optOut` | `false` | Set `true` to disable fleet telemetry routing. |
| `telemetry.customerApm.endpoint` | `""` | OTLP/gRPC endpoint for your APM. Empty disables customer pipeline. |
| `telemetry.customerApm.apiKey` | `""` | API key inline. Prefer `apiKeySecretRef` in production. |
| `telemetry.customerApm.apiKeySecretRef` | `""` | Name of a Kubernetes Secret containing `apiKey`. |
| `otelCollector.image.repository` | `otel/opentelemetry-collector-contrib` | OTel Collector image. |
| `otelCollector.image.tag` | `0.100.0` | OTel Collector version. |

---

## Support

For issues, questions, or enterprise support: **support@cellaflow.com**
