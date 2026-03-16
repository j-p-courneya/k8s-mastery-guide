# Kubernetes Lab Plan on Jetstream2
### A Rational, Budget-Conscious Setup Using ACCESS Credits

---

## 1. Jetstream2 Quick Reference

Before planning, here are the numbers that matter. Jetstream2 charges by vCPU-hour, and the rate depends on the resource type.

### Instance Flavors & SU Burn Rates

**CPU Instances (1 SU per vCPU-hour):**

| Flavor | vCPUs | RAM | Disk | SU/hour |
|--------|-------|-----|------|---------|
| m3.tiny | 1 | 3 GB | 20 GB | 1 |
| m3.small | 2 | 6 GB | 20 GB | 2 |
| m3.quad | 4 | 15 GB | 20 GB | 4 |
| m3.medium | 8 | 30 GB | 60 GB | 8 |
| m3.large | 16 | 60 GB | 60 GB | 16 |
| m3.xl | 32 | 125 GB | 60 GB | 32 |

**GPU Instances (2 SU per vCPU-hour — halved since July 2024):**

| Flavor | vCPUs | RAM | GPU | GPU RAM | SU/hour |
|--------|-------|-----|-----|---------|---------|
| g3.medium | 8 | 30 GB | 25% A100 | 10 GB | 16 |
| g3.large | 16 | 60 GB | 50% A100 | 20 GB | 32 |
| g3.xl | 32 | 120 GB | Full A100 | 40 GB | 64 |

### Critical Cost-Saving Rules

- **SHELVED = 0 SU cost.** Always shelve when not using your instances.
- **STOPPED = 50% burn rate.** Still costs half — don't stop, shelve instead.
- **SUSPENDED = 75% burn rate.** Even worse for GPU instances (and GPU suspend is buggy — avoid it entirely).
- **Running but idle = full cost.** An m3.quad sitting idle overnight for 12 hours burns 48 SUs for nothing.

---

## 2. Lab Architecture Options

I'm recommending three tiers. Start with Tier 1 and graduate up as you get comfortable. This prevents burning credits on a complex cluster before you know what you're doing.

### Tier 1: Single-Node Learning Lab (Start Here)

**Purpose:** Learn kubectl, pods, deployments, services, storage, RBAC — everything in the mastery guide through Phase 8.

**Architecture:**
```
┌──────────────────────────────────────┐
│  m3.quad (4 vCPU, 15 GB RAM)         │
│                                      │
│  k3s single-node cluster             │
│  (control plane + worker combined)   │
│                                      │
│  Runs: pods, deployments, services,  │
│  ingress, helm charts, monitoring    │
└──────────────────────────────────────┘
```

**Why k3s, not kubeadm?** For a single-node learning environment, k3s gives you a fully functional K8s cluster in under 60 seconds, includes built-in ingress (Traefik), local storage, CoreDNS, and a metrics server. It behaves like real Kubernetes (same API, same kubectl) but uses fewer resources. You can always move to kubeadm for the multi-node tier.

**SU Budget:**

| Activity | Hours/week | SU/hour | Weekly SUs | Monthly SUs |
|----------|------------|---------|------------|-------------|
| Active learning sessions | 15 | 4 | 60 | 240 |
| Shelved remainder | — | 0 | 0 | 0 |
| **Total** | | | **60** | **~240** |

That's roughly **2,880 SUs/year** if you study ~15 hours/week. Very cheap.

**Setup Steps:**

```bash
# 1. Create an m3.quad instance via Exosphere
#    - Image: Ubuntu 22.04 Featured
#    - Enable Web Desktop (optional but handy)
#    - Add your SSH key

# 2. SSH in
ssh exouser@<floating-ip>

# 3. Install k3s (one command)
curl -sfL https://get.k3s.io | sh -

# 4. Set up kubectl for your user
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

# 5. Verify
kubectl get nodes
kubectl get pods -A

# 6. Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 7. IMPORTANT: When done for the day, SHELVE the instance via Exosphere
#    Exosphere → Instance → Actions → Shelve
#    This stops ALL SU charges until you unshelve
```

### Tier 2: Multi-Node kubeadm Cluster (Intermediate)

**Purpose:** Learn real cluster administration — node joining, CNI networking, multi-node scheduling, node affinity, taints/tolerations, PersistentVolumes across nodes, and cluster troubleshooting. This is the experience closest to managing a production cluster and maps directly to your Slurm cluster management experience.

**Architecture:**
```
┌──────────────────────────────────────────────────────────┐
│                    Jetstream2 Project                      │
│                                                           │
│  ┌─────────────────┐  ┌───────────────┐  ┌────────────┐  │
│  │ k8s-control      │  │ k8s-worker-1  │  │ k8s-worker-2│ │
│  │ m3.quad           │  │ m3.quad       │  │ m3.quad     │ │
│  │ 4 vCPU, 15 GB     │  │ 4 vCPU, 15 GB│  │ 4 vCPU,15GB│ │
│  │                   │  │              │  │            │  │
│  │ control plane     │  │ worker node  │  │ worker node│  │
│  │ etcd, api-server  │  │ kubelet      │  │ kubelet    │  │
│  │ scheduler         │  │ kube-proxy   │  │ kube-proxy │  │
│  │ controller-mgr    │  │ workloads    │  │ workloads  │  │
│  └────────┬──────────┘  └──────┬───────┘  └─────┬──────┘  │
│           │      Private Network (auto)          │         │
│           └──────────────┴───────────────────────┘         │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ Manila Share or Volume (shared storage for PVs)      │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**SU Budget:**

| Activity | Nodes | Hours/week | SU/hour each | Weekly SUs | Monthly SUs |
|----------|-------|------------|--------------|------------|-------------|
| Active lab (3 nodes running) | 3 | 12 | 4 | 144 | 576 |
| Shelved remainder | 3 | — | 0 | 0 | 0 |
| **Total** | | | | **144** | **~576** |

About **6,912 SUs/year** — still very manageable.

**Setup Steps:**

```bash
# 1. Create 3 x m3.quad instances via Exosphere
#    Names: k8s-control, k8s-worker-1, k8s-worker-2
#    Image: Ubuntu 22.04 Featured
#    Same network/project

# 2. On ALL 3 nodes — run the prep script:

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Install containerd
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml
sudo systemctl restart containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Set hostnames (run the appropriate one on each node)
sudo hostnamectl set-hostname k8s-control    # on control plane
sudo hostnamectl set-hostname k8s-worker-1   # on worker 1
sudo hostnamectl set-hostname k8s-worker-2   # on worker 2

# Add /etc/hosts entries on all nodes
# (use the PRIVATE IPs from Exosphere — run `hostname -I` to find them)
echo "<control-private-ip> k8s-control" | sudo tee -a /etc/hosts
echo "<worker1-private-ip> k8s-worker-1" | sudo tee -a /etc/hosts
echo "<worker2-private-ip> k8s-worker-2" | sudo tee -a /etc/hosts

# 3. On CONTROL PLANE only:

sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --control-plane-endpoint=k8s-control:6443

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico CNI
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

# IMPORTANT for Jetstream2: fix Calico IP detection for ens* interfaces
kubectl set env daemonset/calico-node -n kube-system \
  IP_AUTODETECTION_METHOD=interface=ens\*

# Generate join command
kubeadm token create --print-join-command
# Save the output!

# 4. On EACH WORKER node:
sudo kubeadm join k8s-control:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# 5. Back on CONTROL PLANE — verify:
kubectl get nodes
# All three should show "Ready" within a minute or two

kubectl label node k8s-worker-1 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker-2 node-role.kubernetes.io/worker=worker

# 6. Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 7. WHEN DONE: Shelve ALL 3 instances via Exosphere
```

### Tier 3: GPU + Multi-Node (Advanced)

**Purpose:** GPU workloads, NVIDIA device plugin, distributed training, KubeFlow. Only move here once you're comfortable with Tier 2.

**Architecture:** Same as Tier 2, but add a g3.medium (partial A100, 10 GB VRAM) as a GPU worker node. This is enough for testing GPU scheduling, running inference, and small training jobs.

**Important:** You must have Jetstream2-GPU as a separate resource on your ACCESS allocation. Having Jetstream2 CPU access does NOT include GPU access. You may need to exchange ACCESS credits specifically for Jetstream2-GPU SUs.

**SU Budget:**

| Activity | Detail | Hours/week | SU/hour | Weekly SUs | Monthly SUs |
|----------|--------|------------|---------|------------|-------------|
| 3 CPU nodes (active) | 3 × m3.quad | 10 | 12 total | 120 | 480 |
| 1 GPU node (active) | g3.medium | 6 | 16 | 96 | 384 |
| All shelved rest of time | — | — | 0 | 0 | 0 |
| **Total** | | | | **216** | **~864** |

About **10,368 SUs/year** — still reasonable, but you need to be disciplined about shelving.

---

## 3. Alternative: Magnum (Managed K8s on Jetstream2)

Jetstream2 also offers OpenStack Magnum, which deploys Kubernetes clusters automatically using pre-built images. This is faster (~10 minutes) and supports autoscaling, load balancers, and multi-master HA out of the box.

**When to use Magnum instead of kubeadm:**
- You want a production-like cluster without manual setup
- You need load balancer integration (OpenStack Octavia)
- You want cluster autoscaling (workers spin up/down automatically)
- You've already learned the manual setup and want to move faster

**When to stick with kubeadm:**
- You're learning cluster administration (Magnum hides the internals)
- You want CKA exam-relevant experience (CKA tests kubeadm, not Magnum)
- You need full control over the K8s version and CNI

**Quick Magnum setup:**
```bash
# Requires OpenStack CLI — install it first
pip install python-openstackclient python-magnumclient python-octaviaclient

# Source your Jetstream2 credentials
source app-cred-openrc.sh

# Clone the deployment repo
git clone https://github.com/zonca/jupyterhub-deploy-kubernetes-jetstream
cd jupyterhub-deploy-kubernetes-jetstream/kubernetes_magnum

# Edit create_cluster.sh with your settings, then:
bash create_cluster.sh

# Wait ~10 minutes, then:
openstack coe cluster config <cluster-name> --force
export KUBECONFIG=$(pwd)/config
kubectl get nodes
```

---

## 4. SU Budgeting Strategy

### How to Think About Your Budget

Let's say you have a typical Discover or Accelerate ACCESS allocation. Here's how to plan:

| Allocation Size | Available for K8s Lab | Duration at Tier 1 | Duration at Tier 2 | Duration at Tier 3 |
|-----------------|----------------------|--------------------|--------------------|---------------------|
| 50,000 SUs | ~50,000 | ~17 months | ~7 months | ~5 months |
| 200,000 SUs | ~200,000 | ~5.7 years | ~2.4 years | ~1.6 years |
| 1,000,000 SUs | ~1,000,000 | decades | ~12 years | ~8 years |

For a K8s learning lab, even 50,000 SUs is extremely generous — you won't burn through that unless you forget to shelve.

### The #1 Budget Killer: Forgetting to Shelve

A 3-node Tier 2 cluster left running 24/7 for a month costs:

> 3 nodes × 4 SU/hour × 24 hours × 30 days = **8,640 SUs/month**

The same cluster, shelved except for 12 hours/week of active use:

> 3 nodes × 4 SU/hour × 12 hours × 4 weeks = **576 SUs/month**

That's a **15x difference.** Set a reminder or create a shelve script.

### Automated Shelve Reminder

Add this to your control plane node's crontab to email yourself:

```bash
# Remind to shelve at 6 PM your time
crontab -e
# Add:
0 18 * * * echo "REMINDER: Shelve your K8s nodes on Jetstream2!" | \
  mail -s "Shelve Jetstream2" you@email.com
```

Or create a simple script to check your SU usage:
```bash
# Check current allocation usage
openstack usage show --start $(date -d '-30 days' +%Y-%m-%d) \
  --end $(date +%Y-%m-%d)
```

---

## 5. Recommended Learning Path on Jetstream2

### Week 1-2: Tier 1 — Foundations

1. Launch a single m3.quad, install k3s
2. Work through mastery guide Phases 0-3 (kubectl fluency, core concepts)
3. Deploy nginx, expose it via NodePort, access from browser using Floating IP
4. Practice: create/delete deployments, scale, rollback, check logs
5. **Shelve every night**

### Week 3-4: Tier 1 — Intermediate

1. Same single node
2. Work through Phases 4-6 (workload types, networking, storage)
3. Deploy a multi-container app (frontend + backend + database)
4. Set up Ingress routing (k3s includes Traefik by default)
5. Create PersistentVolumeClaims, attach Jetstream2 volumes
6. **Shelve every night**

### Week 5-6: Tier 1 → Tier 2 Transition

1. Snapshot your Tier 1 instance (save your work)
2. Build the 3-node kubeadm cluster (Tier 2)
3. Work through Phases 7-9 (RBAC, Helm, autoscaling)
4. Practice node affinity, taints — schedule pods on specific workers
5. Install Prometheus + Grafana via Helm
6. **Shelve all 3 nodes every night**

### Week 7-8: Tier 2 — Advanced

1. Deploy apps that use multi-node features (StatefulSets, DaemonSets)
2. Practice cluster troubleshooting (drain a node, cordon, uncordon)
3. Set up NetworkPolicies
4. Try the Magnum deployment as an alternative exercise
5. Snapshot your control plane for quick restore

### Week 9+: Tier 3 — GPU (Optional)

1. Add a g3.medium GPU worker to your cluster
2. Install NVIDIA device plugin
3. Run a GPU pod, verify GPU access with `nvidia-smi`
4. Deploy a PyTorch training job
5. Explore KubeFlow if ML pipelines are your goal

---

## 6. Jetstream2-Specific Tips

### Storage: Volumes vs Root Disk

Jetstream2 instances have limited root disk (20 GB for small flavors, 60 GB for medium+). For persistent data (datasets, container images, PV backing):

```bash
# Create a volume via Exosphere or CLI
openstack volume create --size 100 k8s-data

# Attach to your instance
openstack server add volume k8s-control k8s-data

# Mount it (inside the instance)
sudo mkfs.ext4 /dev/sdb       # Only first time!
sudo mkdir -p /mnt/k8s-data
sudo mount /dev/sdb /mnt/k8s-data
echo '/dev/sdb /mnt/k8s-data ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

Volumes persist independently of instances — you can shelve/delete the instance and the volume survives.

### Manila Shares (Shared Storage Across Nodes)

For multi-node clusters where you need ReadWriteMany PersistentVolumes (like a shared dataset), use Manila shares:

```bash
# Create a Manila share
openstack share create --name k8s-shared NFS 50

# Get the export path
openstack share export location list k8s-shared
```

Then use this as an NFS-backed PersistentVolume in your K8s cluster.

### Networking on Jetstream2

- Instances on the same allocation automatically share a private network
- The private IPs (usually 10.x.x.x) are what you use for K8s inter-node communication
- Floating (public) IPs are limited — only assign them to nodes that need external access (typically just the control plane or an ingress node)
- Security groups: open ports 6443 (API server), 30000-32767 (NodePort) on the control plane's security group if you want external access

### Snapshots: Save Your Progress

Before major changes, snapshot your instances. This is like saving a game — you can restore to a known good state:

```bash
# Via Exosphere: Instance → Actions → Create Snapshot
# Via CLI:
openstack server image create --name "k8s-control-working" k8s-control
```

Snapshots don't cost SUs (they use storage quota) and let you tear down and rebuild without repeating the full setup.

### CACAO: One-Click K3s Clusters

Jetstream2's CACAO interface can deploy multi-VM K3s clusters with a few clicks — no manual setup. This is the fastest way to get a cluster running if you've already learned the manual process and just want a working environment quickly.

Access it at: Exosphere → CACAO → Deploy → Multi-VM Kubernetes Cluster (k3s)

---

## 7. Decision Summary

```
What should I do RIGHT NOW?
│
├── 1. Log into Exosphere (https://jetstream2.exosphere.app)
├── 2. Create one m3.quad instance (Ubuntu 22.04)
├── 3. SSH in, run: curl -sfL https://get.k3s.io | sh -
├── 4. Start working through the mastery guide
├── 5. SHELVE when done for the day
│
└── Budget check: This costs ~4 SUs/hour while active,
    0 SUs while shelved. At 15 hrs/week, that's
    ~240 SUs/month. You have plenty of runway.
```

---

## 8. Reference Links

- Jetstream2 Instance Flavors: https://docs.jetstream-cloud.org/general/instance-flavors/
- Building a K8s Cluster on JS2: https://docs.jetstream-cloud.org/general/k8scluster/
- Magnum (Managed K8s): https://docs.jetstream-cloud.org/general/k8smagnum/
- Kubespray on JS2: https://docs.jetstream-cloud.org/general/k8skubespray/
- CACAO K3s Deployment: https://docs.jetstream-cloud.org/ui/cacao/deployment_kubernetes/
- SU Estimation Calculator: https://docs.jetstream-cloud.org/alloc/estimator/
- ACCESS Allocations: https://allocations.access-ci.org/
- Shelving/Instance Management: https://docs.jetstream-cloud.org/getting-started/instance-management/

*Last updated: March 2026*
