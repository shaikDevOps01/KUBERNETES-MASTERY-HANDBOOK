<div align="center">
```
██╗  ██╗ █████╗ ███████╗    ███╗   ███╗ █████╗ ███████╗████████╗███████╗██████╗ ██╗   ██╗
██║ ██╔╝██╔══██╗██╔════╝    ████╗ ████║██╔══██╗██╔════╝╚══██╔══╝██╔════╝██╔══██╗╚██╗ ██╔╝
█████╔╝ ╚█████╔╝███████╗    ██╔████╔██║███████║███████╗   ██║   █████╗  ██████╔╝ ╚████╔╝
██╔═██╗ ██╔══██╗╚════██║    ██║╚██╔╝██║██╔══██║╚════██║   ██║   ██╔══╝  ██╔══██╗  ╚██╔╝
██║  ██╗╚█████╔╝███████║    ██║ ╚═╝ ██║██║  ██║███████║   ██║   ███████╗██║  ██║   ██║
╚═╝  ╚═╝ ╚════╝ ╚══════╝    ╚═╝     ╚═╝╚═╝  ╚═╝╚══════╝   ╚═╝   ╚══════╝╚═╝  ╚═╝   ╚═╝
                              H A N D B O O K

```

# 📘 Kubernetes Mastery Handbook
### *From Absolute Zero to CKA Certified Administrator*

<br/>

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.29+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![CKA](https://img.shields.io/badge/CKA-Exam%20Ready-blue?logo=linux-foundation&logoColor=white)](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)
[![Chapters](https://img.shields.io/badge/Chapters-12%20Complete-brightgreen)](./chapters/)
[![Lines](https://img.shields.io/badge/Lines%20of%20Content-18%2C000%2B-orange)](./chapters/)
[![Size](https://img.shields.io/badge/Total%20Size-848KB-blue)](./chapters/)
[![Star this repo](https://img.shields.io/github/stars/shaikDevOps01/KUBERNETES-MASTERY-HANDBOOK?style=for-the-badge&logo=github&label=Star%20this%20Handbook)](https://github.com/shaikDevOps01/KUBERNETES-MASTERY-HANDBOOK)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](http://makeapullrequest.com)

<br/>

> **"The journey of a thousand pods begins with a single kubectl command."**

<br/>

**Written by a DevOps Engineer. For engineers who mean business.**
Every concept in plain English first, then technical language.
Every chapter has diagrams, production YAML, labs, troubleshooting, and CKA exam tips.

<br/>

[📖 Start Reading](#-table-of-contents) · [🚀 Quick Start](#-quick-start-lab-setup) · [📊 Stats](#-handbook-statistics) · [🎯 CKA Prep](#-cka-exam-overview) · [🤝 Contribute](#-contributing)

</div>

---

## 🌟 Why This Handbook Exists

Most Kubernetes learning resources fall into one of two failure modes:

**Too shallow** — "Here's a pod YAML, good luck" with no real depth or context.
**Too scattered** — Blog posts that contradict each other, outdated docs, no clear progression.

This handbook is written as a **professional technical book** — not scattered notes. It follows one principle: *everything you need to go from zero Kubernetes knowledge to passing the CKA exam, in one place, in the right order.*

---

## ✅ What Makes This Different

| Feature | This Handbook |
|---|---|
| 📐 **Progressive structure** | Absolute beginner → CKA-ready, one chapter at a time |
| 🗣️ **Dual-mode explanations** | Plain English first, technical language second — always |
| 🖼️ **ASCII architecture diagrams** | Every concept visualised — no external tools needed |
| 🏭 **Production-grade YAML** | Real patterns from live clusters, not toy examples |
| 🔬 **Hands-on labs** | Every chapter has working, tested exercises |
| 🐛 **Troubleshooting guides** | Systematic debugging for every topic area |
| 💼 **Interview preparation** | 8 curated Q&A per chapter with full answers |
| 📌 **CKA exam callouts** | Speed commands, YAML skeletons, common traps |
| ⚠️ **Common mistakes table** | What goes wrong and how to avoid it |

---

## 📊 Handbook Statistics

```
 12 Parts  ·  12 Chapters  ·  18,265 Lines  ·  848 KB  ·  100% Complete
```

| # | Chapter | Lines | Size | Topics |
|---|---|---|---|---|
| 1 | Kubernetes Fundamentals | 1,054 | 65 KB | Architecture, Control Plane, Worker Nodes, kubectl |
| 2 | Core Objects | 1,465 | 75 KB | Pods, ReplicaSets, Deployments, Namespaces, Labels |
| 3 | Networking | 1,722 | 83 KB | Services, Ingress, DNS, CNI, NetworkPolicies |
| 4 | Storage | 1,150 | 46 KB | PV, PVC, StorageClass, Dynamic Provisioning, Snapshots |
| 5 | Configuration | 1,351 | 55 KB | ConfigMaps, Secrets, Downward API, Projected Volumes |
| 6 | Workload Management | 1,481 | 61 KB | DaemonSets, StatefulSets, Jobs, CronJobs |
| 7 | Scheduling | 1,619 | 71 KB | Affinity, Taints, Resources, QoS, Priority Classes |
| 8 | Security | 1,687 | 74 KB | RBAC, ServiceAccounts, SecurityContexts, PSS, Admission |
| 9 | Monitoring & Logging | 1,612 | 70 KB | Prometheus, Grafana, HPA, Fluent Bit, Loki |
| 10 | Cluster Administration | 1,637 | 81 KB | kubeadm, Upgrade, etcd Backup/Restore, Certificates |
| 11 | Production Kubernetes | 1,376 | 59 KB | HA, Helm, Kustomize, GitOps, ArgoCD |
| 12 | CKA Preparation | 1,450 | 57 KB | Exam Strategy, Speed Labs, Mock Exam, Scenarios |
| — | README | 661 | 25 KB | This file |
| **TOTAL** | **13 files** | **18,265** | **848 KB** | **Complete handbook** |

---

## 📚 Table of Contents

<details open>
<summary><b>📗 Part 1 — Kubernetes Fundamentals</b> · Ch 1 · 1,054 lines</summary>

### [Chapter 1: What Is Kubernetes and Why Does It Exist?](./chapters/ch1-kubernetes-fundamentals.md)

**What you will learn:**
- The problem Kubernetes solves — and why it exists at all
- Evolution of deployment: Physical Servers → VMs → Containers → Orchestration
- Containerization basics: Linux namespaces, cgroups, OverlayFS
- Docker vs Kubernetes — the real relationship (they are complementary, not competing)
- Complete Kubernetes cluster architecture with ASCII diagrams
- Control Plane: API Server, etcd, Scheduler, Controller Manager — deep dive
- Worker Nodes: kubelet, kube-proxy, Container Runtime — deep dive
- The full request lifecycle: from `kubectl apply` to a running container (7 steps)
- kubectl setup, kubeconfig structure, context management

**Labs:** Minikube/kind cluster setup · Explore system pods · Your first pod

</details>

<details open>
<summary><b>📗 Part 2 — Kubernetes Core Objects</b> · Ch 2 · 1,465 lines</summary>

### [Chapter 2: Pods, ReplicaSets, Deployments, Namespaces, Labels & Annotations](./chapters/ch2-core-objects.md)

**What you will learn:**
- Pods: anatomy (including the invisible pause container), lifecycle states, multi-container patterns
- Sidecar, Ambassador, and Adapter container patterns with real use cases
- All three health probes: Liveness, Readiness, Startup — when to use each
- Namespaces: logical isolation, ResourceQuota, LimitRange, cross-namespace DNS
- Labels and Selectors: equality-based and set-based selectors, production naming conventions
- Annotations: rich metadata vs labels — what each is for
- ReplicaSets: reconciliation loop, ownership via labels, why you rarely use them directly
- Deployments: the full hierarchy, RollingUpdate vs Recreate, maxSurge/maxUnavailable
- Rolling updates, rollbacks, revision history, change-cause annotations

**Labs:** Pod deep dive with shared volumes · Namespace/label isolation · Full deployment lifecycle (create → update → rollback → scale)

</details>

<details open>
<summary><b>📗 Part 3 — Networking</b> · Ch 3 · 1,722 lines</summary>

### [Chapter 3: Services, Ingress, DNS, CNI & Network Policies](./chapters/ch3-networking.md)

**What you will learn:**
- The Kubernetes network model: 4 rules, 3 IP ranges (Node/Pod/Service)
- CNI plugin comparison: Flannel vs Calico vs Cilium vs Weave — when to use each
- How VXLAN packet encapsulation actually works across nodes (with ASCII trace)
- All 4 Service types: ClusterIP, NodePort, LoadBalancer, ExternalName — full diagrams
- How Endpoints objects work and why empty endpoints = selector mismatch
- Ingress: 4 patterns (single-service, path-based, host-based, TLS/HTTPS)
- CoreDNS: FQDN formats, short names, cross-namespace resolution rules
- Headless services: when and why to use them (StatefulSets, direct pod DNS)
- Network Policies: default-deny baseline, ingress/egress rules, namespace selectors
- Complete 3-tier application with services, ingress, and network policies

**Labs:** All service types · Cross-namespace DNS testing · Ingress path routing · NetworkPolicy allow/block testing

</details>

<details open>
<summary><b>📗 Part 4 — Storage</b> · Ch 4 · 1,150 lines</summary>

### [Chapter 4: Volumes, PersistentVolumes, PVCs & StorageClasses](./chapters/ch4-storage.md)

**What you will learn:**
- Why ephemeral container storage is dangerous and when it destroys your data
- All volume types: emptyDir, hostPath, configMap, secret, downwardAPI, CSI
- PersistentVolumes: cluster-level storage, lifecycle (Available → Bound → Released)
- Reclaim policies: Retain (safe for production) vs Delete (dangerous for databases)
- Access modes: RWO, ROX, RWX, RWOP — capabilities and real storage limitations
- PVCs: the binding algorithm, storageClassName matching, 1:1 exclusive binding
- StorageClasses: dynamic provisioning, CSI drivers for AWS/GCP/NFS/local
- `WaitForFirstConsumer` — why this matters for zone-aware cloud storage
- Volume expansion, PVC resizing, Volume Snapshots for disaster recovery
- Complete production MySQL deployment with all storage best practices

**Labs:** emptyDir shared volumes · Static PV/PVC binding · Dynamic provisioning proof

</details>

<details open>
<summary><b>📗 Part 5 — Configuration Management</b> · Ch 5 · 1,351 lines</summary>

### [Chapter 5: ConfigMaps, Secrets & Environment Variables](./chapters/ch5-configuration.md)

**What you will learn:**
- Why hard-coding config into images is the single biggest production anti-pattern
- The Twelve-Factor App Rule #3 and how Kubernetes implements it
- ConfigMaps: 4 creation methods, all 3 injection methods (env, envFrom, volume)
- Volume mount auto-update: why only file mounts update live (not env vars)
- Secrets: base64 is encoding NOT encryption — what this means for security
- All Secret types: Opaque, TLS, docker-registry, service-account-token
- The three injection methods compared by security level (volume is safest)
- Downward API: pods accessing their own metadata without calling the API server
- Projected volumes: combining ConfigMap + Secret + DownwardAPI in one mount
- Production strategy: external secrets managers (Vault, AWS SM, GCP SM)
- The `echo -n` rule: always avoid newlines in base64-encoded secrets

**Labs:** ConfigMap volume with live nginx reload · Secret injection and tmpfs verification · Downward API pod metadata access

</details>

<details open>
<summary><b>📗 Part 6 — Workload Management</b> · Ch 6 · 1,481 lines</summary>

### [Chapter 6: DaemonSets, StatefulSets, Jobs & CronJobs](./chapters/ch6-workloads.md)

**What you will learn:**
- The workload spectrum: stateless → node-bound → stateful → batch
- DaemonSets: one pod per node, auto-created on new nodes, real-world use cases
- DaemonSet tolerations for control-plane nodes, nodeSelector for targeting
- StatefulSets: the three guarantees (stable identity, stable storage, ordered lifecycle)
- Headless services: why StatefulSets require them and how pod DNS works
- volumeClaimTemplates: auto-creating individual PVCs per pod
- StatefulSet partition: canary upgrades for stateful applications
- Jobs: all 4 patterns (single, sequential, parallel, indexed) with use cases
- The `restartPolicy: OnFailure` rule — why it cannot be Always
- CronJobs: cron syntax reference, concurrencyPolicy (Forbid/Allow/Replace)
- Complete Redis Cluster and Kafka StatefulSet examples
- Manual CronJob trigger: `kubectl create job --from=cronjob/<name>`

**Labs:** DaemonSet node monitoring · StatefulSet ordered startup proof · DB migration Job · CronJob with manual trigger

</details>

<details open>
<summary><b>📗 Part 7 — Scheduling</b> · Ch 7 · 1,619 lines</summary>

### [Chapter 7: Node Selectors, Affinity, Taints, Tolerations & Resources](./chapters/ch7-scheduling.md)

**What you will learn:**
- The Scheduler's two phases: Filtering (eliminate) and Scoring (rank) — detailed
- NodeSelector: simple label equality targeting, AND logic between selectors
- Node Affinity: required (hard) vs preferred (soft), all operators, OR between terms
- Pod Affinity: schedule near matching pods, topologyKey for scope definition
- Pod Anti-Affinity: HA spread patterns for databases and critical services
- TopologySpreadConstraints: the modern way to spread pods across zones/nodes
- Taints: three effects (NoSchedule, PreferNoSchedule, NoExecute) with production patterns
- Tolerations: Equal vs Exists operators, tolerationSeconds for graceful node failure
- 5 production taint patterns: GPU nodes, prod/test isolation, maintenance, spot nodes
- Resource Requests vs Limits: CPU throttling vs memory OOMKill behaviour
- QoS Classes: Guaranteed, Burstable, BestEffort — eviction priority under pressure
- Priority Classes and preemption — protecting critical workloads
- cordon/drain/uncordon workflow for node maintenance

**Labs:** NodeSelector targeting · Taint/toleration blocking test · QoS class assignment verification · Pod anti-affinity spreading

</details>

<details open>
<summary><b>📗 Part 8 — Security</b> · Ch 8 · 1,687 lines</summary>

### [Chapter 8: RBAC, Service Accounts, Security Contexts & Admission Controllers](./chapters/ch8-security.md)

**What you will learn:**
- The four security layers: Authentication → Authorisation → Admission → Runtime
- RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding — the valid combinations
- RBAC verbs, API groups, and the built-in roles (cluster-admin, admin, edit, view)
- Creating users with certificate-based authentication (CSR → approve → kubeconfig)
- ServiceAccounts: pod identity, auto-mounted tokens, TokenRequest API
- The SA format rule: `namespace:sa-name` in bindings — the #1 exam trap
- `kubectl auth can-i` mastery: testing any user or ServiceAccount's permissions
- Security Contexts: pod-level vs container-level settings
- `allowPrivilegeEscalation: false` — why this must ALWAYS be set
- Linux capabilities: drop ALL, add back only NET_BIND_SERVICE if needed
- `readOnlyRootFilesystem: true` + emptyDir volumes for writable paths
- Pod Security Standards: Privileged / Baseline / Restricted applied to namespaces
- Admission Controllers: mutating vs validating webhooks, Kyverno policy examples
- etcd Encryption at Rest: EncryptionConfiguration for true Secret security

**Labs:** RBAC user with limited permissions · ServiceAccount + RBAC + Pod · Security context enforcement · NetworkPolicy allow/block

</details>

<details open>
<summary><b>📗 Part 9 — Monitoring & Logging</b> · Ch 9 · 1,612 lines</summary>

### [Chapter 9: Metrics Server, Prometheus, Grafana & Logging](./chapters/ch9-monitoring-logging.md)

**What you will learn:**
- The three pillars of observability: Metrics, Logs, Traces — role of each
- Metrics Server: architecture (cAdvisor → kubelet → Metrics Server → HPA)
- All `kubectl top` variations: nodes, pods, containers, sort options
- HPA: the scaling formula, v2 YAML with CPU + memory + custom metrics
- HPA `behavior` block: controlling scale-up speed vs scale-down stabilisation
- Prometheus: pull-based architecture, scrape targets, TSDB storage
- kube-prometheus-stack: Helm install — Prometheus + Grafana + AlertManager in one
- ServiceMonitor and PodMonitor: declarative scrape configuration
- PrometheusRule: alerting rules, recording rules, `for` duration, severity labels
- AlertManager: routing to Slack + PagerDuty, grouping, deduplication
- 10 essential PromQL queries: CPU, memory, error rate, P99 latency
- Grafana: popular dashboard IDs (3119, 1860, 6417), GitOps provisioning via ConfigMap
- The three logging levels: Application → Node → Cluster-level (only level 3 survives failures)
- All `kubectl logs` flags including the critical `--previous`
- Fluent Bit DaemonSet: complete production YAML with ConfigMap and RBAC
- EFK vs PLG stack comparison: cost, storage, query language, K8s fit

**Labs:** Metrics Server + HPA load test · kubectl logs deep dive · Prometheus + Grafana install · PrometheusRule alert test

</details>

<details open>
<summary><b>📗 Part 10 — Cluster Administration</b> · Ch 10 · 1,637 lines</summary>

### [Chapter 10: kubeadm, Cluster Upgrade, etcd Backup & Restore](./chapters/ch10-cluster-administration.md)

**What you will learn:**
- Single control plane vs HA topology — when each is appropriate
- etcd Raft quorum: 3 nodes tolerates 1 failure, 5 tolerates 2
- All prerequisites for kubeadm: swap, sysctl, containerd, cgroup driver
- Step-by-step: `kubeadm init` → CNI install → worker node join
- Every key file location: manifests, PKI, etcd data, kubelet config
- kubeconfig management: contexts, credential setup, merging multiple configs
- Cluster upgrade: one minor version at a time — the rule you cannot break
- Full upgrade sequence: etcd backup → CP upgrade → worker drain/upgrade/uncordon
- etcd deep dive: what is stored, key format, ports 2379/2380, Raft leadership
- **etcd backup**: the complete command with all TLS flags — memorised
- **etcd restore**: 6-step procedure, stop cluster → restore to new dir → update manifest → restart
- Certificate management: checking expiry, renewing with kubeadm
- Node maintenance: cordon/drain/uncordon workflows for planned and emergency scenarios
- Cluster troubleshooting: static pods, crictl, kubelet logs, component failures

**Labs:** Full cluster bootstrap from scratch · etcd backup/restore drill · Cluster upgrade 1.28 → 1.29 · Certificate renewal

</details>

<details open>
<summary><b>📗 Part 11 — Production Kubernetes</b> · Ch 11 · 1,376 lines</summary>

### [Chapter 11: High Availability, Helm, Kustomize, GitOps & ArgoCD](./chapters/ch11-production-kubernetes.md)

**What you will learn:**
- Production HA architecture: multi-AZ control plane + multi-AZ workers + external LB
- PodDisruptionBudgets: protecting service availability during node drains and upgrades
- Vertical Pod Autoscaler (VPA): auto-adjusting resource requests from historical data
- Cluster Autoscaler: auto-scaling nodes based on pending pods and utilisation
- Helm: charts, releases, values, upgrade, rollback — the full lifecycle
- Creating a production Helm chart with templates, HPA, and dependent subcharts
- Kustomize: base + overlays pattern — no templates, just patches
- GitOps: the 4 principles, Git as the single source of truth
- ArgoCD architecture: Repo Server, Application Controller, API Server
- ArgoCD Application manifest: source, destination, syncPolicy
- `selfHeal: true` and `prune: true` — automated drift correction
- App of Apps pattern: bootstrapping an entire cluster from one Git commit
- The production readiness checklist: 25 items across availability, security, observability
- The perfect production pod manifest: every best practice in one YAML

**Labs:** Helm install + upgrade + rollback · Kustomize dev/prod overlays · ArgoCD setup + first GitOps application

</details>

<details open>
<summary><b>📗 Part 12 — CKA Preparation</b> · Ch 12 · 1,450 lines</summary>

### [Chapter 12: Exam Strategy, Speed Labs, Troubleshooting Scenarios & Mock Exam](./chapters/ch12-cka-preparation.md)

**What you will learn:**
- The 3-minute exam setup: aliases, vim config, context check
- The 6-step question handling method — for every single task
- Time management: Pass 1 (quick wins) → Pass 2 (medium) → Pass 3 (hard)
- How to use Kubernetes docs effectively: which pages to bookmark
- Top 20 exam tasks ranked by frequency with time estimates
- Speed labs: 8 drills, each under 3 minutes — pods, deployments, services, RBAC, etcd backup, PV/PVC, NetworkPolicy, SecurityContext
- 5 full troubleshooting scenarios: broken node, service not routing, broken static pod, etcd restore, RBAC forbidden
- Complete 17-question mock exam with solutions and explanations
- The complete kubectl reference card organised by category
- Final pre-exam checklist: technical + logistics + mindset
- The CKA in one page — the last thing to read before you start

</details>

---

## 🚀 Quick Start — Lab Setup

### Prerequisites

| Tool | Purpose | Minimum Version |
|---|---|---|
| `kubectl` | Kubernetes CLI | v1.28+ |
| `minikube` | Local single-node cluster | v1.32+ |
| `kind` | Local multi-node cluster | v0.22+ |
| `docker` | Container runtime | v24+ |
| `helm` | Package manager | v3.14+ |

### Option A — Minikube (Recommended for Beginners)

```bash
# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start a 2-node cluster
minikube start --cpus=2 --memory=4096 --nodes=2

# Enable essential addons
minikube addons enable metrics-server
minikube addons enable ingress

# Verify
kubectl get nodes
# NAME            STATUS   ROLES           AGE   VERSION
# minikube        Ready    control-plane   90s   v1.29.0
# minikube-m02    Ready    <none>          60s   v1.29.0
```

### Option B — kind (Multi-Node Simulation)

```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Create a realistic multi-node cluster
cat <<EOF | kind create cluster --config=- --name=cka-lab
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF

# Verify
kubectl get nodes
# NAME                    STATUS   ROLES           AGE   VERSION
# cka-lab-control-plane   Ready    control-plane   90s   v1.29.0
# cka-lab-worker          Ready    <none>          60s   v1.29.0
# cka-lab-worker2         Ready    <none>          60s   v1.29.0
# cka-lab-worker3         Ready    <none>          60s   v1.29.0
```

### Terminal Setup — Productivity Aliases

Add these to `~/.bashrc` or `~/.zshrc`:

```bash
# Essential kubectl shortcuts
alias k=kubectl
alias kgp='kubectl get pods'
alias kgd='kubectl get deployments'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kga='kubectl get all'
alias kdp='kubectl describe pod'
alias klog='kubectl logs'

# CKA exam aliases (paste at start of every exam session)
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'

# Enable kubectl tab completion
source <(kubectl completion bash)
complete -F __start_kubectl k
```

---

## 🎯 CKA Exam Overview

### Exam Format

| Property | Detail |
|---|---|
| **Format** | 100% hands-on, performance-based, online proctored |
| **Duration** | 2 hours |
| **Questions** | 15–20 tasks across 6 live clusters |
| **Pass Score** | 66% |
| **Open Book** | Yes — kubernetes.io/docs, kubernetes.io/blog, github.com/kubernetes |
| **Retake** | 1 free retake included |
| **Validity** | 3 years from passing date |
| **Cost** | $395 USD |
| **Provider** | Linux Foundation + PSI Secure Browser |

### Domain Weights

```
Troubleshooting                        30%  ██████████████████████████████░░░░░░░░░░░░
Cluster Architecture, Install, Config  25%  █████████████████████████░░░░░░░░░░░░░░░░░
Services & Networking                  20%  ████████████████████░░░░░░░░░░░░░░░░░░░░░░
Workloads & Scheduling                 15%  ███████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░
Storage                                10%  ██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
```

### The 5 Commands That Matter Most in the Exam

```bash
# 1. Context switch — run before EVERY question
kubectl config use-context <context-name>

# 2. Generate YAML without creating (fastest way to write manifests)
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml

# 3. Verify your answer worked
kubectl get pods -n <namespace> -o wide

# 4. Debug what's wrong
kubectl describe pod <pod-name> -n <namespace>

# 5. Check permissions
kubectl auth can-i <verb> <resource> --as <user> -n <namespace>
```

---

## 🗂️ Repository Structure

```
kubernetes-mastery-handbook/
│
├── README.md                                  ← You are here
│
├── chapters/
│   ├── ch1-kubernetes-fundamentals.md         1,054 lines
│   ├── ch2-core-objects.md                    1,465 lines
│   ├── ch3-networking.md                      1,722 lines
│   ├── ch4-storage.md                         1,150 lines
│   ├── ch5-configuration.md                   1,351 lines
│   ├── ch6-workloads.md                       1,481 lines
│   ├── ch7-scheduling.md                      1,619 lines
│   ├── ch8-security.md                        1,687 lines
│   ├── ch9-monitoring-logging.md              1,612 lines
│   ├── ch10-cluster-administration.md         1,637 lines
│   ├── ch11-production-kubernetes.md          1,376 lines
│   └── ch12-cka-preparation.md                1,450 lines
│
├── labs/                                      (coming soon)
│   ├── lab-01-cluster-setup/
│   ├── lab-02-pods-and-deployments/
│   ├── lab-03-networking/
│   ├── lab-04-storage/
│   ├── lab-05-configuration/
│   ├── lab-06-workloads/
│   ├── lab-07-scheduling/
│   ├── lab-08-security/
│   ├── lab-09-monitoring/
│   ├── lab-10-cluster-admin/
│   ├── lab-11-helm-argocd/
│   └── lab-12-mock-exam/
│
└── cheatsheets/                               (coming soon)
    ├── kubectl-complete-reference.md
    ├── yaml-templates-library.md
    ├── cka-speed-reference.md
    └── troubleshooting-decision-tree.md
```

---

## 📅 Recommended Study Plan

```
WEEK 1  — Foundation
  ✅ Chapter 1: Kubernetes Architecture
  ✅ Chapter 2: Core Objects (Pods, Deployments)
  ✅ Set up your lab (minikube or kind)
  ✅ Do every lab exercise — type every command yourself

WEEK 2  — Networking & Storage
  ✅ Chapter 3: Services, Ingress, DNS
  ✅ Chapter 4: PV, PVC, StorageClasses
  ✅ Build a 3-tier application from scratch

WEEK 3  — Configuration & Workloads
  ✅ Chapter 5: ConfigMaps, Secrets
  ✅ Chapter 6: DaemonSets, StatefulSets, Jobs
  ✅ Deploy a stateful database with persistent storage

WEEK 4  — Scheduling & Security
  ✅ Chapter 7: Scheduling, Taints, Resources
  ✅ Chapter 8: RBAC, SecurityContexts, NetworkPolicies
  ✅ Build a namespace with full RBAC and network isolation

WEEK 5  — Operations
  ✅ Chapter 9: Prometheus, Grafana, Logging
  ✅ Chapter 10: kubeadm, etcd backup/restore, cluster upgrade
  ✅ Practice etcd backup until it takes under 90 seconds

WEEK 6  — Production & CKA Sprint
  ✅ Chapter 11: HA, Helm, ArgoCD
  ✅ Chapter 12: Speed labs, mock exam
  ✅ Complete the mock exam under real timer pressure
  🎯 Book the CKA — you are ready
```

---

## ⚡ kubectl Quick Reference

```bash
# PODS
k run <name> --image=<img> [--labels=k=v] [--port=N] [--restart=Never]
k run <name> --image=<img> $do > pod.yaml          # Generate YAML
k get pods [-A] [-n ns] [-o wide] [--show-labels] [-l key=val] [-w]
k describe pod <name> [-n ns]                       # Full details + Events
k logs <name> [-f] [--previous] [--tail=N] [-c container]
k exec -it <name> -- bash
k delete pod <name> [$now]

# DEPLOYMENTS
k create deploy <n> --image=<img> --replicas=N [$do > file.yaml]
k scale deploy <n> --replicas=N
k set image deploy/<n> <container>=<newimage>
k rollout status/history/undo/restart deploy/<n>

# SERVICES
k expose deploy <n> --port=80 [--target-port=8080] [--type=NodePort] [--name=svc-name]
k get svc [-o wide]                                 # See ClusterIP and ports
k get endpoints <svc>                               # Must not be empty!

# STORAGE
k get pv,pvc,sc                                     # All storage objects at once
k describe pvc <n>                                  # Always check Events!
k patch pvc <n> -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# CONFIG & SECRETS
k create cm <n> --from-literal=k=v [--from-file=<f>] [$do > cm.yaml]
k create secret generic <n> --from-literal=k=v
k get secret <n> -o jsonpath='{.data.<key>}' | base64 -d

# RBAC
k create role <n> --verb=get,list,watch --resource=pods [-n ns]
k create clusterrole <n> --verb=get,list --resource=nodes
k create rolebinding <n> --role=<r> --user=<u> [-n ns]
k create rolebinding <n> --role=<r> --serviceaccount=ns:sa-name [-n ns]
k create clusterrolebinding <n> --clusterrole=<r> --user=<u>
k auth can-i <verb> <resource> [--as <user>] [-n ns]
k auth can-i --list --as <user> [-n ns]
k auth whoami

# NODES
k taint node <n> key=val:NoSchedule
k taint node <n> key=val:NoSchedule-                # Remove taint
k label node <n> key=val
k cordon <n> && k drain <n> --ignore-daemonsets --delete-emptydir-data
k uncordon <n>
k top nodes && k top pods -A --sort-by=memory

# etcd BACKUP (memorise this)
ETCDCTL_API=3 etcdctl snapshot save /opt/backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# CONTEXT (run before every exam question)
k config current-context
k config use-context <name>
k config set-context --current --namespace=<ns>

# DEBUG
k get events -A --sort-by=.lastTimestamp | tail -20
k describe <resource> <name> | grep -A10 Events
k run debug --image=nicolaka/netshoot --rm -it --restart=Never -- bash
ssh <node> && systemctl status kubelet && journalctl -u kubelet -f
```

---

## 🔧 Recommended Tools

| Tool | What It Does | Install |
|---|---|---|
| **k9s** | Terminal-based Kubernetes dashboard (essential) | [k9s.io](https://k9s.io) |
| **kubectx + kubens** | 1-command context and namespace switching | [github.com/ahmetb/kubectx](https://github.com/ahmetb/kubectx) |
| **stern** | Tail logs from multiple pods simultaneously | [github.com/stern/stern](https://github.com/stern/stern) |
| **Helm** | Kubernetes package manager | [helm.sh](https://helm.sh) |
| **kube-score** | YAML best-practice linting and scoring | [kube-score.com](https://kube-score.com) |
| **ArgoCD** | GitOps continuous delivery | [argoproj.io](https://argoproj.io) |
| **Lens** | Full GUI Kubernetes IDE | [k8slens.dev](https://k8slens.dev) |
| **Popeye** | Live cluster configuration linter | [github.com/derailed/popeye](https://github.com/derailed/popeye) |
| **crontab.guru** | Visual cron expression editor | [crontab.guru](https://crontab.guru) |

---

## 📖 How to Read This Handbook

1. **Read sequentially.** Each chapter builds directly on the previous one. Skipping ahead means missing the foundation that makes later concepts click.
2. **Type every command yourself.** Do not copy-paste in labs. Muscle memory is the difference between passing and failing the CKA.
3. **Do every lab.** Reading about Kubernetes without practicing is like reading about swimming without getting in the water.
4. **Use the CKA Notes boxes.** Every chapter ends with a `📌 CKA EXAM FOCUS` section tuned specifically for the exam.
5. **Save the troubleshooting sections.** You will return to these even after certification when things break in production.
6. **Practice the speed labs until they are automatic.** The mock exam in Chapter 12 should be done with a real 120-minute timer.

---

## 🤝 Contributing

Contributions are welcome. Here is how to help:

```bash
# Fork the repository on GitHub, then:
git clone https://github.com/shaikDevOps01/KUBERNETES-MASTERY-HANDBOOK
cd kubernetes-mastery-handbook
git checkout -b improve/chapter-3-networking-examples

# Make your changes
# Then:
git add .
git commit -m "feat(ch3): add Cilium NetworkPolicy lab example"
git push origin improve/chapter-3-networking-examples

# Open a Pull Request on GitHub
```

**Contribution guidelines:**
- Keep the tone consistent — plain English first, technical second
- All YAML examples must be tested and working
- ASCII diagrams are preferred (portable, version-controllable, no external dependencies)
- Add the chapter number and topic to your PR title
- Update line counts in this README if your changes are significant

---

## 👨‍💻 Author

**Shaik Dasthagiri**
DevOps Engineer · Bengaluru, India 🇮🇳

Building production Kubernetes infrastructure and documenting every hard-won lesson along the way.

- 🌐 
- 💼 [LinkedIn] https://www.linkedin.com/in/shaikdasthagiri/
- 🐙 [GitHub] https://github.com/shaikDevOps01?tab=repositories
- ✈️ [FareFlyers] https://fareflyers.com  — Travel Payment Intelligence Platform

---

## 🙌 Acknowledgements

- The [Kubernetes Documentation](https://kubernetes.io/docs) team — the gold standard of open-source technical writing
- [Kelsey Hightower](https://github.com/kelseyhightower) — for *Kubernetes the Hard Way* which showed what self-teaching looks like
- [Mumshad Mannambeth](https://github.com/mmumshad) — for the CKA course that guided so many engineers including this one
- The CNCF community — for building, maintaining, and documenting this incredible ecosystem
- Every engineer who filed issues, opened PRs, and helped make this handbook better

---

## 📄 License

This handbook is released under the [MIT License](LICENSE).

```
You are free to:
  ✅ Use for personal learning
  ✅ Share with your team or community (with attribution)
  ✅ Use in internal corporate training (with attribution)
  ✅ Fork and build upon it
  ❌ Resell without permission
```

---

## 🏆 CKA Wall of Fame

*Passed the CKA after using this handbook? Open a PR and add your name here.*

| Name | Date | Score | Note |
|---|---|---|---|
| *Be the first!* | | | |

---

<div align="center">

**Built with ❤️ for the Kubernetes community**
**by Shaik Dasthagiri, Bengaluru, India**

*If this handbook helped you pass your CKA — star the repo and share it with the next engineer who needs it.*

<br/>

[![Star this repo](https://img.shields.io/github/stars/shaikDevOps01/KUBERNETES-MASTERY-HANDBOOK?style=for-the-badge&logo=github&label=Star%20this%20Handbook)](https://github.com/shaikDevOps01/KUBERNETES-MASTERY-HANDBOOK)


</div>
