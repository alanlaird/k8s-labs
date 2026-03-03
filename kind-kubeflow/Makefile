KUBERNETES_VERSION := 1.35.1
PIPELINE_VERSION   := 2.15.0

KIND_CLUSTER_NAME=kubeflow

all: help

.PHONY: launch-k8s
launch-k8s: ## Launch Kubernetes cluster with kind
	if [ ! "$(shell kind get clusters | grep $(KIND_CLUSTER_NAME))" ]; then \
		kind create cluster --name=$(KIND_CLUSTER_NAME) --config kind-config.yaml --image kindest/node:v$(KUBERNETES_VERSION) --wait 180s; \
		kubectl wait pod --all -n kube-system --for condition=Ready --timeout 180s; \
	fi

.PHONY: deploy-kubeflow
deploy-kubeflow: ## Deploy Kubeflow Pipelines onto the cluster
	kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$(PIPELINE_VERSION)"
	kubectl wait --for condition=established --timeout=60s crd/applications.app.k8s.io
	kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/platform-agnostic?ref=$(PIPELINE_VERSION)"
	kubectl wait pod --all -n kubeflow --for condition=Ready --timeout=600s
	@echo ""
	@echo "Kubeflow Pipelines $(PIPELINE_VERSION) ready. Run 'make expose-kubeflow' to access the UI."

.PHONY: expose-kubeflow
expose-kubeflow: ## Port-forward Kubeflow UI to http://localhost:8080
	@echo "Kubeflow UI available at http://localhost:8080 (Ctrl+C to stop)"
	kubectl port-forward -n kubeflow svc/ml-pipeline-ui 8080:80

.PHONY: status-k8s
status-k8s: ## Show kind cluster health: nodes and services
	@echo "=== Kind Clusters ==="
	@kind get clusters 2>/dev/null || echo "(none)"
	@echo ""
	@echo "=== Nodes ==="
	@kubectl get nodes -o wide 2>/dev/null || echo "(cluster unreachable)"
	@echo ""
	@echo "=== Services (all namespaces) ==="
	@kubectl get svc --all-namespaces 2>/dev/null || echo "(cluster unreachable)"

.PHONY: shutdown-k8s
shutdown-k8s: ## Shutdown Kubernetes cluster
	if [ "$(shell kind get clusters | grep $(KIND_CLUSTER_NAME))" ]; then \
		kind delete cluster --name=$(KIND_CLUSTER_NAME) || true; \
	fi

.PHONY: help
help: ## Display this help.
	@awk 'BEGIN {FS = ":.*##"; printf "\nUsage:\n  make \033[36m<target>\033[0m\n"} /^[a-zA-Z_0-9-]+:.*?##/ { printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2 } /^##@/ { printf "\n\033[1m%s\033[0m\n", substr($$0, 5) } ' $(MAKEFILE_LIST)
