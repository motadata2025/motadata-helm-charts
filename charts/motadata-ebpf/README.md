# motadata-ebpf

One-shot install of Motadata eBPF APM: [OpenTelemetry eBPF Instrumentation (OBI)](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation) and the Motadata APM agent, in a single `helm install`.

OBI runs as a DaemonSet and instruments applications at the kernel level. It does not need the OpenTelemetry operator, an Instrumentation CR or any language auto-instrumentation images, and it does not modify or restart application pods. Use `motadata-k8s` instead for SDK-based auto-instrumentation. Do not enable both on the same namespace, or every request is traced twice.

## Install

```bash
helm repo add motadata-charts https://motadata2025.github.io/motadata-helm-charts/
helm repo update

helm upgrade --install motadata-ebpf motadata-charts/motadata-ebpf \
  -n obi --create-namespace \
  -f motadata-ebpf-values.yaml
```

`motadata-ebpf-values.yaml`:

```yaml
global:
  otlpEndpoint: http://172.16.15.146:4318
  clusterName: k8s

motadata-apm-agent:
  env:
    clusterName: k8s
    motadataServerUrl: http://172.16.15.52:4318/kubernetes/cluster

# Optional: only instrument these namespaces (default: every process on every node)
opentelemetry-ebpf-instrumentation:
  config:
    data:
      discovery:
        instrument:
          - k8s_namespace: demo-apps
          - k8s_namespace: product-api
```

OBI is installed into the release namespace (`-n obi`). The APM agent creates and uses its own namespace, `motadata-apm-agent-np`.

## Values

| Key | Description |
|---|---|
| `global.otlpEndpoint` | OTLP HTTP URL OBI exports traces and metrics to, e.g. `http://172.16.15.146:4318` |
| `global.clusterName` | Logical cluster name, reported as `k8s.cluster.name` |
| `opentelemetry-ebpf-instrumentation.enabled` | Install OBI (default `true`) |
| `opentelemetry-ebpf-instrumentation.image.*` | OBI image, default `ghcr.io/motadata2025/opentelemetry-ebpf-instrumentation:v0.13.0` |
| `opentelemetry-ebpf-instrumentation.config.data.discovery.instrument` | Namespaces or processes to instrument |
| `opentelemetry-ebpf-instrumentation.k8sCache.replicas` | Kubernetes metadata cache; `0` (default) disables it. Only worth enabling on large clusters |
| `opentelemetry-ebpf-instrumentation.k8sCache.image.*` | Cache image, default `ghcr.io/motadata2025/opentelemetry-ebpf-k8s-cache:v0.13.0` |
| `motadata-apm-agent.enabled` | Install the APM agent (default `true`) |
| `motadata-apm-agent.image.*` | Agent image, default `motadata2026/motadata-apm-agent:8.2.4` (Docker Hub) |
| `motadata-apm-agent.env.motadataServerUrl` | Motadata server ingest URL |
| `motadata-apm-agent.env.clusterName` | Must equal `global.clusterName`; enforced at render time |

Every other value of the two subcharts can be set under its key, as with `--set` on the standalone charts.

`templates/validate.yaml` fails the render with a readable message when a required value is missing or the two cluster names differ:

```
Error: motadata-apm-agent.env.clusterName ("TYPO") must equal global.clusterName ("k8s").
```

`clusterName` has to be set twice because only OBI templates its config (`config.data` goes through `tpl`, so it reads `global.*` directly). The agent chart uses its values as-is.

## Developing

```bash
cd charts/motadata-ebpf
helm dependency update      # writes Chart.lock, vendors subcharts into charts/
helm lint .
helm template motadata-ebpf . -n obi \
  --set global.otlpEndpoint=http://172.16.15.146:4318 \
  --set global.clusterName=k8s \
  --set motadata-apm-agent.env.clusterName=k8s \
  --set motadata-apm-agent.env.motadataServerUrl=http://172.16.15.52:4318/kubernetes/cluster
```

Dependencies are `file://` against the sibling charts, as in `motadata-k8s`, so bumping `opentelemetry-ebpf-instrumentation` or `motadata-apm-agent` also requires bumping the pinned `version:` in this chart's `Chart.yaml`.
