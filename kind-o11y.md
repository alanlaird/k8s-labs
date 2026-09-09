# kind-o11y: Reusable Observability Stack for kind Clusters

A turnkey observability + GitOps stack that can be layered onto any kind cluster.
Useful as a foundation when experimenting with other stacks (Kubeflow, Submariner, etc.)
so you always have metrics, logs, traces, and profiling available.

**Repo:** `~/gitp/kind-o11y`
([github.com/alanlaird/kind-monitoring-prometheus-loki-tempo-phlare-victoriametrics-grafana-playground](https://github.com/alanlaird/kind-monitoring-prometheus-loki-tempo-phlare-victoriametrics-grafana-playground))

---

## What's Included

| Component | Namespace | Purpose |
|---|---|---|
| **ArgoCD** | `argocd` | GitOps CD — drives all other deployments |
| **kube-prometheus-stack** | `prometheus` | Prometheus, Grafana, Alertmanager, kube-state-metrics, node-exporter |
| **Loki + Promtail** | `loki` | Log aggregation; Promtail runs as DaemonSet on all nodes |
| **Tempo** | `tempo` | Distributed tracing (OTLP, Jaeger, Zipkin) |
| **Phlare** | `phlare` | Continuous profiling |
| **VictoriaMetrics** | `victoriametrics` | Parallel metrics backend (operator, vmagent, vmsingle) |
| **prometheus-adapter** | `prometheus-adapter` | Exposes Prometheus metrics as Kubernetes custom metrics API |
| **cert-manager** | `cert-manager` | TLS certificate management |
| **MetalLB** | `metallb-system` | LoadBalancer IPs for kind (optional, used on alsea) |
| **sandbox** | `sandbox` | Sample workloads: `todo`, `request`, `dummy-metrics` |

All components except ArgoCD itself are managed as ArgoCD Applications, synced in dependency order via `argocd.argoproj.io/sync-wave` annotations.

---

## Architecture

```
kind cluster
└── ArgoCD  (reads from github.com/alanlaird/kind-o11y repo)
    ├── namespaces          (wave 0 — create namespaces first)
    ├── cert-manager        (wave 1)
    ├── loki                (wave 1)
    ├── phlare              (wave 1)
    ├── prometheus-adapter  (wave 1)
    ├── tempo               (wave 1)
    ├── victoriametrics     (wave 1)
    ├── prometheus          (wave 2 — depends on above data sources)
    ├── monitoring          (wave 3 — ConfigMaps, PodMonitors, custom dashboards)
    └── sandbox             (wave 3 — sample workloads)
```

Grafana is wired to all four data sources out of the box:
- **Prometheus** (default)
- **Loki** — `http://loki.loki.svc:3100`
- **Tempo** — `http://tempo.tempo.svc:3100`
- **Phlare** — `http://phlare.phlare.svc:4100`

---

## Grafana Dashboards

Dashboards come from two sources:

### 1. Bundled with kube-prometheus-stack (Helm)
The standard set: Kubernetes compute/networking/storage, Node Exporter, Alertmanager, CoreDNS, etcd, Scheduler, Kubelet, API Server, etc.

### 2. Custom dashboards via ConfigMap sidecar
Any ConfigMap in any namespace with label `grafana_dashboard: "1"` is automatically loaded by Grafana's sidecar.

Custom dashboards live in `manifests/monitoring/prometheus/dashboards/` and are wired in via `manifests/monitoring/prometheus/kustomization.yaml`:

```yaml
# manifests/monitoring/prometheus/kustomization.yaml
generatorOptions:
  labels:
    grafana_dashboard: "1"
configMapGenerator:
  - name: grafana-dashboards
    files:
      - ./dashboards/argocd-performance.json
```

**To add a dashboard:** export JSON from Grafana, drop it in `manifests/monitoring/prometheus/dashboards/`, add the filename to the `configMapGenerator` list, and push. ArgoCD will sync it automatically.

---

## Deploying to a New kind Cluster

### Prerequisites (on the target host)
- Docker, kind, kubectl, helm, kustomize, yq, jq, argocd CLI

### Steps

```bash
# 1. Create the cluster (uses kind-config.yaml: 1 control-plane + 3 workers)
#    kind-config.yaml also enables etcd metrics and API server tracing for Tempo
make launch-k8s

# 2. Install ArgoCD and register all Applications
make deploy-argocd

# 3. Sync all Applications in sync-wave order
make sync-applications
```

`make sync-applications` iterates apps sorted by `sync-wave`, running
`argocd app sync && argocd app wait` for each, ensuring dependencies are
ready before dependent apps start.

### Credentials

```bash
# ArgoCD admin password
make argocd-password

# Grafana admin password (default: prom-operator)
make grafana-password
```

### Accessing UIs (port-forward)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# ArgoCD: https://localhost:8080

kubectl port-forward svc/prometheus-grafana -n prometheus 3000:80 &
# Grafana: http://localhost:3000
```

If MetalLB is deployed (e.g. on alsea), Grafana gets a real LoadBalancer IP and
no port-forward is needed.

### Teardown

```bash
make shutdown-k8s
```

---

## Lessons Learned: Deploying to a Fresh Cluster

Observed while deploying to siletz (fresh kind cluster, 2026-03-08).

### Timing

The full deploy takes roughly **25–35 minutes** end to end:

| Phase | Time |
|---|---|
| `kind create cluster` (4 nodes) | ~3 min |
| `helm install argocd` + wait for deployments | ~4 min |
| `kustomize build … | kubectl apply` (register apps) | <1 min |
| Sync wave 0–1 (namespaces through victoriametrics) | ~8 min |
| Sync wave 2 (prometheus — largest chart) | ~8 min |
| Sync wave 3 (monitoring, sandbox) | ~3 min |
| ArgoCD `helm upgrade` with metrics values | ~3 min |

### `argocd app wait` quirks

- **Don't use plain `argocd app wait` on `prometheus-adapter`** — it hangs indefinitely on `APIService` resources which never reach a terminal ArgoCD resource state. Use `--health` flag instead:
  ```bash
  argocd app wait prometheus-adapter --grpc-web --health --timeout 300
  ```
- The `--health` flag is safe to use for all apps; it only waits for pod-level health rather than every resource in the sync tree.

### `monitoring` app timeout

The `monitoring` app (VMAgent, VMPodScrape resources) will time out its `app wait` on the first deploy. The VMPodScrape resources need Prometheus to be fully scraped before ArgoCD considers them healthy. This is cosmetic — the app is actually Synced and the pods are Running. Just re-check with `kubectl get applications -n argocd` after a minute or two; it will show Healthy.

### `promlens` Init:Error

`promlens` consistently fails with `Init:Error` on both alsea and siletz. It's a known upstream issue with the image and doesn't affect anything else. The `monitoring` ArgoCD app shows as Degraded because of it. Safe to ignore.

### Port-forward for ArgoCD CLI

The ArgoCD CLI needs a running port-forward to talk to the server. The port-forward process dies if the SSH session ends:

```bash
kubectl port-forward svc/argocd-server -n argocd 8443:443 &>/tmp/argocd-pf.log &
sleep 4
argocd login localhost:8443 --grpc-web --insecure --username admin \
  --password $(kubectl get secret -n argocd argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d)
```

Always pass `--grpc-web` to all `argocd` commands when going through a port-forward — without it, commands may hang or fail with gRPC framing errors.

### MetalLB and kind network

MetalLB's IP pool (`172.18.255.200-250`) is hard-coded for the kind default bridge network `172.18.0.0/16`. This works on any libvirt VM running a standard kind cluster. If the kind network range differs, update `manifests/metallb/` in the repo before syncing.

### kubeconfig

kind writes the kubeconfig to `~/.kube/config` by default, but on these VMs there's no pre-existing `~/.kube/`. The safest pattern is to always export explicitly:

```bash
kind get kubeconfig --name <cluster-name> > /tmp/kc.yaml
export KUBECONFIG=/tmp/kc.yaml
```

---

## Notes for Layering with Other Stacks

When deploying this alongside another stack (e.g. Kubeflow on noti):

- **Namespaces won't conflict** — all o11y components use dedicated namespaces.
- **Scraping the new stack:** Add a `PodMonitor` or `ServiceMonitor` in `manifests/monitoring/` pointing at the new namespace. Prometheus will pick it up via its `podMonitorSelector: {}` / `serviceMonitorSelector: {}` defaults.
- **Custom dashboards:** Export dashboards from the new stack's own Grafana (if it has one) as JSON and drop them in `manifests/monitoring/prometheus/dashboards/`.
- **Tracing:** Instrument new workloads to send OTLP traces to `tempo.tempo.svc:4317` (gRPC) or `:4318` (HTTP).
- **Profiling:** Add pod annotations `phlare.grafana.com/scrape: "true"` and `phlare.grafana.com/port: "<port>"` to any workload to enable continuous profiling.
- **MetalLB:** Only needed if you want real LoadBalancer IPs inside kind. Can be skipped on clusters that only use port-forwarding.

---

## Current Deployments

| Host | Cluster Name | Status |
|---|---|---|
| alsea (`172.16.11.87`) | `hands-on` | Active — full stack running |
| siletz (`172.16.11.88`) | `kind` (blank) | Reset 2026-03-08 — ready for next lab |
