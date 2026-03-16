# Kubernetes Mastery Guide
### From HPC Admin to K8s Super User

> **Your advantage:** You already know Slurm, nodes, resource scheduling, and containers. This guide maps those concepts to Kubernetes and takes you all the way to expert.

---

## Table of Contents

0. [Prerequisites, System Requirements & Installation](#0-prerequisites-system-requirements--installation)
1. [Mental Model: Slurm → Kubernetes](#1-mental-model-slurm--kubernetes)
2. [Phase 1: Core Architecture](#2-phase-1-core-architecture)
3. [Phase 2: kubectl Fluency](#3-phase-2-kubectl-fluency)
4. [Phase 3: Workload Types Deep Dive](#4-phase-3-workload-types-deep-dive)
5. [Phase 4: Networking & Services](#5-phase-4-networking--services)
6. [Phase 5: Storage](#6-phase-5-storage)
7. [Phase 6: Configuration & Secrets](#7-phase-6-configuration--secrets)
8. [Phase 7: RBAC & Security](#8-phase-7-rbac--security)
9. [Phase 8: Helm & Package Management](#9-phase-8-helm--package-management)
10. [Phase 9: Autoscaling](#10-phase-9-autoscaling)
11. [Phase 10: GPU & HPC Workloads](#11-phase-10-gpu--hpc-workloads)
12. [Phase 11: ML Pipelines with Kubeflow](#12-phase-11-ml-pipelines-with-kubeflow)
13. [Phase 12: Observability](#13-phase-12-observability)
14. [Phase 13: Advanced — CRDs, Operators & GitOps](#14-phase-13-advanced--crds-operators--gitops)
15. [Hands-On Labs](#15-hands-on-labs)
16. [Cheat Sheet](#16-cheat-sheet)
17. [Resources & Certification Path](#17-resources--certification-path)

---

## 0. Prerequisites, System Requirements & Installation

### What You Need Before Starting

Since you're coming from HPC/Slurm, you likely already have most of these. Here's a quick checklist:

- **Container runtime**: Docker Engine 26+ or Docker Desktop 4.37+ (you probably already have this)
- **A terminal** you're comfortable with (bash/zsh)
- **Basic YAML familiarity** — K8s is configured entirely in YAML
- **Git** — for version-controlling your manifests and later for GitOps

### System Requirements Overview

The requirements differ based on whether you're learning locally, running a dev cluster, or deploying production.

| Environment | CPUs | RAM | Disk | Notes |
|---|---|---|---|---|
| **minikube** (learning) | 2+ (4 recommended) | 2 GB min (4-8 GB recommended) | 20 GB+ | Single-node; runs in VM or container |
| **kind** (testing/CI) | 2+ | 2 GB min per node | 10 GB+ | Runs K8s nodes as Docker containers |
| **k3s / k3d** (lightweight) | 1+ | 512 MB min (2 GB recommended) | 5 GB+ | Lightweight K8s; great for edge/dev |
| **kubeadm** (production) | 2+ per node | 2 GB+ per node | 50 GB+ | Full cluster; need separate control plane + workers |
| **Managed cloud** (EKS/GKE/AKS) | Varies | Varies | Varies | Cloud provider handles control plane |

> **Recommendation for learning:** Start with **minikube** on at least 4 CPUs and 8 GB RAM. This gives you room to run real workloads and GPU simulations without everything grinding to a halt. If your machine is constrained, **k3d** is the lightest option.

### Required Ports (for kubeadm / bare-metal clusters)

| Component | Port Range | Purpose |
|---|---|---|
| kube-apiserver | 6443 | API server (all communication) |
| etcd | 2379-2380 | Cluster state store |
| kubelet | 10250 | Node agent API |
| kube-scheduler | 10259 | Scheduling decisions |
| controller-manager | 10257 | Control loops |
| NodePort Services | 30000-32767 | External access to services |

### Installation: Choose Your Path

#### Path A: minikube (Recommended for Learning)

Best for: Getting started, following tutorials, CKA/CKAD exam prep.

```bash
# ── Step 1: Install Docker (if not already installed) ──

# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# Log out and back in for group membership to take effect

# Verify Docker
docker --version
docker run hello-world

# ── Step 2: Install minikube ──

# Linux (x86_64)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

# macOS (Homebrew)
brew install minikube

# Windows (PowerShell as Admin)
# Download from https://minikube.sigs.k8s.io/docs/start/

# Verify
minikube version

# ── Step 3: Install kubectl ──

# Linux (x86_64) — downloads latest stable
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl

# macOS (Homebrew)
brew install kubectl

# Verify
kubectl version --client

# ── Step 4: Start your cluster ──

# Basic start (good for learning)
minikube start --cpus=4 --memory=8g --disk-size=40g

# With specific Kubernetes version
minikube start --cpus=4 --memory=8g --kubernetes-version=v1.35.0

# With GPU support (requires NVIDIA Container Toolkit on host)
minikube start --cpus=4 --memory=8g --gpus=all --driver=docker

# ── Step 5: Verify everything works ──

kubectl cluster-info
kubectl get nodes
minikube status

# Optional: Enable useful addons
minikube addons enable metrics-server    # For 'kubectl top'
minikube addons enable dashboard         # Web UI
minikube addons enable ingress           # Ingress controller

# Open the dashboard (runs in browser)
minikube dashboard
```

#### Path B: kind (Kubernetes IN Docker)

Best for: CI/CD testing, multi-node clusters locally, faster startup than minikube.

```bash
# ── Install kind ──

# Linux
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# macOS
brew install kind

# ── Create a cluster ──

# Simple single-node
kind create cluster --name dev

# Multi-node cluster (create a config file first)
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
kind create cluster --name multi-node --config kind-config.yaml

# Verify
kubectl get nodes
# NAME                       STATUS   ROLES           AGE   VERSION
# multi-node-control-plane   Ready    control-plane   1m    v1.35.0
# multi-node-worker          Ready    <none>          1m    v1.35.0
# multi-node-worker2         Ready    <none>          1m    v1.35.0
# multi-node-worker3         Ready    <none>          1m    v1.35.0

# Delete when done
kind delete cluster --name multi-node
```

#### Path C: k3s / k3d (Lightweight)

Best for: Edge computing, resource-constrained machines, quick experiments.

```bash
# ── k3s (direct install — runs as a system service) ──
curl -sfL https://get.k3s.io | sh -

# kubectl is automatically installed; kubeconfig at /etc/rancher/k3s/k3s.yaml
sudo k3s kubectl get nodes

# To use regular kubectl:
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
kubectl get nodes

# Uninstall
/usr/local/bin/k3s-uninstall.sh

# ── k3d (k3s in Docker — easier to manage multiple clusters) ──

# Install k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Create a cluster
k3d cluster create dev --agents 2    # 1 server + 2 worker nodes

# With port mapping (access services from host)
k3d cluster create dev \
  --agents 2 \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer"

# Verify
kubectl get nodes

# Delete
k3d cluster delete dev
```

#### Path D: kubeadm (Production / Bare-Metal)

Best for: Production clusters, bare-metal HPC infrastructure, learning cluster administration deeply.

This is the closest to how you'd set up a Slurm cluster — you configure each node individually.

```bash
# ══════════════════════════════════════════════════
# Run on ALL nodes (control plane + workers)
# ══════════════════════════════════════════════════

# Disable swap (K8s requirement)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Set required sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
# Enable SystemdCgroup in containerd config:
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl   # Prevent auto-upgrade

# ══════════════════════════════════════════════════
# Run on CONTROL PLANE node only
# ══════════════════════════════════════════════════

# Initialize the cluster
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --control-plane-endpoint=<CONTROL_PLANE_IP>:6443

# Set up kubeconfig for your user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install a CNI (network plugin) — using Calico here
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

# Get the join command (save this for worker nodes!)
kubeadm token create --print-join-command
# Output: kubeadm join <IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# ══════════════════════════════════════════════════
# Run on EACH WORKER node
# ══════════════════════════════════════════════════

# Paste the join command from above
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# ══════════════════════════════════════════════════
# Back on CONTROL PLANE — verify
# ══════════════════════════════════════════════════

kubectl get nodes
# NAME        STATUS   ROLES           AGE   VERSION
# master-01   Ready    control-plane   5m    v1.35.0
# worker-01   Ready    <none>          2m    v1.35.0
# worker-02   Ready    <none>          2m    v1.35.0

# Label worker nodes (like setting up Slurm partitions)
kubectl label node worker-01 node-role.kubernetes.io/worker=worker
kubectl label node worker-02 node-role.kubernetes.io/worker=worker
```

#### Path E: Managed Cloud Services

If you just want a cluster without managing infrastructure:

| Provider | Service | Quick Start |
|---|---|---|
| AWS | **EKS** | `eksctl create cluster --name dev --region us-west-2` |
| Google Cloud | **GKE** | `gcloud container clusters create dev --zone us-central1-a` |
| Azure | **AKS** | `az aks create --resource-group myRG --name dev` |
| DigitalOcean | **DOKS** | Via web console or `doctl` CLI |

> These handle the control plane for you — you only manage worker nodes and workloads. Great for production, but for learning the internals, use minikube or kubeadm.

### Install Helm (You'll Need This Soon)

```bash
# Linux / macOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS (Homebrew)
brew install helm

# Verify
helm version
```

### Post-Installation Checklist

After any installation method, verify your environment:

```bash
# 1. Cluster is running
kubectl cluster-info

# 2. Nodes are Ready
kubectl get nodes -o wide

# 3. System pods are healthy
kubectl get pods -n kube-system

# 4. Can create a pod
kubectl run test --image=nginx --restart=Never
kubectl get pod test
kubectl delete pod test

# 5. DNS is working
kubectl run dns-test --image=busybox --restart=Never \
  --command -- nslookup kubernetes.default
kubectl logs dns-test
kubectl delete pod dns-test

# 6. kubectl autocomplete is set up (huge productivity boost)
# bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc

# zsh
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'alias k=kubectl' >> ~/.zshrc
source ~/.zshrc
```

### Which Path Should You Choose?

```
Are you learning K8s for the first time?
├── Yes → minikube (Path A)
│         Easiest setup, great addon ecosystem, CKA exam uses similar env
│
├── Need multi-node locally?
│   ├── Yes, and lightweight → k3d (Path C)
│   └── Yes, full-featured → kind (Path B)
│
├── Building a production bare-metal cluster?
│   └── kubeadm (Path D)
│       (Most similar to your Slurm cluster setup experience)
│
└── Just want workloads running ASAP?
    └── Managed cloud (Path E) — EKS, GKE, or AKS
```

---

## 1. Mental Model: Slurm → Kubernetes

This is the single most useful table for your transition. Once this clicks, everything else accelerates.

| Slurm Concept | Kubernetes Equivalent | Notes |
|---|---|---|
| `slurmctld` (controller) | **kube-apiserver + controller-manager + scheduler** | K8s splits the control plane into components |
| `slurmd` (node daemon) | **kubelet** | Runs on every worker node |
| Node / compute node | **Node** | Same concept, different management |
| Partition | **Namespace** (logical) or **Node Pool** (physical) | Namespaces isolate resources; node pools group similar hardware |
| Job (`sbatch`) | **Job** or **Pod** | Pod = smallest unit; Job = run-to-completion |
| Job Array | **Job with parallelism** | `spec.parallelism` and `spec.completions` |
| `squeue` | `kubectl get pods` | See what's running |
| `scontrol show job` | `kubectl describe pod` | Detailed status |
| `sinfo` | `kubectl get nodes` | Node status |
| `sacct` | Prometheus / metrics-server | Resource usage history |
| `--gres=gpu:1` | `resources.limits: nvidia.com/gpu: 1` | GPU requests |
| `--constraint` / features | **nodeSelector** / **nodeAffinity** | Place workloads on specific nodes |
| `--exclusive` | **Taints & Tolerations** | Reserve nodes for specific workloads |
| QOS / priority | **PriorityClass** | Preemption support built in |
| Module system (Lmod) | **Container images** | Each image bundles its own dependencies |
| Shared filesystem (Lustre/GPFS) | **PersistentVolume (PV)** | Can mount NFS, Ceph, cloud storage, etc. |

**Key mindset shift:** In Slurm, you submit jobs and wait. In Kubernetes, you declare desired state and the system continuously reconciles reality to match. This is called the **declarative model** — it's the most important concept in K8s.

---

## 2. Phase 1: Core Architecture

### Control Plane Components

```
┌─────────────────────── Control Plane ───────────────────────┐
│                                                              │
│  kube-apiserver     ← All communication goes through here    │
│       │                                                      │
│  etcd              ← Distributed key-value store (state DB)  │
│       │                                                      │
│  kube-scheduler    ← Decides which node runs each pod        │
│       │                                                      │
│  controller-manager ← Runs control loops (Deployment ctrl,   │
│                       ReplicaSet ctrl, Job ctrl, etc.)       │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌─────────── Worker Node ──────────┐
│                                   │
│  kubelet        ← Manages pods    │
│  kube-proxy     ← Network rules   │
│  container-runtime (containerd)   │
│  [pods running here]              │
│                                   │
└───────────────────────────────────┘
```

### The Core Objects

**Pod** — The atomic unit. One or more containers sharing network and storage. Like a single Slurm job step.

**ReplicaSet** — Ensures N copies of a pod are running. You rarely create these directly.

**Deployment** — Manages ReplicaSets. Handles rolling updates, rollbacks. This is what you use 90% of the time for stateless apps.

**Service** — Stable network endpoint for a set of pods. Pods come and go; Services give them a permanent address.

**Namespace** — Logical isolation boundary (like Slurm partitions but for access control and resource quotas).

### How Declarative State Works

```yaml
# You write this (desired state):
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3           # I want 3 copies
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx:1.25
        resources:
          requests:      # Minimum resources (like Slurm --mem, --cpus-per-task)
            cpu: "250m"  # 250 millicores = 0.25 CPU
            memory: "128Mi"
          limits:        # Maximum resources (hard ceiling)
            cpu: "500m"
            memory: "256Mi"
```

K8s then continuously ensures exactly 3 pods are running. If one dies, it spawns another. If you change the image tag, it performs a rolling update.

---

## 3. Phase 2: kubectl Fluency

`kubectl` is your primary interface — like `scontrol`, `squeue`, and `sbatch` rolled into one.

### Essential Commands (Muscle Memory)

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes -o wide            # Like sinfo
kubectl describe node <node-name>    # Like scontrol show node

# Working with pods (like squeue / scontrol show job)
kubectl get pods                     # All pods in current namespace
kubectl get pods -A                  # All pods, ALL namespaces
kubectl get pods -o wide             # Show node placement
kubectl get pods -w                  # Watch (live updates)
kubectl describe pod <name>          # Detailed info + events
kubectl logs <pod>                   # stdout/stderr (like slurm output files)
kubectl logs <pod> -f                # Follow (tail -f)
kubectl logs <pod> -c <container>    # Specific container in multi-container pod
kubectl logs <pod> --previous        # Logs from crashed container

# Interactive access (like ssh into a compute node)
kubectl exec -it <pod> -- /bin/bash
kubectl exec -it <pod> -c <container> -- /bin/sh

# Apply manifests (like sbatch)
kubectl apply -f deployment.yaml     # Create or update
kubectl delete -f deployment.yaml    # Remove everything in file

# Quick resource creation (for testing)
kubectl run test-pod --image=ubuntu --rm -it -- bash   # Ephemeral debug pod
kubectl create deployment nginx --image=nginx:1.25
kubectl expose deployment nginx --port=80 --type=ClusterIP

# Scaling
kubectl scale deployment my-app --replicas=5

# Rolling updates and rollbacks
kubectl set image deployment/my-app my-app=nginx:1.26
kubectl rollout status deployment/my-app
kubectl rollout history deployment/my-app
kubectl rollout undo deployment/my-app                  # Instant rollback!

# Resource usage (like sacct)
kubectl top nodes
kubectl top pods
```

### Power User kubectl Tips

```bash
# Aliases — put these in your .bashrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kaf='kubectl apply -f'
alias kex='kubectl exec -it'

# JSONPath for extracting specific fields
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.nvidia\.com/gpu}{"\n"}{end}'

# Sort and filter
kubectl get pods --sort-by='.status.startTime'
kubectl get pods --field-selector=status.phase=Running
kubectl get pods -l app=my-app       # Filter by label

# Dry run + diff (preview changes before applying)
kubectl apply -f manifest.yaml --dry-run=client -o yaml
kubectl diff -f manifest.yaml        # Show what would change

# Context management (switch between clusters/namespaces)
kubectl config get-contexts
kubectl config use-context my-cluster
kubectl config set-context --current --namespace=my-namespace

# Force delete stuck pods
kubectl delete pod <name> --grace-period=0 --force
```

### 🔬 Lab: kubectl Exploration

```bash
# 1. Set up a local cluster (pick one):
#    - minikube start --cpus=4 --memory=8g
#    - kind create cluster
#    - k3d cluster create mycluster

# 2. Deploy something
kubectl create deployment hello --image=nginx:1.25 --replicas=3

# 3. Explore it
kubectl get all
kubectl describe deployment hello
kubectl get pods -o wide
kubectl logs <one-of-the-pods>
kubectl exec -it <one-of-the-pods> -- cat /etc/nginx/nginx.conf

# 4. Break it and watch recovery
kubectl delete pod <one-of-the-pods>   # Watch: a new one appears!
kubectl get pods -w                     # See the replacement spawn

# 5. Scale and update
kubectl scale deployment hello --replicas=5
kubectl set image deployment/hello nginx=nginx:1.26
kubectl rollout status deployment/hello
```

---

## 4. Phase 3: Workload Types Deep Dive

### When to Use What

| Workload Type | Slurm Analogy | Use Case |
|---|---|---|
| **Deployment** | Long-running service | Web servers, APIs, microservices |
| **StatefulSet** | — | Databases, anything needing stable identity/storage |
| **DaemonSet** | Node-level daemons | Monitoring agents, log collectors, GPU drivers |
| **Job** | `sbatch` (single run) | Batch processing, data migration, one-off tasks |
| **CronJob** | `cron` + `sbatch` | Scheduled tasks, periodic cleanup |
| **ReplicaSet** | — | Managed by Deployments; rarely create directly |

### Job (Your Bread and Butter from HPC)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing
spec:
  parallelism: 4          # Like --array=1-4 (concurrent pods)
  completions: 20          # Total work items to complete
  backoffLimit: 3          # Retry failed pods up to 3 times
  activeDeadlineSeconds: 3600   # Kill after 1 hour (like --time)
  template:
    spec:
      restartPolicy: Never     # Jobs must be Never or OnFailure
      containers:
      - name: processor
        image: myregistry/processor:v1
        command: ["python", "process.py"]
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
            nvidia.com/gpu: 1   # Request a GPU
```

### StatefulSet (For Stateful Services)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:       # Each pod gets its own persistent volume
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

Key properties of StatefulSets: pods get stable names (`postgres-0`, `postgres-1`, `postgres-2`), ordered startup/shutdown, and each pod gets its own persistent storage.

### DaemonSet (Runs on Every Node)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      tolerations:            # Run on ALL nodes, including control plane
      - operator: Exists
      containers:
      - name: monitor
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
```

---

## 5. Phase 4: Networking & Services

### Service Types

```
┌─────────────────────────────────────────────────────┐
│                                                      │
│  ClusterIP (default)                                 │
│  → Internal only. Pods can reach it, outside can't.  │
│  → Like a private DNS name on your HPC cluster.      │
│                                                      │
│  NodePort                                            │
│  → Exposes on every node's IP at a static port.      │
│  → Range: 30000-32767. Good for dev/testing.         │
│                                                      │
│  LoadBalancer                                        │
│  → Provisions an external LB (cloud provider).       │
│  → The standard way to expose to the internet.       │
│                                                      │
│  ExternalName                                        │
│  → CNAME alias to an external service.               │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Service Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: ClusterIP
  selector:
    app: my-app              # Routes to pods with this label
  ports:
  - port: 80                 # Service port
    targetPort: 8080         # Container port
    protocol: TCP
```

### Ingress (HTTP/HTTPS Routing)

Ingress sits in front of Services and routes external HTTP traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
```

### Network Policies (Firewall Rules for Pods)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend       # Only frontend pods can talk to the API
    ports:
    - protocol: TCP
      port: 8080
```

### DNS in Kubernetes

Every Service gets a DNS name: `<service-name>.<namespace>.svc.cluster.local`

```bash
# From any pod, you can reach services by name:
curl http://my-app-service                          # Same namespace
curl http://my-app-service.production               # Cross-namespace
curl http://my-app-service.production.svc.cluster.local  # Fully qualified

# StatefulSet pods also get individual DNS:
# postgres-0.postgres.default.svc.cluster.local
```

---

## 6. Phase 5: Storage

### Storage Hierarchy

```
StorageClass          ← Defines HOW storage is provisioned (SSD, HDD, NFS...)
    │
PersistentVolume (PV) ← A piece of actual storage (like a disk)
    │
PersistentVolumeClaim (PVC) ← A request for storage by a pod
    │
Pod volumeMount       ← Where the storage appears inside the container
```

### Common Patterns

```yaml
# PersistentVolumeClaim — request storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: training-data
spec:
  accessModes:
  - ReadWriteOnce         # RWO = single node
  # - ReadWriteMany       # RWX = multiple nodes (NFS, Ceph, etc.)
  # - ReadOnlyMany        # ROX = read-only from many nodes
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 500Gi

---
# Pod using the PVC
apiVersion: v1
kind: Pod
metadata:
  name: training-job
spec:
  containers:
  - name: trainer
    image: myregistry/trainer:v1
    volumeMounts:
    - name: data
      mountPath: /data
    - name: scratch
      mountPath: /tmp/scratch
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: training-data
  - name: scratch
    emptyDir:              # Ephemeral — deleted when pod dies
      sizeLimit: 50Gi      # Can use memory: emptyDir: { medium: Memory }
```

### For HPC: Mounting Shared Filesystems

```yaml
# Mount an existing NFS share (like Lustre/GPFS access)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: shared-nfs
spec:
  capacity:
    storage: 10Ti
  accessModes:
  - ReadWriteMany
  nfs:
    server: nfs-server.example.com
    path: /exports/shared
  persistentVolumeReclaimPolicy: Retain
```

---

## 7. Phase 6: Configuration & Secrets

### ConfigMaps (Non-Sensitive Config)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      workers: 4
---
# Using in a pod
spec:
  containers:
  - name: app
    envFrom:                     # Load ALL keys as env vars
    - configMapRef:
        name: app-config
    # OR mount as files:
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config
```

### Secrets (Sensitive Data)

```bash
# Create from command line
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password='s3cur3p@ss!'

# Create from files
kubectl create secret tls my-tls \
  --cert=tls.crt \
  --key=tls.key
```

```yaml
# Using secrets in pods
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
    volumeMounts:
    - name: tls-certs
      mountPath: /etc/tls
      readOnly: true
  volumes:
  - name: tls-certs
    secret:
      secretName: my-tls
```

> **Important:** K8s Secrets are base64-encoded, not encrypted at rest by default. For production, enable encryption at rest or use external secret managers (Vault, AWS Secrets Manager, etc.).

---

## 8. Phase 7: RBAC & Security

### RBAC Model

```
User / ServiceAccount
    ↓ bound via
RoleBinding (namespace-scoped) or ClusterRoleBinding (cluster-wide)
    ↓ references
Role (namespace-scoped) or ClusterRole (cluster-wide)
    ↓ contains
Rules: apiGroups + resources + verbs
```

### Example: Read-Only Namespace Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-read
  namespace: production
subjects:
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Pod Security (Hardening Containers)

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    resources:
      limits:
        cpu: "1"
        memory: "512Mi"
```

---

## 9. Phase 8: Helm & Package Management

Helm = package manager for Kubernetes (like `apt` for Ubuntu or `conda` for Python).

### Core Concepts

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo prometheus

# Install a chart
helm install my-prometheus bitnami/kube-prometheus \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.retention=30d

# See what's installed
helm list -A

# Upgrade with new values
helm upgrade my-prometheus bitnami/kube-prometheus \
  --namespace monitoring \
  -f custom-values.yaml

# Rollback
helm rollback my-prometheus 1    # Revision number

# Uninstall
helm uninstall my-prometheus -n monitoring

# See what a chart would render (debug)
helm template my-release bitnami/kube-prometheus -f values.yaml
```

### Creating Your Own Chart

```bash
helm create my-app

my-app/
├── Chart.yaml          # Metadata
├── values.yaml         # Default configuration
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # Template functions
│   └── NOTES.txt       # Post-install message
└── charts/             # Dependencies
```

---

## 10. Phase 9: Autoscaling

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 min before scaling down
```

### Vertical Pod Autoscaler (VPA)

Adjusts CPU/memory requests per pod instead of changing replica count. Useful for workloads that can't scale horizontally.

### Cluster Autoscaler

Adds/removes nodes from the cluster. Works with cloud providers (EKS, GKE, AKS) to provision new VMs when pods can't be scheduled.

```
Pod unschedulable (no room) → Cluster Autoscaler adds node → Pod gets scheduled
Node underutilized → Cluster Autoscaler removes node → Saves cost
```

---

## 11. Phase 10: GPU & HPC Workloads

This is where your HPC background really pays off.

### NVIDIA Device Plugin Setup

```bash
# Install the NVIDIA device plugin (DaemonSet)
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.3/nvidia-device-plugin.yml

# Verify GPUs are detected
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.nvidia\.com/gpu}{"\n"}{end}'
```

### GPU Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training
spec:
  restartPolicy: Never
  containers:
  - name: trainer
    image: nvcr.io/nvidia/pytorch:24.01-py3
    command: ["python", "train.py"]
    resources:
      limits:
        nvidia.com/gpu: 2       # Request 2 GPUs
        cpu: "16"
        memory: "64Gi"
      requests:
        cpu: "8"
        memory: "32Gi"
    volumeMounts:
    - name: datasets
      mountPath: /data
    - name: dshm                # CRITICAL for PyTorch DataLoader
      mountPath: /dev/shm
  volumes:
  - name: datasets
    persistentVolumeClaim:
      claimName: training-datasets
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: "16Gi"         # Shared memory for DataLoader workers
```

### Node Affinity (Target Specific Hardware)

```yaml
# Like Slurm's --constraint or --nodelist
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nvidia.com/gpu.product
            operator: In
            values:
            - NVIDIA-H200           # Only schedule on H200 nodes
            - NVIDIA-H100-SXM5      # Or H100
  tolerations:
  - key: "gpu-type"
    operator: "Equal"
    value: "h200"
    effect: "NoSchedule"
```

### Taints and Tolerations (Reserve GPU Nodes)

```bash
# Taint GPU nodes so only GPU workloads land there (like Slurm partition restrictions)
kubectl taint nodes gpu-node-1 gpu-type=h200:NoSchedule

# Now only pods with matching tolerations can be scheduled there
```

### Multi-Node GPU Training (MPI-Style)

For distributed training across multiple nodes, use the MPI Operator or PyTorch Operator:

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: distributed-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          containers:
          - name: pytorch
            image: myregistry/distributed-trainer:v1
            resources:
              limits:
                nvidia.com/gpu: 8
    Worker:
      replicas: 3                 # 3 additional workers = 4 total nodes
      template:
        spec:
          containers:
          - name: pytorch
            image: myregistry/distributed-trainer:v1
            resources:
              limits:
                nvidia.com/gpu: 8   # 8 GPUs per node = 32 total GPUs
```

---

## 12. Phase 11: ML Pipelines with Kubeflow

### Key Kubeflow Components

| Component | Purpose |
|---|---|
| **Kubeflow Pipelines** | Define, deploy, and manage ML workflows as DAGs |
| **Katib** | Hyperparameter tuning (like Optuna but native to K8s) |
| **KServe** | Model serving and inference |
| **Training Operators** | PyTorch, TensorFlow, MPI, XGBoost distributed training |
| **Notebooks** | JupyterHub on K8s |

### Simple Kubeflow Pipeline

```python
from kfp import dsl, compiler

@dsl.component(base_image="python:3.11")
def preprocess(input_path: str, output_path: dsl.Output[dsl.Dataset]):
    import pandas as pd
    df = pd.read_csv(input_path)
    # ... preprocessing logic
    df.to_parquet(output_path.path)

@dsl.component(base_image="pytorch/pytorch:2.2.0-cuda12.1-cudnn8-runtime")
def train(dataset: dsl.Input[dsl.Dataset], model: dsl.Output[dsl.Model]):
    # ... training logic
    pass

@dsl.pipeline(name="training-pipeline")
def ml_pipeline(data_path: str = "gs://bucket/data.csv"):
    preprocess_task = preprocess(input_path=data_path)
    train_task = train(dataset=preprocess_task.outputs["output_path"])
    train_task.set_gpu_limit(4)
    train_task.set_memory_limit("32Gi")

compiler.Compiler().compile(ml_pipeline, "pipeline.yaml")
```

---

## 13. Phase 12: Observability

### The Monitoring Stack

```
Prometheus     → Metrics collection and alerting
Grafana        → Dashboards and visualization
Loki           → Log aggregation (like ELK but lighter)
Jaeger/Tempo   → Distributed tracing
```

### Quick Install with Helm

```bash
# Prometheus + Grafana stack
helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=changeme

# Access Grafana
kubectl port-forward -n monitoring svc/kube-prom-stack-grafana 3000:80
# Then open http://localhost:3000
```

### Key Metrics to Watch

```
# Node-level (like 'sinfo' detail)
node_cpu_seconds_total
node_memory_MemAvailable_bytes
nvidia_gpu_duty_cycle           # GPU utilization

# Pod-level (like 'sacct')
container_cpu_usage_seconds_total
container_memory_working_set_bytes
kube_pod_status_phase

# Cluster-level
kube_node_status_condition
kube_pod_container_status_restarts_total
```

### Useful Debugging Workflow

```bash
# Pod not starting?
kubectl describe pod <name>          # Check Events section at bottom
kubectl get events --sort-by='.lastTimestamp'

# Pod crashing?
kubectl logs <pod> --previous        # Logs from last crash
kubectl get pod <name> -o yaml | grep -A5 "state:"

# Can't connect to a service?
kubectl exec -it debug-pod -- nslookup my-service
kubectl exec -it debug-pod -- curl http://my-service:80
kubectl get endpoints my-service     # Are pods registered?

# Resource pressure?
kubectl top nodes
kubectl top pods --sort-by=memory
kubectl describe node <name> | grep -A5 "Allocated resources"
```

---

## 14. Phase 13: Advanced — CRDs, Operators & GitOps

### Custom Resource Definitions (CRDs)

CRDs let you extend Kubernetes with your own object types:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: trainingjobs.ml.example.com
spec:
  group: ml.example.com
  names:
    kind: TrainingJob
    plural: trainingjobs
    shortNames: ["tj"]
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              model:
                type: string
              epochs:
                type: integer
              gpus:
                type: integer
```

Now you can do `kubectl get trainingjobs` and manage ML jobs as native K8s objects.

### Operators

An Operator = CRD + Controller. It encodes domain expertise into Kubernetes. Popular operators include the Prometheus Operator, cert-manager, and various database operators (PostgreSQL, MySQL, Redis).

### GitOps with ArgoCD

GitOps principle: Git is the single source of truth. ArgoCD watches your Git repo and automatically applies changes to the cluster.

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Create an application
argocd app create my-app \
  --repo https://github.com/myorg/k8s-manifests.git \
  --path production/ \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated
```

The workflow becomes: push YAML to Git → ArgoCD detects change → applies to cluster → monitors drift.

---

## 15. Hands-On Labs

### Lab 1: Deploy a Full Stack App
Deploy a frontend (nginx), backend (Flask/FastAPI), and database (PostgreSQL) with Services connecting them. Add an Ingress for external access.

### Lab 2: GPU Batch Processing
Create a Job that processes images with a GPU. Use node affinity to target GPU nodes. Mount a PVC for input/output data.

### Lab 3: Auto-Scaling Under Load
Deploy an app with HPA. Use a load generator (`kubectl run loadgen --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://my-app; done"`) and watch it scale.

### Lab 4: Helm Chart from Scratch
Package your Lab 1 app as a Helm chart with configurable replicas, image tags, and resource limits.

### Lab 5: GitOps Pipeline
Set up ArgoCD, connect it to a Git repo with your manifests. Make a change in Git and watch it auto-deploy.

### Lab 6: Distributed GPU Training
Use the PyTorch Operator to run a multi-node training job. Monitor GPU utilization with Prometheus/Grafana.

---

## 16. Cheat Sheet

### Resource Units
| Unit | Meaning |
|---|---|
| `100m` CPU | 0.1 CPU core (100 millicores) |
| `1` CPU | 1 full core |
| `128Mi` | 128 mebibytes (≈134 MB) |
| `1Gi` | 1 gibibyte (≈1.07 GB) |
| `nvidia.com/gpu: 1` | 1 GPU |

### Common Troubleshooting

| Symptom | Check |
|---|---|
| Pod stuck in `Pending` | `kubectl describe pod` → Events. Usually resource shortage or node affinity mismatch |
| Pod in `CrashLoopBackOff` | `kubectl logs <pod> --previous` → app error or misconfiguration |
| Pod in `ImagePullBackOff` | Wrong image name, tag, or missing registry credentials |
| Service not reachable | Check `kubectl get endpoints` — selector might not match pod labels |
| Node `NotReady` | `kubectl describe node` → kubelet or container runtime issues |
| `OOMKilled` | Container exceeded memory limit. Increase `limits.memory` |
| `Evicted` | Node under disk or memory pressure. Check node conditions |

### Quick YAML Starters

```bash
# Generate YAML templates (never write from scratch!)
kubectl create deployment myapp --image=nginx --dry-run=client -o yaml > deployment.yaml
kubectl create service clusterip myapp --tcp=80:8080 --dry-run=client -o yaml > service.yaml
kubectl create job myjob --image=busybox --dry-run=client -o yaml > job.yaml
kubectl create cronjob mycron --image=busybox --schedule="0 * * * *" --dry-run=client -o yaml > cronjob.yaml
```

---

## 17. Resources & Certification Path

### Practice Environments
- **minikube** — Local single-node cluster (easiest start)
- **kind** (Kubernetes IN Docker) — Great for CI/testing, multi-node locally
- **k3s / k3d** — Lightweight K8s, great for edge/dev
- **Killercoda** — Free browser-based K8s labs (killercoda.com)
- **Play with Kubernetes** — Free 4-hour K8s playground

### Documentation
- **kubernetes.io/docs** — Official docs (bookmark the API reference)
- **learnk8s.io** — Excellent visual guides and best practices
- **kubectl cheat sheet** — kubernetes.io/docs/reference/kubectl/cheatsheet/

### Certification Path
1. **CKA** (Certified Kubernetes Administrator) — Operations focus. Start here.
2. **CKAD** (Certified Kubernetes Application Developer) — Developer focus.
3. **CKS** (Certified Kubernetes Security Specialist) — Security hardening. Advanced.

All exams are hands-on in a live terminal — your Slurm CLI skills will help.

### Books
- *Kubernetes Up & Running* (Hightower, Burns, Beda) — The standard intro
- *Kubernetes Patterns* (Ibryam, Huß) — Design patterns for K8s
- *Production Kubernetes* (Rosso et al.) — Operations at scale

### YouTube
- **TechWorld with Nana** — Excellent beginner-friendly K8s course
- **Just me and Opensource** — Deep hands-on tutorials
- **CNCF YouTube** — Conference talks, KubeCon recordings

---

> **Suggested learning order:** Phases 1-3 first week → Phase 4-6 second week → Phase 7-9 third week → Phase 10-13 ongoing as you build real projects. The labs are designed to reinforce each phase.

*Last updated: March 2026*
