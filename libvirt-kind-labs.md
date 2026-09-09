# Kind Lab Nodes

Inventory of what's running on the four homelab nodes provisioned with the `kind` Ansible role.
Surveyed 2026-03-08.

---

## klamath (`172.16.11.31`)

Two kind clusters running a **Submariner** multi-cluster networking lab.

### cluster1

| Namespace | Workload | Status |
|---|---|---|
| kube-system | coredns (×2), etcd, kindnet (×2), kube-apiserver, kube-controller-manager, kube-scheduler | Running |
| kube-system | kube-proxy (×2) | ⚠ CreateContainerError |
| local-path-storage | local-path-provisioner | ⚠ CreateContainerError |
| subm-kindnet-workaround | dummypod (×2) | Running |
| submariner-operator | submariner-gateway | ⚠ CreateContainerError |
| submariner-operator | submariner-lighthouse-agent, lighthouse-coredns (×2), metrics-proxy, route-agent (×2), submariner-operator | Running |

### cluster2

| Namespace | Workload | Status |
|---|---|---|
| kube-system | coredns (×2), etcd, kindnet (×2), kube-apiserver, kube-controller-manager, kube-scheduler | Running |
| kube-system | kube-proxy (×2) | ⚠ CreateContainerError |
| local-path-storage | local-path-provisioner | ⚠ CreateContainerError |
| subm-kindnet-workaround | dummypod (×2) | Running |
| submariner-operator | submariner-operator | ⚠ CreateContainerError |

Also runs a local container registry at `localhost:5000` (`kind-registry`).

---

## alsea (`172.16.11.87`)

One kind cluster: **`hands-on`** (1 control-plane + 3 workers, ~6 days old).

Full **observability + GitOps** stack.

| Namespace | Workload | Notes |
|---|---|---|
| argocd | application-controller, applicationset-controller, dex-server, notifications-controller, redis, repo-server, server | GitOps CD |
| cert-manager | cert-manager, cainjector, webhook | TLS certificate management |
| metallb-system | metallb-controller, metallb-speaker (×4) | Bare-metal load balancer |
| prometheus | kube-prometheus-operator, prometheus, alertmanager, grafana, kube-state-metrics, node-exporter (×4) | Metrics & dashboards |
| prometheus | promlens | ⚠ CrashLoopBackOff |
| prometheus-adapter | prometheus-adapter | Custom metrics API |
| loki | loki, promtail (×4) | Log aggregation |
| tempo | tempo | Distributed tracing |
| phlare | phlare | Continuous profiling |
| victoriametrics | victoria-metrics-operator, vmagent, vmsingle | Alternative metrics backend |
| sandbox | todo, request, dummy-metrics | Sample workloads |

---

## siletz (`172.16.11.88`)

One kind cluster: **`kind`** (1 control-plane + 3 workers, k8s v1.35.1).

Reset 2026-03-08. Redeployed same session with hello-world app + full kind-o11y observability stack.

| Namespace | Workload | Notes |
|---|---|---|
| hello-world | hello-world (×2 replicas) | Sample app — `gcr.io/google-samples/hello-app:1.0`, ClusterIP |
| argocd | application-controller, applicationset-controller, dex-server, notifications-controller, redis, repo-server, server | GitOps CD |
| cert-manager | cert-manager, cainjector, webhook | TLS certificate management |
| metallb-system | metallb-controller, metallb-speaker (×4) | Bare-metal load balancer; pool `172.18.255.200-250` |
| prometheus | kube-prometheus-operator, prometheus, alertmanager, grafana, kube-state-metrics, node-exporter (×4) | Metrics & dashboards — Grafana at `172.18.255.200` |
| prometheus | promlens | ⚠ Init:Error (known upstream bug, non-blocking) |
| prometheus-adapter | prometheus-adapter | Custom metrics API |
| loki | loki, promtail (×4) | Log aggregation |
| tempo | tempo | Distributed tracing |
| phlare | phlare | Continuous profiling |
| victoriametrics | victoria-metrics-operator, vmagent, vmsingle | Alternative metrics backend |
| sandbox | todo, request, dummy-metrics | Sample workloads from kind-o11y |

See `kind-o11y.md` for deployment pattern and lessons learned.

---

## noti (`172.16.11.89`)

One kind cluster: **`kubeflow`** (1 control-plane + 3 workers, ~5 days old).

Running **Kubeflow Pipelines**.

| Namespace | Workload | Notes |
|---|---|---|
| kubeflow | ml-pipeline, ml-pipeline-persistenceagent, ml-pipeline-scheduledworkflow, ml-pipeline-ui, ml-pipeline-viewer-crd, ml-pipeline-visualizationserver | Kubeflow Pipelines core |
| kubeflow | metadata-envoy, metadata-grpc, metadata-writer | ML metadata services |
| kubeflow | cache-deployer, cache-server | Pipeline caching |
| kubeflow | workflow-controller | Argo Workflows |
| kubeflow | mysql | Pipeline metadata DB |
| kubeflow | seaweedfs | Object storage (artifact store) |
| local-path-storage | local-path-provisioner | PV provisioning |
