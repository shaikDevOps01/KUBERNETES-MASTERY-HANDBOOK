# KUBERNETES MASTERY HANDBOOK
# Part 10: Cluster Administration
# Chapter 10: kubeadm, Cluster Setup, Upgrade, etcd Backup & Restore

---

> **"Anyone can deploy an app on Kubernetes.
>  A Kubernetes Administrator can build the cluster, upgrade it safely,
>  and restore it from nothing when disaster strikes."**

---

## Chapter Introduction

This is the most operationally critical chapter in the entire handbook.
Everything you have learned so far assumes a cluster already exists.
This chapter teaches you to **build, maintain, upgrade, and recover** the
cluster itself — the skills that separate a Kubernetes user from a
Kubernetes Administrator.

```
╔══════════════════════════════════════════════════════════════════════╗
║            CHAPTER 10 — CLUSTER ADMINISTRATION ROADMAP               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  10.1 — Cluster Architecture Revisited (production topology)        ║
║  10.2 — kubeadm: Bootstrap a Cluster from Scratch                   ║
║  10.3 — kubeconfig: Managing Cluster Access                         ║
║  10.4 — Cluster Upgrade: Control Plane + Worker Nodes               ║
║  10.5 — etcd: The Source of Truth                                   ║
║  10.6 — etcd Backup: The Most Important Task in CKA                 ║
║  10.7 — etcd Restore: Disaster Recovery in Practice                 ║
║  10.8 — Certificate Management                                       ║
║  10.9 — Node Maintenance Workflows                                   ║
║  10.10 — Cluster Troubleshooting                                     ║
║                                                                      ║
║  CKA EXAM ALERT:                                                     ║
║  etcd backup/restore + cluster upgrade = ~25% of exam tasks        ║
║  Master these two topics above all others in this chapter.          ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 10.1 Cluster Architecture Revisited — Production Topology

### 10.1.1 Single Control Plane vs High Availability

```
╔══════════════════════════════════════════════════════════════════════╗
║          SINGLE CONTROL PLANE (Dev/Lab)                              ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  control-plane-1 (192.168.1.10)                             │    ║
║  │  • kube-apiserver                                           │    ║
║  │  • etcd                                                     │    ║
║  │  • kube-scheduler                                           │    ║
║  │  • kube-controller-manager                                  │    ║
║  └──────────────────────────────┬──────────────────────────────┘    ║
║                                 │                                    ║
║         ┌───────────────────────┼──────────────────────┐            ║
║         ▼                       ▼                      ▼            ║
║  ┌──────────────┐       ┌──────────────┐      ┌──────────────┐      ║
║  │  worker-1    │       │  worker-2    │      │  worker-3    │      ║
║  └──────────────┘       └──────────────┘      └──────────────┘      ║
║                                                                      ║
║  RISK: Control plane = single point of failure.                     ║
║  If control-plane-1 dies → cluster is unmanageable.                 ║
║  (Workloads continue running but cannot be changed/recovered)       ║
╠══════════════════════════════════════════════════════════════════════╣
║          HIGH AVAILABILITY CONTROL PLANE (Production)                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Load Balancer (HAProxy / NLB) : 192.168.1.100 (VIP)               ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │         ┌──────────┐    ┌──────────┐    ┌──────────┐          │ ║
║  │  ctrl-1 │apiserver│    │apiserver │    │apiserver │ ctrl-3   │ ║
║  │         │  etcd   │    │  etcd   │    │  etcd   │           │ ║
║  │         │scheduler│    │scheduler│    │scheduler│           │ ║
║  │         └──────────┘    └──────────┘    └──────────┘          │ ║
║  │                 ctrl-1       ctrl-2          ctrl-3            │ ║
║  │                                                                │ ║
║  │  etcd: 3 members — Raft quorum needs ⌊N/2⌋+1 = 2 nodes alive │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                      ║
║  Worker nodes connect to the LB VIP (192.168.1.100:6443)           ║
║  If ctrl-1 dies → ctrl-2 and ctrl-3 maintain quorum → HA           ║
║                                                                      ║
║  ETCD QUORUM:   3 nodes → tolerate 1 failure                       ║
║                 5 nodes → tolerate 2 failures                      ║
║                 7 nodes → tolerate 3 failures (rarely needed)      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 10.2 kubeadm — Bootstrap a Cluster from Scratch

### 10.2.1 What Is kubeadm?

#### In Plain English

kubeadm is the **official cluster construction tool**. It handles all the
complex certificate generation, component configuration, and bootstrap
sequencing that would take days to do manually. Think of it as the
**construction foreman** that builds your Kubernetes cluster to spec.

#### In Technical Language

`kubeadm` is a tool that provides `kubeadm init` and `kubeadm join` as
best-practice fast-path implementations for creating Kubernetes clusters.
It automates certificate generation, static pod manifests, kubeconfig files,
and component configuration — following best practices defined by the
Kubernetes community.

### 10.2.2 Pre-flight Requirements

```bash
# ═══════════════════════════════════════════════════════════════════
# PREREQUISITES (run on ALL nodes: control plane + workers)
# ═══════════════════════════════════════════════════════════════════

# 1. OS: Ubuntu 20.04/22.04 LTS (or RHEL/CentOS 8+)
# 2. Min specs: 2 CPU, 2GB RAM per node
# 3. Unique hostname, MAC address, and product_uuid on each node
# 4. Swap must be disabled

# Disable swap (kubeadm requirement)
swapoff -a
sed -i '/swap/d' /etc/fstab      # Persist across reboots
cat /proc/swaps                   # Should be empty

# Verify unique hostnames
hostnamectl set-hostname control-plane-1    # On control plane
hostnamectl set-hostname worker-1           # On workers

# Enable required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# Required sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system    # Apply immediately

# Verify
lsmod | grep br_netfilter
sysctl net.bridge.bridge-nf-call-iptables
```

### 10.2.3 Install Container Runtime (containerd)

```bash
# ═══════════════════════════════════════════════════════════════════
# INSTALL containerd (run on ALL nodes)
# ═══════════════════════════════════════════════════════════════════

# Install containerd
apt-get update
apt-get install -y containerd

# Configure containerd for Kubernetes
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup driver (REQUIRED for Kubernetes)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

# Restart and enable containerd
systemctl restart containerd
systemctl enable containerd
systemctl status containerd    # Verify: active (running)
```

### 10.2.4 Install kubeadm, kubelet, kubectl

```bash
# ═══════════════════════════════════════════════════════════════════
# INSTALL kubeadm, kubelet, kubectl (run on ALL nodes)
# ═══════════════════════════════════════════════════════════════════

# Add Kubernetes APT repository
apt-get install -y apt-transport-https ca-certificates curl gnupg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg]
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list

# Install specific version (pin version for cluster consistency)
apt-get update
apt-get install -y kubelet=1.29.0-1.1 kubeadm=1.29.0-1.1 kubectl=1.29.0-1.1

# IMPORTANT: Pin versions to prevent accidental upgrades
apt-mark hold kubelet kubeadm kubectl

# Verify
kubeadm version
kubectl version --client
kubelet --version
```

### 10.2.5 Initialize the Control Plane

```bash
# ═══════════════════════════════════════════════════════════════════
# INITIALIZE CONTROL PLANE (run on control-plane node ONLY)
# ═══════════════════════════════════════════════════════════════════

# Pull images first (speeds up init, useful for offline setups)
kubeadm config images pull

# Initialize the cluster
kubeadm init \
  --kubernetes-version=1.29.0 \
  --pod-network-cidr=10.244.0.0/16 \      # For Flannel CNI
  --apiserver-advertise-address=192.168.1.10 \  # Control plane IP
  --control-plane-endpoint=192.168.1.10:6443    # or VIP for HA

# For HA setup with load balancer:
# --control-plane-endpoint=192.168.1.100:6443 \  # LB VIP
# --upload-certs                                  # Share certs with other control planes

# === SUCCESSFUL OUTPUT (SAVE THIS!) ===
# Your Kubernetes control-plane has initialized successfully!
#
# To start using your cluster, run as a regular user:
#   mkdir -p $HOME/.kube
#   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#   sudo chown $(id -u):$(id -g) $HOME/.kube/config
#
# You should now deploy a pod network to the cluster.
# Then you can join any number of worker nodes by running:
#
#   kubeadm join 192.168.1.10:6443 \
#     --token abcdef.0123456789abcdef \
#     --discovery-token-ca-cert-hash sha256:abc123...

# Step 1: Set up kubeconfig for current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify control plane is up
kubectl get nodes        # STATUS: NotReady (no CNI yet)
kubectl get pods -n kube-system

# Step 2: Install CNI (Flannel example)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# For Calico:
# kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Wait for all system pods to be running
kubectl get pods -n kube-system -w

# Control plane node should now be Ready
kubectl get nodes
# NAME               STATUS   ROLES           AGE   VERSION
# control-plane-1    Ready    control-plane   2m    v1.29.0
```

### 10.2.6 Join Worker Nodes

```bash
# ═══════════════════════════════════════════════════════════════════
# JOIN WORKER NODES (run on each worker node)
# ═══════════════════════════════════════════════════════════════════

# Use the join command from kubeadm init output:
kubeadm join 192.168.1.10:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:abc123...

# If you lost the join token (tokens expire after 24 hours):
# Regenerate on control plane:
kubeadm token create --print-join-command

# Verify all nodes joined
kubectl get nodes -o wide
# NAME               STATUS   ROLES           AGE   VERSION       INTERNAL-IP
# control-plane-1    Ready    control-plane   5m    v1.29.0       192.168.1.10
# worker-1           Ready    <none>          2m    v1.29.0       192.168.1.11
# worker-2           Ready    <none>          1m    v1.29.0       192.168.1.12

# Label worker nodes
kubectl label node worker-1 node-role.kubernetes.io/worker=worker
kubectl label node worker-2 node-role.kubernetes.io/worker=worker
```

### 10.2.7 kubeadm init — Important File Locations

```
╔══════════════════════════════════════════════════════════════════════╗
║           KEY FILE LOCATIONS AFTER kubeadm init                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  CERTIFICATES (PKI):                                                 ║
║  /etc/kubernetes/pki/                                                ║
║  ├── ca.crt / ca.key              Cluster CA                        ║
║  ├── apiserver.crt / .key         API Server cert                   ║
║  ├── apiserver-kubelet-client.crt API→kubelet auth                  ║
║  ├── front-proxy-ca.crt           Front proxy CA                    ║
║  ├── sa.key / sa.pub              Service account signing keys       ║
║  └── etcd/                                                           ║
║      ├── ca.crt / ca.key          etcd CA                           ║
║      ├── server.crt / .key        etcd server cert                  ║
║      └── peer.crt / .key          etcd peer cert                    ║
║                                                                      ║
║  KUBECONFIG FILES:                                                   ║
║  /etc/kubernetes/                                                    ║
║  ├── admin.conf                   Cluster admin kubeconfig          ║
║  ├── controller-manager.conf      Controller manager kubeconfig     ║
║  ├── scheduler.conf               Scheduler kubeconfig              ║
║  └── kubelet.conf                 kubelet kubeconfig                ║
║                                                                      ║
║  STATIC POD MANIFESTS (control plane components):                   ║
║  /etc/kubernetes/manifests/                                          ║
║  ├── kube-apiserver.yaml          API server static pod             ║
║  ├── kube-controller-manager.yaml Controller manager static pod     ║
║  ├── kube-scheduler.yaml          Scheduler static pod              ║
║  └── etcd.yaml                    etcd static pod                   ║
║                                                                      ║
║  kubelet reads /etc/kubernetes/manifests/ directly                  ║
║  Changes to these files = immediate restart of component            ║
║                                                                      ║
║  etcd DATA:                                                          ║
║  /var/lib/etcd/                   All cluster state stored here     ║
║                                                                      ║
║  kubelet config:                                                     ║
║  /var/lib/kubelet/config.yaml                                        ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 10.3 kubeconfig — Managing Cluster Access

```bash
# ── KUBECONFIG BASICS ────────────────────────────────────────────────
# Default location
cat ~/.kube/config

# Use a different kubeconfig
export KUBECONFIG=/path/to/other/config
kubectl get nodes

# Merge multiple kubeconfigs
KUBECONFIG=~/.kube/config:/path/to/cluster2.conf \
  kubectl config view --flatten > ~/.kube/merged.conf
mv ~/.kube/merged.conf ~/.kube/config

# ── CONTEXT MANAGEMENT ────────────────────────────────────────────────
kubectl config get-contexts                     # List all contexts
kubectl config current-context                  # Which am I on?
kubectl config use-context production-admin     # Switch context
kubectl config set-context --current \
  --namespace=development                       # Set default namespace

# ── ADD A NEW CLUSTER TO KUBECONFIG ───────────────────────────────────
kubectl config set-cluster my-cluster \
  --server=https://192.168.1.10:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt

kubectl config set-credentials admin-user \
  --client-certificate=/etc/kubernetes/pki/admin.crt \
  --client-key=/etc/kubernetes/pki/admin.key

kubectl config set-context my-context \
  --cluster=my-cluster \
  --user=admin-user \
  --namespace=default

kubectl config use-context my-context

# ── QUICK KUBECONFIG INSPECTION ───────────────────────────────────────
kubectl config view --minify          # Only current context config
kubectl config view -o jsonpath='{.clusters[0].cluster.server}'  # API server URL
```

---

## 10.4 Cluster Upgrade — The Critical Procedure

### 10.4.1 Why Cluster Upgrades Are High-Risk

```
╔══════════════════════════════════════════════════════════════════════╗
║              CLUSTER UPGRADE — RULES AND RISKS                       ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  KUBERNETES VERSION SKEW POLICY:                                     ║
║  • kube-apiserver: must be upgraded FIRST                           ║
║  • kubelet: can be 2 minor versions BEHIND apiserver               ║
║  • kubectl: can be 1 minor version ahead or behind apiserver       ║
║                                                                      ║
║  UPGRADE ORDER (always follow this sequence):                        ║
║  1. Back up etcd FIRST (before touching anything)                   ║
║  2. Upgrade control plane (kubeadm upgrade plan → apply)            ║
║  3. Upgrade kubelet + kubectl on control plane                      ║
║  4. For each worker node:                                           ║
║     a. kubectl cordon worker-N   (stop new pods)                    ║
║     b. kubectl drain worker-N    (evict existing pods)              ║
║     c. Upgrade kubeadm on worker-N                                  ║
║     d. kubeadm upgrade node      (upgrade node config)              ║
║     e. Upgrade kubelet + kubectl                                    ║
║     f. kubectl uncordon worker-N (re-enable scheduling)             ║
║                                                                      ║
║  ONE MINOR VERSION AT A TIME:                                        ║
║  1.27 → 1.28 → 1.29 (CORRECT)                                      ║
║  1.27 → 1.29 (FORBIDDEN — skipping minor versions not supported)    ║
║                                                                      ║
║  UPGRADE IS NOT INSTANT — average time per cluster:                 ║
║  Small cluster (1 CP + 3 workers): 20-40 minutes                   ║
║  Large cluster (3 CP + 20 workers): 2-4 hours                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 10.4.2 Step-by-Step: Control Plane Upgrade (1.28 → 1.29)

```bash
# ═══════════════════════════════════════════════════════════════════
# STEP 0: BACKUP etcd FIRST (NON-NEGOTIABLE)
# ═══════════════════════════════════════════════════════════════════
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-before-upgrade.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-before-upgrade.db \
  --write-out=table

# ═══════════════════════════════════════════════════════════════════
# STEP 1: Upgrade kubeadm on control plane
# ═══════════════════════════════════════════════════════════════════

# Remove hold to allow upgrade
apt-mark unhold kubeadm

# Install target version
apt-get update
apt-get install -y kubeadm=1.29.0-1.1

# Re-hold kubeadm
apt-mark hold kubeadm

# Verify new kubeadm version
kubeadm version

# ═══════════════════════════════════════════════════════════════════
# STEP 2: Check upgrade plan
# ═══════════════════════════════════════════════════════════════════

kubeadm upgrade plan
# Output shows:
# COMPONENT              CURRENT    TARGET
# kube-apiserver         v1.28.5    v1.29.0
# kube-controller-manager v1.28.5   v1.29.0
# kube-scheduler         v1.28.5    v1.29.0
# kube-proxy             v1.28.5    v1.29.0
# CoreDNS                v1.10.1    v1.11.1
# etcd                   3.5.9-0    3.5.10-0

# ═══════════════════════════════════════════════════════════════════
# STEP 3: Apply the upgrade
# ═══════════════════════════════════════════════════════════════════

kubeadm upgrade apply v1.29.0
# This upgrades:
# - kube-apiserver static pod
# - kube-controller-manager static pod
# - kube-scheduler static pod
# - kube-proxy DaemonSet
# - CoreDNS Deployment
# - etcd (if managed by kubeadm)

# Watch static pods restart
kubectl get pods -n kube-system -w

# Verify control plane components are at new version
kubectl get pods -n kube-system -o wide

# ═══════════════════════════════════════════════════════════════════
# STEP 4: Upgrade kubelet and kubectl on control plane
# ═══════════════════════════════════════════════════════════════════

# Drain control plane node (if running workloads)
kubectl drain control-plane-1 --ignore-daemonsets

# Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
apt-mark hold kubelet kubectl

# Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# Uncordon control plane
kubectl uncordon control-plane-1

# Verify control plane node is Ready at new version
kubectl get nodes
# NAME               STATUS   VERSION
# control-plane-1    Ready    v1.29.0  ← Upgraded!
# worker-1           Ready    v1.28.5  ← Not yet
# worker-2           Ready    v1.28.5  ← Not yet
```

### 10.4.3 Step-by-Step: Worker Node Upgrade

```bash
# ═══════════════════════════════════════════════════════════════════
# REPEAT FOR EACH WORKER NODE (one at a time)
# Run kubectl commands from control plane, ssh commands on worker
# ═══════════════════════════════════════════════════════════════════

# FROM CONTROL PLANE: Cordon and drain worker-1
kubectl cordon worker-1
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60
# Pods move to worker-2, worker-3, etc.
# kubectl get pods -o wide — verify all pods off worker-1

# ON WORKER-1 (via SSH):
ssh worker-1

  # Upgrade kubeadm
  apt-mark unhold kubeadm
  apt-get update
  apt-get install -y kubeadm=1.29.0-1.1
  apt-mark hold kubeadm

  # Upgrade node configuration
  kubeadm upgrade node
  # This updates the kubelet configuration for the node

  # Upgrade kubelet and kubectl
  apt-mark unhold kubelet kubectl
  apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
  apt-mark hold kubelet kubectl

  # Restart kubelet
  systemctl daemon-reload
  systemctl restart kubelet
  exit

# FROM CONTROL PLANE: Uncordon worker-1
kubectl uncordon worker-1

# Verify worker-1 is Ready at new version
kubectl get nodes
# NAME               STATUS   VERSION
# control-plane-1    Ready    v1.29.0  ✅
# worker-1           Ready    v1.29.0  ✅ Just upgraded
# worker-2           Ready    v1.28.5  (next)

# Repeat for worker-2, worker-3...

# ═══════════════════════════════════════════════════════════════════
# VERIFY COMPLETE UPGRADE
# ═══════════════════════════════════════════════════════════════════
kubectl get nodes
# ALL nodes should show v1.29.0

kubectl get pods -n kube-system
# ALL system pods should be running with new version

kubectl version
# Client and Server both at v1.29.0
```

---

## 10.5 etcd — Deep Dive

### 10.5.1 etcd in the Kubernetes Context

```
╔══════════════════════════════════════════════════════════════════════╗
║                   etcd — EVERYTHING YOU NEED TO KNOW                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  WHAT IS STORED IN etcd?                                             ║
║  Every Kubernetes object: pods, deployments, services, secrets,      ║
║  configmaps, nodes, namespaces, RBAC, PVs, PVCs — EVERYTHING.        ║
║  It is the ONLY stateful part of the control plane.                 ║
║                                                                      ║
║  FORMAT:                                                             ║
║  Key: /registry/<resource-type>/<namespace>/<name>                  ║
║  e.g: /registry/pods/default/my-pod-xyz                             ║
║       /registry/secrets/production/db-password                      ║
║       /registry/nodes/worker-1                                       ║
║                                                                      ║
║  RAFT CONSENSUS:                                                     ║
║  etcd uses Raft for distributed consensus.                          ║
║  In a 3-node cluster: 2 nodes must agree (quorum) for writes.       ║
║  Leader election: one etcd node is leader, rest are followers.      ║
║                                                                      ║
║  PORTS:                                                              ║
║  2379 — Client communication (API server → etcd)                    ║
║  2380 — Peer communication (etcd → etcd in HA setup)                ║
║                                                                      ║
║  SECURITY:                                                           ║
║  All etcd communication is TLS-encrypted.                           ║
║  Certs location: /etc/kubernetes/pki/etcd/                          ║
║                                                                      ║
║  FINDING etcd ENDPOINT:                                              ║
║  kubectl describe pod etcd-<cp-name> -n kube-system | grep listen  ║
║  or: cat /etc/kubernetes/manifests/etcd.yaml | grep advertise      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 10.5.2 etcdctl — The etcd CLI

```bash
# etcdctl requires ETCDCTL_API=3 for modern commands
export ETCDCTL_API=3

# Common flags (required for TLS — always needed with Kubernetes etcd)
ETCD_OPTS="\
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

# Check etcd cluster health
etcdctl $ETCD_OPTS endpoint health
# https://127.0.0.1:2379 is healthy: successfully committed proposal

# Check cluster members
etcdctl $ETCD_OPTS member list
# ID              STATUS   NAME    PEER ADDRS              CLIENT ADDRS
# 8e9e05c52164694d  started  control-plane-1  https://192.168.1.10:2380  https://192.168.1.10:2379

# Get cluster status
etcdctl $ETCD_OPTS endpoint status --write-out=table

# Read a key from etcd (for debugging)
etcdctl $ETCD_OPTS get /registry/namespaces/default
etcdctl $ETCD_OPTS get /registry/pods/default/ --prefix --keys-only

# Find all secrets (shows how many exist — NOT their values!)
etcdctl $ETCD_OPTS get /registry/secrets/ --prefix --keys-only

# Get etcd version
etcdctl $ETCD_OPTS version

# Find the etcd advertise address (useful in exam to find endpoint)
cat /etc/kubernetes/manifests/etcd.yaml | \
  grep "\-\-advertise-client-urls"
```

---

## 10.6 etcd Backup — THE Most Critical CKA Skill

### 10.6.1 Why etcd Backup Is Non-Negotiable

```
╔══════════════════════════════════════════════════════════════════════╗
║              WHAT HAPPENS IF etcd DATA IS LOST?                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  SCENARIO: Your etcd data directory (/var/lib/etcd/) is corrupted  ║
║  or a disk fails.                                                    ║
║                                                                      ║
║  RESULT:                                                             ║
║  • kube-apiserver cannot start (no database)                        ║
║  • ALL cluster state is lost:                                       ║
║    - All Deployments, Services, ConfigMaps, Secrets                 ║
║    - All RBAC rules                                                  ║
║    - All PersistentVolumeClaims                                      ║
║    - All network policies                                            ║
║  • Worker nodes continue running existing pods (kubelet is local)   ║
║  • But nobody can manage, scale, update, or fix anything            ║
║                                                                      ║
║  WITHOUT A BACKUP: You rebuild from scratch (days of work)          ║
║  WITH A BACKUP: Restored in 5-10 minutes                            ║
║                                                                      ║
║  PRODUCTION RECOMMENDATION:                                          ║
║  • Backup etcd every hour minimum                                   ║
║  • Store backups off-cluster (S3, NFS, object storage)              ║
║  • Test restores regularly (an untested backup is not a backup!)    ║
║  • Keep last 24 hourly + 7 daily + 4 weekly backups                 ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 10.6.2 etcd Backup — The Complete Procedure

```bash
# ═══════════════════════════════════════════════════════════════════
# STEP 1: Find the etcd endpoint and cert locations
# ═══════════════════════════════════════════════════════════════════

# Method 1: Check etcd static pod manifest
cat /etc/kubernetes/manifests/etcd.yaml | grep -E \
  "listen-client-urls|ca-file|cert-file|key-file|data-dir"

# Method 2: Check running etcd process
ps aux | grep etcd | grep -v grep

# Common values on kubeadm clusters:
ETCD_ENDPOINT="https://127.0.0.1:2379"
ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"

# ═══════════════════════════════════════════════════════════════════
# STEP 2: Create the backup directory
# ═══════════════════════════════════════════════════════════════════
mkdir -p /backup/etcd
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="/backup/etcd/etcd-snapshot-${TIMESTAMP}.db"

# ═══════════════════════════════════════════════════════════════════
# STEP 3: Take the snapshot
# ═══════════════════════════════════════════════════════════════════
ETCDCTL_API=3 etcdctl snapshot save $BACKUP_FILE \
  --endpoints=$ETCD_ENDPOINT \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# Expected output:
# {"level":"info","msg":"created temporary file","path":"/backup/etcd/..."}
# Snapshot saved at /backup/etcd/etcd-snapshot-20240115_020000.db

# ═══════════════════════════════════════════════════════════════════
# STEP 4: Verify the backup
# ═══════════════════════════════════════════════════════════════════
ETCDCTL_API=3 etcdctl snapshot status $BACKUP_FILE \
  --write-out=table

# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | fe01cf57 |       10 |        507 |      2.1 MB|
# +----------+----------+------------+------------+

# Check file size and permissions
ls -lh $BACKUP_FILE

# ═══════════════════════════════════════════════════════════════════
# STEP 5: Copy to off-cluster storage (production practice)
# ═══════════════════════════════════════════════════════════════════
# aws s3 cp $BACKUP_FILE s3://my-cluster-backups/etcd/
# rsync $BACKUP_FILE backup-server:/data/etcd-backups/

echo "Backup complete: $BACKUP_FILE"
```

### 10.6.3 Automated etcd Backup with CronJob

```yaml
# etcd-backup-cronjob.yaml
# Production-grade automated backup with S3 upload
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"           # Every 6 hours
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true           # Access etcd on localhost
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
          - key: node-role.kubernetes.io/control-plane
            operator: Exists
            effect: NoSchedule
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: bitnami/etcd:latest
            command: ["/bin/sh", "-c"]
            args:
            - |
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              FILE="/tmp/etcd-snapshot-${TIMESTAMP}.db"
              ETCDCTL_API=3 etcdctl snapshot save "$FILE" \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
              echo "Snapshot saved: $FILE"
              ETCDCTL_API=3 etcdctl snapshot status "$FILE" --write-out=table
              # Upload to S3
              # aws s3 cp "$FILE" s3://my-backups/etcd/"$TIMESTAMP".db
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
```

---

## 10.7 etcd Restore — Disaster Recovery

### 10.7.1 The Restore Procedure (Step-by-Step)

```
╔══════════════════════════════════════════════════════════════════════╗
║              etcd RESTORE — CRITICAL SEQUENCE                        ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  WARNING: etcd restore is destructive and irreversible.             ║
║  Always confirm you are on the right cluster and node.              ║
║                                                                      ║
║  HIGH-LEVEL SEQUENCE:                                                ║
║                                                                      ║
║  1. STOP API server (so nothing writes to etcd during restore)      ║
║  2. Restore snapshot to a NEW data directory                        ║
║  3. UPDATE etcd manifest to point to new data directory             ║
║  4. RESTART etcd (kubelet auto-restarts static pods)                ║
║  5. VERIFY cluster is healthy                                       ║
╚══════════════════════════════════════════════════════════════════════╝
```

```bash
# ═══════════════════════════════════════════════════════════════════
# STEP 1: Stop API server (move manifest out of watch directory)
# ═══════════════════════════════════════════════════════════════════

# Static pods in /etc/kubernetes/manifests/ are auto-started by kubelet
# Moving them OUT stops them immediately

mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/

# Wait for containers to stop
sleep 30
crictl ps | grep kube-apiserver   # Should show nothing

# ═══════════════════════════════════════════════════════════════════
# STEP 2: Stop etcd
# ═══════════════════════════════════════════════════════════════════
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
sleep 10
crictl ps | grep etcd              # Should show nothing

# ═══════════════════════════════════════════════════════════════════
# STEP 3: Restore snapshot to new data directory
# ═══════════════════════════════════════════════════════════════════

SNAPSHOT="/backup/etcd/etcd-snapshot-20240115_020000.db"
RESTORE_DIR="/var/lib/etcd-restore"    # NEW directory (not original!)

ETCDCTL_API=3 etcdctl snapshot restore $SNAPSHOT \
  --data-dir=$RESTORE_DIR \
  --name=control-plane-1 \
  --initial-cluster=control-plane-1=https://192.168.1.10:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://192.168.1.10:2380

# Output:
# 2024-01-15T02:00:00Z info    restored snapshot
# 2024-01-15T02:00:00Z info    added member

ls -la $RESTORE_DIR/    # Verify data exists

# ═══════════════════════════════════════════════════════════════════
# STEP 4: Update etcd manifest to point to new data directory
# ═══════════════════════════════════════════════════════════════════

# Edit the etcd.yaml that was moved to /tmp
# Find and change: --data-dir=/var/lib/etcd
#              to: --data-dir=/var/lib/etcd-restore
# Also update the hostPath volumes section

sed -i 's|/var/lib/etcd|/var/lib/etcd-restore|g' /tmp/etcd.yaml

# Verify the change
grep "data-dir\|/var/lib" /tmp/etcd.yaml

# ═══════════════════════════════════════════════════════════════════
# STEP 5: Restore all manifests and restart
# ═══════════════════════════════════════════════════════════════════

# Restore etcd first
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
sleep 20

# Wait for etcd to start
crictl ps | grep etcd   # Should show Running

# Verify etcd is healthy before restoring API server
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore other control plane components
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/

# Wait for everything to come up (can take 1-2 minutes)
sleep 60

# ═══════════════════════════════════════════════════════════════════
# STEP 6: Verify restoration
# ═══════════════════════════════════════════════════════════════════

kubectl get nodes
kubectl get pods -n kube-system
kubectl get pods -A

# If successful: all objects from backup time are restored
# Objects created AFTER the backup are lost
```

### 10.7.2 Restore Verification Checklist

```bash
# After restore — run all of these to verify
kubectl cluster-info                   # API server responding?
kubectl get nodes                      # Nodes present?
kubectl get pods -n kube-system        # System pods running?
kubectl get namespaces                 # Namespaces present?
kubectl get deployments -A             # Application deployments restored?
kubectl get pv,pvc -A                  # Storage objects restored?

# Check etcd cluster member status
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Check etcd endpoint status
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

---

## 10.8 Certificate Management

### 10.8.1 Kubernetes Certificate Overview

```
╔══════════════════════════════════════════════════════════════════════╗
║              KUBERNETES CERTIFICATE EXPIRY                           ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  kubeadm-generated certificates expire in 1 YEAR by default.       ║
║  If certificates expire → cluster stops working completely.         ║
║                                                                      ║
║  BEST PRACTICE: Upgrade the cluster at least once per year.         ║
║  kubeadm upgrade renews certificates automatically.                 ║
║                                                                      ║
║  CERTIFICATE TYPES:                                                  ║
║  • Control plane certs: /etc/kubernetes/pki/                        ║
║  • etcd certs:          /etc/kubernetes/pki/etcd/                   ║
║  • Kubeconfig certs:    embedded in /etc/kubernetes/*.conf          ║
╚══════════════════════════════════════════════════════════════════════╝
```

```bash
# ── CHECK CERTIFICATE EXPIRY ──────────────────────────────────────────
# Check all certificate expiry dates
kubeadm certs check-expiration

# Example output:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 Jan 15, 2025 02:00 UTC   364d
# apiserver                  Jan 15, 2025 02:00 UTC   364d
# apiserver-etcd-client      Jan 15, 2025 02:00 UTC   364d
# apiserver-kubelet-client   Jan 15, 2025 02:00 UTC   364d
# controller-manager.conf    Jan 15, 2025 02:00 UTC   364d
# etcd-healthcheck-client    Jan 15, 2025 02:00 UTC   364d
# etcd-peer                  Jan 15, 2025 02:00 UTC   364d
# etcd-server                Jan 15, 2025 02:00 UTC   364d
# front-proxy-client         Jan 15, 2025 02:00 UTC   364d
# scheduler.conf             Jan 15, 2025 02:00 UTC   364d

# ── RENEW ALL CERTIFICATES ────────────────────────────────────────────
# Renew all certs at once
kubeadm certs renew all

# Renew specific cert
kubeadm certs renew apiserver
kubeadm certs renew admin.conf

# After renewal: restart control plane components
# (static pods pick up new certs automatically)
systemctl restart kubelet

# Update your kubeconfig with renewed admin cert
cp /etc/kubernetes/admin.conf ~/.kube/config

# ── INSPECT A CERTIFICATE MANUALLY ────────────────────────────────────
openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -dates -subject -issuer

# openssl for a kubeconfig embedded cert:
kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}' | \
  base64 -d | openssl x509 -noout -dates

# ── CREATE CLIENT CERTIFICATE FOR USER ───────────────────────────────
# (See Chapter 8 for full user certificate flow)
```

---

## 10.9 Node Maintenance — Complete Workflows

```bash
# ══════════════════════════════════════════════════════════════════════
# WORKFLOW 1: Planned Maintenance (kernel patch, hardware replacement)
# ══════════════════════════════════════════════════════════════════════

# Step 1: Cordon the node (no new pods)
kubectl cordon worker-2
kubectl get node worker-2
# STATUS: Ready,SchedulingDisabled

# Step 2: Drain all pods
kubectl drain worker-2 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=120 \        # 2 min for graceful termination
  --timeout=600s              # Total timeout: 10 minutes

# Step 3: Verify no regular pods on node
kubectl get pods -o wide --all-namespaces | grep worker-2
# Should only show DaemonSet pods (if --ignore-daemonsets was used)

# Step 4: Perform maintenance
ssh worker-2
  apt-get update && apt-get upgrade -y
  reboot
# Wait for node to come back up

# Step 5: Verify node is back
kubectl get node worker-2    # STATUS: Ready,SchedulingDisabled

# Step 6: Uncordon
kubectl uncordon worker-2
kubectl get node worker-2    # STATUS: Ready

# ══════════════════════════════════════════════════════════════════════
# WORKFLOW 2: Emergency Node Removal (hardware failure)
# ══════════════════════════════════════════════════════════════════════

# Force delete pods on failed node (they're already dead)
kubectl delete pods \
  --field-selector spec.nodeName=dead-worker-3 \
  --all-namespaces \
  --force --grace-period=0

# Remove the node from cluster
kubectl delete node dead-worker-3

# On the replacement node: kubeadm join (same as initial join)

# ══════════════════════════════════════════════════════════════════════
# WORKFLOW 3: Remove a node cleanly
# ══════════════════════════════════════════════════════════════════════

# From control plane: drain + delete
kubectl drain worker-4 --ignore-daemonsets --delete-emptydir-data
kubectl delete node worker-4

# On the worker node itself: reset kubeadm configuration
kubeadm reset
# This removes all Kubernetes configs, certs, and cleans up containers
# (but does NOT remove kubeadm, kubelet, kubectl packages)
```

---

## 10.10 Cluster Troubleshooting

### 10.10.1 Control Plane Component Failures

```bash
# ══════════════════════════════════════════════════════════════════════
# SYSTEMATIC CLUSTER TROUBLESHOOTING APPROACH
# ══════════════════════════════════════════════════════════════════════

# STEP 1: Can kubectl connect?
kubectl cluster-info
# If error: "The connection to the server was refused"
# → API server is down or kubeconfig is wrong

# STEP 2: Check control plane static pods
kubectl get pods -n kube-system | grep -E "apiserver|etcd|scheduler|controller"
# If all Pending/Error → static pod manifests may be misconfigured

# On the control plane node:
ls /etc/kubernetes/manifests/   # Are manifests present?
crictl ps                        # Are containers actually running?
crictl ps | grep kube            # See control plane containers

# STEP 3: Check kubelet is running (critical!)
systemctl status kubelet
journalctl -u kubelet -f --since "5 min ago"
# Common kubelet errors:
# "Unable to update cni config" → CNI not installed
# "Failed to start container" → image pull error or resource issue

# STEP 4: Check component logs
# API server logs
kubectl logs kube-apiserver-control-plane-1 -n kube-system
# or if kubectl doesn't work:
crictl logs $(crictl ps --name kube-apiserver -q)

# etcd logs
kubectl logs etcd-control-plane-1 -n kube-system
# or:
crictl logs $(crictl ps --name etcd -q)

# Scheduler logs
kubectl logs kube-scheduler-control-plane-1 -n kube-system

# Controller manager logs
kubectl logs kube-controller-manager-control-plane-1 -n kube-system

# STEP 5: Node-level diagnosis
kubectl get nodes
kubectl describe node <problematic-node>
# Look at: Conditions (MemoryPressure, DiskPressure, NetworkUnavailable)
# Look at: Events section

# STEP 6: Events (the most useful output for debugging)
kubectl get events -A --sort-by='.lastTimestamp' | tail -30
kubectl get events -n kube-system --sort-by='.lastTimestamp'
```

### 10.10.2 Common Control Plane Failure Scenarios

```bash
# ══════════════════════════════════════════════════════════════════════
# SCENARIO 1: kube-apiserver won't start
# ══════════════════════════════════════════════════════════════════════
crictl logs $(crictl ps --name kube-apiserver -q 2>/dev/null || \
  crictl ps -a --name kube-apiserver -q | head -1)

# Check the static pod manifest for syntax errors
cat /etc/kubernetes/manifests/kube-apiserver.yaml | python3 -c \
  "import sys,yaml; yaml.safe_load(sys.stdin)"

# Check certificate paths in manifest exist
grep "cert\|key\|ca" /etc/kubernetes/manifests/kube-apiserver.yaml | \
  awk '{print $NF}' | while read f; do
    [ -f "$f" ] && echo "EXISTS: $f" || echo "MISSING: $f"
  done

# ══════════════════════════════════════════════════════════════════════
# SCENARIO 2: etcd is unhealthy
# ══════════════════════════════════════════════════════════════════════
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check disk space (etcd can fail if disk is full)
df -h /var/lib/etcd

# Check etcd data dir permissions
ls -la /var/lib/etcd

# ══════════════════════════════════════════════════════════════════════
# SCENARIO 3: Worker node shows NotReady
# ══════════════════════════════════════════════════════════════════════
kubectl describe node worker-1 | grep -A10 Conditions
# NetworkUnavailable=True → CNI not working on node
# MemoryPressure=True → Node is out of memory
# DiskPressure=True → Disk is nearly full

# SSH to worker node
systemctl status kubelet            # Is kubelet running?
journalctl -u kubelet -f            # Kubelet errors

# Check CNI
ls /etc/cni/net.d/                  # CNI config present?
ls /opt/cni/bin/                    # CNI binaries present?

# Check network connectivity from node to control plane
curl -k https://192.168.1.10:6443/healthz   # Can reach API server?
```

---

## Chapter 10: Hands-On Labs

### Lab 10.1 — kubeadm Cluster Bootstrap (on VMs or cloud)

```bash
# This lab requires 3 VMs or cloud instances:
# 1x control-plane: 2 CPU, 2GB RAM, Ubuntu 22.04
# 2x workers: 2 CPU, 2GB RAM, Ubuntu 22.04

# === RUN ON ALL NODES ===
# Disable swap
swapoff -a && sed -i '/swap/d' /etc/fstab

# Kernel modules
modprobe overlay && modprobe br_netfilter
echo -e "overlay\nbr_netfilter" > /etc/modules-load.d/k8s.conf

# Sysctl
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
EOF
sysctl --system

# Install containerd
apt-get update && apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd && systemctl enable containerd

# Install kubeadm, kubelet, kubectl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' > \
  /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet=1.29.0-1.1 kubeadm=1.29.0-1.1 kubectl=1.29.0-1.1
apt-mark hold kubelet kubeadm kubectl

# === RUN ON CONTROL PLANE ONLY ===
kubeadm init --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=$(hostname -I | awk '{print $1}')

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Print join command
kubeadm token create --print-join-command

# === RUN ON WORKER NODES (with join command from above) ===
kubeadm join <CP_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>

# === VERIFY (from control plane) ===
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

### Lab 10.2 — etcd Backup and Restore Practice

```bash
# This is the most important lab — practice until it's automatic

# === SETUP: Create some test resources to verify restore ===
kubectl create namespace restore-test
kubectl create deployment before-backup \
  --image=nginx --replicas=3 --namespace=restore-test
kubectl create configmap backup-marker \
  --from-literal=created="$(date)" \
  --namespace=restore-test

kubectl get all -n restore-test     # Note what exists

# === BACKUP ===
ETCDCTL_API=3 etcdctl snapshot save /tmp/lab-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl snapshot status /tmp/lab-backup.db --write-out=table

# === SIMULATE DISASTER: Delete namespace ===
kubectl delete namespace restore-test
kubectl get namespaces | grep restore-test   # Gone!

# === RESTORE ===
# Step 1: Stop control plane
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
sleep 30

# Step 2: Restore to new data dir
ETCDCTL_API=3 etcdctl snapshot restore /tmp/lab-backup.db \
  --data-dir=/var/lib/etcd-lab-restore \
  --name=$(hostname) \
  --initial-cluster=$(hostname)=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-lab-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# Step 3: Update etcd manifest
sed 's|/var/lib/etcd|/var/lib/etcd-lab-restore|g' \
  /tmp/etcd.yaml > /etc/kubernetes/manifests/etcd.yaml
sleep 20

# Step 4: Restore other manifests
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
sleep 60

# Step 5: Verify
kubectl get namespace restore-test         # Should be back!
kubectl get all -n restore-test            # before-backup deployment back!
kubectl get cm backup-marker -n restore-test  # ConfigMap back!
```

### Lab 10.3 — Cluster Upgrade Practice

```bash
# On a kubeadm cluster running v1.28.x

# Step 0: Check current version
kubectl get nodes
kubeadm version

# Step 1: Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /tmp/pre-upgrade.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Step 2: Upgrade kubeadm
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.29.0-1.1
apt-mark hold kubeadm

# Step 3: Plan and apply
kubeadm upgrade plan
kubeadm upgrade apply v1.29.0 --yes

# Step 4: Upgrade kubelet on control plane
kubectl drain control-plane-1 --ignore-daemonsets
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon control-plane-1

# Verify
kubectl get nodes | grep control-plane   # Should show v1.29.0

# Step 5: Upgrade worker-1 (SSH to worker for apt commands)
kubectl cordon worker-1
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
# ... (SSH to worker-1 and run apt upgrade + kubeadm upgrade node)
kubectl uncordon worker-1
```

### Lab 10.4 — Certificate Check and Renewal

```bash
# Check all certificate expiry dates
kubeadm certs check-expiration

# Renew all certificates
kubeadm certs renew all

# Verify renewed
kubeadm certs check-expiration

# Update your kubeconfig with new certs
cp /etc/kubernetes/admin.conf ~/.kube/config

# Restart control plane components to pick up new certs
kubectl delete pod -n kube-system \
  -l component=kube-apiserver \
  -l component=kube-controller-manager \
  -l component=kube-scheduler
# (or: systemctl restart kubelet which restarts static pods)

kubectl get pods -n kube-system   # Verify all come back up
```

---

## Chapter 10: Troubleshooting Guide

### Issue 1: kubeadm init fails — preflight errors

```bash
kubeadm init
# [ERROR Swap]: running with swap on is not supported

# Fix: Disable swap
swapoff -a

# [ERROR CRI]: container runtime is not running
# Fix: Check containerd is running
systemctl status containerd
systemctl start containerd

# [ERROR Port-6443]: Port 6443 is in use
# Fix: Something else is using API server port
ss -tlnp | grep 6443
# Kill that process or use --apiserver-bind-port=6444

# [ERROR NumCPU]: the number of available CPUs 1 is less than the required 2
# Fix: Add more CPUs to VM or use --ignore-preflight-errors=NumCPU (dev only)
```

### Issue 2: Worker node can't join cluster

```bash
kubeadm join ...
# error: couldn't validate the identity of the API Server

# CAUSE 1: Expired token (tokens expire after 24 hours)
# On control plane: regenerate token
kubeadm token create --print-join-command

# CAUSE 2: Wrong CA cert hash
# Get correct hash:
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
```

### Issue 3: etcd restore failed — API server won't come back

```bash
# Check if etcd is actually running
crictl ps | grep etcd

# Check etcd logs
crictl logs $(crictl ps --name etcd -q 2>/dev/null || \
  crictl ps -a --name etcd -q | head -1)

# COMMON ERROR: "data dir already exists"
# This means the restore data dir already exists from a previous attempt
rm -rf /var/lib/etcd-restore
# Then re-run the restore command

# COMMON ERROR: wrong --initial-cluster value
# Must match the node name exactly
hostname   # Get exact hostname
# Use this in --name AND in --initial-cluster flags

# COMMON ERROR: API server still connecting to old etcd dir
grep "data-dir" /etc/kubernetes/manifests/etcd.yaml
# Must show the NEW restore directory, not the old one

# VERIFY etcd is responding before restoring apiserver
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Issue 4: Upgrade stuck — node not coming back Ready

```bash
kubectl get nodes
# worker-1   NotReady   ...

# SSH to worker-1
systemctl status kubelet
journalctl -u kubelet | tail -20

# COMMON CAUSE: kubelet version mismatch
kubelet --version   # Is it the new version?
# If not: re-run the apt install commands

# COMMON CAUSE: kubelet not restarted after upgrade
systemctl daemon-reload
systemctl restart kubelet
```

---

## Chapter 10: Interview Questions

**Q1: What is kubeadm and what does kubeadm init do?**

> kubeadm is the official Kubernetes cluster bootstrap tool. `kubeadm init` performs: (1) preflight checks (swap disabled, ports available, container runtime running); (2) generates all PKI certificates and kubeconfig files; (3) creates static pod manifests for control plane components; (4) bootstraps etcd; (5) configures the API server, scheduler, and controller manager; (6) installs CoreDNS and kube-proxy; (7) outputs a join command for workers. It follows Kubernetes best practices and is the recommended way to create production clusters.

**Q2: What is the order of operations for a Kubernetes cluster upgrade?**

> (1) Back up etcd — always first, non-negotiable. (2) Upgrade kubeadm on the control plane. (3) Run `kubeadm upgrade plan` to verify the upgrade path. (4) Run `kubeadm upgrade apply <version>` to upgrade control plane components. (5) Upgrade kubelet and kubectl on the control plane. (6) For each worker: cordon → drain → upgrade kubeadm → kubeadm upgrade node → upgrade kubelet/kubectl → uncordon. Always upgrade one minor version at a time (1.27→1.28→1.29, not 1.27→1.29).

**Q3: Where are Kubernetes control plane component configurations stored?**

> Control plane components (API server, etcd, scheduler, controller manager) run as **static pods** — their manifests are in `/etc/kubernetes/manifests/`. kubelet watches this directory and automatically starts/restarts/stops pods based on changes. Certificates are in `/etc/kubernetes/pki/`. kubeconfig files for components are in `/etc/kubernetes/*.conf`. etcd data is in `/var/lib/etcd/`.

**Q4: How do you take an etcd backup and what does it contain?**

> Use `etcdctl snapshot save <file>` with TLS flags pointing to etcd's CA cert, server cert, and server key. The snapshot is a point-in-time copy of the entire etcd keyspace — every Kubernetes object (pods, deployments, services, secrets, RBAC, PVs, PVCs, configmaps, namespaces) is stored. The backup file is a binary snapshot of the etcd B-tree database. It should be stored off-cluster and tested regularly with restore drills.

**Q5: Walk me through restoring an etcd backup.**

> (1) Stop the API server, scheduler, and controller manager by moving their manifests out of `/etc/kubernetes/manifests/`. (2) Stop etcd the same way. (3) Run `etcdctl snapshot restore <file> --data-dir=<new-dir>` with the node's name and cluster config. (4) Edit the etcd static pod manifest to point `--data-dir` to the new restore directory. (5) Move etcd manifest back to trigger restart. (6) Verify etcd is healthy. (7) Move the other control plane manifests back. (8) Verify the cluster is fully restored.

**Q6: How do you renew Kubernetes certificates with kubeadm?**

> Run `kubeadm certs check-expiration` to see current expiry dates. Run `kubeadm certs renew all` to renew all certificates at once. After renewal, copy the updated `/etc/kubernetes/admin.conf` to `~/.kube/config`, and restart kubelet (`systemctl restart kubelet`) so control plane static pods reload with new certificates. Certificates can also be renewed automatically when performing a cluster upgrade with kubeadm.

**Q7: What is the Raft consensus algorithm and why does it matter for etcd?**

> Raft is a distributed consensus algorithm that ensures all etcd nodes agree on the same data. In a cluster of N etcd nodes, writes require a quorum of ⌊N/2⌋+1 nodes to agree. With 3 nodes: 2 must agree (tolerates 1 failure). With 5 nodes: 3 must agree (tolerates 2 failures). This matters because if too many etcd nodes fail (losing quorum), the cluster becomes read-only — no writes can be made and new pods cannot be scheduled. This is why production clusters run 3 or 5 etcd nodes, never 2 or 4.

**Q8: What happens to running pods when the control plane goes down?**

> Running pods continue running. kubelet on each worker node operates independently — it maintains pod lifecycle based on its local state and does not require constant API server contact to keep pods running. However, NO NEW pods can be scheduled, existing pods cannot be updated or scaled, failed pods cannot be replaced, and kubectl commands fail. This is why control plane HA (3 control plane nodes) is critical for production — if one control plane goes down, the other two maintain cluster management capability.

---

## CKA Exam Notes — Chapter 10

```
╔════════════════════════════════════════════════════════════════════╗
║                  CKA EXAM FOCUS — CHAPTER 10                      ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: Cluster Architecture & Admin = ~25%                 ║
║  etcd backup/restore + upgrade = highest-value exam tasks         ║
║                                                                    ║
║  ETCD BACKUP ONE-LINER (memorize letter-perfect):                  ║
║  ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db          ║
║    --endpoints=https://127.0.0.1:2379                             ║
║    --cacert=/etc/kubernetes/pki/etcd/ca.crt                       ║
║    --cert=/etc/kubernetes/pki/etcd/server.crt                     ║
║    --key=/etc/kubernetes/pki/etcd/server.key                      ║
║                                                                    ║
║  ETCD RESTORE ONE-LINER:                                           ║
║  ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db       ║
║    --data-dir=/var/lib/etcd-restore                               ║
║                                                                    ║
║  CLUSTER UPGRADE SEQUENCE (5 steps):                              ║
║  1. BACKUP etcd                                                    ║
║  2. apt install kubeadm=<new> → kubeadm upgrade apply <ver>       ║
║  3. apt install kubelet=<new> kubectl=<new> → restart kubelet     ║
║  4. For each worker: cordon → drain → ssh → upgrade → uncordon    ║
║  5. kubectl get nodes → all should show new version               ║
║                                                                    ║
║  FINDING ETCD CERT PATHS (critical for exam):                     ║
║  cat /etc/kubernetes/manifests/etcd.yaml | grep "\-\-"            ║
║  Look for: --cert-file, --key-file, --trusted-ca-file             ║
║  Also check: --listen-client-urls for endpoint                    ║
║                                                                    ║
║  KEY FILE LOCATIONS (memorize):                                    ║
║  Static pod manifests: /etc/kubernetes/manifests/                 ║
║  PKI certs:            /etc/kubernetes/pki/                       ║
║  etcd certs:           /etc/kubernetes/pki/etcd/                  ║
║  etcd data:            /var/lib/etcd/                             ║
║  kubelet config:       /var/lib/kubelet/config.yaml               ║
║  admin kubeconfig:     /etc/kubernetes/admin.conf                 ║
║                                                                    ║
║  EXAM TRAPS:                                                       ║
║  Always verify backup: etcdctl snapshot status <file>             ║
║  Restore MUST use --data-dir (new dir, not old /var/lib/etcd)     ║
║  etcd manifest hostPath MUST also be updated after restore        ║
║  Upgrade: cordon+drain worker BEFORE upgrading its packages       ║
║  Upgrade: kubeadm upgrade NODE (not apply) on workers             ║
║  Certs expire in 1 year — check with: kubeadm certs check-expiry  ║
║                                                                    ║
║  CONTEXT CHECK REMINDER:                                           ║
║  kubectl config current-context ← Run before EVERY exam task!    ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Common Mistakes — Chapter 10

| Mistake | Why It Happens | How to Avoid |
|---|---|---|
| Skipping etcd backup before upgrade | "It'll be fine" | Non-negotiable: backup → upgrade, never skip |
| Using old join token (expired) | Tokens expire after 24h | Always regenerate: `kubeadm token create --print-join-command` |
| Restoring to original `/var/lib/etcd` | Seems simpler | Always restore to NEW directory; original may be corrupted |
| Forgetting to update hostPath in etcd.yaml | Only updates `--data-dir` flag | Both the `--data-dir` arg AND the volume hostPath must point to new dir |
| Upgrading workers before control plane | Skipping order | Control plane ALWAYS upgraded first |
| Not apt-marking hold after upgrade | Next `apt upgrade` breaks cluster | Always `apt-mark hold kubelet kubeadm kubectl` |
| `kubeadm upgrade apply` on workers | Workers use `kubeadm upgrade node` | `apply` only on first control plane; `node` on all workers |
| Not draining before worker upgrade | Rolling upgrade while under load | Always cordon + drain worker before touching packages |
| Not restarting kubelet after kubelet upgrade | New binary not loaded | Always `systemctl daemon-reload && systemctl restart kubelet` |

---

## Chapter 10 Summary

1. **Cluster topology** — single vs HA control plane; etcd quorum (3 nodes = tolerate 1 failure)
2. **kubeadm init** — prerequisites (swap, sysctl, containerd) → init → CNI → workers join
3. **Key file locations** — manifests (`/etc/kubernetes/manifests/`), PKI (`/etc/kubernetes/pki/`), etcd data (`/var/lib/etcd/`)
4. **kubeconfig** — manages cluster access; context switching; merging multiple configs
5. **Cluster upgrade** — backup first; one minor version at a time; control plane first then workers
6. **Worker upgrade** — cordon → drain → upgrade packages → kubeadm upgrade node → uncordon
7. **etcd fundamentals** — Raft consensus; ports 2379/2380; stores all cluster state
8. **etcd backup** — `etcdctl snapshot save` with all TLS flags; verify with `snapshot status`
9. **etcd restore** — stop cluster → restore to new dir → update manifest → restart → verify
10. **Certificate management** — expire in 1 year; `kubeadm certs check-expiration`; `kubeadm certs renew all`
11. **Node maintenance** — cordon (stop new), drain (evict existing), maintenance, uncordon
12. **Cluster troubleshooting** — static pods, kubelet status, crictl, component logs

---

*Next: Chapter 11 — Production Kubernetes: HA, GitOps, Helm & ArgoCD*

*"You can build it, upgrade it, and restore it from disaster.
 Now let's talk about running Kubernetes the way world-class engineering teams do."*
