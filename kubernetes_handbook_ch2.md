# KUBERNETES MASTERY HANDBOOK
# Part 2: Kubernetes Core Objects
# Chapter 2: Pods, ReplicaSets, Deployments, Namespaces, Labels & Annotations

---

> **"Everything in Kubernetes is an object. Learn to think in objects, and Kubernetes becomes readable."**

---

## Chapter Introduction

In Chapter 1 you learned *how* Kubernetes works internally — the brain (Control Plane) and the muscles (Worker Nodes). Now you'll learn *what* Kubernetes manages.

Every application in Kubernetes is described as an **object** — a structured declaration of what you want. You write it in YAML, submit it to Kubernetes, and the system makes it real.

This chapter covers the foundational objects you will use *every single day*:

```
╔══════════════════════════════════════════════════════════════════╗
║              CORE OBJECTS — CHAPTER ROADMAP                      ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Namespace ──────────────────────────────────────────────────    ║
║  │  (logical boundary for all other objects)                     ║
║  │                                                               ║
║  ├── Pod ──────────────────────────────────────────────────      ║
║  │   │  (smallest deployable unit — one or more containers)      ║
║  │   └── Labels/Annotations (metadata attached to any object)    ║
║  │                                                               ║
║  ├── ReplicaSet ───────────────────────────────────────────      ║
║  │   │  (ensures N copies of a Pod are always running)           ║
║  │   └── (uses Labels/Selectors to find its Pods)                ║
║  │                                                               ║
║  └── Deployment ──────────────────────────────────────────       ║
║      │  (manages ReplicaSets — adds rolling updates, rollback)   ║
║      └── (the object you'll use 90% of the time)                 ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 2.1 Pods — The Atomic Unit of Kubernetes

### 2.1.1 What Is a Pod?

#### In Plain English

A Pod is like a **room in a house**. The house is the worker node. Containers are the people living in the room. Containers in the same pod share everything in that room — the same address (IP), the same storage space, and they can talk to each other through a local intercom (localhost).

Most of the time, one room has one person (one container per pod). But sometimes you need a roommate — a helper container that does a supporting job (like a sidecar).

#### In Technical Language

A **Pod** is the smallest and simplest deployable unit in Kubernetes. It is a **wrapper around one or more containers** that:

- Share the same **network namespace** (same IP address and port space)
- Share the same **IPC namespace** (can communicate via shared memory)
- Can share **storage volumes** (mounted into each container)
- Are **always co-located** on the same node
- Are **co-scheduled** — they start and stop together

```
╔═══════════════════════════════════════════════════════════════════╗
║                        POD ANATOMY                                ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  ┌─────────────────────────────────────────────────────────────┐  ║
║  │                    POD (IP: 10.244.1.5)                     │  ║
║  │                                                             │  ║
║  │  ┌─────────────────┐    ┌─────────────────┐                 │  ║
║  │  │  Main Container │    │Sidecar Container│                 │  ║
║  │  │  (nginx:1.21)   │    │ (log-collector) │                    ║
║  │  │                 │    │                 │                 │  ║
║  │  │  Port 80        │    │  Port 9090      │                 │  ║
║  │  └────────┬────────┘    └────────┬────────┘                 │  ║
║  │           │                      │                          │  ║
║  │    Both containers share:        │                          │  ║
║  │    • Same IP (10.244.1.5)        │                          │  ║
║  │    • Same localhost              │                          │  ║
║  │    • Same /shared-logs volume    │                          │  ║
║  │           │                      │                          │  ║
║  │  ┌────────┴──────────────────────┘                          │  ║
║  │  │         Shared Volume: /shared-logs                      │  ║
║  │  └──────────────────────────────────────────────────────    │  ║
║  │                                                             │  ║
║  │  pause container (invisible) ← holds network namespace      │  ║
║  └─────────────────────────────────────────────────────────────┘  ║
║                                                                   ║
║  NOTE: The "pause" (or "infra") container is created first.       ║
║  It holds the network namespace open so containers can restart    ║
║  without losing the Pod's IP address.                             ║
╚═══════════════════════════════════════════════════════════════════╝
```

### 2.1.2 Pod Lifecycle

Understanding pod lifecycle is critical for debugging.

```
╔═══════════════════════════════════════════════════════════════════╗
║                     POD LIFECYCLE STATES                          ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  Pending ──→ Running ──→ Succeeded (for Jobs)                     ║
║     │                                                             ║
║     │        Running ──→ Failed                                   ║
║     │                                                             ║
║     └──→ Unknown (node communication lost)                        ║
║                                                                   ║
║  STATE       MEANING                                              ║
║  ─────────   ───────────────────────────────────────────────      ║
║  Pending     Pod accepted by API server, not yet running.         ║
║              Could be: scheduling, image pulling, init            ║
║                                                                   ║
║  Running     At least one container is running. Others may        ║
║              still be starting or restarting.                     ║
║                                                                   ║
║  Succeeded   All containers exited with code 0 (success).         ║
║              Won't restart. (Common for Job pods)                 ║
║                                                                   ║
║  Failed      All containers have stopped; at least one exited     ║
║              with non-zero code or was killed by system.          ║
║                                                                   ║
║  Unknown     Pod state cannot be determined (node issue).         ║
║                                                                   ║
║  CONTAINER STATES (inside a pod):                                 ║
║  Waiting   → Running → Terminated                                 ║
╚═══════════════════════════════════════════════════════════════════╝
```

### 2.1.3 Pod YAML — The Anatomy

```yaml
# pod-definition.yaml
# Every Kubernetes object has these 4 mandatory top-level fields:

apiVersion: v1              # Which API version defines this object
kind: Pod                   # What type of object
metadata:                   # Data ABOUT the object (name, labels, etc.)
  name: my-web-pod          # Must be unique within namespace
  namespace: default        # Which namespace (default if not specified)
  labels:                   # Key-value pairs for selection/grouping
    app: web
    version: v1
    env: production
  annotations:              # Key-value pairs for metadata (not for selection)
    description: "Frontend web server pod"
    team: "platform-engineering"
spec:                       # The DESIRED STATE — what should exist
  containers:
  - name: nginx-container   # Container name (unique within pod)
    image: nginx:1.21       # Image to run (imagename:tag)
    ports:
    - containerPort: 80     # Port the container listens on (informational only)
    resources:              # CPU/Memory limits
      requests:             # Minimum guaranteed
        memory: "64Mi"
        cpu: "250m"         # 250 millicores = 0.25 CPU
      limits:               # Maximum allowed
        memory: "128Mi"
        cpu: "500m"
    env:                    # Environment variables
    - name: ENV_NAME
      value: "production"
    volumeMounts:           # Where to mount volumes inside container
    - name: config-volume
      mountPath: /etc/config
  
  initContainers:           # Run BEFORE main containers (in order)
  - name: init-db-check
    image: busybox
    command: ['sh', '-c', 'until nslookup database; do sleep 2; done']
  
  volumes:                  # Volumes available to containers in this pod
  - name: config-volume
    configMap:
      name: app-config

  restartPolicy: Always     # Always | OnFailure | Never
  nodeName: worker-node-1   # Pin to specific node (usually let scheduler decide)
  serviceAccountName: default
```

### 2.1.4 Multi-Container Pod Patterns

In production, you'll encounter three established multi-container patterns:

```
╔══════════════════════════════════════════════════════════════════╗
║              MULTI-CONTAINER PATTERNS                            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  PATTERN 1: SIDECAR                                              ║
║  ┌──────────────────────────────────────┐                        ║
║  │ POD                                  │                        ║
║  │  ┌──────────────┐  ┌──────────────┐  │                        ║
║  │  │ Main App     │  │  Sidecar     │  │                       ║
║  │  │ (writes logs │  │ (ships logs  │  │                        ║
║  │  │  to /logs)   │  │  to ELK)    │  │                       ║
║  │  └──────────────┘  └──────────────┘  │                       ║
║  │  └──── Shared volume: /logs ──────┘  │                       ║
║  └──────────────────────────────────────┘                       ║
║  Use case: Log shipping, metrics collection, config sync        ║
║                                                                  ║
║  PATTERN 2: AMBASSADOR                                          ║
║  ┌──────────────────────────────────────┐                       ║
║  │ POD                                  │                       ║
║  │  ┌──────────────┐  ┌──────────────┐  │                       ║
║  │  │ Main App     │  │ Ambassador   │  │                       ║
║  │  │ (connects to │→ │ (proxies to  │→ External DB            ║
║  │  │  localhost)  │  │  real DB)    │  │                       ║
║  │  └──────────────┘  └──────────────┘  │                       ║
║  └──────────────────────────────────────┘                       ║
║  Use case: DB proxy, service mesh (Envoy), SSL termination      ║
║                                                                  ║
║  PATTERN 3: ADAPTER                                             ║
║  ┌──────────────────────────────────────┐                       ║
║  │ POD                                  │                       ║
║  │  ┌──────────────┐  ┌──────────────┐  │                       ║
║  │  │ Main App     │  │  Adapter     │  │                       ║
║  │  │ (custom      │→ │ (transforms  │→ Monitoring System      ║
║  │  │  metrics     │  │  to standard │  │                       ║
║  │  │  format)     │  │  Prometheus) │  │                       ║
║  │  └──────────────┘  └──────────────┘  │                       ║
║  └──────────────────────────────────────┘                       ║
║  Use case: Metrics normalization, protocol translation          ║
╚══════════════════════════════════════════════════════════════════╝
```

### 2.1.5 Pod Health Checks (Probes)

Kubernetes uses **probes** to know if your container is healthy:

```yaml
# health-probes.yaml
apiVersion: v1
kind: Pod
metadata:
  name: healthy-app
spec:
  containers:
  - name: app
    image: my-app:v1
    
    # LIVENESS PROBE: Is the container alive? 
    # If fails → container is killed and restarted
    livenessProbe:
      httpGet:
        path: /healthz       # Kubernetes calls this endpoint
        port: 8080
      initialDelaySeconds: 15   # Wait 15s before first probe
      periodSeconds: 20         # Check every 20s
      failureThreshold: 3       # Restart after 3 consecutive failures
    
    # READINESS PROBE: Is the container ready to receive traffic?
    # If fails → removed from Service endpoints (no traffic sent)
    # Pod keeps running but gets no requests
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      successThreshold: 1       # Mark ready after 1 success
    
    # STARTUP PROBE: For slow-starting containers.
    # Kubernetes won't run liveness/readiness until startup probe succeeds.
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30      # 30 * 10s = 5 minutes to start
      periodSeconds: 10
```

```
╔══════════════════════════════════════════════════════════════╗
║             PROBE TYPES COMPARISON                          ║
╠════════════════╦═════════════════════╦═════════════════════╣
║  Probe Type    ║  Failure Action      ║  Use Case           ║
╠════════════════╬═════════════════════╬═════════════════════╣
║  Liveness      ║  Restart container  ║  Detect deadlocks,  ║
║                ║                     ║  frozen apps        ║
╠════════════════╬═════════════════════╬═════════════════════╣
║  Readiness     ║  Remove from        ║  App still loading  ║
║                ║  Service endpoints  ║  DB not connected   ║
╠════════════════╬═════════════════════╬═════════════════════╣
║  Startup       ║  Restart container  ║  Slow-starting apps ║
║                ║  (but generous time)║  (JVM, legacy apps) ║
╚════════════════╩═════════════════════╩═════════════════════╝
```

### 2.1.6 Pod Commands

```bash
# ─── CREATING PODS ───────────────────────────────────────────────────
# Imperative (quick, for testing):
kubectl run nginx-pod --image=nginx:1.21
kubectl run busybox-pod --image=busybox --command -- sleep 3600
kubectl run redis --image=redis:6 --port=6379 --env="REDIS_PASSWORD=secret"

# Generate YAML without creating (your best friend):
kubectl run nginx-pod --image=nginx --dry-run=client -o yaml
kubectl run nginx-pod --image=nginx --dry-run=client -o yaml > pod.yaml

# Declarative (from YAML file):
kubectl apply -f pod.yaml

# ─── INSPECTING PODS ─────────────────────────────────────────────────
kubectl get pods                         # List pods in current namespace
kubectl get pods -n kube-system          # Pods in kube-system namespace
kubectl get pods --all-namespaces        # All pods across all namespaces
kubectl get pods -A                      # Shorthand for --all-namespaces
kubectl get pods -o wide                 # Include node, IP info
kubectl get pods -o yaml                 # Full YAML output
kubectl get pods -o json                 # Full JSON output
kubectl get pod my-pod -o jsonpath='{.status.podIP}'  # Get specific field
kubectl get pods --watch                 # Watch for real-time changes
kubectl get pods -w                      # Shorthand

# ─── DESCRIBING PODS ─────────────────────────────────────────────────
kubectl describe pod my-pod              # Full details including Events
# Events section is THE most useful for debugging

# ─── LOGS ────────────────────────────────────────────────────────────
kubectl logs my-pod                      # Current logs
kubectl logs my-pod -f                   # Follow logs (stream)
kubectl logs my-pod --previous           # Logs from previous (crashed) container
kubectl logs my-pod -c container-name    # Specific container in multi-container pod
kubectl logs my-pod --tail=100           # Last 100 lines
kubectl logs my-pod --since=1h           # Last 1 hour of logs

# ─── EXECUTING COMMANDS ──────────────────────────────────────────────
kubectl exec my-pod -- ls /app           # Run command in pod
kubectl exec my-pod -- env               # Print environment variables
kubectl exec -it my-pod -- bash          # Interactive shell (bash)
kubectl exec -it my-pod -- sh            # Interactive shell (sh, for alpine)
kubectl exec -it my-pod -c sidecar -- sh # Shell into specific container

# ─── PORT FORWARDING ─────────────────────────────────────────────────
kubectl port-forward pod/my-pod 8080:80  # localhost:8080 → pod:80
kubectl port-forward pod/my-pod 8080:80 &  # Run in background

# ─── DELETING PODS ───────────────────────────────────────────────────
kubectl delete pod my-pod                # Graceful delete (30s termination)
kubectl delete pod my-pod --grace-period=0 --force  # Immediate delete
kubectl delete pod my-pod1 my-pod2       # Delete multiple
kubectl delete pods --all                # Delete ALL pods in namespace
kubectl delete pods -l app=web           # Delete pods by label
```

---

## 2.2 Namespaces — Logical Boundaries

### 2.2.1 What Is a Namespace?

#### In Plain English

A Namespace is like a **department in a company**. HR, Engineering, and Finance all work in the same building (cluster), but they have their own offices (namespaces), their own whiteboards (resources), and their own rules about who can enter (RBAC). Resources in different departments don't conflict even if they have the same name.

#### In Technical Language

A **Namespace** is a Kubernetes mechanism for isolating groups of resources within a single cluster. Namespaces provide:

- **Name scoping** — Two pods can have the same name in different namespaces
- **Resource quotas** — Limit CPU/memory per namespace
- **Access control** — RBAC policies can be namespace-scoped
- **Network isolation** — With Network Policies

```
╔══════════════════════════════════════════════════════════════════════╗
║                    NAMESPACES IN A CLUSTER                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  KUBERNETES CLUSTER                                                  ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │                                                                │  ║
║  │  ┌──────────────────┐  ┌──────────────────┐                   │  ║
║  │  │  kube-system     │  │   kube-public    │                   │  ║
║  │  │  (system pods)   │  │  (public info)   │                   │  ║
║  │  │  etcd, apiserver │  │  cluster-info    │                   │  ║
║  │  │  scheduler, etc. │  │                  │                   │  ║
║  │  └──────────────────┘  └──────────────────┘                   │  ║
║  │                                                                │  ║
║  │  ┌──────────────────┐  ┌──────────────────┐                   │  ║
║  │  │  kube-node-lease │  │    default       │                   │  ║
║  │  │  (node heartbeat)│  │  (your apps if   │                   │  ║
║  │  │                  │  │   no NS specified)│                  │  ║
║  │  └──────────────────┘  └──────────────────┘                   │  ║
║  │                                                                │  ║
║  │  ┌──────────────────┐  ┌──────────────────┐                   │  ║
║  │  │   development    │  │   production     │ ← Your namespaces │  ║
║  │  │  ┌────┐ ┌────┐   │  │  ┌────┐ ┌────┐  │                   │  ║
║  │  │  │pod │ │pod │   │  │  │pod │ │pod │  │                   │  ║
║  │  │  └────┘ └────┘   │  │  └────┘ └────┘  │                   │  ║
║  │  │  "web" pod       │  │  "web" pod ← same name, no conflict  │  ║
║  │  └──────────────────┘  └──────────────────┘                   │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
║                                                                      ║
║  Default Namespaces (auto-created):                                  ║
║  • default       — Where resources go if no namespace specified      ║
║  • kube-system   — Kubernetes internal system components             ║
║  • kube-public   — Readable by all (unauthenticated) users           ║
║  • kube-node-lease — Node heartbeat lease objects                    ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 2.2.2 Namespace YAML

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    env: dev
    team: backend
---
# ResourceQuota — limit what a namespace can consume
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    pods: "20"                    # Max 20 pods
    requests.cpu: "4"             # Max 4 CPU requested
    requests.memory: 8Gi          # Max 8GB memory requested
    limits.cpu: "8"               # Max 8 CPU limit
    limits.memory: 16Gi           # Max 16GB memory limit
    persistentvolumeclaims: "10"  # Max 10 PVCs
---
# LimitRange — set defaults for containers in namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - default:           # Default limits if not specified
      cpu: 500m
      memory: 256Mi
    defaultRequest:    # Default requests if not specified
      cpu: 100m
      memory: 64Mi
    type: Container
```

### 2.2.3 Namespace Commands

```bash
# ─── NAMESPACE OPERATIONS ────────────────────────────────────────────
kubectl get namespaces                   # List all namespaces
kubectl get ns                           # Shorthand
kubectl create namespace staging         # Create imperatively
kubectl apply -f namespace.yaml          # Create declaratively
kubectl delete namespace staging         # DELETE namespace (deletes all resources inside!)

# ─── WORKING WITHIN A NAMESPACE ──────────────────────────────────────
kubectl get pods -n development          # Pods in 'development' ns
kubectl get all -n production            # All resources in 'production'
kubectl apply -f app.yaml -n staging     # Apply to specific namespace

# Set default namespace for current context (avoid typing -n constantly):
kubectl config set-context --current --namespace=development
# Now all commands default to 'development' namespace
kubectl config set-context --current --namespace=default  # Reset

# ─── CROSS-NAMESPACE DNS ─────────────────────────────────────────────
# Service in another namespace is reachable via:
# <service-name>.<namespace>.svc.cluster.local
# Example: database.production.svc.cluster.local
```

---

## 2.3 Labels and Selectors — The Glue of Kubernetes

### 2.3.1 What Are Labels?

#### In Plain English

Labels are **sticky notes** you put on your Kubernetes objects. They're key-value pairs like `env=production` or `app=web`. The magic is that other objects (like Services and ReplicaSets) use these sticky notes to *find* and *select* the right pods.

#### In Technical Language

**Labels** are key-value pairs attached to Kubernetes objects. They are:
- Arbitrary (you decide what labels mean)
- Used for **selection** — identifying groups of objects
- Used for **filtering** — `kubectl get pods -l app=web`
- NOT meant for storing detailed non-identifying info (use Annotations for that)

**Selectors** are how you query for objects by their labels.

```
╔══════════════════════════════════════════════════════════════════════╗
║                   LABELS & SELECTORS IN ACTION                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Pods with labels:                                                   ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  Pod-A  │  app=web  │  env=prod  │  version=v1  │ tier=front │   ║
║  │  Pod-B  │  app=web  │  env=prod  │  version=v2  │ tier=front │   ║
║  │  Pod-C  │  app=web  │  env=dev   │  version=v1  │ tier=front │   ║
║  │  Pod-D  │  app=db   │  env=prod  │  version=v1  │ tier=back  │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  Service selector: app=web, env=prod                                 ║
║  → Selects: Pod-A, Pod-B  (both have app=web AND env=prod)           ║
║  → Excludes: Pod-C (env=dev), Pod-D (app=db)                         ║
║                                                                      ║
║  ReplicaSet selector: app=web                                        ║
║  → Selects: Pod-A, Pod-B, Pod-C  (all with app=web)                 ║
║                                                                      ║
║  kubectl selector: version=v1                                        ║
║  → Selects: Pod-A, Pod-C, Pod-D                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 2.3.2 Label Naming Conventions (Production Best Practice)

```yaml
# Recommended label set (from Kubernetes docs)
metadata:
  labels:
    app.kubernetes.io/name: my-app           # App name
    app.kubernetes.io/instance: my-app-prod  # Unique instance name
    app.kubernetes.io/version: "1.2.3"       # App version/SemVer
    app.kubernetes.io/component: frontend    # Component type
    app.kubernetes.io/part-of: my-platform   # Larger app this belongs to
    app.kubernetes.io/managed-by: helm       # Tool used to manage it
    
    # Custom labels for your team:
    team: platform-engineering
    env: production
    cost-center: "engineering-001"
```

### 2.3.3 Selector Types

```yaml
# EQUALITY-BASED SELECTORS (simple matching)
selector:
  matchLabels:
    app: web             # app EQUALS web
    env: production      # AND env EQUALS production

# SET-BASED SELECTORS (more expressive)
selector:
  matchExpressions:
  - key: app
    operator: In              # Value must be IN this set
    values: [web, api]
  - key: env
    operator: NotIn           # Value must NOT be in this set
    values: [test]
  - key: tier
    operator: Exists          # Key must exist (any value)
  - key: deprecated
    operator: DoesNotExist    # Key must NOT exist
```

### 2.3.4 Annotations — Metadata Without Selection

**In Plain English:** Annotations are like the full description on the back of a business card — richer information that's not used for identification.

**In Technical Language:** Annotations store arbitrary non-identifying metadata. Unlike labels, annotations are NOT used by selectors. They're used for:
- Build/release info
- Tool-specific configuration (ingress controllers, monitoring)
- Human-readable descriptions
- Timestamps and audit info

```yaml
metadata:
  annotations:
    # Kubernetes built-in annotations
    kubernetes.io/change-cause: "Updated nginx to 1.21 for security patch"
    
    # Ingress controller config
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Monitoring
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
    
    # Custom metadata
    contact: "platform-team@company.com"
    documentation: "https://wiki.company.com/service-x"
    last-reviewed: "2024-01-15"
```

### 2.3.5 Label Commands

```bash
# ─── ADD LABELS ──────────────────────────────────────────────────────
kubectl label pod my-pod env=production          # Add label
kubectl label pod my-pod env=staging --overwrite # Update label
kubectl label node worker-1 disk=ssd             # Label a node
kubectl label pod my-pod env-                    # REMOVE label (note the dash)

# ─── FILTER BY LABELS (Selectors) ────────────────────────────────────
kubectl get pods -l app=web                      # Equality
kubectl get pods -l "app=web,env=prod"           # Multiple (AND)
kubectl get pods -l "env in (prod,staging)"      # Set-based
kubectl get pods -l "env!=test"                  # Not equal
kubectl get pods -l "team"                       # Key exists
kubectl get pods --show-labels                   # Show all labels

# ─── ANNOTATIONS ─────────────────────────────────────────────────────
kubectl annotate pod my-pod description="Web server"
kubectl annotate pod my-pod description="Updated" --overwrite
kubectl annotate pod my-pod description-          # Remove annotation
```

---

## 2.4 ReplicaSets — Ensuring Availability

### 2.4.1 What Is a ReplicaSet?

#### In Plain English

A ReplicaSet is a **guarantee**. You tell Kubernetes: "I need exactly 3 copies of my web app running at all times." The ReplicaSet enforces this contract. If one pod crashes, it creates a replacement. If a node dies, it starts pods on surviving nodes.

#### In Technical Language

A **ReplicaSet** ensures that a specified number of Pod replicas are running at any given time. It uses a **label selector** to identify which Pods it manages, and continuously reconciles the actual count with the desired count.

```
╔═══════════════════════════════════════════════════════════════════╗
║                  REPLICASET RECONCILIATION LOOP                  ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  DESIRED: replicas: 3                                             ║
║                                                                   ║
║  Scenario 1 — Pod crashes:                                        ║
║  ┌────────┐  ┌────────┐  ┌────────X  ← Pod-C crashes            ║
║  │ Pod-A  │  │ Pod-B  │  │ Pod-C  │                              ║
║  └────────┘  └────────┘  └─────────                              ║
║                                ↓                                  ║
║  ReplicaSet sees 2 running, wants 3 → Creates Pod-D             ║
║  ┌────────┐  ┌────────┐  ┌────────┐                              ║
║  │ Pod-A  │  │ Pod-B  │  │ Pod-D  │  ✅ Back to 3               ║
║  └────────┘  └────────┘  └────────┘                              ║
║                                                                   ║
║  Scenario 2 — Someone manually adds a matching pod:              ║
║  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ← 4 pods!     ║
║  │ Pod-A  │  │ Pod-B  │  │ Pod-D  │  │ Pod-E  │                  ║
║  └────────┘  └────────┘  └────────┘  └───────X                  ║
║                                ↓                                  ║
║  ReplicaSet sees 4 running, wants 3 → Terminates Pod-E          ║
║                                                                   ║
║  The ReplicaSet OWNS pods via label selectors.                   ║
║  DANGER: Never create a pod with labels that match an existing   ║
║  ReplicaSet — it will get immediately terminated!                ║
╚═══════════════════════════════════════════════════════════════════╝
```

### 2.4.2 ReplicaSet YAML

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-replicaset
  namespace: default
  labels:
    app: web
    tier: frontend
spec:
  replicas: 3                    # How many pod copies we want

  selector:                      # How RS finds pods it manages
    matchLabels:                 # Pods with THESE labels are "owned" by this RS
      app: web
      tier: frontend

  template:                      # Blueprint for creating new pods
    metadata:
      labels:                    # MUST include all labels in selector
        app: web
        tier: frontend
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
```

### 2.4.3 ReplicaSet Commands

```bash
# ─── REPLICASET OPERATIONS ───────────────────────────────────────────
kubectl get replicasets                    # List ReplicaSets
kubectl get rs                             # Shorthand
kubectl describe rs web-replicaset         # Full details
kubectl get rs -o wide                     # With more info

# ─── SCALING ─────────────────────────────────────────────────────────
kubectl scale rs web-replicaset --replicas=5   # Scale up to 5
kubectl scale rs web-replicaset --replicas=1   # Scale down to 1
kubectl scale rs web-replicaset --replicas=0   # Scale to 0 (deletes all pods)

# ─── DELETING ────────────────────────────────────────────────────────
kubectl delete rs web-replicaset               # Deletes RS and its pods
kubectl delete rs web-replicaset --cascade=orphan  # Deletes RS but KEEPS pods

# ─── CHECK WHICH RS OWNS A POD ───────────────────────────────────────
kubectl get pod my-pod -o yaml | grep -A5 ownerReferences
```

> ⚠️ **Important:** You will rarely create ReplicaSets directly in production. Deployments manage ReplicaSets for you and add rolling updates and rollback. Always use Deployments in practice.

---

## 2.5 Deployments — The Workhorse of Kubernetes

### 2.5.1 What Is a Deployment?

#### In Plain English

A Deployment is a ReplicaSet with **superpowers**. It not only keeps the right number of pods running — it also handles updates safely. When you release a new version of your app, a Deployment rolls it out gradually (so users never see downtime), and if something goes wrong, rolls it back instantly.

#### In Technical Language

A **Deployment** provides declarative updates for Pods and ReplicaSets. You describe the *desired state* in the Deployment, and the Deployment Controller changes the actual state at a controlled rate. Deployments manage ReplicaSets — when you update a Deployment, it creates a *new* ReplicaSet and scales it up while scaling down the old one.

```
╔═══════════════════════════════════════════════════════════════════════╗
║              DEPLOYMENT → REPLICASET → POD HIERARCHY                ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ┌────────────────────────────────────────────────────────────────┐   ║
║  │                   DEPLOYMENT: web-app                          │   ║
║  │  strategy: RollingUpdate                                       │   ║
║  │  replicas: 3                                                   │   ║
║  └──────────────────────────────┬─────────────────────────────────┘   ║
║                                 │ manages                             ║
║                    ┌────────────┴──────────────┐                      ║
║                    │                           │                      ║
║          ┌─────────▼──────────┐     ┌──────────▼─────────┐           ║
║          │  ReplicaSet v1     │     │  ReplicaSet v2     │           ║
║          │  (image: app:v1)   │     │  (image: app:v2)   │           ║
║          │  replicas: 0 ← being│     │  replicas: 3 ←new  │           ║
║          │  scaled down        │     │  being scaled up   │           ║
║          └────────────────────┘     └────────────────────┘           ║
║          Pods:  (none left)          Pods: P4, P5, P6                 ║
║                                                                       ║
║  Rolling Update in Progress:                                         ║
║  Time→  [v1,v1,v1] → [v2,v1,v1] → [v2,v2,v1] → [v2,v2,v2]         ║
║                                                                       ║
║  Old RS kept at 0 replicas (not deleted) for ROLLBACK                ║
║  kubectl rollout undo deployment/web-app → scales old RS back up     ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### 2.5.2 Deployment YAML — Production-Grade

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
    version: v2
  annotations:
    kubernetes.io/change-cause: "Update nginx to 1.21 - security patch CVE-2021-xxxx"
spec:
  replicas: 3              # Desired number of pods

  selector:
    matchLabels:
      app: web-app         # Must match pod template labels below

  # ── Rolling Update Strategy ──────────────────────────────────────
  strategy:
    type: RollingUpdate    # Default. Alternative: Recreate
    rollingUpdate:
      maxSurge: 1          # Max extra pods during update (above desired count)
                           # Can be number or percentage: "25%"
      maxUnavailable: 0    # Max pods that can be down during update
                           # 0 = zero-downtime deployment

  # ── Pod Template ─────────────────────────────────────────────────
  template:
    metadata:
      labels:
        app: web-app       # MUST match selector above
        version: v2
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      # Security: don't run as root
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000

      # Graceful shutdown
      terminationGracePeriodSeconds: 30

      containers:
      - name: nginx
        image: nginx:1.21   # Change this to trigger rolling update
        imagePullPolicy: IfNotPresent  # Always | Never | IfNotPresent
        ports:
        - name: http
          containerPort: 80

        # Resources — ALWAYS set these in production
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"

        # Health checks
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20

        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10

        # Lifecycle hooks
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]  # Allow in-flight requests to complete

      # Restart pods across nodes, not together on one node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: web-app
              topologyKey: kubernetes.io/hostname

  # Keep old ReplicaSet history for rollback
  revisionHistoryLimit: 10

  # If a pod takes more than 600s to become Ready → deployment fails
  progressDeadlineSeconds: 600
```

### 2.5.3 Deployment Update Strategies

```
╔══════════════════════════════════════════════════════════════════════╗
║                  UPDATE STRATEGIES COMPARED                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  RECREATE STRATEGY                                                   ║
║  ─────────────────                                                   ║
║  Step 1: Kill ALL old pods                                           ║
║  Step 2: Create ALL new pods                                         ║
║                                                                      ║
║  Timeline:  [v1,v1,v1] → [☠,☠,☠] → [v2,v2,v2]                    ║
║  Downtime:  YES (gap between kill and new pods ready)               ║
║  Use when:  Database schema changes require complete version switch  ║
║                                                                      ║
║  ROLLING UPDATE STRATEGY (Default)                                  ║
║  ─────────────────────────────────                                   ║
║  maxSurge: 1, maxUnavailable: 0                                     ║
║                                                                      ║
║  Timeline:                                                           ║
║  Start:    [v1, v1, v1]          (3 pods, 0 surge, 0 unavailable)  ║
║  Step 1:   [v1, v1, v1, v2]     (4 pods, 1 surge) — v2 starting   ║
║  Step 2:   [v1, v1, ☠, v2,v2]  wait for v2 ready, kill 1 v1      ║
║  Step 3:   [v1, v2, v2]         → [v1, v2, v2, v2-new]            ║
║  Final:    [v2, v2, v2]                                             ║
║                                                                      ║
║  Downtime:  NONE (maxUnavailable: 0)                                ║
║  Speed:     Slower (controlled rollout)                             ║
║  Use when:  Most production deployments                              ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 2.5.4 Deployment Commands

```bash
# ─── CREATING DEPLOYMENTS ────────────────────────────────────────────
# Imperative:
kubectl create deployment web-app --image=nginx:1.21 --replicas=3
# Generate YAML:
kubectl create deployment web-app --image=nginx:1.21 --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

# Declarative:
kubectl apply -f deployment.yaml

# ─── INSPECTING DEPLOYMENTS ──────────────────────────────────────────
kubectl get deployments                    # List deployments
kubectl get deploy                         # Shorthand
kubectl describe deployment web-app        # Full details
kubectl get deployment web-app -o yaml     # Full YAML
kubectl get deploy web-app -o wide         # With extra info

# ─── SCALING ─────────────────────────────────────────────────────────
kubectl scale deployment web-app --replicas=5
kubectl scale deployment web-app --replicas=0   # Stop all pods

# ─── UPDATING ────────────────────────────────────────────────────────
# Change the image (triggers rolling update):
kubectl set image deployment/web-app nginx=nginx:1.22
kubectl set image deployment/web-app nginx=nginx:1.22 --record  # (deprecated but used)

# Edit deployment directly:
kubectl edit deployment web-app   # Opens in $EDITOR

# Apply updated YAML:
kubectl apply -f deployment.yaml

# ─── ROLLOUT MANAGEMENT ──────────────────────────────────────────────
kubectl rollout status deployment/web-app     # Watch rollout progress
kubectl rollout history deployment/web-app    # View revision history
kubectl rollout history deployment/web-app --revision=2  # Details of revision 2
kubectl rollout undo deployment/web-app       # Rollback to previous version
kubectl rollout undo deployment/web-app --to-revision=1  # Rollback to revision 1
kubectl rollout pause deployment/web-app      # Pause rolling update
kubectl rollout resume deployment/web-app     # Resume rolling update
kubectl rollout restart deployment/web-app    # Restart all pods (rolling)

# ─── AUTOSCALING (preview) ───────────────────────────────────────────
kubectl autoscale deployment web-app --min=2 --max=10 --cpu-percent=70
# Creates a HorizontalPodAutoscaler (HPA)
kubectl get hpa
```

---

## 2.6 Putting It All Together — Real-World Example

Let's build a complete, production-style application stack using everything from this chapter:

```yaml
# complete-app-stack.yaml
# A real microservice: frontend web app

# --- Namespace -------------------------------------------------------
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    env: production
    team: platform

---
# --- Deployment -------------------------------------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: my-app
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/part-of: my-app
  annotations:
    kubernetes.io/change-cause: "v1.0.0 initial release"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: frontend
        version: v1
        env: production
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
      - name: nginx
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 256Mi
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
  revisionHistoryLimit: 5
```

---

## Chapter 2: Hands-On Labs

### Lab 2.1 — Pod Deep Dive

```bash
# === EXERCISE 1: Create your first Pod from YAML ===

cat <<EOF > my-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
    env: lab
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: 50m
        memory: 32Mi
      limits:
        cpu: 200m
        memory: 64Mi
EOF

kubectl apply -f my-pod.yaml
kubectl get pod web-pod -w    # Watch it go from Pending → Running

# === EXERCISE 2: Inspect the Pod ===
kubectl describe pod web-pod
kubectl get pod web-pod -o yaml
kubectl get pod web-pod -o jsonpath='{.status.podIP}'

# === EXERCISE 3: Access the Pod ===
kubectl exec -it web-pod -- sh
  # Inside the pod:
  hostname           # Should show pod name
  env                # Environment variables
  cat /etc/hosts     # Pod hostname and IP
  exit

# === EXERCISE 4: Port-forward and test ===
kubectl port-forward pod/web-pod 8080:80 &
curl http://localhost:8080   # Should see nginx welcome page
kill %1                      # Stop port-forward

# === EXERCISE 5: Multi-container pod ===
cat <<EOF > multi-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
  - name: main
    image: nginx:alpine
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: content-generator
    image: busybox
    command: ['/bin/sh', '-c']
    args:
    - while true; do
        echo "<h1>Generated at $(date)</h1>" > /data/index.html;
        sleep 5;
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

kubectl apply -f multi-pod.yaml
kubectl get pod multi-pod
kubectl exec -it multi-pod -c content-generator -- sh
  cat /data/index.html
  exit
kubectl port-forward pod/multi-pod 8080:80 &
curl http://localhost:8080   # See dynamically generated content!
kill %1
```

### Lab 2.2 — Namespaces and Labels

```bash
# === Create namespaces ===
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod

# === Deploy to different namespaces ===
kubectl create deployment web --image=nginx --namespace=dev
kubectl create deployment web --image=nginx --namespace=staging
kubectl create deployment web --image=nginx:1.22 --namespace=prod
# Same name, different namespaces, no conflict!

# === List with namespace filters ===
kubectl get pods -n dev
kubectl get pods -n staging
kubectl get pods --all-namespaces

# === Labels ===
kubectl run app-v1 --image=nginx --labels="app=myapp,version=v1,env=prod"
kubectl run app-v2 --image=nginx:1.22 --labels="app=myapp,version=v2,env=prod"
kubectl run app-dev --image=nginx --labels="app=myapp,version=v1,env=dev"

kubectl get pods --show-labels
kubectl get pods -l app=myapp             # All app pods
kubectl get pods -l "version=v1"          # Only v1
kubectl get pods -l "env=prod,version=v1" # Prod v1 only
kubectl get pods -l "env in (prod)"       # Set-based
kubectl get pods -l "version!=v2"         # Not v2
```

### Lab 2.3 — Deployment Mastery

```bash
# === Create Deployment ===
kubectl create deployment webapp --image=nginx:1.20 --replicas=3
kubectl get deployment webapp
kubectl get replicaset           # Note the RS created
kubectl get pods -l app=webapp   # Pods created by deployment

# === Rolling Update ===
kubectl set image deployment/webapp nginx=nginx:1.21
kubectl rollout status deployment/webapp    # Watch update progress

# === See the history ===
kubectl rollout history deployment/webapp
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# === Update with change cause ===
kubectl annotate deployment webapp \
  kubernetes.io/change-cause="Upgrade to nginx 1.21 for security"
kubectl rollout history deployment/webapp

# === Rollback ===
kubectl set image deployment/webapp nginx=nginx:DOESNOTEXIST
kubectl rollout status deployment/webapp    # Watch it fail!
kubectl rollout undo deployment/webapp      # Rollback!
kubectl rollout status deployment/webapp    # Back to healthy

# === Scale ===
kubectl scale deployment webapp --replicas=5
kubectl get pods -l app=webapp   # 5 pods now!
kubectl scale deployment webapp --replicas=2
kubectl get pods -l app=webapp   # 2 pods now

# === Pause & Resume (batch update) ===
kubectl rollout pause deployment/webapp
kubectl set image deployment/webapp nginx=nginx:1.22
kubectl set resources deployment/webapp -c nginx \
  --limits=cpu=500m,memory=256Mi
# Multiple changes staged, not applied yet
kubectl rollout resume deployment/webapp    # Now apply all changes at once
kubectl rollout status deployment/webapp
```

---

## Chapter 2: Troubleshooting Guide

### Issue 1: Pod stuck in `Pending` state

```bash
kubectl describe pod <pod-name>
# Look at EVENTS section at the bottom

# Common causes:
# 1. Insufficient resources
# Events: 0/3 nodes are available: 3 Insufficient cpu
kubectl top nodes                    # Check node resources
kubectl describe nodes | grep -A5 "Allocated resources"

# Fix: Reduce pod resource requests or add more nodes
# Or: Scale down other deployments

# 2. Node Selector / Affinity not matching
# Events: 0/3 nodes are available: 3 node(s) didn't match Pod's node affinity
kubectl get nodes --show-labels      # See node labels
# Fix: Add correct label to node or fix the pod selector

# 3. PersistentVolumeClaim not bound
# Events: waiting for first consumer to be created before binding
kubectl get pvc                      # Check PVC status
```

### Issue 2: Pod in `CrashLoopBackOff`

```bash
kubectl describe pod <pod-name>      # Check Events and last state
kubectl logs <pod-name>              # Current logs
kubectl logs <pod-name> --previous   # Logs from BEFORE crash (crucial!)

# Common causes:
# 1. Application error at startup
# Fix: Check logs above for error message

# 2. Missing environment variables / secrets
# Fix: Check that required ConfigMaps/Secrets exist

# 3. Wrong image or entrypoint
kubectl get pod <pod-name> -o yaml | grep -A5 "image:"
# Fix: Verify image name and tag, test locally with docker run

# 4. Liveness probe too aggressive
# Fix: Increase initialDelaySeconds in livenessProbe
```

### Issue 3: Pod in `ImagePullBackOff` or `ErrImagePull`

```bash
kubectl describe pod <pod-name>
# Events: Failed to pull image "my-app:v99": not found

# Causes:
# 1. Wrong image name or tag
# 2. Private registry without credentials
# 3. Registry unreachable

# Fix 1: Verify image exists
docker pull nginx:99     # Test locally (will fail = bad tag)

# Fix 2: Create registry secret and reference in pod
kubectl create secret docker-registry regcred \
  --docker-server=registry.company.com \
  --docker-username=myuser \
  --docker-password=mypassword

# Reference in pod spec:
# imagePullSecrets:
# - name: regcred
```

### Issue 4: Deployment rollout stuck

```bash
kubectl rollout status deployment/web-app
# Waiting for deployment "web-app" rollout to finish:
#   1 out of 3 new replicas have been updated...

kubectl describe deployment web-app  # Check events
kubectl get pods -l app=web-app      # Which pods have issues?

# Common causes:
# 1. New pods failing readiness probe
#    → Check new pod logs, fix readiness probe path/timing

# 2. Insufficient resources for new pods
#    → Scale up cluster or reduce requests

# 3. progressDeadlineSeconds exceeded
# Fix: Pause and rollback
kubectl rollout undo deployment/web-app
```

### Issue 5: `Error from server (NotFound)` — wrong namespace

```bash
# This is the MOST COMMON mistake
kubectl get pod my-pod
# Error from server (NotFound): pods "my-pod" not found

# Check: What namespace is the pod actually in?
kubectl get pods --all-namespaces | grep my-pod

# Fix: Use the correct namespace
kubectl get pod my-pod -n production
# OR set your default namespace:
kubectl config set-context --current --namespace=production
```

---

## Chapter 2: Interview Questions

**Q1: What is a Pod and why is it the smallest deployable unit?**

> *Answer:* A Pod is a wrapper around one or more containers that share the same network namespace (IP address), IPC namespace, and optionally storage volumes. It's the smallest deployable unit because Kubernetes doesn't manage individual containers — it manages Pods. The Pod abstraction is necessary because some workloads genuinely need tightly-coupled containers (like an app and its log shipper) that must share network and storage.

**Q2: What is the difference between a ReplicaSet and a Deployment?**

> *Answer:* A ReplicaSet ensures a specified number of pod replicas are always running. A Deployment is a higher-level abstraction that manages ReplicaSets and adds rolling update and rollback capabilities. When you update a Deployment (e.g., change the image), it creates a new ReplicaSet and gradually scales it up while scaling down the old one. Deployments keep old ReplicaSets (at 0 replicas) for easy rollback. In practice, you almost never create ReplicaSets directly — you use Deployments.

**Q3: Explain the difference between `kubectl apply` and `kubectl create`.**

> *Answer:* `kubectl create` is imperative — it creates the resource and fails if it already exists. `kubectl apply` is declarative — it creates the resource if it doesn't exist, or updates it if it does. `apply` compares the submitted manifest with the last-applied configuration stored as an annotation, allowing incremental updates. In production, always use `apply` for idempotent, version-controlled deployments.

**Q4: How do Labels and Selectors work? Why are they important?**

> *Answer:* Labels are key-value pairs attached to objects. Selectors query for objects matching specific labels. They are critical because they are the mechanism by which Kubernetes objects relate to each other — a Service selects pods to route traffic to using labels, a ReplicaSet selects pods it owns using labels, a Deployment finds its ReplicaSets using labels. Without labels and selectors, Kubernetes has no way to know which pods belong to which service.

**Q5: What are Namespaces used for?**

> *Answer:* Namespaces provide logical isolation within a cluster. They're used for: (1) multi-team environments — each team gets their own namespace; (2) multi-environment setups — dev/staging/prod in one cluster; (3) resource quotas — limit CPU/memory per namespace; (4) access control — RBAC policies scoped to namespaces; (5) network isolation — with NetworkPolicies. Note: Namespaces don't provide security isolation by default — two pods in different namespaces can still communicate.

**Q6: What is a RollingUpdate deployment strategy and what do `maxSurge` and `maxUnavailable` mean?**

> *Answer:* RollingUpdate gradually replaces old pods with new ones. `maxSurge` is the maximum number of extra pods allowed above the desired replica count during the update (allows old and new pods to coexist temporarily). `maxUnavailable` is the maximum number of pods that can be unavailable (not ready) during the update. Setting `maxUnavailable: 0` and `maxSurge: 1` gives zero-downtime deployments — a new pod must be ready before an old one is removed.

**Q7: How do you roll back a Deployment?**

> *Answer:* `kubectl rollout undo deployment/<name>` rolls back to the previous revision. You can roll back to a specific version with `kubectl rollout undo deployment/<name> --to-revision=<number>`. The revision history is stored as old ReplicaSets (at 0 replicas), controlled by `revisionHistoryLimit`. Always annotate deployments with `kubernetes.io/change-cause` so revision history is human-readable.

**Q8: What happens to pods when you delete a Deployment vs. a ReplicaSet?**

> *Answer:* Deleting a Deployment deletes the Deployment, all managed ReplicaSets, and all their Pods (cascade delete by default). Deleting a ReplicaSet with `--cascade=foreground` (default) deletes the RS and its pods. But you can delete a ReplicaSet with `--cascade=orphan` which deletes the RS object but leaves the pods running (they become unmanaged). Deleting a Deployment does NOT have an orphan option — it always cascades.

---

## 📌 CKA Exam Notes — Chapter 2

```
╔════════════════════════════════════════════════════════════════════╗
║                CKA EXAM FOCUS — CHAPTER 2                         ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: Core objects are in EVERY CKA domain.               ║
║  Deployments alone could be 15-20% of your exam.                  ║
║                                                                    ║
║  SPEED TIPS — Set these aliases at exam start:                    ║
║  alias k=kubectl                                                   ║
║  alias kgp='kubectl get pods'                                      ║
║  alias kgs='kubectl get svc'                                       ║
║  alias kgd='kubectl get deploy'                                    ║
║  export do='--dry-run=client -o yaml'                             ║
║  export now='--force --grace-period=0'                            ║
║                                                                    ║
║  FASTEST WAY TO GENERATE YAML (use this in exam!):               ║
║  k create deploy myapp --image=nginx $do > deploy.yaml            ║
║  k run mypod --image=nginx $do > pod.yaml                         ║
║                                                                    ║
║  MUST-KNOW COMMANDS:                                               ║
║  ✅ kubectl run <name> --image=<img> --dry-run=client -o yaml     ║
║  ✅ kubectl create deployment <n> --image=<img> --replicas=<n>    ║
║  ✅ kubectl scale deployment <name> --replicas=<n>                ║
║  ✅ kubectl set image deployment/<n> <container>=<image>          ║
║  ✅ kubectl rollout undo deployment/<name>                         ║
║  ✅ kubectl rollout status deployment/<name>                       ║
║  ✅ kubectl rollout history deployment/<name>                      ║
║  ✅ kubectl get pods --show-labels                                 ║
║  ✅ kubectl get pods -l "key=value"                                ║
║  ✅ kubectl config set-context --current --namespace=<ns>         ║
║                                                                    ║
║  EXAM TRAPS:                                                       ║
║  ⚠️  Always check kubectl config current-context FIRST            ║
║  ⚠️  Labels in selector MUST match labels in pod template         ║
║  ⚠️  If asked for a ReplicaSet, give a Deployment unless told     ║
║      explicitly to use RS directly                                 ║
║  ⚠️  kubectl rollout undo goes ONE step back only                 ║
║      Use --to-revision=N for specific version                     ║
║  ⚠️  Namespace flag -n is NOT sticky unless you set context       ║
║                                                                    ║
║  EXAM SCENARIO EXAMPLES:                                           ║
║  "Create a deployment named web with image nginx:1.21,            ║
║   3 replicas, in namespace production"                             ║
║  → k create deploy web --image=nginx:1.21 --replicas=3            ║
║       -n production                                                ║
║                                                                    ║
║  "Scale the deployment frontend to 5 replicas"                    ║
║  → k scale deploy frontend --replicas=5                           ║
║                                                                    ║
║  "Update the image to nginx:1.22 and ensure rollout succeeds"     ║
║  → k set image deploy/frontend nginx=nginx:1.22                  ║
║  → k rollout status deploy/frontend                               ║
║                                                                    ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Common Mistakes — Chapter 2

| Mistake | Why It Happens | How to Avoid |
|---|---|---|
| Pod labels don't match ReplicaSet selector | Typo or copy-paste | Always validate: selector labels ⊆ template labels |
| Using `kubectl create` then failing on re-apply | Habit from scripting | Always use `kubectl apply` for deployments |
| Forgetting `-n <namespace>` | Working in wrong context | Set default namespace with `config set-context` |
| No resource requests/limits in production | Seems fine locally | Always set both; missing = cluster instability |
| No readiness probe | App works fine in testing | Always add readiness probe to every container |
| Rolling update with `maxUnavailable: 100%` | Default config | Set `maxUnavailable: 0` for zero-downtime apps |
| Deleting a Deployment intending to just restart pods | Misunderstanding | Use `kubectl rollout restart deployment/<name>` |
| Running as root in containers | Default container behavior | Set `runAsNonRoot: true` in securityContext |

---

## Chapter 2 Summary

You now know:

1. **Pods** — the atomic unit, wrapping 1+ containers sharing network/storage
2. **Pod lifecycle** — Pending → Running → Succeeded/Failed/Unknown
3. **Pod probes** — Liveness (restart), Readiness (traffic), Startup (slow apps)
4. **Namespaces** — logical cluster partitions for isolation and organization
5. **Labels** — key-value metadata for selecting and grouping objects
6. **Selectors** — matchLabels and matchExpressions to query by labels
7. **Annotations** — rich non-identifying metadata for tools and humans
8. **ReplicaSets** — ensures N pod replicas always run
9. **Deployments** — manages ReplicaSets, adds rolling updates and rollback
10. **Deployment strategies** — RollingUpdate vs Recreate, maxSurge vs maxUnavailable

---

*Next: Chapter 3 — Kubernetes Networking: Services, ClusterIP, NodePort, LoadBalancer, Ingress, DNS, and CNI*

*"Your pods are running. Now let's make them talk to each other — and to the world."*
