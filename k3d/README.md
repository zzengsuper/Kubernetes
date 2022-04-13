# K3d cluster

1. Lightway and quick to spin up the cluster with container
2. Loadbalancer svc created 
3. Traefik proxy node created 
4. Support multi node cluster 
5. Support multiple cluster 

## Setup k3d on Linux

1. Go to the github release page download the xxx packages(v5.3.0)
2. chmod +x xxx
3. mv xxx /usr/local/bin/k3d

## Create k3d cluster

```shell
k3d cluster create cluster1 --agents 2
kubectl cluster-info
kubectl get nodes
kubectl get all -n kube-system
k3d node list
```
