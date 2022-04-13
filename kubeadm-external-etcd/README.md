## Vagrant Environment

| Role | Hostname | IP            | OS          | RAM  | CPU  |
| ---- | -------- | ------------- | ----------- | ---- | ---- |
| ETCD | etcd1    | 172.16.16.221 | Ubuntu20.04 | 1G   | 1    |
| ETCD | etcd2    | 172.16.16.222 | Ubuntu20.04 | 1G   | 1    |
| ETCD | etcd3    | 172.16.16.223 | Ubuntu20.04 | 1G   | 1    |

## Setup ETCD Cluster

Disable firewall

```shell
ufw disable
```

Disable swap
```shell
swapoff -a; sed -i '/swap/d' /etc/fstab
```

Enable kernel modules and configure sysctl

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
sudo apt install -y docker-ce containerd.io

# Configure containerd and start service
sudo su -
mkdir -p /etc/containerd
containerd config default>/etc/containerd/config.toml

# restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd
```

Install kubectl kubeadm kubelet

```shell\
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
}
```

```shell
apt update && apt install -y kubeadm=1.23.0-00 kubelet=1.23.0-00 kubectl=1.23.0-00
```

kubelet configuration
```shell
cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
[Service]
ExecStart=
#  Replace "systemd" with the cgroup driver of your container runtime. The default value in the kubelet is "cgroupfs".
ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd
Restart=always
EOF

systemctl daemon-reload
systemctl restart kubelet
```

```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
}
EOF
  
mkdir -p /etc/systemd/system/docker.service.d
```

kubeadm configuration

```shell
# Update HOST0, HOST1 and HOST2 with the IPs of your hosts
export HOST0=172.16.16.221
export HOST1=172.16.16.222
export HOST2=172.16.16.223

# Update NAME0, NAME1 and NAME2 with the hostnames of your hosts
export NAME0="etcd1"
export NAME1="etcd2"
export NAME2="etcd3"

# Create temp directories to store files that will end up on other hosts.
mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

HOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=(${NAME0} ${NAME1} ${NAME2})

for i in "${!HOSTS[@]}"; do
HOST=${HOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: InitConfiguration
nodeRegistration:
    name: ${NAME}
localAPIEndpoint:
    advertiseAddress: ${HOST}
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: ${NAMES[0]}=https://${HOSTS[0]}:2380,${NAMES[1]}=https://${HOSTS[1]}:2380,${NAMES[2]}=https://${HOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done
```

Generate cert on etcd1
```shell
kubeadm init phase certs etcd-ca
```

Generate etcd cert for each node
```shell
kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/
# cleanup non-reusable certificates
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
# No need to move the certs because they are for HOST0

# clean up certs that should not be copied off this host
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete

```

Copy the cert to etcd2 and etcd3
```shell
scp -r /tmp/${HOST1}/* root@${HOST1}:/etc/kubernetes
scp -r /tmp/${HOST2}/* root@${HOST2}:/etc/kubernetes
```

Check the files should be existed in the each of etcd node
```shell
[root@etcd1 ~]# tree /etc/kubernetes/
/etc/kubernetes/
├── kubeadmcfg.yaml
├── manifests
│   └── etcd.yaml
└── pki
    ├── apiserver-etcd-client.crt
    ├── apiserver-etcd-client.key
    └── etcd
        ├── ca.crt
        ├── ca.key
        ├── healthcheck-client.crt
        ├── healthcheck-client.key
        ├── peer.crt
        ├── peer.key
        ├── server.crt
        └── server.key

3 directories, 12 files

[root@etcd2 kubernetes]# tree /etc/kubernetes/
.
├── kubeadmcfg.yaml
├── manifests
│   └── etcd.yaml
└── pki
    ├── apiserver-etcd-client.crt
    ├── apiserver-etcd-client.key
    └── etcd
        ├── ca.crt
        ├── healthcheck-client.crt
        ├── healthcheck-client.key
        ├── peer.crt
        ├── peer.key
        ├── server.crt
        └── server.key

3 directories, 11 files


[root@etcd3 ~]# tree /etc/kubernetes/
/etc/kubernetes/
├── kubeadmcfg.yaml
├── manifests
│   └── etcd.yaml
└── pki
    ├── apiserver-etcd-client.crt
    ├── apiserver-etcd-client.key
    └── etcd
        ├── ca.crt
        ├── healthcheck-client.crt
        ├── healthcheck-client.key
        ├── peer.crt
        ├── peer.key
        ├── server.crt
        └── server.key

3 directories, 11 files

```

Create static pod manifests
```shell
[root@etcd1 ~]#  kubeadm init phase etcd local --config=/etc/kubernetes/kubeadmcfg.yaml  
[root@etcd2 ~]#  kubeadm init phase etcd local --config=/etc/kubernetes/kubeadmcfg.yaml  
[root@etcd3 ~]#  kubeadm init phase etcd local --config=/etc/kubernetes/kubeadmcfg.yaml  
```

Check the etcd cluster status

```shell
[root@etcd1 kubernetes]# docker run --rm -it --name etcd-check \
-v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:3.4.3-0 etcdctl \
--cert /etc/kubernetes/pki/etcd/peer.crt \
--key /etc/kubernetes/pki/etcd/peer.key \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--endpoints https://172.16.16.221:2379 endpoint health --cluster
https://192.168.1.244:2379 is healthy: successfully committed proposal: took = 29.827986ms
https://192.168.1.243:2379 is healthy: successfully committed proposal: took = 30.169169ms
https://192.168.1.242:2379 is healthy: successfully committed proposal: took = 31.270748ms
```
