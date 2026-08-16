# motadata-k8s

One-shot install of the Motadata Kubernetes APM stack. Replaces five separate commands (cert-manager, operator, Instrumentation CR, APM agent) with a single `helm install`.

## Install

```bash
helm repo add motadata-charts https://motadata2025.github.io/motadata-helm-charts/
helm repo update

helm upgrade --install motadata motadata-charts/motadata-k8s \
  -n motadata --create-namespace --wait \
  -f motadata-values.yaml
```

`motadata-values.yaml`:

```yaml
global:
  otlpEndpoint: http://172.16.9.243:4318
  collectorIp: 172.16.9.243
  clusterName: k8s

motadata-apm-agent:
  env:
    clusterName: k8s
    motadataServerUrl: https://172.16.9.243:9477/api/v2/rum
```

### `--wait` is mandatory

Not optional, not a nicety. The Instrumentation CR is applied as a `post-install,post-upgrade` hook because the operator's `minstrumentation.kb.io` and `vinstrumentationcreateupdate.kb.io` webhooks are `failurePolicy: Fail` — the CR is rejected outright if it lands before the operator is serving.

Helm's lifecycle is: render → pre-install hooks → apply resources → **wait for readiness (only with `--wait`)** → post-install hooks. The operator Deployment has a `readinessProbe` on `/readyz`, so `--wait` is a real gate. Without it the hook fires immediately after apply and the install fails.

### Use a values file, not `--set`

Helm does not carry `--set` values forward across upgrades. A forgotten `--set global.collectorIp=...` reverts to the chart default **silently** and every instrumented pod starts reporting a placeholder. A forgotten `certManager.enabled=false` at least fails loudly. Keep one version-controlled values file and audit live releases with `helm get values motadata -n motadata`.

## Values

| Key | Description |
|---|---|
| `global.otlpEndpoint` | OTLP HTTP URL the SDKs export to, e.g. `http://172.16.9.243:4318` |
| `global.collectorIp` | Bare collector IP, reported as a resource attribute |
| `global.clusterName` | Logical cluster name, reported as `k8s.cluster.name` |
| `instrumentation.enabled` | Create the Instrumentation CR (default `true`) |
| `instrumentation.namespace` / `.name` | CR location, default `motadata-cr/central-auto` |
| `instrumentation.images.*` | Auto-instrumentation images per language |
| `opentelemetry-operator.enabled` | Install the operator (set `false` if the cluster already runs one) |
| `motadata-apm-agent.enabled` | Install the discovery agent |
| `motadata-apm-agent.env.motadataServerUrl` | Motadata ingest URL — **different host/port/path** from `global.otlpEndpoint` |
| `motadata-apm-agent.env.clusterName` | Must equal `global.clusterName`; enforced at render time |

All four required values are validated by `templates/validate.yaml`, which fails the render with a readable message rather than shipping a placeholder.

`clusterName` has to be set in two places. That is not an oversight: Helm cannot template `values.yaml`, and `motadata-apm-agent` reads its own scope rather than `.Values.global.*`. The validation template turns the resulting drift risk into a hard error:

```
Error: motadata-apm-agent.env.clusterName ("TYPO") must equal global.clusterName ("k8s").
```

The clean fix is to teach the subcharts to read globals with a local fallback —
`{{ .Values.global.clusterName | default .Values.clusterName }}` — which is backward compatible. Then this chart can depend on all three subcharts and the duplication disappears.

## No cert-manager

Chart defaults set `admissionWebhooks.certManager.enabled: false` with `autoGenerateCert.enabled: true`, so Helm mints the webhook certificate itself and cert-manager is not required.

`autoGenerateCert.recreate` is set to `false` so upgrades reuse the existing cert instead of minting a new one on every `helm upgrade`. This is implemented with Helm's `lookup`, which returns nothing when rendering offline — under `helm template`, ArgoCD or Flux it silently regenerates the cert on every sync. **If you render offline, set `certManager.enabled: true` and install cert-manager instead.**

Do not set `certManager.enabled` and `autoGenerateCert.enabled` both to `false` unless you are supplying your own `certFile`/`keyFile`/`caFile`. That combination renders an empty TLS secret and empty `caBundle` with no error — `mpod.kb.io` is `failurePolicy: Ignore`, so pods keep starting and injection is silently skipped.

## Structure

```
motadata-k8s/
├── Chart.yaml                    # file:// deps on the sibling charts
├── values.yaml
└── templates/
    ├── validate.yaml             # renders nothing; fails fast on bad input
    ├── namespace.yaml            # motadata-cr namespace
    └── instrumentation.yaml      # Instrumentation CR, as a post-install hook
```

### Why `file://` dependencies

The operator and agent are pulled from the sibling directories in this repo rather than the published index:

- **No version skew.** The umbrella ships the subchart revisions from the same commit — the ones it was tested against. The published index currently serves `opentelemetry-operator` 0.99.2 while this repo is on 0.102.0.
- **No CI configuration.** Neither `.github/workflows/lint.yaml` (`ct lint --chart-repos`) nor `release.yaml` (*Add dependent repositories*) lists the `motadata-charts` repo. An HTTPS dependency would fail resolution in CI until someone adds it there; `file://` needs nothing.

The trade-off is that bumping `opentelemetry-operator` requires bumping the pinned `version:` in this chart's `Chart.yaml` too.

### Why the Instrumentation CR is not a subchart

`motadata-cr-instrumentation` stays published and standalone-usable, but this chart owns a copy of the CR template. Two independent blockers:

1. **Helm cannot template `values.yaml`.** A subchart sees only `.Values.global.*` plus its own scope, and that chart reads `.Values.clusterName` / `.Values.collectorIp` at *its* scope — so a single global input can never reach it.
2. **A parent cannot annotate a subchart's resources**, and the CR needs the `post-install` hook described above.

Cost: the CR spec is now duplicated. If `motadata-cr-instrumentation` changes, update this chart too.

## Developing

```bash
cd charts/motadata-k8s
helm dependency update      # writes Chart.lock, vendors subcharts into charts/
helm lint .
helm template motadata . -n motadata \
  --set global.otlpEndpoint=http://172.16.9.243:4318 \
  --set global.collectorIp=172.16.9.243 \
  --set global.clusterName=k8s \
  --set motadata-apm-agent.env.clusterName=k8s \
  --set motadata-apm-agent.env.motadataServerUrl=https://172.16.9.243:9477/api/v2/rum
```

Render assertions worth keeping green:

| Check | Expected |
|---|---|
| `kind: Certificate` present | no — cert-manager not used |
| `kubernetes.io/tls` Secret present, `caBundle` non-empty | yes |
| Operator resources named `motadata-operator` | yes |
| `dummy.value` anywhere in output | no |
| Instrumentation CR carries `helm.sh/hook: post-install,post-upgrade` | yes |
| `--set opentelemetry-operator.enabled=false` drops operator, CRDs, webhooks | yes |

`helm template` does **not** execute hooks, so it proves the CR renders correctly but says nothing about ordering. Ordering is only exercised by a real `helm install --wait` against a cluster.

Commit `Chart.lock`; `charts/*.tgz` is already covered by the repo `.gitignore`.

## Releasing

Publishing is automated. `.github/workflows/release.yaml` runs `helm/chart-releaser-action` over `charts/` on every push to `main`, so no manual `helm package` / `helm repo index` is needed — running those by hand would fight chart-releaser.

Bump `version:` in `Chart.yaml` for every release; chart-releaser skips a chart whose version already has a release tag.

## Known limitations

- **`helm uninstall` does not remove the Instrumentation CR.** Hook-annotated resources are not tracked in the release, and `helm.sh/hook-delete-policy` does not change that — it governs the hook's lifecycle around execution, not uninstall. In practice the `motadata-cr` Namespace *is* tracked, so deleting it cascades and takes the CR with it. If `instrumentation.namespace` points at a pre-existing namespace, clean up manually with `kubectl delete instrumentation central-auto -n motadata-cr`.
- **`inject-go` is configured but incomplete.** The CR sets only `go.image`; eBPF Go auto-instrumentation additionally requires `OTEL_GO_AUTO_TARGET_EXE` and an elevated sidecar. Annotating a Go workload today is a silent no-op.
- **Auto-instrumentation images are not mirrored.** `manager.image` and `kubeRBACProxy.image` were repointed to `ghcr.io/motadata2025`, but the language SDK images here still pull from `ghcr.io/open-telemetry`. That gap matters on air-gapped or egress-restricted clusters.
