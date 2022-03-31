[toc]



# Kind

This document shows you how to create cluster with Kind.
While kind uses docker or podman on your host, it uses containers/CRI inside the nodes and does not use dockershim.

## Create kind cluster

```shell
Kind create cluster --name kind2 --image xxx --config xxx
```

- default name is kind
- config file will be saved at ~/.kube/config
- --image can specify the version of image
- --config can create multi-node cluster

## List kind clusters

```shell
kind get clusters
```

## Delete a kind cluster

```shell
kind delete cluster --name kind2
```

- It will delete default kind cluster if you don't specify the cluster name

## Interact with a specific cluster

```shell
kubectl cluster-info --context kind-kind2
```

## Kind node network (docker network)

```shell
docker network ls 
```

- kind bridge network will be created
- you can also check it from the inspect command

## Storage class

- Kind cluster will install storage class by default (standard)
- accessModes: ReadWriteOnce
- Volumebindmode: is WaitForFirstConsumer

## Create dynamic volume 

- Create PVC
- Create pod and mount

## Use LoadBalancer (Metallb)

```shell
kubectl -n metallb-system get all
```

```shell
kubectl expose deploy nginx --port 80 type LoadBalancer
kubectl get all
```

Please note: On macOS and Windows, docker does not expose the docker network to the host. Because of this limitation, containers (including kind nodes) are only reachable from the host via port-forwards

## Kubectl command

```shell
kubectl cluster-info
kubectl get nodes -A 
kubectl get nodes -o wide
kubectl config get-contexts
kubectl config use-context kind2
kubectl get sc
kubectl get pvc,pv
kubectl get pods
```

## Kind command

```shell
kind create cluster --help
kind create cluster
kind create cluster --name xx --image xx --config xx
kind get clusters
kind delete cluster --name
```

## Docker command

```shell
docker ps
docker exec -it kind-control-plane sh
docker network ls
docker inspect kind-cluster
grep -ef
```

## 
