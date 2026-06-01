
# Kubernetes 3-Node Cluster Homelab

I built a fully functional three-node Kubernetes cluster using three repurposed spare machines, consisting of one control 
plane node and two worker nodes. The environment was set up on a local network with Ubuntu, containerd, and kubeadm, and 
includes core cluster components such as networking, DNS, and workload scheduling. This project demonstrates hands-on 
experience with cluster initialization, node management, troubleshooting real-world issues like network disruption and 
swap misconfiguration, and restoring node health through kubelet and system-level debugging. The result is a stable, 
self-managed lab environment used to explore and validate Kubernetes concepts in a practical, production-like setup.

The cluster consists of an Intel NUC, two Apple MacBook laptops and a 5 port network switch.
 
<img src="images/k8s-cluster.jpg" width="600"/>

## Step 1 — Configure Hostnames
Run on each node.

Control Plane
Bash
````text
sudo hostnamectl set-hostname k8s-master
````

Bash
````text
sudo hostnamectl set-hostname k8s-worker1
````

Bash
````text
sudo hostnamectl set-hostname k8s-worker2
````

## Step 2 — Configure /etc/hosts
Run on ALL nodes.

Edit hosts file:

Bash
```text
sudo nano /etc/hosts
````
Add:
````text
192.168.1.10 k8s-master
192.168.1.11 k8s-worker1
192.168.1.12 k8s-worker2
````
Test connectivity:

Bash
````text
ping k8s-master
ping k8s-worker1
ping k8s-worker2
````

## Step 3 — Disable Swap
Kubernetes requires swap disabled.

Run on ALL nodes:

Bash
````text
sudo swapoff -a
````
Disable permanently:

Bash
````text
sudo sed -i '/ swap / s/^/#/' /etc/fstab
````

Verify:

Bash
````text
free -h
````
Swap should show 0B.

## Step 4 — Enable Required Kernel Modules
Run on ALL nodes.

Create modules config:

Bash
````text
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
````
Load modules:

Bash
````text
sudo modprobe overlay
sudo modprobe br_netfilter
````

## Step 5 — Configure Kubernetes Networking Sysctl Settings
Run on ALL nodes.

Create sysctl config:

Bash
````text
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
````
Apply settings:

Bash
````text
sudo sysctl --system
````
Verify:

Bash
````text
sysctl net.ipv4.ip_forward
````
Should return:

net.ipv4.ip_forward = 1

## Step 6 — Install containerd
Run on ALL nodes.

Update packages:

Bash
```text
sudo apt update
````
Install dependencies:

Bash
````text
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
````  
Install containerd:

Bash
sudo apt install -y containerd
Create default config:

Bash
````text
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
````
Enable Systemd cgroups (required/recommended)

Edit config:

Bash
````text
sudo nano /etc/containerd/config.toml
````
Find:

TOML
SystemdCgroup = false
Change to:

TOML
SystemdCgroup = true
Restart containerd:

Bash
````text
sudo systemctl restart containerd
sudo systemctl enable containerd
````
Verify:

Bash
````text
sudo systemctl status containerd
````
## Step 7 — Install Kubernetes Packages
Run on ALL nodes.

Create Kubernetes keyring directory:

Bash
````text
sudo mkdir -p /etc/apt/keyrings
````
Add Kubernetes GPG key:

Bash
````text
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
````
Add repository:

Bash
````text
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
````
Update:

Bash
````text
sudo apt update
````
Install Kubernetes tools:

Bash
````text
sudo apt install -y kubelet kubeadm kubectl
````
Prevent automatic upgrades:

Bash
````text
sudo apt-mark hold kubelet kubeadm kubectl
````
Enable kubelet:

Bash
````text
sudo systemctl enable kubelet
````
## Step 8 — Initialize the Control Plane
Run ONLY on the control plane node.

Initialize cluster:

Bash
````text
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16
  ````
This takes several minutes.

## Step 9 — Configure kubectl Access
After initialization completes, run on the control plane node:

Bash
````text
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
````
Test:

Bash
````text
kubectl get nodes
````
You should see the master node in NotReady state initially.

## Step 10 — Install Pod Network (Calico)
Run ONLY on the control plane node.

Apply Calico:

Bash
````text
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
````
Wait a few minutes.

Verify pods:

Bash
````text
kubectl get pods -A
````
Eventually all pods should become Running.

## Step 11 — Join Worker Nodes
During kubeadm init, a join command is displayed.

Example:

Bash
````text
sudo kubeadm join 192.168.1.10:6443 \
  --token abcdef.1234567890abcdef \
  --discovery-token-ca-cert-hash sha256:xxxxxxxx
````
Run that command on BOTH worker nodes.

## Step 12 — Verify Cluster
Run on control plane:

Bash
````text
kubectl get nodes
````
Expected output:
````text
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   10m   v1.30.x
k8s-worker1    Ready    <none>          5m    v1.30.x
k8s-worker2    Ready    <none>          5m    v1.30.x
````

## Step 13 — Test the Cluster
Deploy nginx:

Bash
````text
kubectl create deployment nginx --image=nginx
````
Scale deployment:

Bash
````text
kubectl scale deployment nginx --replicas=3
````
Check pods:

Bash
````text
kubectl get pods -o wide
````
You should see pods distributed across workers.

Useful Administrative Commands
Cluster Info
Bash
````text
kubectl cluster-info
````
Node Status
Bash
````text
kubectl get nodes -o wide
````
Pod Status
Bash
````text
kubectl get pods -A
````
Describe Problematic Pod
Bash
````text
kubectl describe pod <pod-name>
````
View Logs
Bash
````text
kubectl logs <pod-name>
````

Optional Next Steps

After the cluster is working, consider adding:
- Ingress Controller 
 -	NGINX Ingress Controller 
 -	Traefik 
-	Storage 
 -	Longhorn 
 -	Rook 
•	Observability 
o	Prometheus 
o	Grafana 
•	GitOps 
o	Argo CD 
o	Flux 
Install NGINX Ingress Controller on Your Kubernetes Cluster
This guide assumes:
•	Your 3-node Kubernetes cluster is operational 
•	kubectl get nodes shows all nodes as Ready 
•	You are using: 
o	Kubernetes 
o	containerd 
o	Calico 
We will install:
•	NGINX Ingress Controller 
using the official bare-metal deployment method.
 
# Install Ingress Controller 
The ingress controller:
•	receives HTTP/HTTPS traffic 
•	routes traffic to services inside Kubernetes 
•	replaces NodePort-only access 
•	enables: 
o	hostnames 
o	TLS 
o	reverse proxy 
o	path routing 

 
## IMPORTANT NODE INFORMATION
Task	Run On
kubectl apply commands	Control Plane ONLY
Helm installation	Control Plane ONLY
Ingress pods	Automatically scheduled to workers
Browser testing	Any machine on network
You do NOT manually install ingress software on workers.
Kubernetes schedules the ingress controller pods automatically.
 
## Step 1 — Verify Cluster Health
Run on CONTROL PLANE node:
kubectl get nodes
Expected:
NAME            STATUS   ROLES
k8s-master      Ready    control-plane
k8s-worker1     Ready    <none>
k8s-worker2     Ready    <none>
 
## Step 2 — Install Helm (Recommended)
Run ONLY on control plane.
Install:
•	Helm 
Download Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
Verify:
helm version
 
## Step 3 — Add NGINX Ingress Helm Repository
Run ONLY on control plane.
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
Update repos:
helm repo update
 
## Step 4 — Create Namespace
Run ONLY on control plane.
kubectl create namespace ingress-nginx
 
## Step 5 — Install NGINX Ingress Controller
Run ONLY on control plane.
For bare-metal/home-lab clusters, use NodePort service type.
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=NodePort
This deploys:
•	ingress controller deployment 
•	services 
•	admission webhooks 
•	RBAC 
•	configmaps 
 
## Step 6 — Verify Installation
Run on control plane:
kubectl get pods -n ingress-nginx
Expected:
NAME                                        READY   STATUS
ingress-nginx-controller-xxxxx              1/1     Running
 
## Step 7 — Verify Service
Run:

```bash
kubectl get svc -n ingress-nginx
```

Example Output:

```text
NAME                        TYPE       PORT(S)
ingress-nginx-controller    NodePort   80:32080/TCP,443:32443/TCP
```

### Important

- `32080` = HTTP NodePort
- `32443` = HTTPS NodePort
- Your port numbers may differ.
 
## Step 8 — Verify Where Ingress Is Running
kubectl get pods -n ingress-nginx -o wide
You should see it scheduled on a worker node.
Example:
NODE
k8s-worker1
 
## Step 9 — Deploy a Test Application
Run on control plane.
Create nginx deployment:
kubectl create deployment nginx --image=nginx
Scale:
kubectl scale deployment nginx --replicas=3
Expose service internally:
kubectl expose deployment nginx --port=80
 
## Step 10 — Create an Ingress Resource
Create file:
nano nginx-ingress.yaml
Paste:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```
Save file.
Apply:
kubectl apply -f nginx-ingress.yaml
 
## Step 11 — Verify Ingress
kubectl get ingress
Example:
NAME            CLASS   HOSTS
nginx-ingress   nginx   nginx.local
 
## Step 12 — Update Your LOCAL Machine Hosts File
On your laptop/desktop (NOT cluster node):
Linux/macOS
Edit:
sudo nano /etc/hosts
Windows
Edit:
C:\Windows\System32\drivers\etc\hosts
Add:
192.168.1.11 nginx.local
Use:
•	worker node IP 
•	or control plane IP 
 
## Step 13 — Access Application
Open browser:
http://nginx.local:<nodeport>
Example:
http://nginx.local:32080
You should see:
•	nginx welcome page 
 
## Step 14 — Understand Traffic Flow
Traffic path:
Browser
  ↓
Ingress Controller
  ↓
Ingress Rule
  ↓
Kubernetes Service
  ↓
Pods
This is the standard Kubernetes application routing model.
 
Helpful Commands
View ingress controller logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
 
Describe ingress
kubectl describe ingress nginx-ingress
 
Watch ingress events
kubectl get events -A --sort-by=.metadata.creationTimestamp
 

