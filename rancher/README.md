# Set up a HA RKE Cluster

Follow the guide to setup a HA Kubernetes cluster on Ubuntu 20.04 LTS machines using Rancher RKE which three nodes all of which play control plane, etcd and worker role.


## Vagrant Environment

| Role                      | FQDN              | IP            | OS           | RAM  | CPU  |
| ------------------------- | ----------------- | ------------- | ------------ | ---- | ---- |
| Control plane,etcd,worker | node1.example.com | 172.16.16.121 | Ubuntu 20.04 | 2G   | 2    |
| Control plane,etcd,worker | node2.example.com | 172.16.16.122 | Ubuntu 20.04 | 2G   | 2    |
| Control plane,etcd,worker | node3.example.com | 172.16.16.123 | Ubuntu 20.04 | 2G   | 2    |

## Bring up all the virtual machines

```shell
vagrant up
```

## Set up password SSH Logins on all nodes

We will be using SSH Keys to login to root account on all the nodes and save the keypair at /home/fiona/.ssh/rke_rsa.pub

### Create an ssh keypair on the host machine

```shell
ssh-keygen -t rsa -b 2048 -C "rke@ubuntu"
```

### Copy SSH keys to all the nodes

```shell
ssh-copy-id -i /home/fiona/.ssh/rke_rsa.pub root@172.16.16.121
ssh-copy-id -i /home/fiona/.ssh/rke_rsa.pub root@172.16.16.122
ssh-copy-id -i /home/fiona/.ssh/rke_rsa.pub root@172.16.16.123
```

## Prepare the node1, node2, node3

### Disable Firewall

```shell
ufw disable
```

### Disable swap

```shell
swapoff -a; sed -i '/swap/d' /etc/fstab
```

### Update sysctl settings for Kubernetes networking

```shell
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### Install docker engine

```shell
{
  apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  apt update && apt install -y docker-ce=5:19.03.10~3-0~ubuntu-focal containerd.io
}
```

## Bring up RKE Cluster

### Create cluster configuration

```shell
rke config --name rkecluster.yml 
```

Once gone through this interactive cluster configuration, you will end up with configuration file (rkecluster.yml) in the current directory.

### Provision the cluster

```shell
rke up --config rkecluster.yml
```

Once this command completed provisioning the cluster, you will have cluster state file (rkecluster.rkestate)  and kube config file (kube_config_rkecluster.yml) in the current directory.

## Copy kube config to your local machine (Optional)

```shell
mkdir ~/.kube
cp kube_config_rkecluster.yml ~/.kube/config
```

## Verify the RKE Cluster (Optional)

```shell
kubectl cluster-info
kubectl get nodes
kubectl get cs
```

That's it!
