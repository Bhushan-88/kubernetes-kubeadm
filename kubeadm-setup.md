# FOR Virtualbox vms ubuntu-24.04.3-live-server-amd64
# Hadware rqeuires
Master Node VM (2 CPUs, 2–4 GB RAM, 20 GB disk).
Worker Node(s) VM (1–2 CPUs, 2 GB RAM each, 15 GB disk).

MASTER NODE SETUP (192.168.1.10)
Step 0: Prerequisites
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Add hosts entry (do this on all nodes)
sudo vim /etc/hosts
# Add lines:
192.168.1.10  k8s-master
192.168.1.11  k8s-worker1
192.168.1.12  k8s-worker2

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
Step 1: Install Docker
```bash
sudo apt install -y curl apt-transport-https ca-certificates software-properties-common gnupg lsb-release

# Add Docker GPG key and repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list

# Install Docker and plugins
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable Docker service
sudo systemctl enable --now docker

# Configure cgroup driver
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
sudo systemctl restart docker
```
Step 2: Install Kubernetes Binaries
```bash
# Add Kubernetes repo
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubeadm, kubelet, kubectl
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Step 3: Install cri-dockerd (Docker shim)
```bash
# Install Go (needed for building cri-dockerd)
sudo apt install -y golang-go git

# Clone cri-dockerd repo
git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd

# Build cri-dockerd
mkdir bin
go build -o bin/cri-dockerd

# Move binary
sudo mv bin/cri-dockerd /usr/local/bin/

# Setup systemd services
sudo mkdir -p /etc/systemd/system
sudo cp packaging/systemd/* /etc/systemd/system
sudo sed -i 's:/usr/bin/cri-dockerd:/usr/local/bin/cri-dockerd:g' /etc/systemd/system/cri-docker.service

# Enable services
sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.service
sudo systemctl enable --now cri-docker.socket

```
Step 4: Initialize Kubernetes Master
```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.1.10 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///var/run/cri-dockerd.sock

```
Step 5: Configure kubectl
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes

```

Step 6: Install Calico CNI
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
kubectl get pods -n kube-system

```
------------------------------------------------------------------------------------------------------------
# Note
Issue-------------if calico-node fails 
kubectl get pods -n kube-system
calico-node-52fm8                          0/1     Running   4 (13s ago)   7m15s
kubectl describe pod calico-node-52fm8 -n kube-system
sudo systemctl restart kubelet
----- kubectl delete pod -n kube-system -l k8s-app=calico-node
Ensure localhost entry exists 
On your master node and worker, check:
echo "127.0.0.1 localhost" | sudo tee -a /etc/hosts
echo "::1 localhost ip6-localhost ip6-loopback" | sudo tee -a /etc/hosts
echo "192.168.1.10   k8s-master " | sudo tee -a /etc/hosts
echo "192.168.1.11   k8s-worker" | sudo tee -a /etc/hosts
sudo systemctl restart kubelet

Check resolv.conf inside pod
Sometimes the pod inherits wrong DNS.
Run:

kubectl exec -n kube-system calico-node-52fm8 -c calico-node -- cat /etc/resolv.conf
Should contain your cluster DNS (10.96.0.10 by default).
If only 8.8.8.8, that’s the problem → CoreDNS not applied or /etc/resolv.conf is overridden.

------------------------------------------------------------------------------------------------------------
# Same steps as master

# Update system packages
```bash
Kubernetes requires swap to be off, and hostname helps in cluster identification.
sudo apt update && sudo apt upgrade -y

# Set hostname
sudo hostnamectl set-hostname k8s-master

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
Step 1: Install Docker (same as master)
```bash
# Same Docker install and cgroup configuration as master
Why:
Docker is the container runtime.
native.cgroupdriver=systemd ensures Docker uses the same cgroup driver as kubelet.

sudo apt install -y curl apt-transport-https ca-certificates software-properties-common gnupg lsb-release

# Add Docker GPG key and repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list

# Install Docker and plugins
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable Docker service
sudo systemctl enable --now docker

# Configure cgroup driver
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
sudo systemctl restart docker
```

Step 2: Install Kubernetes Binaries
```bash
# Same kubeadm, kubelet, kubectl installation as master
Why:
kubeadm → bootstrap cluster
kubelet → agent that runs on each node
kubectl → CLI to manage cluster

# Add Kubernetes repo
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubeadm, kubelet, kubectl
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Step 3: Install cri-dockerd (Docker shim)
```bash
# Install Go (needed for building cri-dockerd)
sudo apt install -y golang-go git

# Clone cri-dockerd repo
git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd

# Build cri-dockerd
mkdir bin
go build -o bin/cri-dockerd

# Move binary
sudo mv bin/cri-dockerd /usr/local/bin/

# Setup systemd services
sudo mkdir -p /etc/systemd/system
sudo cp packaging/systemd/* /etc/systemd/system
sudo sed -i 's:/usr/bin/cri-dockerd:/usr/local/bin/cri-dockerd:g' /etc/systemd/system/cri-docker.service

# Enable services
sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.service
sudo systemctl enable --now cri-docker.socket

Why:
Kubernetes v1.24+ removed Docker support.
cri-dockerd allows kubelet to use Docker as container runtime.
```
Step 4: Join Worker Node
```bash
kubeadm join 192.168.1.10:6443 
--token epe931.le2jijd0fup4v0rx \
--discovery-token-ca-cert-hash sha256:18d4bce45c1c24f92ce34f35654975134b89c23da6723886cfa69a3e9b51f66b 
--cri-socket unix:///var/run/cri-dockerd.sock

Why:
--apiserver-advertise-address → master IP that workers will connect to.
--pod-network-cidr → network range for pods (required for Calico).
--cri-socket → tells kubeadm to use Docker via cri-dockerd.
```