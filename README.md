# kubernetes-kubeadm
All objects with kubeadm
## What is Kubernetes?

Kubernetes (often called K8s) is an open-source container orchestration platform developed by Google and now maintained by the Cloud Native Computing Foundation (CNCF).
It helps you automate the deployment, scaling, and management of containerized applications (apps running inside containers like Docker).

## Kubernetes has two main layers:
1. Control Plane (Master components) → Manages the cluster.
1. Worker Nodes (Data Plane) → Runs the actual applications (containers).

### Uses of Kubernetes
- Automates container deployment & scaling
- Self-healing (restarts failed containers)
- Load balancing
- Service discovery
- Storage orchestration
- Rolling updates & rollbacks
- Resource utilization optimization

---

## Core Components of Kubernetes
1. API Server – Entry point for all commands.
2. etcd – Key-value store for cluster state.
3. Controller Manager – Ensures cluster desired state.
4. Scheduler – Assigns Pods to Nodes.
5. Kubelet – Agent running on each node.
6. Kube-proxy – Handles networking & load balancing.
7. Pod – Smallest deployable unit.

---

## Minikube, Kind, Kubeadm
- **Minikube** – Run Kubernetes locally in VM/Container (for practice).
- **Kind** – Runs Kubernetes clusters inside Docker containers (lightweight).
- **Kubeadm** – Tool to bootstrap a production-like Kubernetes cluster.

---

##  Kubernetes Objects
- **Service** → Exposes Pods to network.
- **Replication Controller** → Ensures a specific number of Pods.
- **ReplicaSet** → Newer version of Replication Controller.
- **Deployment** → Manages ReplicaSets & supports rollout strategies.
- **StatefulSet** → For stateful apps (databases, unique Pod IDs).
- **DaemonSet** → Runs one Pod on every Node.
- **ConfigMap** → Stores configuration data.
- **Secret** → Stores sensitive data (passwords, keys).

---

## General Requirements for Installing Kubernetes
1. Hardware Requirements

Control Plane Node (Master)
Minimum: 2 CPUs, 2 GB RAM
Recommended: 2+ CPUs, 4 GB RAM or more
Worker Nodes
Minimum: 1 CPU, 2 GB RAM
Recommended: 2+ CPUs, 4 GB RAM (depends on workloads)
Disk Space
At least 20 GB free disk space per node

2. Operating System

3. Networking Requirements

4. Software Requirements
Container Runtime (choose one):
1.containerd (preferred)
2.CRI-O
3.Docker (deprecated, but still usable via cri-dockerd)
kubeadm, kubelet, kubectl (if using kubeadm)
CNI (Container Network Interface) plugin (like Calico, Flannel, Weave, Cilium)

## minikube installation On Windows:
1.Install docker - docker --version
2.Install minikube  - minikube version 
3.Install kubectl - kubectl version --client

Installation link: https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2Fwindows+package+manager
```bash
winget install Kubernetes.minikube
minikube version 
minikube start --driver=docker OR//
minikube start --driver=virtualbox
minikube status
minikube stop 
minikube delete

kubectl cluster-info
kubectl get nodes -o wide 
```
## Deploy First Application
```bash
kubectl run my-pod --image nginx 
kubectl get pods -o wide 
kubectl describe pod my-pod 
kubectl delete pod my-pod 
kubectl get pods -w

# application access 
minikube ssh 
curl <pod Ip>:port (inside the cluster)
```
# expose pod 
```bash
kubectl expose pod my-pod --name my-svc --type NodePort --port 80
minikube service my-pod curl 192.168.49.2:31698 (access ur pod outside the container after expose)
minikube service my-pod ->(If you want to avoid remembering NodeIP + NodePort, you can directly run)
OR/ curl <node_IP>:<node_port>
```

##  Namespaces
- Logical partitions in a cluster for organizing resources.
 Default Namespaces

When you create a cluster, Kubernetes already provides some namespaces:
default → Used when you don’t specify a namespace.
kube-system → For system Pods (DNS, networking, controllers).
kube-public → Publicly readable resources (like cluster info).
kube-node-lease → Stores heartbeat information from nodes.

### ✅ Commands:
```bash
kubectl get namespaces
kubectl get pods --namespace kube-system 
kubectl create namespace dev
kubectl get pods -n dev
kubectl run <podname> --image=nginx -n dev (#Create a Pod in dev namespace)
minikube service nginx -n dev 
```
## Lists all available resource types in Kubernetes.
```bash
kubectl api-resources
```
## Services 

A Service in Kubernetes is like a permanent doorway that allows communication between different parts of your cluster (pods, users, or external traffic).

-Problem: Pods get a new IP each time they restart or scale → very unstable.
-Solution: A Service gives a stable IP + DNS name that stays the same, even if the underlying pods change.

## Types of Services

1. ClusterIP (default)
Accessible only inside the cluster.

2. NodePort
Opens a port on each Node so traffic can come from outside.
Access format: NodeIP:NodePort

3. LoadBalancer
Creates a cloud Load Balancer (AWS, GCP, Azure) to expose the service externally.


## Exposing a Pod (service)
# Check service
```bash
kubectl get svc -n dev
kubectl expose pod my-pod 
kubect expose my-app --image nginx --port 80
kubectl expose pod nginx -n dev --type NodePort --port 80 (-n dev / if our service is in diff namespace)

# Get URL (Minikube only)
minikube service nginx -n dev
```
## Goto inside the container
```bash
kubectl exec -it -n dev mypod -- bash
kubectl exec -it -n dev mypod -c nginx-container -- bash
```
##  Create Manifest file YAMAL
pod.yaml
service.yaml
```bash
kubectl apply -f pod.yaml
kubectl delete -f pod.yaml

```
# Replication Controller
ReplicationController (RC) in Kubernetes is an object that makes sure a specific number of identical Pods are always running.

Keeps a fixed number of pods running: You specify a desired number of replicas in the ReplicationController's definition. The controller then creates new pods until the actual number of running pods matches the desired count.

Automatic self-healing: If a pod crashes, fails, or is deleted for any reason, the controller automatically replaces it with a new one. This ensures high availability and resilience for your application.

The selector decides which Pods the controller should manage.

## Create ReplicationController.yaml
```bash 
kubectl apply -f ReplicationController.yaml
kubectl get rc -n dev
kubectl get -n dev all -A
kubectl get all
kubectl get pod
kubectl get svc 
kubectl scale rc my-rc --replicas 5 ##Also we can change desire capacity using rc yaml file OR// 
kubectl edit rc my-rc
kubectl delete rc myapp-rc
kubectl describe rc my-rc 
minikube service my-svc #to access 

```
# Replicaset 
A ReplicaSet (RS) in Kubernetes is a controller that ensures a specific number of replica Pods are running at all times, just like a ReplicationController but more advanced.
It can use advanced selectors to manage Pods.

Diff: 
RC → only supports equality-based selectors (e.g., app=nginx).
RS → supports both equality and set-based selectors (more flexible).
ex:-1. matchLabels:
         app: myapp # Selector to match the pods
    2.matchExpressions:
      - key: app
        operator: In
        values: ["nginx", "tomcat"]
      - key: env
        operator: NotIn
        values: ["dev"]

API Group
RC → belongs to the core/v1 API.
RS → belongs to apps/v1 API, which is the modern, scalable, and extensible API group.

```bash
kubectl apply -f replicaset.yaml
kubectl create -f replicaset.yaml
kubectl get rs
kubectl get replicaset -o wide
kubectl describe rs my-rs
kubectl get pods
kubectl get pods -l app=myapp   # filter by label
kubectl scale rs my-rs --replicas=4
#edit
kubectl edit rs my-rs
#delete
kubectl delete rs my-rs
kubectl delete rs my-rs --cascade=orphan # delete only rs
#View logs of a Pod in ReplicaSet
kubectl get pods -l app=myapp
kubectl logs <pod-name>
#Execute a command inside a Pod container
kubectl exec -it <pod-name> -- bash
#Dry-run (test without applying)
kubectl apply -f replicaset.yaml --dry-run=client
#Export ReplicaSet YAML from cluster
kubectl get rs my-rs -o yaml > my-rs-export.yaml

```
# Deployment Strategies 
A Deployment in Kubernetes is a higher-level abstraction that manages ReplicaSets for you.
features like:
Automatic ReplicaSet creation & management
Rolling updates (update Pods gradually without downtime)
Rollbacks (go back to an older version if something fails)
Easier scaling (kubectl scale deployment)

```bash 
kubectl scale deployment my-deployment --replicas=5 #Scale deployment (say 5 Pods instead of 2)
kubectl set image deployment/my-deployment nginx-container=nginx:1.27.2 #Update image (rolling update)
kubectl rollout undo deployment my-deployment #Rollback to previous version
kubectl rollout status deployment my-deployment #Check rollout status
```
# StatefulSet
A StatefulSet is a Kubernetes workload object just like a Deployment, but designed for stateful applications.

Pods get stable names (pod-0, pod-1, pod-2 …).
Pods are deleted in reverse order (2 → 1 → 0).
Each Pod can have its own persistent storage (using PersistentVolumeClaim).
Even if Pods are deleted, when they come back, they keep the same name+same storage.

ex:- env 
      - name: MYSQL_ROOT_PASSWORD
      value: admin123
```bash 
kubectl exec -it my-statefulset-0 -- mariadb -uroot -prootpassword
kubectl scale sts my-statefulset --replicas 5
kubectl get pvc
```
# A DaemonSet ensures that one Pod runs on every Node (or on a selected group of Nodes).
Running monitoring agents (e.g., Prometheus Node Exporter).
Running logging agents (e.g., Fluentd, Logstash).
Running networking components (e.g., kube-proxy, CNI plugins).
Running security agents (antivirus, firewalls).
How it works:
If you have 3 nodes in the cluster → Kubernetes schedules 1 nginx Pod per node.
If a new node is added → DaemonSet automatically schedules a Pod there.
If a node is removed → Pod on that node is deleted.

Run practical on killerkoda 

```bash
kubectl describe ds nginx-daemonset
kubectl get daemonsets
kubectl get pods -o wide
```

# ConfigMap
A ConfigMap in Kubernetes is an API object used to store configuration data (in key-value pairs, JSON, YAML, or entire config files) that your Pods can consume.
Use case: Instead of hardcoding environment variables or config files inside Pods, you externalize them in a ConfigMap. This way, you can change configuration without rebuilding your image.

Ways to use ConfigMap in Pods
As environment variables inside a container.
As configuration files mounted into a container (volume).
As command-line arguments for a container.
ConfigMap → Non-sensitive data (URLs, ENV, feature flags).

# Secret
A Secret is just like a ConfigMap, but it’s designed to store sensitive data such as:
Passwords
API keys
Tokens
SSH keys

Secrets keep this data hidden (base64 encoded) and are not exposed in plain text like ConfigMaps.
```bash
echo -n "admin" | base64
echo -n "admin123" | base64
echo -n "YWRtaW4xMjM=" |base64 -d
kubectl get secret 
kubectl describe secret db-secret
kubectl exec -it my-statefulset-0 -- mariadb -uroot -padmin123
```

# Ingress

Ingress is a Kubernetes API object that manages external access to services in a cluster, typically HTTP and HTTPS traffic.

Instead of exposing each service with a NodePort or LoadBalancer, you use an Ingress to provide a single entry point for multiple services.

🔹 Without Ingress:

Each service needs a NodePort or LoadBalancer.
Hard to manage multiple apps.

🔹 With Ingress:
One external IP for multiple services.
Route traffic based on hostnames or paths.
Supports SSL/TLS termination.

🔹 Ingress vs Ingress Controller

Ingress Resource (YAML):
Just a set of rules (e.g., route /api → backend-service).

Ingress Controller (Pod running inside cluster):
The actual implementation (usually a reverse proxy like NGINX, Traefik, HAProxy) that reads those rules and forwards traffic correctly.

# Why Do We Need an Ingress Controller?

Kubernetes doesn’t have built-in ingress routing.
By default, K8s only supports ClusterIP, NodePort, LoadBalancer services.
No smart routing or domain-based access.

Ingress Controller implements reverse proxy logic.
Handls HTTP/HTTPS traffic.
Routes requests based on hosts or paths.

TLS/SSL termination.
You can configure HTTPS certificates in Ingress, and the Ingress Controller will terminate TLS.

Load balancing.
Distributes traffic across multiple Pods behind a Service.

Centralized traffic entry point.
Instead of exposing every app via NodePort/LoadBalancer, you expose only the Ingress Controller.

```bash 
kubectl get ing 
minikube addons enable ingress
kubectl get all  -n ingress-nginx
```