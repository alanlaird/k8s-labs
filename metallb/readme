# create kind cluster
kind create cluster --name metallb-kind --config kind-config.yaml
kubectl cluster-info --context kind-metallb-kind

# install metallb
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=300s

docker network inspect kind --format='{{json .IPAM.Config}}'
# Output will be something like [{"Subnet":"172.18.0.0/16", ...}]

# apply config
kubectl apply -f metallb-config.yaml

# deploy app and expose it
kubectl create deployment echo --image inanimate/echo-server --replicas 3 --port 8080
kubectl expose deployment echo --type=LoadBalancer
kubectl get svc

kubectl create deployment nginx --image nginx --replicas 3 --port 80
kubectl expose deployment nginx --type=LoadBalancer
kubectl get svc

# expose the services
./metallb-dnat.sh

# cleanup
kind delete clusters metallb-kind

