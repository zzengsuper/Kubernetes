# Set up a Highly Available Kubernetes Cluster using kubeadm

This document guides you in setting up a cluster with two master nodes, two worker nodes and two load balancer nodes using HAProxy.

## VMvare Environment

| Role                 | Hostname    | IP            | OS          | RAM  | CPU  |
| -------------------- | ----------- | ------------- | ----------- | ---- | ---- |
| Keepalived & HAproxy | k8s-lb1     | 172.16.16.141 | Ubuntu20.04 | 1G   | 1    |
| Keepalived & HAproxy | k8s-lb2     | 172.16.16.142 | Ubuntu20.04 | 1G   | 1    |
| Virtual IP address   |             | 172.16.16.140 |             |      |      |
| Master               | k8s-master1 | 172.16.16.151 | Ubuntu20.04 | 2G   | 2    |
| Master               | k8s-master2 | 172.16.16.152 | Ubuntu20.04 | 2G   | 2    |
| Worker               | k8s-worker1 | 172.16.16.161 | Ubuntu20.04 | 1G   | 1    |
| Worker               | k8s-worker2 | 172.16.16.162 | Ubuntu20.04 | 1G   | 1    |

## Install Kubernetes Server

Bring up the servers, once ready update them.

```shell
sudo apt update
sudo apt -y full-upgrade
[ -f /var/run/reboot-required ] && sudo reboot -f
```

## Install kubelet, kubeadm and kubectl

Once the servers are rebooted, add the Kubernetes repository for Ubuntu 20.04 to all the servers.

```shell
sudo apt -y install curl apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Then install required packages

```shell
sudo apt update
sudo apt -y install vim git curl wget kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Confirm installation by checking the version of kubectl

```shell
kubectl version --client && kubeadm version
```

## Disable Swap

Turn off swap

```shell
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```

Enable kernel modules and configure sysctl

```shell
# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Add some settings to sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system
```

## Install Container runtime

Installing Containers:

```shell
# Configure persistent loading of modules
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

# Load at runtime
sudo modprobe overlay
sudo modprobe br_netfilter

# Ensure sysctl params are set
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload configs
sudo sysctl --system

# Install required packages
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Add Docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install containerd
sudo apt update
sudo apt install -y containerd.io

# Configure containerd and start service
sudo su -
mkdir -p /etc/containerd
containerd config default>/etc/containerd/config.toml

# restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status  containerd
```

## Initialize master node

Login to the server to be used as master and make sure that br_netfilter module is loaded:

```shell
lsmod | grep br_netfilter
```

Enable kubelet service

```shell
sudo systemctl enable kubelet
```

Initialize the machine that will run the control plane components which includes etcd and the API Server.

```shell
sudo kubeadm config images pull
```

```shell
# Containerd
sudo kubeadm config images pull --cri-socket /run/containerd/containerd.sock
```

Create cluster (on k8s-master1)

```shell
sudo kubeadm init \
>   --pod-network-cidr=192.168.0.0/16 \
>   --upload-certs \
>   --control-plane-endpoint=172.16.16.151
```

Configure kubectl using commands in the output:

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check the cluster status:

```shell
kubectl cluster-info
```

Additional Master node (k8s-master2) can be added using the command in installation output:

```shell
kubeadm join 172.16.16.151:6443 --token xumxit.a2fa72cdsbxwvrof \
	--discovery-token-ca-cert-hash sha256:59c778c94386ea046b923cc46ba57f52e942026b7e45db7cf138a70fd8e4ccc3 \
	--control-plane --certificate-key 5fc319cfc09949c0f87f59178a69f37e84f609b67d692cb554b966e8a8b45fe7
```

## Install network plugin on  Master

Install Calico network plugins

```shell
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

Confirm that all of the pods are running:

```shell
watch kubectl get pods --all-namespaces
```

## Add worker nodes

Worker nodes (k8s-node1, k8s-node2) can be added using the command in installation output:

```shell
kubeadm join 172.16.16.151:6443 --token xumxit.a2fa72cdsbxwvrof \
> --discovery-token-ca-cert-hash sha256:59c778c94386ea046b923cc46ba57f52e942026b7e45db7cf138a70fd8e4ccc3
```

Two master nodes and two worker nodes should be up and running:

```shell
kubectl get nodes
```

## Configure Load Balancing 

Run the following command to install Keepalived and HAproxy first.

```shell
sudo apt update && sudo apt install -y haproxy keepalived psmisc
```

## HAproxy Configuration

Run the following command to configure HAproxy.

```shell
sudo nano /etc/haproxy/haproxy.cfg
```

Configuration for the reference

```shell
fronted kubernetes
bind *:6443
option tcplog
mode tcp
default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
mode tcp
balance roundrobin
option tcp-check
server kubernetes-master1 172.16.16.151:6443 check fall 3 rise 2
server kubernetes-master2 172.16.16.152:6443 check fall 3 rise 2
```

Save the file and run the following command to restart and enable HAproxy.

```shell
sudo systemctl restart haproxy
sudo systemctl enable haproxy
sudo systemctl status haproxy
```

## Keepalived Configuration

Run the following command to configure Keepalived.

```shell
nano /etc/keepalived/keepalived.conf
```

Configuration (lb1) for your reference:

```shell
global_defs {
  notification_email {
  }
  router_id LVS_DEVEL
  vrrp_skip_check_adv_addr
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}
   
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
   
vrrp_instance haproxy-vip {
  state BACKUP
  priority 100
  interface eth0                       # Network card
  virtual_router_id 60
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  unicast_src_ip 172.16.16.141      # The IP address of current machine
  unicast_peer {
    172.16.16.142                         # The IP address of peer machines
  }
   
  virtual_ipaddress {
    172.16.16.140/24                  # The VIP address
  }
   
  track_script {
    chk_haproxy
  }
}
```

Save the file and run the following command to restart and enable Keepalived.

```shell
systemctl restart keepalived
systemctl enable keepalived
systemctl enable haproxy
```

Make sure configure Keepalived on the other machine (lb2) as well.

## Verify High Availability

On the lb1, run the following command :
You will be able to see the virtual IP address is successfully added.

```shell
ip a s
```

Simulate a failure on lb1:
You check the floating IP address again and you can see it disappears on lb1. 

```shell
systemctl stop haproxy
```

Theoretically the virtual IP will be failed over to the lb2 if the configuration is successful.
On the lb2, run the following command:
You should be able to see the virtual IP address.

```shell
ip a s
```

Higly availability is successfully configured.
