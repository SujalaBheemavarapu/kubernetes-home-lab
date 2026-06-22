# 🚀 On-Premises Kubernetes Home Lab

> A complete hands-on Kubernetes cluster built from scratch on a MacBook Air M3 using Multipass Ubuntu VMs — simulating a real on-premises production environment.

**Author:** Sujala Bheemavarapu  
**Environment:** MacBook Air M3, 8GB RAM  
**Cluster:** 2-node on-premises Kubernetes v1.28.15  
**Completed:** June 2026

---

## 📋 Table of Contents

- [Why I Built This](#why-i-built-this)
- [Environment & Tools](#environment--tools)
- [Architecture](#architecture)
- [Week 1 — Building the Cluster](#week-1--building-the-cluster)
- [Week 2 — Deploying Apps & Monitoring](#week-2--deploying-apps--monitoring)
- [Week 3 — Operating the Cluster](#week-3--operating-the-cluster)
- [Errors Faced & How I Fixed Them](#errors-faced--how-i-fixed-them)
- [Key Learnings](#key-learnings)
- [CV Bullet Points](#cv-bullet-points)

---

## Why I Built This

I have 6+ years of experience managing Linux infrastructure on AWS. However, most of my experience was with cloud-managed services (EC2, ECS, RDS) rather than on-premises Kubernetes.

To bridge this gap, I built a real on-premises Kubernetes cluster from scratch on my MacBook — simulating exactly the kind of environment used by companies running self-managed Kubernetes on physical servers or on-premises VMs.

This project gave me hands-on experience with:
- The full Kubernetes setup process that cloud providers hide from you
- Real troubleshooting of network, storage, and scheduling issues
- Production-grade tools: Calico CNI, Helm, Prometheus, Grafana, RBAC
- The "You build it, you run it" ownership mindset

---

## Environment & Tools

| Component | Details | Why I chose it |
|---|---|---|
| **Hardware** | MacBook Air M3, 8GB RAM | Personal laptop |
| **VM Tool** | Multipass | VirtualBox doesn't work on Apple Silicon M3 chips. Multipass is made by Canonical (Ubuntu's creator) and works perfectly on M3 |
| **OS** | Ubuntu 26.04 LTS | Industry standard for Kubernetes nodes |
| **Kubernetes** | v1.28.15 | Stable LTS version |
| **Container Runtime** | Containerd | Default and recommended runtime for Kubernetes |
| **CNI Plugin** | Calico v3.26.0 | More stable than Flannel on Multipass — stores config in etcd so survives reboots |
| **Package Manager** | Helm v3.x | Industry standard for deploying Kubernetes applications |
| **Monitoring** | kube-prometheus-stack | Bundles Prometheus + Grafana + Alertmanager in one Helm chart |

---

## Architecture

```
MacBook Air M3 (Host Machine)
│
├── k8s-master VM (2 CPU, 4GB RAM, 20GB disk)
│   IP: 192.168.252.2
│   │
│   ├── Control Plane Components (brain of cluster):
│   │   ├── kube-apiserver      → receives all kubectl commands
│   │   ├── etcd                → database storing all cluster state
│   │   ├── kube-scheduler      → decides which node runs which pod
│   │   └── kube-controller-manager → ensures desired state matches reality
│   │
│   ├── Networking:
│   │   └── calico-node         → manages pod network on this node
│   │
│   └── Workloads (since worker was unstable):
│       ├── Prometheus          → collects and stores metrics
│       ├── Grafana             → visualises metrics as dashboards
│       ├── Alertmanager        → sends alerts when issues detected
│       └── nginx deployment    → sample web application (2 replicas)
│
└── k8s-worker1 VM (2 CPU, 2GB RAM, 20GB disk)
    IP: 192.168.252.x
    │
    ├── Worker Components:
    │   ├── kubelet             → manages pods on this node
    │   ├── kube-proxy          → handles network routing
    │   └── containerd          → actually runs containers
    │
    └── calico-node             → manages pod network on this node
```

**Three networks in the cluster:**
```
Node network:    192.168.252.0/24  → VMs talk to each other and Mac
Pod network:     10.244.0.0/16    → pods talk to each other (Calico)
Service network: 10.96.0.0/12    → Kubernetes services (ClusterIP)
```

---

## Week 1 — Building the Cluster

### Step 1 — Install tools on Mac

**What we're doing:**
Before creating any VMs, we need three tools on the Mac:
- **Homebrew** — Mac's package manager (like `apt` on Ubuntu)
- **Multipass** — creates and manages Ubuntu VMs on Apple Silicon
- **kubectl** — command line tool to control Kubernetes

```bash
# Install Homebrew — Mac's package manager
# This lets us install developer tools with simple commands
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Multipass — our VM creation tool
# We use this instead of VirtualBox because VirtualBox
# doesn't work reliably on Apple M1/M2/M3 chips
brew install --cask multipass

# Install kubectl — the Kubernetes command line tool
# This lets us run commands like: kubectl get pods
brew install kubectl

# Verify everything installed correctly
multipass version
kubectl version --client
```

---

### Step 2 — Create Ubuntu VMs

**What we're doing:**
We create 2 virtual machines that will act as our "servers":
- `k8s-master` — the brain of the cluster (control plane)
- `k8s-worker1` — where our actual applications run

Think of these VMs as physical servers in a data centre — just running inside your MacBook instead.

**Why 4GB for master?**
The monitoring stack (Prometheus + Grafana + Alertmanager) needs significant memory. Initially I used 2GB which caused CrashLoopBackOff errors. Increasing to 4GB fixed this.

```bash
# Create master node with 4GB RAM
# --cpus 2       = 2 virtual CPU cores
# --memory 4G    = 4GB RAM
# --disk 20G     = 20GB storage
multipass launch --name k8s-master --cpus 2 --memory 4G --disk 20G

# Create worker node with 2GB RAM
# Worker only runs application pods — less memory needed
multipass launch --name k8s-worker1 --cpus 2 --memory 2G --disk 20G

# Verify both VMs are running with their IP addresses
multipass list

# Enter master VM — prompt changes to: ubuntu@k8s-master:~$
# This is like SSH-ing into a server
multipass shell k8s-master
```

---

### Step 3 — Install Containerd (Container Runtime)

**What we're doing:**
Kubernetes cannot run containers by itself — it needs a container runtime to actually start and stop containers. Containerd is that runtime.

**Analogy:**
```
Kubernetes = Restaurant manager (decides what to cook and where)
Containerd = Kitchen equipment (actually does the cooking)
Pods/containers = The dishes being cooked
```

Without Containerd, Kubernetes has nothing to run containers on.

**Run on BOTH master and worker VMs:**

```bash
# Step 1: Disable swap memory
# Kubernetes requires swap to be OFF
# Swap is when the OS uses disk as "slow RAM"
# K8s wants to manage memory itself — not share with swap
sudo swapoff -a
# Make it permanent across reboots:
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Step 2: Install Containerd
# apt update = refresh list of available packages (like refreshing App Store)
# apt install containerd = download and install Containerd
sudo apt update && sudo apt install -y containerd

# Step 3: Create config directory for Containerd settings
# mkdir -p = create folder, no error if already exists
sudo mkdir -p /etc/containerd

# Step 4: Generate default Containerd configuration
# containerd config default = generate default settings
# | tee = save those settings to a file
containerd config default | sudo tee /etc/containerd/config.toml

# Step 5: Enable SystemdCgroup
# Kubernetes and Containerd must use the same cgroup manager
# SystemdCgroup = true makes Containerd use systemd (same as K8s)
# Without this: they conflict and cluster won't work
# sed -i = find and replace inside the config file
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Step 6: Restart Containerd to apply new config
# Like restarting an app after changing settings
sudo systemctl restart containerd

# Step 7: Verify Containerd is running
# Look for: Active: active (running)
sudo systemctl status containerd --no-pager
```

---

### Step 4 — Install Kubernetes Components

**What we're doing:**
Installing three Kubernetes tools on both nodes:

| Tool | Role | Analogy |
|---|---|---|
| `kubeadm` | Sets up and joins cluster | Construction worker who builds the restaurant |
| `kubelet` | Manages pods on each node | Chef working in each kitchen |
| `kubectl` | Your command line control | Remote control / manager's walkie-talkie |

**Run on BOTH master and worker VMs:**

```bash
# Step 1: Install helper tools needed for secure downloads
# apt-transport-https = allows apt to download from https
# ca-certificates    = SSL certificates to trust secure sites
# curl               = download tool (like clicking download button)
# gpg                = verifies downloaded software is genuine
sudo apt install -y apt-transport-https ca-certificates curl gpg

# Step 2: Download Kubernetes's official security key
# This key proves the K8s software is genuine and not tampered with
# Think of it as Kubernetes's official ID card
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Step 3: Add Kubernetes download source
# Tells apt WHERE to find Kubernetes packages
# Like adding a new store to your shopping list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Step 4: Refresh package list
# Now apt knows about Kubernetes packages
sudo apt update

# Step 5: Install the three K8s tools
sudo apt install -y kubelet kubeadm kubectl

# Step 6: Lock versions — very important!
# apt-mark hold = freeze these at current version
# Never auto-upgrade them
# If master runs v1.28 and worker upgrades to v1.29
# → cluster breaks! Versions must always match
sudo apt-mark hold kubelet kubeadm kubectl

# Verify installation
kubeadm version
```

---

### Step 5 — Initialise the Cluster (Master Only)

**What we're doing:**
`kubeadm init` sets up the entire Kubernetes control plane on the master node — API server, etcd, scheduler, controller manager. This is the most important step that creates the cluster.

```bash
# First get the master VM's IP address
hostname -I | awk '{print $1}'
# Note this IP — you'll need it in the next command

# Initialise the Kubernetes cluster
# --apiserver-advertise-address = master's IP address
#   This tells all nodes where the API server lives
# --pod-network-cidr = IP range for pods
#   10.244.0.0/16 is the range Calico will use
sudo kubeadm init \
  --apiserver-advertise-address=192.168.252.2 \
  --pod-network-cidr=10.244.0.0/16

# IMPORTANT: Save the "kubeadm join" command printed at end!
# It looks like: kubeadm join 192.168.252.2:6443 --token xxx
# You'll need this to connect the worker node

# Set up kubectl access for your user (not root)
# Kubernetes config file stores cluster connection details
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify cluster is running
kubectl get nodes
# Master should show: NotReady (normal — no CNI yet)
```

---

### Step 6 — Install Calico CNI (Network Plugin)

**What we're doing:**
After `kubeadm init`, nodes show "NotReady" because there's no network plugin installed. Without a CNI (Container Network Interface) plugin, pods on different nodes can't communicate.

**Why Calico instead of Flannel?**

I initially tried Flannel but it broke every time VMs restarted because it stores its config in `/run/flannel/subnet.env` — a temporary directory cleared on reboot. Calico stores config in etcd (Kubernetes database) which persists across reboots.

```
Flannel  → config in /run/ (temporary, lost on reboot) ❌
Calico   → config in etcd  (permanent, survives reboot) ✅
```

```bash
# Install Calico network plugin
# This creates all the networking components Calico needs:
# - ServiceAccounts, ClusterRoles (permissions)
# - ConfigMaps (settings)
# - DaemonSet (runs calico-node on every node)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Wait 2 minutes then verify Calico pods are running
kubectl get pods -n kube-system | grep calico

# Check nodes — should now show Ready!
kubectl get nodes
```

Expected output:
```
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   5m    v1.28.15
```

---

### Step 7 — Join Worker Node

**What we're doing:**
The worker node needs to join the cluster so Kubernetes can schedule pods on it. We use the `kubeadm join` command printed during `kubeadm init`.

```bash
# On worker VM — paste the join command from kubeadm init:
sudo kubeadm join 192.168.252.2:6443 \
  --token <token-from-init> \
  --discovery-token-ca-cert-hash sha256:<hash-from-init>

# Verify both nodes are Ready (run on master):
kubectl get nodes
```

Expected output:
```
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   8m    v1.28.15
k8s-worker1   Ready    <none>          2m    v1.28.15
```

---

## Week 2 — Deploying Apps & Monitoring

### Step 1 — Install Helm

**What Helm is:**
Helm is the package manager for Kubernetes — like `apt` for Ubuntu but for K8s applications. Instead of writing 20+ YAML files manually, Helm bundles everything into a "chart" that deploys with one command.

```
Without Helm:              With Helm:
kubectl apply -f file1.yaml    helm install myapp
kubectl apply -f file2.yaml    Done! ✅
kubectl apply -f file3.yaml
... 20 more files
```

```bash
# Install Helm using the official install script
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

---

### Step 2 — Deploy Sample Application

**What we're doing:**
Deploying nginx as our sample application. This demonstrates how workloads are deployed and managed in Kubernetes.

```bash
# Create a namespace — like a folder to organise resources
# Keeps our app separate from monitoring and system components
kubectl create namespace myapp

# Create nginx deployment with 2 replicas
# --image=nginx  = use official nginx container image
# --replicas=2   = run 2 copies for high availability
kubectl create deployment nginx --image=nginx --replicas=2 -n myapp

# Expose nginx as a NodePort service
# NodePort makes it accessible from outside the cluster
# --port=80 = nginx listens on port 80
kubectl expose deployment nginx --port=80 --type=NodePort -n myapp

# Verify pods are running
kubectl get pods -n myapp
# Both pods should show: Running ✅
```

---

### Step 3 — Install Prometheus + Grafana Monitoring Stack

**What we're doing:**
Installing a complete monitoring solution using the `kube-prometheus-stack` Helm chart. This one chart installs:

| Component | Purpose |
|---|---|
| **Prometheus** | Collects and stores metrics from all pods and nodes |
| **Grafana** | Beautiful dashboards to visualise those metrics |
| **Alertmanager** | Sends alerts when something goes wrong |
| **Node Exporter** | Collects CPU/RAM/disk metrics from each node |
| **Kube State Metrics** | Collects Kubernetes object metrics (pod status etc) |

**How they connect:**
```
Node Exporter → collects node metrics
Kube State Metrics → collects K8s object metrics
        ↓
Prometheus → scrapes and stores all metrics
        ↓
Grafana → reads Prometheus → shows dashboards
        ↓
Alertmanager → sends alerts when thresholds exceeded
```

```bash
# Add Prometheus community Helm repository
# Like adding a new app store to Helm
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

# Create dedicated namespace for monitoring
kubectl create namespace monitoring

# Install the full monitoring stack
# sujala-stack = name I gave to this installation (can be anything)
# --set flags  = override default memory limits for 8GB MacBook
helm install sujala-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.resources.requests.memory=256Mi \
  --set prometheus.prometheusSpec.resources.limits.memory=512Mi \
  --set grafana.resources.requests.memory=128Mi \
  --set grafana.resources.limits.memory=256Mi \
  --set prometheusOperator.resources.requests.memory=64Mi \
  --set prometheusOperator.resources.limits.memory=128Mi

# Wait 3-5 minutes then verify all pods are running
kubectl get pods -n monitoring
```

---

### Step 4 — Access Grafana Dashboard

**What we're doing:**
By default, Grafana is only accessible inside the cluster (ClusterIP). We change it to NodePort to access it from our Mac browser.

**ClusterIP vs NodePort:**
```
ClusterIP = internal phone (only inside cluster) ❌
NodePort  = mobile phone (accessible from outside) ✅
```

```bash
# Change Grafana service from ClusterIP to NodePort
# This "opens a door" from outside to Grafana inside the cluster
kubectl patch svc sujala-stack-grafana \
  -n monitoring \
  -p '{"spec": {"type": "NodePort"}}'

# Find which port Kubernetes assigned
kubectl get svc sujala-stack-grafana -n monitoring
# Look for: 80:3XXXX/TCP — the 3XXXX number is your port

# Get the Grafana admin password
kubectl --namespace monitoring get secrets sujala-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

Open Chrome and go to: `http://192.168.252.2:<your-port>`
- Username: `admin`
- Password: from command above

---

### Step 5 — Install Metrics Server

**What we're doing:**
`kubectl top nodes` and `kubectl top pods` require Metrics Server to collect real-time CPU and RAM usage. It's not installed by default.

```
kubectl top = needs Metrics Server
              (like Grafana needs Prometheus)

Prometheus   = stores ALL historical metrics
Metrics Server = only current metrics for kubectl top
```

```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for lab environment (self-signed certificates)
# --kubelet-insecure-tls = skip TLS verification
# In production: use proper certificates instead
kubectl patch deployment metrics-server \
  -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Wait 1 minute then verify
kubectl top nodes
# Should show CPU% and MEMORY% for each node
```

---

## Week 3 — Operating the Cluster

### RBAC — Role Based Access Control

**What RBAC is:**
RBAC controls who can do what in Kubernetes — just like AWS IAM but for K8s.

**Why it matters:**
```
Without RBAC:
Everyone has access to everything!
Junior developer can accidentally:
→ Delete production database ❌
→ Stop all running pods ❌
→ Bring down entire cluster 😱

With RBAC:
Junior dev  → read only ✅
Senior dev  → deploy apps ✅
SysOps      → full access ✅
```

**AWS comparison:**
```
Kubernetes ServiceAccount  =  AWS IAM User
Kubernetes Role            =  AWS IAM Policy
Kubernetes RoleBinding     =  Attach Policy to User
```

```bash
# Step 1: Create a ServiceAccount (user identity in K8s)
# Like creating an IAM user in AWS
# K8s doesn't have "human users" — it uses ServiceAccounts
kubectl create serviceaccount dev-user -n myapp

# Step 2: Create a Role (set of permissions)
# --verb=get,list,watch = read-only actions (cannot create/delete)
# --resource=pods,services = what objects this applies to
# This is like an IAM policy saying "ReadOnly on pods and services"
kubectl create role dev-reader \
  --verb=get,list,watch \
  --resource=pods,services \
  -n myapp

# Step 3: Bind the Role to the ServiceAccount
# RoleBinding = "give dev-user the dev-reader permissions"
# Like attaching an IAM policy to an IAM user
kubectl create rolebinding dev-reader-binding \
  --role=dev-reader \
  --serviceaccount=myapp:dev-user \
  -n myapp

# Step 4: Test that it works correctly
# kubectl auth can-i = check if a user has permission
# --as = impersonate this user for the check

# Should return: yes ✅ (dev-user can READ pods)
kubectl auth can-i get pods \
  --as=system:serviceaccount:myapp:dev-user -n myapp

# Should return: no ✅ (dev-user CANNOT delete pods)
kubectl auth can-i delete pods \
  --as=system:serviceaccount:myapp:dev-user -n myapp
```

---

### Rolling Updates & Rollback

**What rolling updates are:**
When deploying a new version, Kubernetes replaces pods one at a time instead of all at once — ensuring the application stays available throughout the update.

```
Without rolling update:       With rolling update:
Stop all old pods             Replace pod 1 (service still up)
↓ DOWNTIME 😱                Replace pod 2 (service still up)
Start all new pods            All pods updated ✅
                              Zero downtime ✅
```

```bash
# Update nginx to version 1.25
# Kubernetes will replace pods one by one
kubectl set image deployment/nginx \
  nginx=nginx:1.25 -n myapp

# Watch the rolling update happen in real time
# Shows progress: "1 out of 2 replicas updated..."
kubectl rollout status deployment/nginx -n myapp

# Check new pods are running
kubectl get pods -n myapp

# --- Simulate a bad deployment ---
# Use an invalid image version to trigger failure
kubectl set image deployment/nginx \
  nginx=nginx:invalid-version -n myapp

# Kubernetes detects new pods are failing
# Old pods STAY RUNNING to protect service availability
kubectl get pods -n myapp
# Shows: new pod = ImagePullBackOff ❌, old pods = Running ✅

# Rollback to previous working version
# Like Ctrl+Z for Kubernetes deployments
kubectl rollout undo deployment/nginx -n myapp

# Verify recovery — all pods running again
kubectl get pods -n myapp
```

---

### Persistent Volume Claims (PVCs)

**What PVCs solve:**
By default, pod storage is temporary — when a pod dies, all its data is lost. PVCs provide permanent storage that survives pod restarts and deletions.

**Real world impact:**
```
Database without PVC:
Pod crashes → ALL DATA GONE 😱

Database with PVC:
Pod crashes → PVC keeps data safe ✅
Pod restarts → connects to PVC → data back ✅
```

**AWS comparison:**
```
PersistentVolume (PV)   =  EBS Volume (the actual disk)
PersistentVolumeClaim   =  Mounting EBS to EC2 (the request)
```

**Why separate PV and PVC?**
```
SysOps team creates PVs  → they know storage infrastructure
Developers create PVCs   → they just say "I need X GB"
Kubernetes matches them  → hotel receptionist finding a room
```

```bash
# Step 1: Create PersistentVolume (the actual storage)
# hostPath = stores data on the node's filesystem at /tmp/my-pv-data
# In production: would use NFS, Ceph, or cloud storage instead
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi          # 1GB of storage
  accessModes:
    - ReadWriteOnce       # Only one pod can write at a time
  hostPath:
    path: /tmp/my-pv-data # Where data is stored on the node
EOF

# Step 2: Create PersistentVolumeClaim (request for storage)
# The application says "I need 1GB of ReadWriteOnce storage"
# Kubernetes finds a matching PV and binds them together
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: myapp
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Verify PVC found and bound to PV
kubectl get pvc -n myapp
# STATUS should show: Bound ✅

# Step 3: Create a pod that uses the PVC
# volumeMounts = mount the PVC at /data inside the container
# volumes = reference to the PVC we created
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-test-pod
  namespace: myapp
spec:
  containers:
  - name: test-container
    image: nginx
    volumeMounts:
    - mountPath: /data    # inside the container
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc   # use our PVC
EOF

# Step 4: Write data to the persistent volume
kubectl exec pvc-test-pod -n myapp -- \
  sh -c "echo 'Sujala K8s Lab Data' > /data/test.txt"

# Step 5: Read data back — should work
kubectl exec pvc-test-pod -n myapp -- cat /data/test.txt
# Output: Sujala K8s Lab Data ✅

# Step 6: DELETE the pod — data should survive!
kubectl delete pod pvc-test-pod -n myapp

# Step 7: Recreate the pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-test-pod
  namespace: myapp
spec:
  containers:
  - name: test-container
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc
EOF

# Step 8: Read data again — still there after pod deletion!
kubectl exec pvc-test-pod -n myapp -- cat /data/test.txt
# Output: Sujala K8s Lab Data ✅ (data survived!)
```

---

### Troubleshooting Commands

**Daily health check routine:**

```bash
# 1. Check all nodes are healthy
# Look for: STATUS = Ready ✅
kubectl get nodes

# 2. Find all non-running pods across entire cluster
# grep -v Running = show everything EXCEPT Running pods
# Anything appearing here needs investigation
kubectl get pods --all-namespaces | grep -v Running

# 3. Check resource usage on nodes
# Watch for: CPU > 80% or Memory > 85% = take action!
kubectl top nodes

# 4. Check resource usage per pod
kubectl top pods -n myapp

# 5. Deep dive into a specific pod
# Most important section: EVENTS at the bottom
# Normal  = healthy ✅
# Warning = needs investigation ❌
kubectl describe pod <pod-name> -n myapp

# 6. Check application logs
# Look for: [error] or [warn] level messages
kubectl logs <pod-name> -n myapp

# 7. Follow logs in real time (like tail -f)
# Very useful during incident response
kubectl logs <pod-name> -n myapp -f

# 8. See logs from before a crash
# --previous = logs from the crashed container
kubectl logs <pod-name> -n myapp --previous

# 9. Check cluster events sorted by time
# grep Warning = show only problems
kubectl get events --all-namespaces \
  --sort-by='.lastTimestamp' | grep Warning

# 10. Get inside a running container
# Like SSH-ing into a server — for live debugging
kubectl exec -it <pod-name> -n myapp -- /bin/bash
# Type 'exit' to come back out
```

**How to read Events output:**
```
TYPE     REASON              MESSAGE
Normal   Scheduled    ✅     pod assigned to node
Normal   Pulling      ✅     downloading image
Normal   Pulled       ✅     image downloaded
Normal   Created      ✅     container created
Normal   Started      ✅     container started

Warning  FailedScheduling ❌  no node available
Warning  BackOff          ❌  container keeps crashing
Warning  OOMKilling       ❌  out of memory
Warning  FailedMount      ❌  PVC not found
Warning  ImagePullBackOff ❌  image doesn't exist
```

---

## Errors Faced & How I Fixed Them

### ❌ Error 1 — Flannel subnet.env not found

**Error message:**
```
Failed to create pod sandbox: plugin type="flannel" failed (add):
failed to load flannel subnet.env file:
open /run/flannel/subnet.env: no such file or directory
```

**What happened:**
Pods were stuck in `ContainerCreating` for over 30 minutes. Every new pod failed to start with this error. The issue happened after every VM restart.

**Root cause:**
Flannel (the network plugin) stores its subnet configuration in `/run/flannel/subnet.env`. The `/run` directory is temporary storage — like RAM — it gets completely cleared every time the VM reboots. So every restart wiped Flannel's config and broke pod networking.

**What I tried first:**
- Manually creating the subnet.env file
- Reinstalling Flannel
- Force deleting stuck pods

**Final solution — switch to Calico:**
Calico stores its configuration in etcd (Kubernetes's own database) which persists across reboots. This completely solved the problem.

```bash
# Remove Flannel completely
kubectl delete -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml \
  --force --grace-period=0
kubectl delete namespace kube-flannel --force --grace-period=0

# Install Calico instead
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Verify Calico running
kubectl get pods -n kube-system | grep calico
```

**Lesson learned:**
Always check CNI plugin compatibility with your specific environment. Calico is better for on-premises and lab environments because it uses etcd for storage. Flannel is simpler but has this /run directory limitation.

---

### ❌ Error 2 — Namespace stuck in Terminating forever

**Error message:**
```
Error from server (Forbidden): unable to create new content
in namespace kube-flannel because it is being terminated
```

**What happened:**
The `kube-flannel` namespace showed `Terminating` status for 2 days and 21 hours — never completing deletion. Any attempt to reinstall Flannel failed because the namespace was stuck.

**Root cause:**
Kubernetes namespaces have "finalizers" — a checklist that must be completed before deletion can finish. When the worker node was gone, Kubernetes couldn't complete the checklist (couldn't verify all pods were removed from the missing node), so deletion hung forever.

```
Finalizer checklist:
✅ Remove services
✅ Remove configmaps
⏳ Remove pods from worker... (worker is gone, stuck here forever)
```

**Solution:**
Patch the namespace to remove all finalizers — skipping the stuck checklist:

```bash
# Remove finalizers from the stuck namespace
kubectl get namespace kube-flannel -o json | \
  tr -d "\n" | \
  sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/" | \
  kubectl replace --raw /api/v1/namespaces/kube-flannel/finalize -f -

# Verify it's gone
kubectl get namespace kube-flannel
# Should show: Error from server (NotFound) ✅
```

**Lesson learned:**
Finalizers protect against accidental deletion but create stuck states when nodes are missing. The `--force --grace-period=0` flag handles stuck pods. For stuck namespaces, finalizer patching via the raw API is needed.

---

### ❌ Error 3 — Pods stuck in Pending after worker node failure

**Error message:**
```
0/1 nodes are available:
1 node(s) had untolerated taint
{node-role.kubernetes.io/control-plane: NoSchedule}
```

**What happened:**
After worker node went down, all new pods showed `Pending` status — never getting scheduled anywhere.

**Root cause:**
Kubernetes master nodes have a special "taint" called `NoSchedule` applied by default. This prevents normal application pods from running on master — protecting the control plane from resource contention.

With worker gone, pods had nowhere to go:
```
Master → "Staff Only" sign = pods not allowed here
Worker → gone = no room anywhere
Result → pods wait forever in Pending
```

**Solution:**
Temporarily remove the NoSchedule taint from master to allow pod scheduling:

```bash
# Remove NoSchedule taint from master
# The - at the end means REMOVE (without - it would ADD the taint)
kubectl taint nodes k8s-master \
  node-role.kubernetes.io/control-plane:NoSchedule-

# Pods can now schedule on master
# Verify pods start running
kubectl get pods -n monitoring
```

**Lesson learned:**
In production with multiple worker nodes, this wouldn't happen — pods reschedule to healthy workers automatically. The NoSchedule taint is a best practice that should be restored once the worker is back. This incident showed the importance of having multiple worker nodes for resilience.

---

### ❌ Error 4 — Grafana and Prometheus CrashLoopBackOff

**Error message:**
```
sujala-stack-grafana-xxx          0/3  CrashLoopBackOff
prometheus-sujala-stack-xxx       0/2  CrashLoopBackOff
```

**What happened:**
After installing the monitoring stack, most pods kept crashing and restarting in a loop. Even after waiting 30+ minutes, they never stabilised.

**Root cause:**
Initial VM had only 2GB RAM. Running Kubernetes itself plus the full monitoring stack (Prometheus + Grafana + Alertmanager + Node Exporter + kube-state-metrics) required more memory than available.

```bash
# Evidence: only 54MB free!
free -h
# Mem: 1.9Gi total, 1.2Gi used, 54Mi free
```

With only 54MB free, pods couldn't get enough memory to start — causing them to crash immediately.

**Solution — two fixes:**

1. Increase master VM memory:
```bash
multipass stop k8s-master
multipass set local.k8s-master.memory=4G
multipass start k8s-master
# Now 2.9GB available — plenty of room
```

2. Set memory limits in Helm installation:
```bash
helm install sujala-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.resources.limits.memory=512Mi \
  --set grafana.resources.limits.memory=256Mi \
  --set prometheusOperator.resources.limits.memory=128Mi
```

**Lesson learned:**
Always check available resources before installing monitoring stacks. Use `free -h` and `kubectl top nodes` to monitor memory. Resource limits in Helm values prevent pods from consuming all available memory and starving other components.

---

### ❌ Error 5 — kubectl top: Metrics API not available

**Error message:**
```
error: Metrics API not available
```

**What happened:**
Running `kubectl top nodes` to check resource usage returned this error immediately.

**Root cause:**
`kubectl top` requires Metrics Server — a separate component that collects real-time CPU and memory data. It is NOT installed by default in Kubernetes.

```
kubectl top nodes → asks Metrics API for data
Metrics API       → doesn't exist yet!
Result            → error
```

**Solution:**
```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For lab/self-signed certs — add insecure TLS flag
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Now works!
kubectl top nodes
# NAME         CPU(cores)  CPU%  MEMORY(bytes)  MEMORY%
# k8s-master   497m        24%   2251Mi         59%
```

**Lesson learned:**
`--kubelet-insecure-tls` is only for lab environments with self-signed certificates. In production, proper TLS certificates should be configured for kubelet and Metrics Server.

---

### ❌ Error 6 — Worker node stuck in Starting/Restarting

**Symptom:**
```
k8s-worker1   Restarting   --   Ubuntu 26.04 LTS
```
Worker node stayed in `Restarting` or `Starting` state for 10+ minutes, never becoming `Running`.

**Root cause:**
Known instability with Multipass on Apple M3 after multiple VM restarts and resource pressure. The VM state became corrupted.

**Solution:**
Delete and recreate the worker node completely — this is the same process used in production when a physical server fails:

```bash
# Force stop the stuck VM
multipass stop --force k8s-worker1

# Permanently delete it (--purge = no recovery possible)
multipass delete --purge k8s-worker1

# Create fresh worker VM
multipass launch --name k8s-worker1 --cpus 2 --memory 2G --disk 20G

# Reinstall Containerd and Kubernetes on new VM
# Then rejoin the cluster with kubeadm join
```

**Lesson learned:**
Node replacement is a normal operational task in production Kubernetes. The process mirrors real-world scenarios — drain the node, remove it from the cluster, provision a fresh replacement, and rejoin. This is exactly the kind of incident a SysOps engineer handles during on-call duties.

---

## Key Learnings

### Technical Learnings:

```
Kubernetes Architecture:
✅ How control plane components work together
   (API server, etcd, scheduler, controller manager)
✅ Role of kubelet and kube-proxy on worker nodes
✅ How pods get scheduled across nodes

Networking:
✅ CNI plugins and how they provide pod networking
✅ Why Calico is better than Flannel for on-premises
✅ Difference between ClusterIP, NodePort, LoadBalancer
✅ Three separate networks in a K8s cluster

Storage:
✅ Why default pod storage is temporary
✅ How PVs and PVCs solve stateful storage
✅ Data persistence across pod deletion

Operations:
✅ Rolling updates — zero downtime deployments
✅ Rollback — recovering from bad deployments
✅ RBAC — principle of least privilege in Kubernetes
✅ Node failure recovery and replacement
✅ Troubleshooting with describe, logs, events, top
```

### Operational Learnings:

```
Real incident experience:
✅ Diagnosed CNI failures using kubectl describe events
✅ Recovered from node failures using taint management
✅ Fixed stuck namespaces using finalizer patching
✅ Resolved memory pressure by tuning resource limits
✅ Replaced corrupted worker node following
   production-standard procedures

Mindset:
✅ "You build it, you run it" ownership
✅ Always check resource availability before deploying
✅ Document every error and fix for future reference
✅ Understanding root cause, not just applying fixes
```

---

## CV Bullet Points

> These bullet points are based on real work done in this project:

- Built and operated a 2-node on-premises Kubernetes cluster (1 control-plane + 1 worker) using kubeadm on Ubuntu VMs via Multipass — directly mirroring production on-premises Kubernetes setup

- Diagnosed and resolved persistent CNI networking failures by switching from Flannel to Calico after identifying that Flannel's subnet.env configuration was lost on every VM reboot — Calico's etcd-backed storage solved the issue permanently

- Deployed full Prometheus + Grafana monitoring stack using Helm (kube-prometheus-stack); configured resource limits to run within 8GB RAM constraints; accessed live Kubernetes cluster dashboards showing real-time CPU and memory metrics

- Implemented RBAC following principle of least privilege — created ServiceAccounts, Roles with specific verb/resource combinations, and RoleBindings; verified permissions using kubectl auth can-i

- Performed zero-downtime rolling deployments and rollbacks using kubectl set image and kubectl rollout undo; demonstrated Kubernetes's built-in safety mechanism of keeping old pods running during failed deployments

- Configured Persistent Volume Claims for stateful storage; proved data persistence by writing data, deleting pod, recreating pod, and confirming data survived — equivalent to EBS volume management in AWS

- Performed production-standard node failure recovery: diagnosed stuck Terminating pods, removed NoSchedule taint from master to restore scheduling, patched namespace finalizers to clear stuck deletions, and replaced corrupted worker node

- Used full troubleshooting toolkit: kubectl describe (Events section), logs with --previous flag, top for resource monitoring, and events sorted by timestamp to diagnose and resolve cluster issues

---

## Tools & Technologies

```
Container Orchestration:  Kubernetes v1.28, kubeadm, kubelet, kubectl
Container Runtime:        Containerd v2.2.2
Network Plugin:           Calico v3.26.0
Package Management:       Helm v3.x
Monitoring:               Prometheus, Grafana, Alertmanager, Metrics Server
Infrastructure:           Multipass, Ubuntu 26.04 LTS
Platform:                 MacBook Air M3, Apple Silicon
```

---

*This project was completed as part of hands-on preparation for on-premises Kubernetes SysOps/DevOps engineering roles.*
*All errors documented are real issues encountered and resolved during the project.*
