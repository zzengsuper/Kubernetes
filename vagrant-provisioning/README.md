# Set up a Kubernetes Cluster using Vagrant

Follow this documentation to set up a Kubernetes cluster which one master node with two worker nodes using Vagrant.

## Vagrant Environment

| Role   | FQDN                 | IP            | OS          | RAM  | CPU  |
| ------ | -------------------- | ------------- | ----------- | ---- | ---- |
| Master | kmaster.example.com  | 172.16.16.100 | Ubuntu20.04 | 2G   | 2    |
| Worker | kworker1.example.com | 172.16.16.101 | Ubuntu20.04 | 1G   | 1    |
| Worker | kworker2.example.com | 172.16.16.102 | Ubuntu20.04 | 1G   | 1    |

## Bring up all the virtual machines

```shell
vagrant up
```

