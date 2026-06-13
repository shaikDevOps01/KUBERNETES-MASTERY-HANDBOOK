# KUBERNETES MASTERY HANDBOOK
# Part 7: Scheduling
# Chapter 7: Node Selectors, Affinity, Taints & Tolerations, Resource Management

---

> **"The Scheduler is a matchmaker. Your job is to give it the right rules —
>  and get out of the way."**

---

## Chapter Introduction

By default, the Kubernetes Scheduler places pods on whatever node has the most
available resources. That works for simple clusters — but production reality is
far more complex:

```
╔══════════════════════════════════════════════════════════════════════╗
║           REAL-WORLD SCHEDULING REQUIREMENTS                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  "My ML training job needs GPU nodes."                               ║
║  → Node Selector / Node Affinity                                     ║
║                                                                      ║
║  "My database pods must NEVER share a node."                         ║
║  → Pod Anti-Affinity                                                 ║
║                                                                      ║
║  "My frontend pods should prefer nodes near the backend pods."       ║
║  → Pod Affinity                                                      ║
║                                                                      ║
║  "These nodes are reserved for system components only."              ║
║  → Taints and Tolerations                                            ║
║                                                                      ║
║  "This pod needs 4 CPU cores and 16GB RAM guaranteed."               ║
║  → Resource Requests and Limits                                      ║
║                                                                      ║
║  "If the cluster is under pressure, evict test pods first."          ║
║  → Quality of Service Classes + Priority Classes                     ║
║                                                                      ║
║  SCHEDULING TOOLBOX — CHAPTER ROADMAP:                               ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │  NodeSelector      — simple label-based node selection         │  ║
║  │  Node Affinity     — expressive node selection (required/pref) │  ║
║  │  Pod Affinity      — schedule near other pods                  │  ║
║  │  Pod Anti-Affinity — spread pods away from each other          │  ║
║  │  Taints            — mark nodes as "repelling" by default      │  ║
║  │  Tolerations       — allow pods to overcome node taints        │  ║
║  │  Resource Requests — guaranteed minimums for scheduling        │  ║
║  │  Resource Limits   — hard caps for containers                  │  ║
║  │  QoS Classes       — eviction priority under pressure          │  ║
║  │  Priority Classes  — preemption — high-priority pods evict low │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 7.1 How the Kubernetes Scheduler Works — Recap

Before diving into controls, understand the Scheduler's two-phase process:

```
╔══════════════════════════════════════════════════════════════════════╗
║                  SCHEDULER — TWO-PHASE DECISION                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  NEW POD (unscheduled) arrives                                       ║
║         │                                                            ║
║         ▼                                                            ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  PHASE 1: FILTERING  (eliminate unfit nodes)                 │   ║
║  │                                                              │   ║
║  │  • Node is Ready?           Yes → keep   No → remove        │   ║
║  │  • Enough CPU/Memory?       Yes → keep   No → remove        │   ║
║  │  • nodeSelector matches?    Yes → keep   No → remove        │   ║
║  │  • Node Affinity satisfied? Yes → keep   No → remove        │   ║
║  │  • Tolerates all taints?    Yes → keep   No → remove        │   ║
║  │  • Port conflicts?          No  → keep   Yes → remove       │   ║
║  │  • PVC zone available?      Yes → keep   No → remove        │   ║
║  │                                                              │   ║
║  │  Result: Feasible node set (0..N nodes)                     │   ║
║  └──────────────────────────────┬───────────────────────────────┘   ║
║                                 │ if 0 nodes → Pod stays Pending    ║
║                                 ▼                                    ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  PHASE 2: SCORING  (rank feasible nodes 0-100)               │   ║
║  │                                                              │   ║
║  │  • LeastAllocated     — prefer nodes with most free CPU/Mem  │   ║
║  │  • BalancedAllocation — prefer balanced CPU+memory usage     │   ║
║  │  • NodeAffinityPriority — prefer nodes matching affinity     │   ║
║  │  • PodAffinityPriority — prefer nodes co-located with others │   ║
║  │  • ImageLocality      — prefer nodes that already have image │   ║
║  │  • TaintToleration    — prefer nodes with fewer taints       │   ║
║  │                                                              │   ║
║  │  Result: Node with highest score wins                       │   ║
║  └──────────────────────────────┬───────────────────────────────┘   ║
║                                 │                                    ║
║                                 ▼                                    ║
║              Pod assigned to winning node                            ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 7.2 NodeSelector — Simple Node Targeting

### 7.2.1 What Is NodeSelector?

#### In Plain English

NodeSelector is like putting a **"GPU ONLY" sign** on a job application.
Only candidates (nodes) with the matching skill (label) get considered for
the role (pod). It is the simplest node selection mechanism.

#### In Technical Language

`nodeSelector` is the simplest form of node selection constraint. It is a
field of the pod spec that specifies a map of key-value pairs. The pod can
only be scheduled on nodes that have ALL specified labels.

```
╔══════════════════════════════════════════════════════════════════════╗
║                   NODESELECTOR IN ACTION                             ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Node Labels:                                                        ║
║  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ║
║  │  worker-1        │  │  worker-2        │  │  worker-3        │  ║
║  │  disk=ssd        │  │  disk=hdd        │  │  disk=ssd        │  ║
║  │  gpu=nvidia      │  │  region=us-east  │  │  region=us-west  │  ║
║  └──────────────────┘  └──────────────────┘  └──────────────────┘  ║
║                                                                      ║
║  Pod with nodeSelector: {disk: ssd}                                  ║
║  → Eligible: worker-1 ✅, worker-3 ✅                                ║
║  → Excluded: worker-2 ❌ (disk=hdd)                                  ║
║                                                                      ║
║  Pod with nodeSelector: {disk: ssd, gpu: nvidia}                     ║
║  → Eligible: worker-1 ✅ only                                        ║
║  → Excluded: worker-2 ❌, worker-3 ❌ (no gpu label)                ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 7.2.2 NodeSelector YAML and Commands

```yaml
# nodeselector-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  nodeSelector:
    accelerator: nvidia-gpu          # MUST have this label
    disk: ssd                        # AND this label
  containers:
  - name: ml-trainer
    image: tensorflow/tensorflow:latest-gpu
    resources:
      limits:
        nvidia.com/gpu: 1            # Request 1 GPU (requires GPU plugin)
```

```bash
# Add labels to nodes
kubectl label node worker-1 disk=ssd
kubectl label node worker-1 accelerator=nvidia-gpu
kubectl label node worker-2 disk=hdd
kubectl label node worker-3 disk=ssd

# Remove a label
kubectl label node worker-1 disk-        # Trailing dash removes label

# Verify node labels
kubectl get nodes --show-labels
kubectl get nodes -l disk=ssd            # Filter nodes by label

# Check why pod is pending (nodeSelector mismatch)
kubectl describe pod gpu-workload
# Events: 0/3 nodes available: 1 node(s) didn't match Pod's nodeAffinity
```

---

## 7.3 Node Affinity — Expressive Node Selection

### 7.3.1 What Is Node Affinity?

Node Affinity is a more expressive, powerful version of nodeSelector.
It supports **operators** (In, NotIn, Exists, DoesNotExist, Gt, Lt)
and **two modes** — required (hard) and preferred (soft).

```
╔══════════════════════════════════════════════════════════════════════╗
║              NODE AFFINITY — REQUIRED vs PREFERRED                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  REQUIRED (Hard Rule) — Pod WILL NOT schedule if unmet:              ║
║  requiredDuringSchedulingIgnoredDuringExecution                      ║
║  ─────────────────────────────────────────────────                   ║
║  "I MUST run on a node in us-east-1a OR us-east-1b"                 ║
║  If no such node exists → Pod stays Pending forever                  ║
║  "IgnoredDuringExecution" = if label removed after scheduling,       ║
║   pod is NOT evicted (continues running)                             ║
║                                                                      ║
║  PREFERRED (Soft Rule) — Scheduler tries, but can ignore:            ║
║  preferredDuringSchedulingIgnoredDuringExecution                     ║
║  ─────────────────────────────────────────────────                   ║
║  "I PREFER nodes with disk=ssd, but any node is OK"                 ║
║  If no matching node exists → pod still schedules on best match      ║
║  Each preference has a WEIGHT (1-100) for scoring                   ║
║                                                                      ║
║  COMBINING BOTH:                                                     ║
║  Required: MUST be in us-east-1 region                              ║
║  Preferred: WOULD LIKE disk=ssd (weight: 80)                        ║
║  Preferred: WOULD LIKE instance-type=m5.xlarge (weight: 40)         ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 7.3.2 Node Affinity YAML

```yaml
# node-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-demo
spec:
  affinity:
    nodeAffinity:

      # ── REQUIRED (hard) ─────────────────────────────────────────
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:              # OR between terms
        - matchExpressions:            # AND between expressions
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b              # Node must be in these AZs
          - key: node-type
            operator: NotIn
            values:
            - spot                    # Not on spot instances
        - matchExpressions:           # This is OR'd with above
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - eu-west-1a              # OR can be in this AZ

      # ── PREFERRED (soft) ─────────────────────────────────────────
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80                    # High weight = strong preference
        preference:
          matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd                     # Prefer SSD nodes
      - weight: 40                    # Lower weight = weaker preference
        preference:
          matchExpressions:
          - key: instance-type
            operator: In
            values:
            - m5.2xlarge
            - m5.4xlarge              # Prefer these instance types

  containers:
  - name: app
    image: my-app:v1

---
# REAL-WORLD: Run only on nodes with sufficient resources in right AZ
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-replica
spec:
  replicas: 2
  selector:
    matchLabels:
      app: db-replica
  template:
    metadata:
      labels:
        app: db-replica
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/region
                operator: In
                values: [us-east-1]
              - key: node-role
                operator: NotIn
                values: [spot, preemptible]
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: disk
                operator: In
                values: [nvme-ssd]
      containers:
      - name: db
        image: postgres:14
```

---

## 7.4 Pod Affinity and Anti-Affinity — Scheduling Relative to Other Pods

### 7.4.1 What Are Pod Affinity and Anti-Affinity?

#### In Plain English

**Pod Affinity** says: "Schedule me **near** pods with these labels."
Like a new employee who wants to sit near their team.

**Pod Anti-Affinity** says: "Schedule me **away from** pods with these labels."
Like a database that should never share a node with another database instance
(to avoid single-node failure taking both down).

#### In Technical Language

Pod Affinity and Anti-Affinity let you constrain which nodes a pod can be
scheduled on based on the labels of **other pods already running** on those
nodes, rather than node labels. They use **topology keys** to define the
scope of "near" or "away" — typically `kubernetes.io/hostname` (same node)
or `topology.kubernetes.io/zone` (same AZ).

```
╔══════════════════════════════════════════════════════════════════════╗
║              POD AFFINITY / ANTI-AFFINITY — VISUALISED               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  POD AFFINITY (schedule NEAR matching pods):                         ║
║                                                                      ║
║  Node-1                    Node-2                    Node-3          ║
║  ┌──────────────────┐      ┌──────────────────┐      ┌───────────┐  ║
║  │  cache-pod ✅    │      │  (empty)         │      │ (empty)   │  ║
║  │  web-pod   ?     │      │                  │      │           │  ║
║  └──────────────────┘      └──────────────────┘      └───────────┘  ║
║                                                                      ║
║  web-pod has affinity: "prefer nodes where cache-pod runs"           ║
║  → web-pod scheduled on Node-1 (near cache-pod) ✅                  ║
║  Use case: web server co-located with its Redis cache for low latency║
║                                                                      ║
║  POD ANTI-AFFINITY (schedule AWAY from matching pods):               ║
║                                                                      ║
║  Node-1                    Node-2                    Node-3          ║
║  ┌──────────────────┐      ┌──────────────────┐      ┌───────────┐  ║
║  │  db-primary ✅  │      │                  │      │           │  ║
║  │  db-replica  ❌ │      │  db-replica ✅   │      │           │  ║
║  └──────────────────┘      └──────────────────┘      └───────────┘  ║
║                                                                      ║
║  db-replica anti-affinity: "NEVER schedule on node with db-primary" ║
║  → db-replica correctly lands on Node-2 ✅                           ║
║  Use case: HA databases — never put primary and replica on same node ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 7.4.2 Pod Affinity and Anti-Affinity YAML

```yaml
# pod-affinity-antiaffinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:

        # ── POD AFFINITY: schedule near cache pods ───────────────
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: redis-cache      # Find nodes with redis-cache pods
              topologyKey: kubernetes.io/hostname  # "near" = same node

        # ── POD ANTI-AFFINITY: spread web pods across nodes ──────
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web              # Don't schedule near other web pods
            topologyKey: kubernetes.io/hostname
            # Effect: max 1 web pod per node (hard spread)

      containers:
      - name: web
        image: nginx:alpine

---
# DATABASE HA PATTERN — Primary and Replica must be on different nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-replica
spec:
  replicas: 2
  selector:
    matchLabels:
      app: postgres
      role: replica
  template:
    metadata:
      labels:
        app: postgres
        role: replica
    spec:
      affinity:
        podAntiAffinity:
          # REQUIRED: Never place replica with primary
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: postgres
                role: primary
            topologyKey: kubernetes.io/hostname
          # PREFERRED: Also keep replicas on different nodes from each other
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: postgres
                  role: replica
              topologyKey: kubernetes.io/hostname
      containers:
      - name: postgres
        image: postgres:14

---
# ZONE SPREAD — spread pods across AZs (higher availability)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
spec:
  replicas: 6
  selector:
    matchLabels:
      app: critical-service
  template:
    metadata:
      labels:
        app: critical-service
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: critical-service
              topologyKey: topology.kubernetes.io/zone   # Prefer different AZs
      containers:
      - name: svc
        image: my-critical-svc:v1
```

### 7.4.3 Topology Spread Constraints — Better Pod Spreading

In modern Kubernetes (1.19+), **TopologySpreadConstraints** is the preferred
way to spread pods evenly across zones or nodes:

```yaml
# topology-spread.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-spread
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web-spread
  template:
    metadata:
      labels:
        app: web-spread
    spec:
      topologySpreadConstraints:
      - maxSkew: 1                # Max difference in pod count between zones
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule  # Hard: DoNotSchedule | Soft: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web-spread
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname  # Also spread across nodes within AZ
        whenUnsatisfiable: ScheduleAnyway    # Soft constraint for nodes
        labelSelector:
          matchLabels:
            app: web-spread
      containers:
      - name: web
        image: nginx:alpine
```

---

## 7.5 Taints and Tolerations — Repelling Pods from Nodes

### 7.5.1 What Are Taints and Tolerations?

#### In Plain English

A **Taint** is a **"No Unauthorized Personnel"** sign on a node.
By default, all pods are repelled (they cannot schedule there).
A **Toleration** is the pod's **security badge** — "I'm authorized to
enter despite the sign." A pod with the right toleration can overcome the taint.

Think of it this way:
- **Taint** = the bouncer at the door
- **Toleration** = the VIP pass in your pocket
- Without the pass, you can't get in

#### In Technical Language

**Taints** are applied to nodes and **repel** pods that don't tolerate them.
**Tolerations** are applied to pods and allow them to schedule on tainted nodes.
Together they ensure nodes are dedicated to specific workloads.

```
╔══════════════════════════════════════════════════════════════════════╗
║             TAINTS AND TOLERATIONS — THREE EFFECTS                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  TAINT EFFECTS:                                                      ║
║                                                                      ║
║  NoSchedule                                                          ║
║  ─────────────                                                       ║
║  New pods WITHOUT matching toleration are NOT scheduled here.        ║
║  Existing pods already running are NOT evicted.                      ║
║  Most common taint for node reservation.                             ║
║                                                                      ║
║  PreferNoSchedule                                                    ║
║  ─────────────────                                                   ║
║  Scheduler TRIES to avoid placing untolerated pods here.             ║
║  If no other nodes available, pod CAN still schedule here.           ║
║  Soft version of NoSchedule.                                         ║
║                                                                      ║
║  NoExecute                                                           ║
║  ──────────                                                          ║
║  New pods WITHOUT toleration are NOT scheduled.                      ║
║  Existing pods WITHOUT toleration are EVICTED immediately.           ║
║  Used for node maintenance (drain) and node failures.               ║
║  Kubernetes itself uses NoExecute taints for NotReady nodes.        ║
║                                                                      ║
║  REAL-WORLD EXAMPLE:                                                 ║
║                                                                      ║
║  GPU Node: taint = "gpu-only:NoSchedule"                             ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │  GPU Node (gpu=true, taint: dedicated=gpu-only:NoSchedule)    │  ║
║  │                                                                │  ║
║  │  ✅ ML Training Pod (has toleration: dedicated=gpu-only)      │  ║
║  │  ❌ Web Server Pod  (no toleration → not scheduled here)      │  ║
║  │  ❌ Database Pod    (no toleration → not scheduled here)      │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 7.5.2 Taint Commands

```bash
# Add a taint to a node
kubectl taint node worker-1 dedicated=gpu-only:NoSchedule
kubectl taint node worker-2 environment=production:NoSchedule
kubectl taint node worker-3 maintenance=true:NoExecute

# Taint format: key=value:Effect   OR   key:Effect  (no value)
kubectl taint node worker-1 special-hardware:NoSchedule

# Remove a taint (trailing dash removes it)
kubectl taint node worker-1 dedicated=gpu-only:NoSchedule-
kubectl taint node worker-1 special-hardware:NoSchedule-

# View taints on a node
kubectl describe node worker-1 | grep Taints

# Kubernetes automatically adds these taints:
# node.kubernetes.io/not-ready:NoExecute              (node not ready)
# node.kubernetes.io/unreachable:NoExecute            (node unreachable)
# node.kubernetes.io/disk-pressure:NoSchedule         (disk full)
# node.kubernetes.io/memory-pressure:NoSchedule       (memory low)
# node-role.kubernetes.io/control-plane:NoSchedule    (control plane)
```

### 7.5.3 Tolerations YAML

```yaml
# tolerations-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  # ── TOLERATIONS: What taints this pod can tolerate ─────────────
  tolerations:
  # Exact match: tolerate key=dedicated, value=gpu-only, effect=NoSchedule
  - key: "dedicated"
    operator: Equal               # Equal | Exists
    value: "gpu-only"
    effect: "NoSchedule"          # NoSchedule | PreferNoSchedule | NoExecute

  # Exists match: tolerate ANY value for this key/effect
  - key: "special-hardware"
    operator: Exists              # Ignores value
    effect: "NoSchedule"

  # Tolerate all taints (dangerous — use sparingly)
  - operator: Exists              # No key = matches ALL taints

  # Tolerate NoExecute with grace period (for node failures)
  - key: "node.kubernetes.io/not-ready"
    operator: Exists
    effect: "NoExecute"
    tolerationSeconds: 300        # Tolerate for 5 minutes before eviction

  containers:
  - name: ml-trainer
    image: tensorflow/tensorflow:latest-gpu

---
# FULL PATTERN: Taint + Toleration + NodeAffinity together
# (Belt AND suspenders — most robust node dedication)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-training-job
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-trainer
  template:
    metadata:
      labels:
        app: gpu-trainer
    spec:
      # Toleration = allowed on GPU nodes (overcomes the repel)
      tolerations:
      - key: dedicated
        operator: Equal
        value: gpu-only
        effect: NoSchedule

      # Affinity = MUST go to GPU nodes (not just allowed, but required)
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: accelerator
                operator: In
                values: [nvidia-gpu]

      containers:
      - name: trainer
        image: pytorch/pytorch:latest
        resources:
          limits:
            nvidia.com/gpu: 1
```

### 7.5.4 Common Taint Patterns

```
╔══════════════════════════════════════════════════════════════════════╗
║              PRODUCTION TAINT PATTERNS                               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PATTERN 1: Dedicated Node Pool (e.g., GPU cluster)                 ║
║  kubectl taint node gpu-node-1 dedicated=gpu:NoSchedule             ║
║  kubectl label node gpu-node-1 accelerator=nvidia-gpu               ║
║  Pod: toleration + nodeAffinity → guaranteed on GPU nodes           ║
║                                                                      ║
║  PATTERN 2: Production vs Test Isolation                             ║
║  kubectl taint node prod-node-1 env=production:NoSchedule           ║
║  Production pods: have toleration for env=production                 ║
║  Test pods: no toleration → cannot schedule on prod nodes           ║
║                                                                      ║
║  PATTERN 3: Node Maintenance (drain + NoExecute)                    ║
║  kubectl drain node worker-1 --ignore-daemonsets                    ║
║  → Adds node.kubernetes.io/unschedulable:NoSchedule taint           ║
║  → Evicts all pods (moves them to other nodes)                      ║
║  kubectl uncordon worker-1 (removes the taint)                      ║
║                                                                      ║
║  PATTERN 4: Control Plane Protection                                 ║
║  Auto-taint: node-role.kubernetes.io/control-plane:NoSchedule       ║
║  System DaemonSets tolerate this                                    ║
║  User pods do NOT → cannot schedule on master nodes                 ║
║                                                                      ║
║  PATTERN 5: Spot/Preemptible Node Handling                          ║
║  kubectl taint node spot-node cloud.google.com/gke-spot:NoSchedule  ║
║  Only fault-tolerant workloads with toleration run on spot nodes    ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 7.6 Resource Requests and Limits

### 7.6.1 What Are Requests and Limits?

#### In Plain English

**Requests** are what your container is **guaranteed** — like booking a seat
on a flight. The scheduler finds a node with enough free seats.

**Limits** are the **maximum** your container can use — like a baggage weight
cap. Exceed the memory limit and your container gets killed. Exceed the CPU
limit and it gets throttled (slowed down).

#### In Technical Language

- **Requests**: Used by the scheduler to find a node with enough capacity.
  The container is guaranteed this amount. If a node has 4 CPU and your pods
  request 1 CPU each, only 4 pods can schedule there (regardless of actual usage).
- **Limits**: Enforced at runtime by cgroups. CPU limit = throttling.
  Memory limit = OOMKill (container killed and restarted).

```
╔══════════════════════════════════════════════════════════════════════╗
║            REQUESTS vs LIMITS — THE CONTRACT                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Node: 4 CPU, 8 GB RAM                                               ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  Pod-A: request=1CPU,1GB  limit=2CPU,2GB                    │   ║
║  │  Pod-B: request=1CPU,2GB  limit=2CPU,4GB                    │   ║
║  │  Pod-C: request=1CPU,1GB  limit=2CPU,2GB                    │   ║
║  │                                                              │   ║
║  │  Total requested: 3 CPU, 4 GB  (node has 4 CPU, 8 GB → OK) │   ║
║  │  All 3 pods can schedule on this node ✅                    │   ║
║  │                                                              │   ║
║  │  AT RUNTIME (each pod uses full limit):                     │   ║
║  │  Total used: 6 CPU, 8 GB ← exceeds node capacity!          │   ║
║  │  CPU: pods throttled (slowed down, not killed)              │   ║
║  │  Memory: if Pod-B tries to use 4GB → OOMKilled              │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  UNITS:                                                              ║
║  CPU:    1 = 1 core, 0.5 = 500m (millicores), 0.1 = 100m           ║
║  Memory: 1Gi = 1 gibibyte, 512Mi = 512 mebibytes, 1G = 1 gigabyte  ║
║                                                                      ║
║  CPU is COMPRESSIBLE: throttled when exceeded (pod survives)        ║
║  Memory is NON-COMPRESSIBLE: killed when exceeded (OOMKill)         ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 7.6.2 Resource YAML — Production Patterns

```yaml
# resource-requests-limits.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: web-app
    image: nginx:alpine
    resources:
      requests:                  # Scheduler guarantee (minimum needed)
        cpu: "250m"              # 0.25 CPU core
        memory: "128Mi"          # 128 mebibytes
      limits:                    # Hard cap (maximum allowed)
        cpu: "500m"              # 0.5 CPU core
        memory: "256Mi"          # 256 mebibytes

  - name: sidecar-logger
    image: fluent/fluent-bit:latest
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"

---
# LimitRange — default resource values for a namespace
# Any container without explicit requests/limits gets these defaults
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:                     # Default limits (applied if not set)
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:              # Default requests (applied if not set)
      cpu: "100m"
      memory: "64Mi"
    max:                         # Maximum any container can request
      cpu: "2"
      memory: "2Gi"
    min:                         # Minimum any container must request
      cpu: "50m"
      memory: "32Mi"
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"

---
# ResourceQuota — limit total resource use per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"           # Total CPU requests in namespace
    requests.memory: "40Gi"      # Total memory requests
    limits.cpu: "40"             # Total CPU limits
    limits.memory: "80Gi"
    pods: "100"                  # Max pods in namespace
    services: "20"               # Max services
    persistentvolumeclaims: "30" # Max PVCs
    services.loadbalancers: "5"  # Max LoadBalancer services
    secrets: "50"
    configmaps: "50"
```

---

## 7.7 Quality of Service (QoS) Classes

### 7.7.1 What Is QoS?

When a node runs out of memory, Kubernetes must evict pods. Which pods get
evicted first? That is decided by **QoS class** — automatically assigned based
on how you set requests and limits.

```
╔══════════════════════════════════════════════════════════════════════╗
║                THREE QoS CLASSES                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  CLASS: Guaranteed  (evicted LAST — safest)                          ║
║  ──────────────────────────────────────────                          ║
║  Requirement: EVERY container has requests == limits (CPU + Memory)  ║
║  Example:                                                            ║
║    resources:                                                        ║
║      requests:  { cpu: "500m", memory: "256Mi" }                    ║
║      limits:    { cpu: "500m", memory: "256Mi" }   ← same values!  ║
║  Use for: Production databases, critical services                    ║
║                                                                      ║
║  CLASS: Burstable  (evicted SECOND)                                  ║
║  ──────────────────────────────────                                  ║
║  Requirement: At least one container has requests < limits, OR       ║
║               only limits set (no requests), OR only requests set   ║
║  Example:                                                            ║
║    resources:                                                        ║
║      requests:  { cpu: "250m", memory: "128Mi" }                    ║
║      limits:    { cpu: "500m", memory: "256Mi" }   ← different!    ║
║  Use for: Most application workloads                                 ║
║                                                                      ║
║  CLASS: BestEffort  (evicted FIRST — most dangerous)                 ║
║  ─────────────────────────────────────────────────                   ║
║  Requirement: NO requests AND NO limits set on any container         ║
║  Example:                                                            ║
║    resources: {}   ← missing entirely!                               ║
║  Use for: Dev/test only — NEVER in production                        ║
║                                                                      ║
║  EVICTION ORDER UNDER MEMORY PRESSURE:                               ║
║  BestEffort → Burstable (highest usage over request) → Guaranteed   ║
╚══════════════════════════════════════════════════════════════════════╝
```

```bash
# Check QoS class of a pod
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'
# Output: Guaranteed | Burstable | BestEffort

kubectl describe pod my-pod | grep QoS
```

---

## 7.8 Priority Classes — Pod Scheduling Priority

### 7.8.1 What Is a Priority Class?

When resources are scarce and a high-priority pod cannot be scheduled,
Kubernetes can **preempt** (evict) lower-priority pods to make room.

```yaml
# priority-classes.yaml

# SYSTEM-LEVEL (built-in, highest)
# system-cluster-critical — kube-apiserver, etcd, coredns
# system-node-critical    — kube-proxy, calico-node

---
# Create custom priority classes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-production
value: 1000000                  # Higher = more important
globalDefault: false
preemptionPolicy: PreemptLowerPriority   # Can evict lower-priority pods
description: "Critical production workloads"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 500000
globalDefault: true             # Default for pods with no priorityClassName
preemptionPolicy: PreemptLowerPriority
description: "Standard workloads"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority-batch
value: 100000
preemptionPolicy: Never         # Cannot preempt other pods
description: "Batch and test workloads"

---
# Use PriorityClass in a pod
apiVersion: v1
kind: Pod
metadata:
  name: critical-payment-pod
spec:
  priorityClassName: high-priority-production   # Reference priority class
  containers:
  - name: payment
    image: payment-service:v1
    resources:
      requests:
        cpu: "1"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "2Gi"
```

---

## 7.9 Node Maintenance — Cordon, Drain, Uncordon

```
╔══════════════════════════════════════════════════════════════════════╗
║              NODE MAINTENANCE WORKFLOW                                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  STEP 1: CORDON — mark node unschedulable (no new pods)             ║
║  kubectl cordon worker-1                                             ║
║  Adds taint: node.kubernetes.io/unschedulable:NoSchedule            ║
║  Existing pods continue running — only NEW pods are blocked          ║
║                                                                      ║
║  STEP 2: DRAIN — evict existing pods                                 ║
║  kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data  ║
║  → Evicts all pods (moves them to other nodes)                      ║
║  → DaemonSet pods ignored (they're node-specific by design)         ║
║  → Pods with emptyDir: data lost (--delete-emptydir-data required)  ║
║  → Pods with PodDisruptionBudget respected (waits for new pod ready)║
║                                                                      ║
║  STEP 3: MAINTENANCE                                                 ║
║  Apply OS patches, kernel updates, hardware replacement...           ║
║                                                                      ║
║  STEP 4: UNCORDON — re-enable scheduling                             ║
║  kubectl uncordon worker-1                                           ║
║  Removes NoSchedule taint. Pods can schedule here again.            ║
╚══════════════════════════════════════════════════════════════════════╝
```

```bash
# Full maintenance workflow
kubectl cordon worker-1
kubectl get node worker-1   # SchedulingDisabled status

kubectl drain worker-1 \
  --ignore-daemonsets \          # Don't try to evict DaemonSet pods
  --delete-emptydir-data \       # Allow deleting pods with emptyDir volumes
  --grace-period=60 \            # Give pods 60s to terminate gracefully
  --timeout=300s                 # Total timeout for drain operation

# Apply maintenance...
# ssh worker-1 && apt upgrade -y && reboot

kubectl uncordon worker-1
kubectl get node worker-1   # Ready status restored
```

---

## 7.10 Complete Scheduling Example — Production Cluster

```yaml
# production-scheduling-complete.yaml
# A production-grade deployment using all scheduling concepts

---
# Priority Class
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: payment-critical
value: 900000
globalDefault: false
preemptionPolicy: PreemptLowerPriority

---
# LimitRange for namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: production-defaults
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"

---
# The Deployment with full scheduling controls
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 6
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
        tier: critical
    spec:
      priorityClassName: payment-critical

      # Tolerate production node taint
      tolerations:
      - key: environment
        operator: Equal
        value: production
        effect: NoSchedule

      affinity:
        # MUST run on production-labelled nodes
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: environment
                operator: In
                values: [production]
              - key: node-type
                operator: NotIn
                values: [spot, preemptible]

        # MUST NOT run multiple instances on same node
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: payment-service
            topologyKey: kubernetes.io/hostname

        # PREFER to spread across AZs
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: payment-service
              topologyKey: topology.kubernetes.io/zone

      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: payment-service

      containers:
      - name: payment
        image: payment-service:v3.1
        resources:
          requests:               # Guaranteed (Burstable QoS)
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "1Gi"
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 10
```

---

## Chapter 7: Hands-On Labs

### Lab 7.1 — NodeSelector and Node Affinity

```bash
# Label nodes for targeting
kubectl label node minikube disk=ssd env=production
kubectl get nodes --show-labels

# Deploy with nodeSelector
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disk: ssd
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl get pod ssd-pod -o wide    # Verify it runs on labelled node

# Now try a pod that CAN'T be placed (impossible nodeSelector)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: impossible-pod
spec:
  nodeSelector:
    disk: nvme-ultra-fast
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl get pod impossible-pod     # STATUS: Pending
kubectl describe pod impossible-pod | grep -A5 Events:
# 0/1 nodes are available: 1 node(s) didn't match Pod's node affinity

# Fix: Update nodeSelector OR label the node
kubectl label node minikube disk=nvme-ultra-fast --overwrite
kubectl get pod impossible-pod     # Now Running!

# Cleanup
kubectl delete pod ssd-pod impossible-pod
```

### Lab 7.2 — Taints and Tolerations

```bash
# Taint the node (minikube has 1 node)
kubectl taint node minikube special=gpu-only:NoSchedule
kubectl describe node minikube | grep Taint

# Try to deploy WITHOUT toleration
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl get pod no-toleration-pod   # STATUS: Pending!
kubectl describe pod no-toleration-pod | grep -A3 Events
# 1 node(s) had untolerated taint {special: gpu-only}

# Deploy WITH toleration
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  tolerations:
  - key: "special"
    operator: Equal
    value: "gpu-only"
    effect: NoSchedule
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl get pod toleration-pod      # Running!

# Remove the taint
kubectl taint node minikube special=gpu-only:NoSchedule-
kubectl get pod no-toleration-pod   # Still Pending (needs manual start)
kubectl delete pod no-toleration-pod toleration-pod

# Reapply - now works without taint
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
EOF
kubectl get pod no-toleration-pod   # Running now!
```

### Lab 7.3 — Resource Requests and QoS Classes

```bash
# GUARANTEED QoS: requests == limits
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "200m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
EOF

# BURSTABLE QoS: requests < limits
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
EOF

# BESTEFFORT QoS: no resources set
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
EOF

# Check QoS class for each
kubectl get pod guaranteed-pod \
  -o jsonpath='{.status.qosClass}'   # Guaranteed
kubectl get pod burstable-pod \
  -o jsonpath='{.status.qosClass}'   # Burstable
kubectl get pod besteffort-pod \
  -o jsonpath='{.status.qosClass}'   # BestEffort

# Check resource allocations on node
kubectl describe node minikube | grep -A10 "Allocated resources"

# Cleanup
kubectl delete pod guaranteed-pod burstable-pod besteffort-pod
```

### Lab 7.4 — Pod Anti-Affinity (Spread Pods)

```bash
# Deploy with anti-affinity (for multi-node clusters)
# In minikube: only 1 node, so pods after the first will be Pending
# Use kind with multiple nodes for this lab

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread-demo
  template:
    metadata:
      labels:
        app: spread-demo
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: spread-demo
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: nginx:alpine
EOF

kubectl get pods -l app=spread-demo -o wide
# In a multi-node cluster: pods distributed across nodes
# NODE column should show different nodes for each pod
```

---

## Chapter 7: Troubleshooting Guide

### Issue 1: Pod stuck in Pending due to resource constraints

```bash
kubectl describe pod my-pod
# Events: 0/3 nodes available: 3 Insufficient cpu

# Check current node resource usage
kubectl top nodes                     # Requires metrics-server
kubectl describe nodes | grep -A6 "Allocated resources:"

# Find nodes with enough free resources
kubectl get nodes -o json | jq '
  .items[] | {
    name: .metadata.name,
    cpuCapacity: .status.capacity.cpu,
    memCapacity: .status.capacity.memory
  }'

# Fix option 1: Reduce pod resource requests
# Fix option 2: Add more nodes to cluster
# Fix option 3: Scale down other deployments
kubectl scale deployment other-app --replicas=1

# Identify resource hogs
kubectl top pods -A --sort-by=memory | head -20
```

### Issue 2: Pod not scheduling on expected node

```bash
kubectl describe pod my-pod
# Check for node affinity / nodeSelector mismatches
# Events: didn't match node selector

# Check node labels
kubectl get nodes --show-labels
kubectl describe node worker-1 | grep Labels

# Check pod's nodeSelector / affinity
kubectl get pod my-pod -o yaml | grep -A20 affinity
kubectl get pod my-pod -o yaml | grep -A5 nodeSelector

# Fix: Add missing label to node
kubectl label node worker-1 disk=ssd

# Debug scheduling decisions using scheduler events
kubectl get events --field-selector reason=FailedScheduling
```

### Issue 3: Drain fails with PodDisruptionBudget

```bash
kubectl drain worker-1 --ignore-daemonsets
# Error: cannot evict pod "web-pod-xyz" as it would violate PDB

# Check PDBs in cluster
kubectl get pdb -A
kubectl describe pdb my-pdb

# PDB prevents drain if too few pods would remain available
# Fix option 1: Temporarily delete the PDB (risky)
kubectl delete pdb my-pdb

# Fix option 2: Scale up the deployment first so PDB is satisfied
kubectl scale deploy my-app --replicas=5
kubectl drain worker-1 --ignore-daemonsets

# Fix option 3: Force the drain (last resort — may cause downtime)
kubectl drain worker-1 --ignore-daemonsets --disable-eviction=true
```

### Issue 4: OOMKilled containers

```bash
kubectl get pod my-pod
# STATUS: OOMKilled

kubectl describe pod my-pod | grep -A5 "Last State:"
# Last State: Terminated  Reason: OOMKilled  Exit Code: 137

# Check memory usage trends
kubectl top pod my-pod --containers

# Fix: Increase memory limit
kubectl set resources deployment/my-app \
  --limits=memory=512Mi \
  --requests=memory=256Mi

# If OOMKill is repeated — the app has a memory leak
# Check app-level metrics and logs for leak diagnosis
kubectl logs my-pod --previous
```

### Issue 5: Pods evicted unexpectedly

```bash
kubectl get events --field-selector reason=Evicted
kubectl describe pod evicted-pod   # Should show eviction reason

# CAUSE: Node under memory pressure
kubectl describe node worker-1 | grep -A5 Conditions
# MemoryPressure = True

# CAUSE: Pod QoS class was BestEffort
kubectl get pod evicted-pod -o jsonpath='{.status.qosClass}'
# BestEffort pods evicted first!

# Fix: Set proper resource requests and limits
# This changes QoS from BestEffort to Burstable or Guaranteed
```

---

## Chapter 7: Interview Questions

**Q1: What is the difference between nodeSelector and Node Affinity?**

> `nodeSelector` is the simplest node selection — it only supports equality matching (key=value) and is an AND of all conditions. Node Affinity is more expressive — it supports operators (In, NotIn, Exists, DoesNotExist, Gt, Lt), multiple terms (OR logic between terms), and two modes: `requiredDuringScheduling` (hard rule — pod stays Pending if unmet) and `preferredDuringScheduling` (soft rule — scheduler prefers but doesn't require). Use nodeSelector for simple cases; Node Affinity for production where you need set-based rules and weighted preferences.

**Q2: What is the difference between Pod Affinity and Pod Anti-Affinity?**

> Pod Affinity schedules a pod **near** (on the same node or AZ as) pods matching a label selector. Pod Anti-Affinity schedules a pod **away from** pods matching a label selector. Both support required (hard) and preferred (soft) variants, and use a `topologyKey` to define the scope of "near" — `kubernetes.io/hostname` means same node, `topology.kubernetes.io/zone` means same availability zone. Anti-affinity is commonly used to spread replicas across nodes for HA, or ensure a database primary and replica never share a node.

**Q3: Explain Taints and Tolerations. How are they different from Affinity?**

> Taints are applied to **nodes** and **repel** all pods that don't tolerate them. Tolerations are applied to **pods** and allow them to schedule on tainted nodes despite the repulsion. There are three taint effects: `NoSchedule` (prevents new pods, doesn't evict existing), `PreferNoSchedule` (soft — avoids if possible), `NoExecute` (prevents new pods AND evicts existing ones). The key difference from affinity: taints+tolerations work as a DENY mechanism (nodes reject pods) while affinity is an ALLOW/PREFER mechanism (pods seek nodes). A complete node dedication uses BOTH: taint the node to repel others, and add affinity to make the target pod actively seek that node.

**Q4: What is the difference between CPU and memory resource behavior when limits are exceeded?**

> CPU is **compressible** — when a container exceeds its CPU limit, it is throttled (slowed down) but NOT killed. The container continues running at a reduced speed. Memory is **non-compressible** — when a container exceeds its memory limit, it is immediately killed with OOMKill (exit code 137) and restarted. This is why memory limits must be set conservatively and why OOMKill loops indicate either a memory leak or an undersized limit.

**Q5: What are the three QoS classes and how are they assigned?**

> Kubernetes automatically assigns QoS class based on resource settings. **Guaranteed** — every container has CPU and memory requests equal to limits; evicted last. **Burstable** — at least one container has requests less than limits, or only one of requests/limits is set; evicted second. **BestEffort** — no containers have any resource requests or limits set; evicted first under node pressure. For production workloads, aim for Guaranteed or Burstable. Never use BestEffort in production.

**Q6: What does `kubectl drain` do and how is it different from `kubectl cordon`?**

> `kubectl cordon` marks a node as unschedulable — it adds a taint so no NEW pods schedule there, but existing pods continue running. `kubectl drain` does both cordon AND evicts all existing pods from the node, moving them to other nodes. During maintenance, you cordon first (optional — drain does it automatically), then drain to empty the node safely. Drain respects PodDisruptionBudgets, ensuring enough replicas remain available. DaemonSet pods are ignored with `--ignore-daemonsets`. After maintenance, `kubectl uncordon` re-enables scheduling.

**Q7: What is a PodDisruptionBudget?**

> A PodDisruptionBudget (PDB) limits the number of pods that can be simultaneously unavailable during voluntary disruptions (like node drains or rolling updates). `minAvailable: 2` ensures at least 2 pods are always running. `maxUnavailable: 1` ensures at most 1 pod is down at a time. PDBs are respected by `kubectl drain` — if draining would violate the PDB, the drain is blocked. They protect services from unintended downtime during maintenance. PDBs do NOT protect against node failures (involuntary disruptions).

**Q8: What is a Priority Class and when would you use preemption?**

> A PriorityClass assigns numerical priority to pods. When a high-priority pod cannot be scheduled due to resource constraints, Kubernetes can **preempt** (evict) lower-priority pods to make room — this is called preemption. Use Priority Classes to ensure critical workloads (payment service, authentication) always get resources, even at the expense of lower-priority batch jobs or development workloads. System-critical built-in classes (`system-cluster-critical`, `system-node-critical`) protect Kubernetes system components from being evicted.

---

## CKA Exam Notes — Chapter 7

```
╔════════════════════════════════════════════════════════════════════╗
║                  CKA EXAM FOCUS — CHAPTER 7                       ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: Workloads & Scheduling = ~15%                       ║
║  Taints/tolerations and resource requests are heavily tested      ║
║                                                                    ║
║  FASTEST COMMANDS:                                                 ║
║  k taint node <n> key=val:NoSchedule                              ║
║  k taint node <n> key=val:NoSchedule-   ← remove taint           ║
║  k label node <n> key=value                                       ║
║  k label node <n> key-                  ← remove label           ║
║  k cordon <node>                                                   ║
║  k drain <node> --ignore-daemonsets --delete-emptydir-data        ║
║  k uncordon <node>                                                 ║
║  k top nodes                                                       ║
║  k top pods -A --sort-by=memory                                   ║
║                                                                    ║
║  TOLERATION YAML SKELETON (memorize):                              ║
║  tolerations:                                                      ║
║  - key: "dedicated"                                                ║
║    operator: Equal                                                 ║
║    value: "gpu-only"                                               ║
║    effect: NoSchedule                                              ║
║                                                                    ║
║  NODE AFFINITY SKELETON (memorize):                                ║
║  affinity:                                                         ║
║    nodeAffinity:                                                   ║
║      requiredDuringSchedulingIgnoredDuringExecution:               ║
║        nodeSelectorTerms:                                          ║
║        - matchExpressions:                                         ║
║          - key: disk                                               ║
║            operator: In                                            ║
║            values: [ssd]                                           ║
║                                                                    ║
║  EXAM TRAPS:                                                       ║
║  Taint on node + pod has NO toleration = Pending (not error)      ║
║  nodeSelector is under spec, NOT spec.affinity                    ║
║  drain needs --ignore-daemonsets or it fails                      ║
║  drain needs --delete-emptydir-data for emptyDir pods             ║
║  QoS class auto-assigned — cannot be set manually                 ║
║  requests == limits → Guaranteed (lowest eviction risk)           ║
║  No resources at all → BestEffort (first to be evicted!)          ║
║                                                                    ║
║  DECODE SCHEDULING FAILURE FAST:                                   ║
║  1. k describe pod → Events section                               ║
║  2. "insufficient cpu/memory" → k top nodes, reduce requests      ║
║  3. "didn't match node affinity" → k get nodes --show-labels      ║
║  4. "had untolerated taint" → k describe node | grep Taint        ║
║  5. "didn't match nodeSelector" → k get nodes --show-labels       ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Common Mistakes — Chapter 7

| Mistake | Why It Happens | How to Avoid |
|---|---|---|
| nodeSelector under wrong spec level | Misplacing it in spec.template.spec.affinity | nodeSelector is directly under `spec:` not under `affinity:` |
| Taint without matching toleration | Testing taint works but forgetting toleration | Always add toleration to intended pods BEFORE tainting node |
| Drain without `--ignore-daemonsets` | Drain tries to evict DaemonSet pods, fails | Always use `--ignore-daemonsets` flag with drain |
| No resource requests in production | "It works without them" | Always set requests; missing = BestEffort = first evicted |
| requests > limits (impossible) | Typo | Requests must always be ≤ limits |
| Memory in CPU units or vice versa | e.g., memory: 256m (millicores!) | Memory uses Mi/Gi; CPU uses m (millicores) or plain numbers |
| Anti-affinity with required on single-node | Testing on minikube | Use preferred for dev; required for multi-node prod |
| Forgetting `topologyKey` in affinity | Optional-looking but required field | topologyKey is REQUIRED in pod affinity/anti-affinity |

---

## Chapter 7 Summary

1. **Scheduler phases** — Filtering (eliminate unfit nodes) → Scoring (rank feasible nodes)
2. **NodeSelector** — simple label equality matching; AND between all selectors
3. **Node Affinity** — expressive (In/NotIn/Exists operators); required (hard) vs preferred (soft)
4. **Pod Affinity** — schedule near pods with matching labels; use topologyKey for scope
5. **Pod Anti-Affinity** — spread pods apart; HA pattern for databases and critical services
6. **TopologySpreadConstraints** — modern, declarative pod spreading across zones/nodes
7. **Taints** — repel pods from nodes; effects: NoSchedule, PreferNoSchedule, NoExecute
8. **Tolerations** — allow pods to overcome node taints; Equal or Exists operators
9. **Resource Requests** — scheduler guarantee; used for bin-packing decisions
10. **Resource Limits** — runtime cap; CPU throttles, memory OOMKills
11. **QoS Classes** — Guaranteed (requests=limits) → Burstable → BestEffort (evicted first)
12. **Priority Classes** — preemption; high-priority pods can evict lower-priority pods
13. **cordon/drain/uncordon** — node maintenance workflow; respects PodDisruptionBudgets

---

*Next: Chapter 8 — Security: RBAC, Service Accounts, Network Policies & Security Contexts*

*"You can schedule pods anywhere with perfect efficiency — but can you prove
 that only the right people and processes can do so? That's Chapter 8."*
