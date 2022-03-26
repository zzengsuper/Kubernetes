# Set up Rook Ceph Cluster

Follow this documentation to set up a highly available Ceph storage cluster to a Kubernetes cluster.

## Vagrant Environment

| Role   | Host     | IP            | OS          | RAM  | CPU  |
| ------ | -------- | ------------- | ----------- | ---- | ---- |
| Master | cmaster  | 172.16.16.130 | Ubuntu20.04 | 2G   | 2    |
| Worker | cworker1 | 172.16.16.131 | Ubuntu20.04 | 2G   | 2    |
| Worker | cworker2 | 172.16.16.132 | Ubuntu20.04 | 2G   | 2    |
| Worker | cworker3 | 172.16.16.133 | Ubuntu20.04 | 2G   | 2    |

## Bring up all the vagrant machines

```shell
vagrant up
```

Kubernetes cluster should be up and running.
Export the kube config file to the local machine.

```shell
kubectl version --short
Client Version: v1.23.5
Server Version: v1.23.5
```

## Add raw partions to the worker nodes 

Each of the worker nodes will have one raw device /dev/vdb which we will add 40GB

```shell
sudo apt-get install -y lvm2
```

## Deploy Rook Storage Orchestrator

Clone the rook project from Github to the local machine

```shell
git clone --single-branch --branch release-1.8 https://github.com/rook/rook.git
```

Deploy the Rook operator
```shell
cd rook/deploy/examples/
kubectl create -f crds.yaml
kubectl create -f common.yaml
kubectl create -f operator.yaml
```

After few seconds Rook components should be up and running as seen below:
```shell
kubectl get all -n rook-ceph
```

## Create Rook Ceph Cluster

```shell
kubectl create -f cluster.yaml
```

Watching Pods creation in rook-ceph namespace:
```shell
watch kubectl get pods -n rook-ceph
```

This is a list of Pods running in the namespace after a successful deployment:
```shell
kubectl get pods -n rook-ceph
NAME                                                 READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-4k594                               3/3     Running     0          20m
csi-cephfsplugin-mnk4j                               3/3     Running     0          20m
csi-cephfsplugin-provisioner-5dc9cbcc87-7z79s        6/6     Running     0          20m
csi-cephfsplugin-provisioner-5dc9cbcc87-j9prg        6/6     Running     0          20m
csi-cephfsplugin-vk5w2                               3/3     Running     0          20m
csi-rbdplugin-5rvdx                                  3/3     Running     0          20m
csi-rbdplugin-dqdnq                                  3/3     Running     0          20m
csi-rbdplugin-provisioner-58f584754c-c6tfn           6/6     Running     0          20m
csi-rbdplugin-provisioner-58f584754c-w6547           6/6     Running     0          20m
csi-rbdplugin-qp6nh                                  3/3     Running     0          20m
rook-ceph-crashcollector-cworker1-576cdbc48c-j9vdc   1/1     Running     0          14m
rook-ceph-crashcollector-cworker2-58d496cc8d-9mnpg   1/1     Running     0          14m
rook-ceph-crashcollector-cworker3-f989c5b5c-8gwf8    1/1     Running     0          14m
rook-ceph-mgr-a-86cd5fd4db-d485d                     1/1     Running     0          14m
rook-ceph-mon-a-5bc98c8484-2gmb7                     1/1     Running     0          21m
rook-ceph-mon-b-64d7c56f67-vnsdt                     1/1     Running     0          20m
rook-ceph-mon-c-597cc64d54-9x2wl                     1/1     Running     0          17m
rook-ceph-operator-846695c777-97448                  1/1     Running     0          32m
rook-ceph-osd-0-c84f89587-6bxhq                      1/1     Running     0          14m
rook-ceph-osd-1-7888dd4b59-kv294                     1/1     Running     0          13m
rook-ceph-osd-2-86cdbf44cb-sfhdh                     1/1     Running     0          12m
rook-ceph-osd-prepare-cworker1-wrwnp                 0/1     Completed   0          14m
rook-ceph-osd-prepare-cworker2-s2x8f                 0/1     Completed   0          14m
rook-ceph-osd-prepare-cworker3-j46gx                 0/1     Completed   0          14m
```

Each worker node will have a Job to add OSDs into Ceph Cluster:
```shell
kubectl get -n rook-ceph jobs.batch
```

Verify that the cluster CR has been created and active:
```shell
kubectl -n rook-ceph get cephcluster
```

## Deploy Rook Ceph toolbox in kubernetes

```shell
kubectl apply -f toolbox.yaml
```

Connect to the pod using kubectl command with *exec* option:
```shell
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

Check Ceph Storage Cluster Status:
```shell
ceph status
```

List all OSDs to check their current status. They should exist and be up
```shell
ceph osd status
```

Check raw storage and pools:
```shell
ceph df
rados df
```

## Access Ceph Dashboard

Create external dashboard

```shell
kubectl create -f dashboard-external-https.yaml
```

Check new service created.
In my case, port 31266 will be opened to expose port 8443 from the ceph-mgr-pod.
Now you can enter the URL in your browser with any NodeIP:31266 and the dashboard will appear.

```shell
kubectl get svc -n rook-ceph
service/rook-ceph-mgr-dashboard-external-https created
```

Login with admin and password decoded from rook-ceph-dashboard-password secret

```shell
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```