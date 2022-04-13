# MatalLB

## matalLB Installation By Manifest

https://metallb.universe.tf/installation/
Note: before apply any file or url, please check/verify it in advance 

## matalLB layer 2 Configuration

https://metallb.universe.tf/configuration/

## To test LoadBalancer

```shell
kubectl create deploy nginx --image nginx
kubectl expose deploy nginx --port 80 --type LoadBalancer
kubectl get svc 
w3m 172.16.16.240

kubectl create deploy nginx2 --image nginx
kubectl expose deploy nginx2 --port 80 --type LoadBalancer
kubectl get all
w3m 172.16.16.241
```

## Nginx Ingress

### Using helm to add ingress repo

```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm search repo ingress

# Check for values before install
helm show values ingress-nginx/ingress-nginx > ingress-nginx.yaml
# Make value changes below 3 section
vi ingress-nginx.yaml 

hostNetwork: true
hostPort: enabled: true
kind -> DaemonSet

# Create ingress-nginx namespace
kubectl create ns ingress-nginx
kubectl get ns

# Install ingress-nginx with value changed
helm install myingress ingress-nginx/ingress-nginx -n ingress-nginx --values ingress-nignx-value.yaml   

# Will show myingress here
helm list -n ingress-nginx 
```

### Create Ingress policy

```shell
kubectl create -f nginx-deploy-main.yaml
kubectl expose deploy nginx-deploy-main --port 80
kubectl create -f ingress-resource-1.yaml
```

### Nginx Ingress command

```shell
kubectl -n ingress-nginx get all 
kubectl get ing
kubectl describe ingress ingress-resource-1
```

### LoadBalancer IP is reachable

```shell
curl --header "host: nginx.example.com" 172.16.16.240
<h1>Welcome to nginx!</h1>
```









