# kubeflow on kind

# reading
https://www.kubeflow.org/docs/components/pipelines/legacy-v1/installation/localcluster-deployment/

# do it

export PIPELINE_VERSION=2.15.0
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"
kubectl wait --for condition=established --timeout=60s crd/applications.app.k8s.io
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/platform-agnostic?ref=$PIPELINE_VERSION"


kubectl port-forward -n kubeflow svc/ml-pipeline-ui 8080:80


