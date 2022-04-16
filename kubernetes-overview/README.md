

[toc]

# Kubernetes Overview

## Intro to K8s

### What is kubernetes

- open source container orchestration tool

- developed by Google

- Helps you manage containerized applications in different deployment environments (physical, virtual, cloud, hybrid)


### What problems does kubernetes solve? What are the tasks of an orchestration tool?

- The rise of microservices caused increated usage of container technologies. Because the containers actually offer the perfect host for small independent applications like microservices, and the rises of containers and the microservice technology actually resulted in applications that they're now comprised of hundreds sometimes maybe even thousands of containers. now managing those loads of containers across multiple environments using scripts and self-made tools can be really complex and sometimes even impossible. So that specific scenario actually caused the need for having container orchestration technologies.



### The need for a container orchestration tool

- Trend from Monolith to Microservices

- Increased usage of containers

- Demand for a proper way of managing those hundreds of containers

  

### What features do orchestration tools offer?

- High Availability or no downtime

- Scalability or high performance

- Disaster recovery - backup and restore

  

### K8s architecture

![Kubernetes cluster](https://github.com/zzengsuper/Kubernetes/blob/master/kubernetes-overview/K8s_png/Kubernetes%20cluster.png)

#### 3 process running on every worker node

- container runtime

- kubelet 

  - interact with both the container and node
  - starts pod with a container inside and then assigning the node to the container like cpu, ram and storage
  
- kube proxy

  - the way communication between node 1 and node 2 works is Services. which is sort of load balancer that basically catches the request directly to the pod or the application
  - process that is responsible for forwarding requests from the services to pods is actually kube proxy
  - kube proxy is intelligent forwarding logic inside that make sure communication also works in a performant way with low overhead (for example if application my-app replica is making a requested database instead of service just randomly forwarding the request to any replica it will actually forward it to the replica that is running on the same node as the pod that initiated the request , so this way avoiding the network overhead of sending the request to another machine)

  

#### 4 process running on master node (control cluster state and worker nodes)

- API Server 

  - cluster gateway which gets the initial request of any updates into the cluster or even the queries)
  - acts as a gatekeeper for authentication
  - main entry point into the kubernetes cluster (so you need to first talk to API Server)

- Scheduler 

  - send an API Server a request to schedule a new pod , API server after it validates your request actually hand it over to the schedular in order to start that application pod on one node
  - instead of just randomly assigning it to any node scheduler has this whole intelligent way of deciding on which specific worker node the next pod will be scheduled 
  - Schedular just decides on which Node new Pod should be scheduled

- Controller manager

  - pods dies on any node there must be a way to detect the ndes dies and reschedule those pods as soon as possible
  - detect cluster state changes, and it makes a request for the scheduler to reschedule those dead pods and the same cycle happens here where the schedular decides based on the resource calculation which worker nodes should restart those pods again and makes requests to the corresponding cubelets on those worker nodes to actually restart the pods

- etcd

  - key value to store the cluster state
  - etcd is the cluster brain which means every change in the cluster for example when a new pod gets scheduled when a pod dies all of these changes get stored in the key value 
  - all of these mechanism with scheduler and controller manager works because of its (etcd) data
  - Is the cluster healthy?
  - what resources are available?
  - Dif the cluster state change?

  

### main K8s components

- Pod

  - smallest unit of K8s
  - abstraction over container
  - usually 1 application per pod
  - each pod gets its own IP address
  - new IP address on re-creation

- Service

  - permanent IP address
  - lifecycle of pod and service not connected (even pod dies the service and its IP address will stay)
  - external-service(my-app) http://my-app.com:8080   http://124/89.101.2:8080 
  - internal-service(DB) 
  - load balancer

- Ingress

  - https://my-app.com  

  - external service with the secure protocol and domain name is another component of kubernetes called as Ingress

  - used to route traffic into the cluster

- Volumes

  - persistence using volume 

  - DB stores on local or remote
  - Ks8 explicitly doesn't manage any data persistence which means someone are responsible backing up data replicating and managing it and making sure it's kept on a proper hardware etc.

- Scretes

  - Don't put credentials into ConfigMap with plain text
  - external configuration used to store secret data
  - base64 encoded
  - use it as environment variables or as a properties file

- ConfigMap

  - external configuration of you application (if you change the name of the DB endpoint of the service, you just need to adjust the configMap and that's it)

- StatefulSet

  - statefulset just like deployment would take care of replicating the pods and scaling them up or scaling then down but making sure the database reads and writes are synchronized so that no database inconsistencies are offered
  - statefulset for stateful apps or databases
  - DB are often hosted outside of K8s cluster (statefulsets in k8s can be somewhat tedious-just have the deployments or stateless applications that replicate and scale with no problem inside of the kubernetes cluster and communicate with the external database)

- Deployment

  - deployment for stateless apps

  - blueprint for my-app pods
  
  - you create deployments
  
  - abstract of pods (another abstraction on top of pods which makes it more convenient to interact with the pods ,replicate them and do some other configuration)
  
    

### minikube and kubectl - Local Setup

2 Master Nodes - less resource 
3 Worker Nodes - more resource

- Hardware resources of master and node servers actually differ.
- master processes are more important but they actually have less load of work, so they need less resources like cpu, ram and storage
- whereas the worker nodes do the actual job of running those pods with containers inside therefore they need more resources 
- as your application complexity and its demand of resources increases you may actually add more master node servers to your cluster and thus forming a more powerful and robust cluster to meet your application resources requirements 

#### how to add new Master/Node server

1. get new bare server

2. install all the master/worker node processes

3. add it to the cluster

#### minikube

- create virtualbox on your laptop
- masteer and worker nodes run in that virtualbox
- 1 node K8s cluster
- for testing purposes

#### kubectl

- command line tool for K8s cluster 
- kubectl  is the most powerful way to talk to API Server than UI and API
- kubctl isn't just for minikube cluster, cloud cluster, hybrid cluster whatever kubectl is the tool to use to interact with any type of kubernetes cluster setup

#### installtion and create minikube cluster

[Installation guide link](https://v1-18.docs.kubernetes.io/docs/tasks/tools/install-minikube/)

How to install on MacOS

1. brew update

2. brew install hyperkit (minikube needs to run in virtual environment)

3. brew install minikube

4. minikube start --vm-driver=hyperkit (tell minikube use the hyperkit hypervisor to start this virtual minikube cluster)

5. kubectl get nodes (get status of nodes)

6. minikube status 

7. kubectl version

8. [more kubectl command](https://gitlab.com/nanuchi/youtube-tutorial-series/-/blob/master/basic-kubectl-commands/cli-commands.md)

   

### main kubectl commands - K8s CLI

#### Create and debug pods in a minikube cluster

- CRUD commands:
  - create deployment : kubectl create deployment [name]
  - edit deployment: kubectl edit deployment [name]
  - delete deployment: kubectl delete deployment [name]
- Status of different K8s components
  - kubectl get nodes | pod | services | replicates | deployment
- Debugging pods
  - log to console : kubectl logs [pod name]
  - get interactive terminal : kubectl exec -it [pod name] --bin/bash
  - get info about pod : kubectl describe pod [pod name]
- Use configuration file for CRUD
  - apply a configuration file : kubectl apply -f [file name]
  - delete with configuration file : kubectl delete -f [file name]



#### Why you couldn't create pods directly?

- because pod is the smallest unit, you're not working with pods directly

- There is an abstration layer over the pods, that is called deployment
- kubectl create deployment nginx-depl --image=nginx
  - blueprint for creating pods
  - another layer which is automatically managed by kubernetes deployment called repliaset
  - kubectl get replicaset (nginx-depl-xxxxxx has been created)
- Replicaset is managing the replicas of a pod
- you in practice will never have to create replicaset or delete/update in any way, you're going to be working with deployment directly 



#### Layers of abstration

- deployment manages a replica set 
- replicaset manages all the replicas of that pod
- pod is an abstration of a container 
- container
- everything below deployment is handled by Kubernetes



### K8s yamls configuration file

#### 3 parts of K8s configuration file

- metadata (name)
- specification
- status (automatically generated and added by kubernetes, compare desired and actual state and self-healing )
  - where does K8s get this status data?
    etcd, cluster brain one of the master processes that actually stores the cluster data, so etcd holds at any time the current status of any kubernetes component and that's where the status information comes from

#### YAML configuration files

- human friendly data serialization standard for all programming language
- syntax: strict indentation! (yaml validator to see where need to fix that, code editors also have plugins for YAML syntax validation)
- store the config file with your code or own git repository 

#### Connecting deployments to service to pods

- The way the connection is established is using the lables and selectors 
- connecting deployment to pods
  - any key-value pair for component (labels: app: nginx)
  - pods get the label through the template blueprint
  - This label is matched by the selector

### Hands-on demo

mongoDB

- Deployment config file is checked into repository
- Username and password should not go there!
- Secret lives in K8s, not in the repository!
- Storing the data in a secret component doesn't automatically make it secure. 
- There are built-in mechanism (lee encryption) for basic security, which are not enabled by default
- The value must be base64 encoded
  - echo -n 'username' | base64 (exec on terminal)
  - echo -n 'password' | base64 (exec on terminal)
- Secret must be created before the Deployment!
- After secret has been created, secret can be referenced now in Deployment

mongo-express

- which database to connect?
  - mongodb address / internal server
  - mongodb_server
- ConfigMap
  - external configuration
  - centralized
  - other components can use it
  - configmap must already be in the K8s cluster, when referencing it
  - confgMapKeyRef instead of secretKeyRef 

- which credentials to authenticate?
  - adminusername
  - adminpassword
- How to make it an External Service?
  - type : "Loadbalancer" (But internal service also acts as a load balancer!)
  - This type of LoadBalancer is chosen basically is it accepts requests by assigning the service an external ip address 
  - Cluster IP difference with LoadBalancer IP
    - Cluster IP will give the service an internal IP address
    - LoadBalancer IP also gives an internal IP address but in addition to that it will also give the service an external ip address where the external requests will be coming from 
  - nodePort :  must be between 30000-32767 (port for external IP address/port you need to put into browser)

- How to assign an external service a public IP address
  - minikube service mongo-express-service



## Advanced concepts

#### K8s namespaces - organize your components

What is namespaces?
why use namespces?
namespaces yaml configuration file
Kubectx installation guides

#### K8s ingress

What is Ingress?
Ingress YAML configuration
When do you need Ingress?
Ingress controller

- evaluates all the rules

- manage redirections

- entrypoint to cluster

- many third-party implementations

- k8s Nginx Ingress Controller 

  - command line : minikube addons enable ingress 
  - it will automatically starts the K8s Nginx implementation of Ingress Controller
  - Ingress Controller configured with just nana command
  - kubectl get pod -n ingress-nginx
  - you will see the nginx ingress controller pod running in your cluster
  - once ingress controller installed now I can create an ingress rule that the controller can evaluate

- Configure dashboard-ingress yaml file

- Configure default backend in ingress

  - kubectl describe ingress myapp-ingress

- multiple paths for same host
  http://myapp.com/service1

  http://myapp.com/service2

- multi sub-domains or domains

  http://service1.myapp.com

- Configure TLS Certificate

- Configure Secret

  - Data keys need to be "tls.crt" and "tls.key"
  - Values are file contents NOT file paths/locations
  - Secret component must be in the same namespace as the Ingress component

#### Helm - package manager

#### Volumes - persisting data in K8s 

Persistent Volume
Persistent Volume ClaimStorage Class

- Storage Class usage
  1. pod claims storage via PVC
  2. PVC requests storage from SC
  3. SC creates PV that meets the needs of the Claim

#### K8s StatefulSet - deploying stateful apps

#### K8s Services







