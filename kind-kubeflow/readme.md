# Kubeflow Pipelines on kind

Deploys [Kubeflow Pipelines](https://www.kubeflow.org/docs/components/pipelines/) onto a local [kind](https://kind.sigs.k8s.io/) cluster (1 control-plane + 3 workers).

## Prerequisites

- `kind` — local Kubernetes cluster tool
- `kubectl` — Kubernetes CLI
- `kustomize` — (or a recent `kubectl` with built-in kustomize support)

## Quick start

```bash
# 1. Create the kind cluster
make launch-k8s

# 2. Deploy Kubeflow Pipelines
make deploy-kubeflow

# 3. Open the UI
make expose-kubeflow
# Browse to http://localhost:8080
```

## Targets

| Target             | Description                                              |
|--------------------|----------------------------------------------------------|
| `launch-k8s`       | Create the kind cluster (skips if already running)       |
| `deploy-kubeflow`  | Install Kubeflow Pipelines and wait for pods to be ready |
| `expose-kubeflow`  | Port-forward the UI to http://localhost:8080             |
| `shutdown-k8s`     | Delete the kind cluster                                  |

## Versions

| Component            | Version |
|----------------------|---------|
| Kubernetes           | 1.35.1  |
| Kubeflow Pipelines   | 2.15.0  |

Override at the command line:

```bash
make deploy-kubeflow PIPELINE_VERSION=2.16.0
```

## External access via MetalLB DNAT

If you have MetalLB installed and want to reach the UI from other machines on the network without a port-forward, use the included `metallb-dnat.sh` script (requires root):

```bash
# Apply DNAT rules for all LoadBalancer services
sudo ./metallb-dnat.sh apply

# Check status
sudo ./metallb-dnat.sh status

# Remove rules
sudo ./metallb-dnat.sh remove

# Install as a systemd service (survives reboots)
sudo ./metallb-dnat.sh install
```

## Cluster config

The kind cluster (`kind-config.yaml`) enables:
- API server distributed tracing (OpenTelemetry)
- Scheduler and controller-manager metrics on all interfaces
- 3 worker nodes for workload distribution

## References

- https://www.kubeflow.org/docs/components/pipelines/legacy-v1/installation/localcluster-deployment/
- https://github.com/kubeflow/pipelines
