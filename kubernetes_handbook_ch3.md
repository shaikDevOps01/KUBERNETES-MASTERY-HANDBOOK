# KUBERNETES MASTERY HANDBOOK
# Part 3: Networking
# Chapter 3: Services, Ingress, DNS, and CNI

---

> **"In Kubernetes, nothing is permanent — pods die and are reborn with new IPs.
>  Services are the promise that someone will always answer the phone."**

---

## Chapter Introduction

Your pods from Chapter 2 are running. But there's a fundamental problem: **pods are ephemeral and unstable by design**. Every time a pod restarts, it gets a new IP address. You can't hard-code `10.244.1.5` into your app configuration — that IP will be gone after the next crash.

Kubernetes Networking solves three problems:

```
╔══════════════════════════════════════════════════════════════════════╗
║              THE THREE NETWORKING PROBLEMS KUBERNETES SOLVES        ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PROBLEM 1: Pod-to-Pod Communication                                ║
║  "How does my frontend pod talk to my backend pod?"                 ║
║  Solution: Flat network — every pod gets a unique cluster IP.       ║
║  Every pod can reach every other pod directly (no NAT).             ║
║                                                                      ║
║  PROBLEM 2: Stable Endpoint for a Group of Pods                     ║
║  "Pod IPs change on restart. How do I find my backend reliably?"    ║
║  Solution: Services — a stable virtual IP that load-balances        ║
║  across all healthy pods matching a label selector.                 ║
║                                                                      ║
║  PROBLEM 3: External Access                                         ║
║  "How do users on the internet reach my app inside the cluster?"    ║
║  Solution: NodePort, LoadBalancer, and Ingress controllers.         ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

This chapter covers the full networking stack — from pods talking to each other, all the way to an HTTPS request from a user's browser reaching the right microservice inside your cluster.

---

## 3.1 The Kubernetes Network Model

### 3.1.1 The Fundamental Rules

Before any specific concept, internalize these four rules that every Kubernetes network must follow:

```
╔══════════════════════════════════════════════════════════════════════╗
║           KUBERNETES NETWORKING RULES (The Contract)                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  RULE 1: Every Pod gets its own unique IP address.                  ║
║          No two pods share an IP, even on different nodes.          ║
║                                                                      ║
║  RULE 2: Pods on the same node can communicate with each other      ║
║          directly using those IPs — no NAT required.                ║
║                                                                      ║
║  RULE 3: Pods on DIFFERENT nodes can communicate directly using     ║
║          those IPs — no NAT required.                               ║
║          (This is the hard part — CNI plugins solve this)           ║
║                                                                      ║
║  RULE 4: The IP a pod sees for itself is the same IP others use     ║
║          to reach it. No address translation games.                 ║
║                                                                      ║
║  These rules create a FLAT network — as if all pods are on         ║
║  the same giant LAN switch, regardless of physical node.            ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 3.1.2 The IP Address Spaces in a Kubernetes Cluster

A Kubernetes cluster has THREE distinct IP ranges. Confusing them is one of the most common networking mistakes:

```
╔══════════════════════════════════════════════════════════════════════╗
║                   THREE IP RANGES IN KUBERNETES                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. NODE NETWORK (Physical/VM Network)                              ║
║     ─────────────────────────────────                               ║
║     Range:   e.g., 192.168.1.0/24                                   ║
║     Who:     The actual machines (control plane + workers)          ║
║     Example: worker-1 = 192.168.1.10                                ║
║                                                                      ║
║  2. POD NETWORK (Virtual Overlay Network)                           ║
║     ──────────────────────────────────────                          ║
║     Range:   e.g., 10.244.0.0/16    (set by --pod-network-cidr)    ║
║     Who:     Every pod gets one IP from this range                  ║
║     Example: pod-A = 10.244.1.5, pod-B = 10.244.2.3               ║
║     Managed by: CNI plugin (Flannel, Calico, Weave, Cilium)        ║
║                                                                      ║
║  3. SERVICE NETWORK (Virtual, kube-proxy managed)                   ║
║     ─────────────────────────────────────────────                   ║
║     Range:   e.g., 10.96.0.0/12     (set by --service-cluster-ip-range)
║     Who:     Every Kubernetes Service gets one IP from this range   ║
║     Example: my-service = 10.96.45.123                              ║
║     NOTE:    These IPs don't exist on any real interface!           ║
║              They're virtual — implemented via iptables/IPVS rules  ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │ CLUSTER VIEW                                                 │   ║
║  │                                                              │   ║
║  │  Node: 192.168.1.10          Node: 192.168.1.11             │   ║
║  │  ┌─────────────────────┐     ┌─────────────────────┐        │   ║
║  │  │ Pod: 10.244.1.5     │     │ Pod: 10.244.2.3     │        │   ║
║  │  │ Pod: 10.244.1.6     │     │ Pod: 10.244.2.4     │        │   ║
║  │  └─────────────────────┘     └─────────────────────┘        │   ║
║  │                                                              │   ║
║  │  Service "web-svc" → 10.96.45.123 (virtual, cluster-wide)  │   ║
║  │  Routes to:  10.244.1.5, 10.244.1.6, 10.244.2.3            │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 3.2 CNI — Container Network Interface

### 3.2.1 What Is CNI?

#### In Plain English

CNI is the **wiring standard** for Kubernetes networking. Kubernetes itself doesn't know *how* to set up networking between nodes — it just knows *what* it needs (pods to communicate across nodes). CNI plugins are the actual electricians that wire everything up.

#### In Technical Language

The **Container Network Interface (CNI)** is a specification and set of libraries for configuring network interfaces in Linux containers. Kubernetes uses CNI plugins to:

- Assign IP addresses to pods
- Set up network routes so pods on different nodes can communicate
- Enforce Network Policies (in supported plugins)
- Clean up network resources when pods are deleted

```
╔══════════════════════════════════════════════════════════════════════╗
║                  CNI PLUGIN COMPARISON                               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PLUGIN       OVERLAY      NETWORK POLICY   PERFORMANCE  USE CASE  ║
║  ──────────   ─────────    ──────────────   ───────────  ────────  ║
║  Flannel      VXLAN/UDP    ❌ No            Good         Simplest  ║
║  Calico       BGP/IPIP     ✅ Yes           Excellent    Production ║
║  Cilium       eBPF         ✅ Yes           Best         Advanced  ║
║  Weave Net    VXLAN        ✅ Yes           Good         Mid-size  ║
║  Canal        Flannel+     ✅ Yes           Good         Balanced  ║
║               Calico NP                                             ║
║                                                                      ║
║  HOW FLANNEL WORKS (simplest example):                              ║
║                                                                      ║
║  Node-1 (192.168.1.10)        Node-2 (192.168.1.11)                ║
║  Pod CIDR: 10.244.1.0/24      Pod CIDR: 10.244.2.0/24              ║
║  ┌─────────────────────┐      ┌─────────────────────┐              ║
║  │ Pod-A: 10.244.1.5   │      │ Pod-C: 10.244.2.3   │              ║
║  │      │              │      │      ▲              │              ║
║  │   cni0 bridge       │      │   cni0 bridge       │              ║
║  │      │              │      │      │              │              ║
║  │   flannel0          │      │   flannel0          │              ║
║  │   (VXLAN tunnel)    │──────│   (VXLAN tunnel)   │              ║
║  │   eth0:192.168.1.10 │      │   eth0:192.168.1.11 │              ║
║  └─────────────────────┘      └─────────────────────┘              ║
║                                                                      ║
║  Packet from Pod-A → Pod-C:                                         ║
║  1. Src: 10.244.1.5 → Dst: 10.244.2.3                              ║
║  2. Flannel wraps in UDP packet (VXLAN encapsulation)               ║
║  3. Outer packet: Src: 192.168.1.10 → Dst: 192.168.1.11            ║
║  4. Node-2 receives, unwraps, delivers to Pod-C                     ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 3.2.2 CNI Configuration

```bash
# CNI plugins are installed as files in:
ls /etc/cni/net.d/          # CNI config files
ls /opt/cni/bin/            # CNI plugin binaries

# Check which CNI is running
kubectl get pods -n kube-system | grep -E "flannel|calico|cilium|weave"

# Flannel installation (example)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Calico installation (example)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

## 3.3 Services — The Heart of Kubernetes Networking

### 3.3.1 What Is a Service?

#### In Plain English

A Service is like a **restaurant's phone number**. The restaurant (your app) might have multiple cooks (pods) on different shifts. The phone number (Service) always stays the same. When you call, it connects you to whoever is available. Even if a cook quits and is replaced, the phone number doesn't change.

#### In Technical Language

A **Service** is a Kubernetes object that provides a stable network endpoint (IP + DNS name) for a set of pods. It:

- Maintains a stable **ClusterIP** (virtual IP) even as pods come and go
- Uses **label selectors** to find backing pods dynamically
- Distributes traffic using **kube-proxy** (iptables/IPVS rules on each node)
- Registers in **CoreDNS** so pods can reach it by name

```
╔══════════════════════════════════════════════════════════════════════╗
║                 HOW A SERVICE WORKS                                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Service: web-svc                                                    ║
║  ClusterIP: 10.96.45.123                                             ║
║  Selector: app=web                                                   ║
║  Port: 80 → targetPort: 8080                                        ║
║                                                                      ║
║  ENDPOINTS (auto-managed, updated dynamically):                     ║
║  10.96.45.123:80  →  [10.244.1.5:8080,  ← Pod-A (healthy)          ║
║                        10.244.1.6:8080,  ← Pod-B (healthy)          ║
║                        10.244.2.3:8080]  ← Pod-C (healthy)          ║
║                                                                      ║
║  When Pod-B becomes unhealthy (fails readiness probe):              ║
║  ENDPOINTS: [10.244.1.5:8080, 10.244.2.3:8080]  ← Pod-B removed    ║
║                                                                      ║
║  Endpoint Controller watches pods, updates Endpoints object         ║
║  kube-proxy watches Endpoints, updates iptables rules               ║
║                                                                      ║
║  CLIENT REQUEST FLOW:                                                ║
║                                                                      ║
║  Client Pod → 10.96.45.123:80                                       ║
║                    │                                                 ║
║                    │ iptables rule on node intercepts virtual IP     ║
║                    │                                                 ║
║                    ├──── 33% → 10.244.1.5:8080  (Pod-A)            ║
║                    ├──── 33% → 10.244.1.6:8080  (Pod-B)            ║
║                    └──── 33% → 10.244.2.3:8080  (Pod-C)            ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 3.3.2 Service Types — The Four Types Explained

Kubernetes has four Service types, each exposing your app differently:

```
╔══════════════════════════════════════════════════════════════════════╗
║              SERVICE TYPES — EXPOSURE LEVELS                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ClusterIP (default)                                                 ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │ INTERNET                                                       │  ║
║  │    ╳  NOT reachable                                            │  ║
║  │ CLUSTER ──────────────────────────────────────────────────     │  ║
║  │    ✅  Pod-to-Pod communication only                           │  ║
║  │    ✅  Internal microservices                                  │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
║                                                                      ║
║  NodePort                                                            ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │ INTERNET                                                       │  ║
║  │    ✅  Reachable via <NodeIP>:<NodePort>                       │  ║
║  │       e.g., http://192.168.1.10:30080                          │  ║
║  │    NodePort range: 30000-32767                                 │  ║
║  │    Use for: Dev/test, on-premise without LoadBalancer          │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
║                                                                      ║
║  LoadBalancer                                                        ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │ INTERNET                                                       │  ║
║  │    ✅  Cloud provider creates external LB automatically        │  ║
║  │       e.g., http://34.102.123.45 (GCP/AWS public IP)          │  ║
║  │    Extends NodePort — also opens NodePort on every node        │  ║
║  │    Use for: Production on cloud (GKE, EKS, AKS)               │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
║                                                                      ║
║  ExternalName                                                        ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │ No selector, no endpoints                                      │  ║
║  │ Maps a Service to an external DNS name                         │  ║
║  │ e.g., db.company.com → maps to prod-db.rds.amazonaws.com       │  ║
║  │ Use for: Integrating external services into cluster DNS        │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 3.4 ClusterIP — Internal Communication

### 3.4.1 ClusterIP Deep Dive

ClusterIP is the default and most common Service type. It creates a stable internal IP accessible **only within the cluster**.

```
╔══════════════════════════════════════════════════════════════════════╗
║              CLUSTERIP — INTERNAL SERVICE                            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  EXTERNAL USER  ────╳────  (Cannot reach ClusterIP)                 ║
║                                                                      ║
║  INSIDE CLUSTER:                                                     ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │                                                              │   ║
║  │  frontend-pod ──→ http://backend-svc:8080                    │   ║
║  │                              │                               │   ║
║  │                   CoreDNS resolves to                        │   ║
║  │                   10.96.45.123 (ClusterIP)                   │   ║
║  │                              │                               │   ║
║  │                   kube-proxy routes to:                      │   ║
║  │                     ├── backend-pod-1:8080                   │   ║
║  │                     ├── backend-pod-2:8080                   │   ║
║  │                     └── backend-pod-3:8080                   │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  DNS names for ClusterIP (auto-registered in CoreDNS):             ║
║  • backend-svc                         (same namespace)            ║
║  • backend-svc.production              (with namespace)            ║
║  • backend-svc.production.svc          (with svc suffix)           ║
║  • backend-svc.production.svc.cluster.local  (fully qualified)     ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 3.4.2 ClusterIP YAML

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: production
  labels:
    app: backend
spec:
  type: ClusterIP           # Default type — can be omitted
  selector:
    app: backend            # Selects pods with this label
    env: production
  ports:
  - name: http              # Name is important for Ingress references
    protocol: TCP           # TCP (default) | UDP | SCTP
    port: 8080              # Port the Service listens on (cluster-wide)
    targetPort: 8080        # Port on the pod containers
    # targetPort can also use port NAME from container spec:
    # targetPort: app-port  (matches containerPort name: app-port)
  
  # Optional: Session affinity — route same client to same pod
  sessionAffinity: None     # None (default) | ClientIP

---
# Headless Service — no ClusterIP (for StatefulSets/direct pod DNS)
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  clusterIP: None           # "None" = headless
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
  # DNS returns ALL pod IPs instead of single ClusterIP
  # Used by StatefulSets for stable pod identity
```

---

## 3.5 NodePort — External Access (Basic)

### 3.5.1 NodePort Deep Dive

NodePort extends ClusterIP by also opening a port (30000–32767) on **every node** in the cluster. External traffic hitting any node's IP on that port gets routed to the backend pods.

```
╔══════════════════════════════════════════════════════════════════════╗
║                   NODEPORT — TRAFFIC FLOW                            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  EXTERNAL CLIENT                                                     ║
║       │                                                              ║
║       │  http://192.168.1.10:30080  (any node IP + NodePort)        ║
║       │  http://192.168.1.11:30080  (same port on every node)       ║
║       │                                                              ║
║       ▼                                                              ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  Node-1 (192.168.1.10)       Node-2 (192.168.1.11)          │    ║
║  │  Port 30080 OPEN             Port 30080 OPEN                 │    ║
║  │  ┌─────────────────┐         ┌─────────────────┐            │    ║
║  │  │  kube-proxy     │         │  kube-proxy     │            │    ║
║  │  │  iptables rule: │         │  iptables rule: │            │    ║
║  │  │  30080 → pods   │         │  30080 → pods   │            │    ║
║  │  └────────┬────────┘         └────────┬────────┘            │    ║
║  │           │                           │                      │    ║
║  │      ┌────┴────┐                 ┌────┴────┐                 │    ║
║  │      │ Pod-A   │                 │ Pod-B   │                 │    ║
║  │      │:8080    │                 │:8080    │                 │    ║
║  │      └─────────┘                 └─────────┘                 │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  NOTE: Traffic hitting Node-1 can be routed to Pod-B on Node-2!     ║
║  kube-proxy handles cross-node routing transparently.               ║
║                                                                      ║
║  LIMITATIONS:                                                        ║
║  • Exposes a port on EVERY node (security concern)                  ║
║  • Client must know a valid node IP (no single entry point)         ║
║  • NodePort range limited to 30000-32767                            ║
║  • Not suitable for production internet traffic                     ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 3.5.2 NodePort YAML

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport-svc
  namespace: default
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80           # ClusterIP port (internal access)
    targetPort: 80     # Pod port
    nodePort: 30080    # External port (30000-32767)
                       # If omitted, Kubernetes assigns randomly
```

### 3.5.3 NodePort Commands

```bash
# Create a NodePort service imperatively
kubectl expose deployment web-app --type=NodePort --port=80 --target-port=80

# Get the NodePort assigned
kubectl get svc web-app-svc
# NAME          TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
# web-app-svc   NodePort   10.96.45.123   <none>       80:30080/TCP  5m

# Access the service
# minikube:
minikube service web-app-svc --url
# On real cluster:
curl http://<any-node-ip>:30080
```

---

## 3.6 LoadBalancer — Production External Access

### 3.6.1 LoadBalancer Deep Dive

LoadBalancer is NodePort with an automatic cloud load balancer in front. When you create a LoadBalancer Service on GKE, EKS, or AKS, the cloud provider automatically provisions an external load balancer and assigns a public IP.

```
╔══════════════════════════════════════════════════════════════════════╗
║               LOADBALANCER SERVICE — FULL FLOW                       ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  USER (Internet)                                                     ║
║       │                                                              ║
║       │  http://34.102.123.45  (Public IP from cloud provider)       ║
║       │                                                              ║
║       ▼                                                              ║
║  ┌─────────────────────────────────────────┐                         ║
║  │  CLOUD LOAD BALANCER                    │                         ║
║  │  (AWS ELB / GCP GLB / Azure LB)         │                         ║
║  │  Public IP: 34.102.123.45               │                         ║
║  │  Automatically provisioned by K8s       │                         ║
║  └──────────────────┬──────────────────────┘                         ║
║                     │  Forwards to NodePort (e.g., :31000)           ║
║            ┌────────┴─────────┐                                      ║
║            ▼                  ▼                                      ║
║    Node-1:31000          Node-2:31000                                ║
║    ┌──────────┐           ┌──────────┐                               ║
║    │ kube-    │           │ kube-    │                               ║
║    │ proxy    │           │ proxy    │                               ║
║    └────┬─────┘           └────┬─────┘                               ║
║         │                      │                                     ║
║    ┌────┴────────────────────┐  │                                     ║
║    │ Pods on Node-1 or Node-2│◄─┘                                    ║
║    └─────────────────────────┘                                       ║
║                                                                      ║
║  HIERARCHY:                                                          ║
║  LoadBalancer builds ON TOP OF NodePort                              ║
║  NodePort builds ON TOP OF ClusterIP                                 ║
║                                                                      ║
║  LoadBalancer = ClusterIP + NodePort + Cloud LB                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 3.6.2 LoadBalancer YAML

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb-svc
  namespace: production
  annotations:
    # Cloud-specific annotations (AWS example):
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    # GCP example:
    # cloud.google.com/load-balancer-type: "External"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
  # Optional: Restrict which IPs can access this LoadBalancer
  loadBalancerSourceRanges:
  - "203.0.113.0/24"         # Only allow this IP range
  externalTrafficPolicy: Local  # Local | Cluster
  # Local: Preserves client IP, only routes to local node pods (avoid extra hop)
  # Cluster: Default, routes to any pod, client IP is NATed
```

```bash
# Watch LoadBalancer get its external IP (takes 1-2 minutes on cloud)
kubectl get svc web-lb-svc -w
# NAME        TYPE          CLUSTER-IP    EXTERNAL-IP    PORT(S)       AGE
# web-lb-svc  LoadBalancer  10.96.1.100   <pending>      80:31234/TCP  10s
# web-lb-svc  LoadBalancer  10.96.1.100   34.102.123.45  80:31234/TCP  90s

# Access the service
curl http://34.102.123.45
```

> **💡 On-Premise Alternative:** Without a cloud provider, use **MetalLB** — it implements LoadBalancer Services for bare-metal clusters by assigning IPs from a configured pool.

---

## 3.7 Ingress — Smart HTTP/HTTPS Routing

### 3.7.1 The Problem with LoadBalancer Services

In a real application, you might have 10 different microservices all needing external access. With LoadBalancer Services, you'd need 10 separate cloud load balancers = **10 separate public IPs = high cost**.

Ingress solves this by acting as a **single entry point** that routes HTTP/HTTPS traffic to different services based on the URL path or hostname.

### 3.7.2 What Is Ingress?

#### In Plain English

Ingress is like a **smart hotel concierge**. One person at the front desk — one door into the hotel (one IP). You tell the concierge "I need Room 301 (frontend service)" or "I need the restaurant (API service)." The concierge directs you to the right place based on what you asked for.

#### In Technical Language

**Ingress** is a Kubernetes object that manages external HTTP/HTTPS access to Services. It requires an **Ingress Controller** (like NGINX, Traefik, or HAProxy) that reads Ingress rules and configures itself accordingly.

```
╔══════════════════════════════════════════════════════════════════════╗
║                   INGRESS ARCHITECTURE                               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  INTERNET                                                            ║
║     │                                                                ║
║     │  https://myapp.com/         → frontend service                 ║
║     │  https://myapp.com/api/v1   → backend service                  ║
║     │  https://api.myapp.com/     → api-gateway service              ║
║     │                                                                ║
║     ▼                                                                ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │  INGRESS CONTROLLER (e.g., NGINX Ingress Controller)           │  ║
║  │  Single LoadBalancer Service with 1 Public IP                  │  ║
║  │  Reads all Ingress objects → configures NGINX routing rules    │  ║
║  └──────────────────────────┬─────────────────────────────────────┘  ║
║                             │                                        ║
║    INGRESS OBJECT RULES:    │                                        ║
║    ┌────────────────────────┼──────────────────────────────────┐     ║
║    │  Host: myapp.com       │  Host: api.myapp.com             │     ║
║    │  Path: /        → ─────┼──→ frontend-svc:80               │     ║
║    │  Path: /api/v1  → ─────┼──→ backend-svc:8080              │     ║
║    │                        │  Path: /  → api-gateway-svc:8000 │     ║
║    └────────────────────────┴──────────────────────────────────┘     ║
║                                                                      ║
║    ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐        ║
║    │ frontend-svc│  │ backend-svc  │  │ api-gateway-svc    │        ║
║    │ (ClusterIP) │  │ (ClusterIP)  │  │ (ClusterIP)        │        ║
║    └──────┬──────┘  └──────┬───────┘  └─────────┬──────────┘        ║
║           │                │                     │                   ║
║         Pods             Pods                  Pods                  ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 3.7.3 Installing NGINX Ingress Controller

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml

# For minikube:
minikube addons enable ingress

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
# ingress-nginx-controller   LoadBalancer  10.96.x.x  <EXTERNAL-IP>  80,443

# Check Ingress Controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller -f
```

### 3.7.4 Ingress YAML — All Patterns

```yaml
# ══════════════════════════════════════════════════════════════════
# PATTERN 1: Simple single-service ingress
# ══════════════════════════════════════════════════════════════════
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"           # Which controller to use
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix          # Prefix | Exact | ImplementationSpecific
        backend:
          service:
            name: frontend-svc
            port:
              number: 80

---
# ══════════════════════════════════════════════════════════════════
# PATTERN 2: Path-based routing (most common)
# ══════════════════════════════════════════════════════════════════
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /$2   # Strip /api prefix
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /                   # Traffic to /  → frontend
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api(/|$)(.*)       # Traffic to /api/* → backend
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 8080
      - path: /admin              # Exact match for /admin
        pathType: Exact
        backend:
          service:
            name: admin-svc
            port:
              number: 3000

---
# ══════════════════════════════════════════════════════════════════
# PATTERN 3: Host-based routing (virtual hosting)
# ══════════════════════════════════════════════════════════════════
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: www.myapp.com           # Main website
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
  - host: api.myapp.com           # API subdomain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
  - host: admin.myapp.com         # Admin panel
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-svc
            port:
              number: 3000

---
# ══════════════════════════════════════════════════════════════════
# PATTERN 4: TLS/HTTPS with cert-manager
# ══════════════════════════════════════════════════════════════════
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"   # Auto-cert via cert-manager
    nginx.ingress.kubernetes.io/ssl-redirect: "true"      # Force HTTPS
spec:
  tls:
  - hosts:
    - myapp.com
    - www.myapp.com
    secretName: myapp-tls-secret     # TLS cert stored here (auto-managed by cert-manager)
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

### 3.7.5 Ingress Commands

```bash
# List Ingress resources
kubectl get ingress                          # Current namespace
kubectl get ing                              # Shorthand
kubectl get ingress -A                       # All namespaces
kubectl describe ingress path-based-ingress  # Details including rules

# Test Ingress routing (using /etc/hosts trick for local testing)
echo "$(minikube ip) myapp.example.com" >> /etc/hosts
curl http://myapp.example.com/
curl http://myapp.example.com/api/users

# Watch Ingress for changes
kubectl get ingress -w
```

---

## 3.8 DNS in Kubernetes — CoreDNS

### 3.8.1 How Kubernetes DNS Works

#### In Plain English

CoreDNS is the **phone book** of your Kubernetes cluster. When a pod wants to reach the "database-service," it asks CoreDNS: "What's the IP for database-service?" CoreDNS looks it up and responds instantly. Without DNS, every pod would need to know the IP addresses of every other service — and those IPs change.

#### In Technical Language

**CoreDNS** is the default DNS server in Kubernetes (since 1.12, replacing kube-dns). It runs as a Deployment in `kube-system` and automatically registers DNS records for every Service and Pod.

```
╔══════════════════════════════════════════════════════════════════════╗
║                    KUBERNETES DNS RECORDS                            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  SERVICE DNS RECORDS:                                                ║
║  ─────────────────────────────────────────────────────────────────  ║
║  Format: <service>.<namespace>.svc.<cluster-domain>                 ║
║  Default cluster-domain: cluster.local                              ║
║                                                                      ║
║  Service: backend-svc in namespace: production                      ║
║  Full FQDN: backend-svc.production.svc.cluster.local               ║
║                                                                      ║
║  SHORT FORMS (based on calling pod's namespace):                    ║
║  From pod in SAME namespace (production):                           ║
║    backend-svc                              ← works                 ║
║    backend-svc.production                   ← works                 ║
║    backend-svc.production.svc               ← works                 ║
║    backend-svc.production.svc.cluster.local ← works (FQDN)         ║
║                                                                      ║
║  From pod in DIFFERENT namespace (default):                         ║
║    backend-svc                              ← FAILS (not same NS)  ║
║    backend-svc.production                   ← works                 ║
║    backend-svc.production.svc.cluster.local ← works (FQDN)         ║
║                                                                      ║
║  POD DNS RECORDS:                                                    ║
║  ─────────────────────────────────────────────────────────────────  ║
║  Format: <pod-ip-dashes>.<namespace>.pod.<cluster-domain>           ║
║  Pod IP: 10.244.1.5 → 10-244-1-5.default.pod.cluster.local        ║
║  (Usually use Service DNS instead of Pod DNS)                       ║
║                                                                      ║
║  SRV RECORDS (for headless services):                               ║
║  Format: _<port-name>._<protocol>.<svc>.<ns>.svc.cluster.local     ║
║  StatefulSet pods: pod-0.svc.namespace.svc.cluster.local            ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 3.8.2 CoreDNS Configuration

```yaml
# CoreDNS is configured via a ConfigMap
kubectl get configmap coredns -n kube-system -o yaml

# Default Corefile:
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {    # Forward unknown queries to node's DNS
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### 3.8.3 DNS Debugging Commands

```bash
# Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS resolution from inside a pod
kubectl run dns-test --image=busybox --rm -it --restart=Never -- sh
  # Inside the pod:
  nslookup kubernetes                              # Built-in service
  nslookup backend-svc.production                  # Service in another NS
  nslookup backend-svc.production.svc.cluster.local  # Full FQDN
  cat /etc/resolv.conf                             # DNS config injected into pod
  # nameserver 10.96.0.10    ← CoreDNS service IP
  # search default.svc.cluster.local svc.cluster.local cluster.local

# Check DNS from a running pod
kubectl exec -it my-pod -- nslookup my-service

# View CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns -f

# Diagnose DNS issues
kubectl run debug --image=nicolaka/netshoot --rm -it --restart=Never -- bash
  # Rich networking tools (nslookup, dig, curl, tcpdump, etc.)
  dig backend-svc.production.svc.cluster.local
  dig +short backend-svc.production
```

---

## 3.9 Service Discovery — The Endpoints Object

### 3.9.1 Understanding Endpoints

Behind every Service is an **Endpoints** object (automatically managed) that lists the actual pod IPs and ports the Service routes to:

```bash
# View Endpoints for a Service
kubectl get endpoints backend-svc
# NAME          ENDPOINTS                               AGE
# backend-svc   10.244.1.5:8080,10.244.2.3:8080        5m

kubectl describe endpoints backend-svc
# Shows which pods are included and which are excluded (not ready)

# When a pod fails readiness probe → it's removed from Endpoints
# When a pod becomes ready → it's added back to Endpoints
# The Service NEVER routes traffic to pods not in Endpoints
```

---

## 3.10 Network Policies — Firewall Rules for Pods

### 3.10.1 What Is a NetworkPolicy?

#### In Plain English

By default, all pods in a Kubernetes cluster can talk to all other pods — like an office with no doors. NetworkPolicies add doors with locks. You decide: "Only pods with the label `role=frontend` can talk to my database pods."

#### In Technical Language

A **NetworkPolicy** is a specification of how groups of pods are allowed to communicate with each other and with network endpoints. Network Policies require a CNI plugin that supports them (Calico, Cilium, Weave — NOT Flannel alone).

```yaml
# network-policy.yaml

# SCENARIO: Database pod should ONLY accept connections from backend pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-network-policy
  namespace: production
spec:
  podSelector:           # WHICH pods this policy applies to
    matchLabels:
      role: database     # This policy governs database pods

  policyTypes:           # What types of traffic to control
  - Ingress              # Incoming traffic to database pods
  - Egress               # Outgoing traffic from database pods

  ingress:               # ALLOW incoming traffic:
  - from:
    - podSelector:       # ONLY from pods with role=backend
        matchLabels:
          role: backend
    - namespaceSelector: # AND only from production namespace
        matchLabels:
          env: production
    ports:
    - protocol: TCP
      port: 5432         # Only on PostgreSQL port

  egress:                # ALLOW outgoing traffic:
  - to: []               # Allow nothing (or specify destinations)
    ports:
    - protocol: TCP
      port: 53           # Allow DNS lookups
    - protocol: UDP
      port: 53

---
# DENY ALL ingress (default-deny policy — security best practice)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}        # Applies to ALL pods in namespace
  policyTypes:
  - Ingress              # No ingress rules = deny ALL ingress
```

```
╔══════════════════════════════════════════════════════════════════════╗
║              NETWORK POLICY LOGIC                                    ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Default (no NetworkPolicy): ALL traffic allowed pod-to-pod         ║
║                                                                      ║
║  Once ANY NetworkPolicy selects a pod:                               ║
║  → That pod's selected traffic type becomes DENY-ALL by default     ║
║  → Only explicitly allowed traffic passes                           ║
║                                                                      ║
║  EXAMPLE with default-deny + allow policy:                          ║
║                                                                      ║
║  frontend-pod ──→ database-pod:5432  ❌ (frontend not allowed)      ║
║  backend-pod  ──→ database-pod:5432  ✅ (explicitly allowed)        ║
║  backend-pod  ──→ database-pod:22    ❌ (only port 5432 allowed)    ║
║  database-pod ──→ backend-pod        ❌ (egress denied)             ║
║  database-pod ──→ 8.8.8.8:53        ✅ (DNS allowed in egress)     ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 3.11 Complete Networking Stack — Real-World Example

Let's deploy a complete 3-tier application with proper networking:

```yaml
# complete-network-stack.yaml
# Three tiers: Frontend, Backend API, Database

# ── NAMESPACE ─────────────────────────────────────────────────────────
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    env: production

---
# ── FRONTEND DEPLOYMENT ───────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
        role: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80

---
# ── BACKEND DEPLOYMENT ────────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: api
  template:
    metadata:
      labels:
        app: backend
        tier: api
        role: backend
    spec:
      containers:
      - name: api
        image: my-api:v1
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "database-svc.myapp.svc.cluster.local"   # DNS name
        - name: DB_PORT
          value: "5432"

---
# ── DATABASE STATEFULSET (simplified) ────────────────────────────────
apiVersion: apps/v1
kind: Deployment  # (In practice use StatefulSet — see Chapter 6)
metadata:
  name: database
  namespace: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
        role: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "changeme"   # Use Secrets in production!

---
# ── SERVICES ──────────────────────────────────────────────────────────

# Frontend Service — NodePort for external access (dev) or use Ingress
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: myapp
spec:
  type: ClusterIP         # Ingress will handle external access
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80

---
# Backend Service — ClusterIP (internal only)
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080

---
# Database Service — ClusterIP (internal only)
apiVersion: v1
kind: Service
metadata:
  name: database-svc
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432

---
# ── INGRESS ───────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 8080

---
# ── NETWORK POLICIES ──────────────────────────────────────────────────

# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: myapp
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Allow frontend → backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - port: 8080

---
# Allow backend → database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      role: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - port: 5432
```

---

## Chapter 3: Hands-On Labs

### Lab 3.1 — Service Types Exploration

```bash
# === SETUP: Create a deployment to test services ===
kubectl create deployment web --image=nginx --replicas=3
kubectl get pods -l app=web -o wide    # Note pod IPs

# === ClusterIP Service ===
kubectl expose deployment web --type=ClusterIP --port=80 --name=web-clusterip

kubectl get svc web-clusterip
# NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# web-clusterip   ClusterIP   10.96.45.123    <none>        80/TCP

# Test from inside cluster
kubectl run test-pod --image=busybox --rm -it --restart=Never -- \
  wget -qO- http://web-clusterip
# Should see nginx welcome page!

kubectl get endpoints web-clusterip    # See which pods are backing it

# === NodePort Service ===
kubectl expose deployment web --type=NodePort --port=80 --name=web-nodeport

kubectl get svc web-nodeport
# web-nodeport   NodePort  10.96.45.124  <none>   80:31xxx/TCP

NODE_PORT=$(kubectl get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
echo "Access at: http://$NODE_IP:$NODE_PORT"
curl http://$NODE_IP:$NODE_PORT

# === Scale deployment and watch endpoints update ===
kubectl scale deployment web --replicas=5
kubectl get endpoints web-clusterip -w   # Watch endpoints update

kubectl scale deployment web --replicas=1
kubectl get endpoints web-clusterip      # Fewer endpoints now
```

### Lab 3.2 — DNS Resolution Testing

```bash
# === DNS Lab ===
# Create services in two namespaces
kubectl create namespace ns-a
kubectl create namespace ns-b

kubectl create deployment app-a --image=nginx --namespace=ns-a
kubectl expose deployment app-a --port=80 --namespace=ns-a --name=app-a-svc

kubectl create deployment app-b --image=nginx --namespace=ns-b
kubectl expose deployment app-b --port=80 --namespace=ns-b --name=app-b-svc

# === Test DNS resolution ===
# From ns-a: Can we reach app-a-svc (same namespace)?
kubectl run dns-test --image=busybox --rm -it --restart=Never \
  --namespace=ns-a -- sh
  # Short name (same namespace):
  nslookup app-a-svc
  wget -qO- http://app-a-svc    # Should work

  # Cross-namespace (must use full name):
  nslookup app-b-svc            # FAILS (different namespace)
  nslookup app-b-svc.ns-b       # Works!
  wget -qO- http://app-b-svc.ns-b  # Should work

  # Full FQDN always works:
  nslookup app-b-svc.ns-b.svc.cluster.local

  # Look at DNS configuration:
  cat /etc/resolv.conf           # See search domains

  exit

# === View CoreDNS ===
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Lab 3.3 — Ingress Setup and Testing

```bash
# === Enable Ingress (minikube) ===
minikube addons enable ingress
kubectl get pods -n ingress-nginx

# === Create two apps ===
kubectl create deployment blue --image=docker.io/nginx:alpine
kubectl create deployment green --image=docker.io/nginx:alpine

# Customize each to identify itself
kubectl exec $(kubectl get pod -l app=blue -o name) -- \
  sh -c 'echo "<h1>BLUE SERVICE</h1>" > /usr/share/nginx/html/index.html'
kubectl exec $(kubectl get pod -l app=green -o name) -- \
  sh -c 'echo "<h1>GREEN SERVICE</h1>" > /usr/share/nginx/html/index.html'

# Expose as ClusterIP
kubectl expose deployment blue --port=80 --name=blue-svc
kubectl expose deployment green --port=80 --name=green-svc

# === Create Ingress rules ===
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: color-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: colors.local
    http:
      paths:
      - path: /blue
        pathType: Prefix
        backend:
          service:
            name: blue-svc
            port:
              number: 80
      - path: /green
        pathType: Prefix
        backend:
          service:
            name: green-svc
            port:
              number: 80
EOF

# === Test ===
INGRESS_IP=$(kubectl get ingress color-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$INGRESS_IP colors.local" | sudo tee -a /etc/hosts

curl http://colors.local/blue    # Should see "BLUE SERVICE"
curl http://colors.local/green   # Should see "GREEN SERVICE"

kubectl describe ingress color-ingress   # See rules and backend
```

### Lab 3.4 — Network Policy Testing

```bash
# === NetworkPolicy Lab (requires Calico or Cilium CNI) ===
# If using minikube, start with: minikube start --cni=calico

kubectl create namespace netpol-test

# Deploy two apps
kubectl run server --image=nginx --labels="app=server" -n netpol-test
kubectl run allowed-client --image=busybox --labels="role=allowed" \
  -n netpol-test --command -- sleep 3600
kubectl run blocked-client --image=busybox --labels="role=blocked" \
  -n netpol-test --command -- sleep 3600

kubectl expose pod server --port=80 -n netpol-test --name=server-svc

# Without NetworkPolicy — both clients can reach server:
kubectl exec -n netpol-test allowed-client -- wget -qO- http://server-svc  # Works
kubectl exec -n netpol-test blocked-client -- wget -qO- http://server-svc  # Works

# Apply NetworkPolicy — allow only "allowed" role
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: server-policy
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: allowed
    ports:
    - port: 80
EOF

# Test again with NetworkPolicy applied:
kubectl exec -n netpol-test allowed-client -- wget -qO- http://server-svc   # Still works!
kubectl exec -n netpol-test blocked-client -- wget -qO- --timeout=5 http://server-svc  # BLOCKED!
```

---

## Chapter 3: Troubleshooting Guide

### Issue 1: Service not routing traffic to pods

```bash
# SYSTEMATIC DIAGNOSIS:

# Step 1: Verify service exists and has correct selector
kubectl get svc my-service -o yaml
# Check: spec.selector matches pod labels?

# Step 2: Check Endpoints — are pods listed?
kubectl get endpoints my-service
# If Endpoints is empty (<none>):
#   → Pod labels DON'T match Service selector
#   → Pods are not Ready (failing readiness probe)
#   → Pods are in wrong namespace

# Step 3: Verify pod labels match service selector
kubectl get pods --show-labels
kubectl get svc my-service -o jsonpath='{.spec.selector}'
# Compare the two — they must match!

# Step 4: Check if pods are Ready
kubectl get pods -l app=my-app
# Look at READY column — must be 1/1, not 0/1

# Step 5: Test direct pod connectivity
POD_IP=$(kubectl get pod my-pod -o jsonpath='{.status.podIP}')
kubectl run test --image=busybox --rm -it --restart=Never -- \
  wget -qO- http://$POD_IP:8080
# If this works but Service doesn't → Service config issue
# If this also fails → App/container issue
```

### Issue 2: DNS resolution failing inside pod

```bash
# Test DNS from a debug pod
kubectl run debug --image=nicolaka/netshoot --rm -it --restart=Never -- bash
  cat /etc/resolv.conf          # Check search domains
  nslookup kubernetes           # Test built-in K8s service
  nslookup my-svc               # Test your service
  nslookup my-svc.my-namespace  # Cross-namespace

# If DNS fails:
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl describe pod -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check CoreDNS ConfigMap for misconfig
kubectl get configmap coredns -n kube-system -o yaml

# Restart CoreDNS (if config changed)
kubectl rollout restart deployment coredns -n kube-system
```

### Issue 3: Ingress not routing traffic

```bash
# Step 1: Check Ingress Controller is running
kubectl get pods -n ingress-nginx
# Must be Running and Ready

# Step 2: Check Ingress object
kubectl describe ingress my-ingress
# Look for: Backend and Events sections

# Step 3: Check if backend service exists and has endpoints
kubectl get svc my-svc
kubectl get endpoints my-svc
# Endpoints must not be empty!

# Step 4: Check Ingress Controller logs
kubectl logs -n ingress-nginx \
  deployment/ingress-nginx-controller -f

# Step 5: Test backend service directly
kubectl port-forward svc/my-svc 8080:80
curl http://localhost:8080   # Does the service work directly?

# Step 6: Common Ingress issues:
# - Wrong ingress class annotation
# - Path type mismatch (Prefix vs Exact)
# - Service name typo
# - Port number wrong
# - TLS secret missing
```

### Issue 4: NetworkPolicy blocking unexpected traffic

```bash
# Diagnose with netshoot
kubectl run debug --image=nicolaka/netshoot \
  --labels="role=debug" --rm -it --restart=Never -- bash
  curl http://my-service:8080   # Test connectivity
  # If blocked → NetworkPolicy issue

# List all NetworkPolicies in namespace
kubectl get networkpolicies -n my-namespace
kubectl describe networkpolicy my-policy

# Check if policy matches your pod
kubectl get pod my-pod --show-labels
# Compare with policySelector — do they match?

# Temporarily delete policy to confirm it's the cause
kubectl delete networkpolicy my-policy
# Test again — if traffic now works → policy was the issue

# Check Calico policy rules (if using Calico)
calicoctl get networkpolicy -n my-namespace
```

### Issue 5: LoadBalancer stuck in `<pending>` for external IP

```bash
kubectl get svc my-lb-svc -w
# Stays at EXTERNAL-IP: <pending>

# On cloud: Check cloud provider permissions
# The cluster must have permission to create load balancers

# On bare metal: Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

# Configure MetalLB IP pool
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.250  # Your available IP range
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
EOF
```

---

## Chapter 3: Interview Questions

**Q1: Explain the difference between ClusterIP, NodePort, and LoadBalancer Services.**

> *Answer:* ClusterIP creates a stable virtual IP accessible only within the cluster — ideal for internal microservice communication. NodePort extends ClusterIP by opening a port (30000–32767) on every node, making the service accessible from outside the cluster via `<nodeIP>:<nodePort>`. LoadBalancer extends NodePort further by automatically provisioning a cloud load balancer (on GKE/EKS/AKS) with a public IP. Each type builds on the previous: LoadBalancer = ClusterIP + NodePort + Cloud LB.

**Q2: What is an Ingress and how does it differ from a LoadBalancer Service?**

> *Answer:* A LoadBalancer Service creates a single L4 load balancer per service (TCP/UDP, no HTTP awareness). With 10 services you'd need 10 public IPs — expensive. Ingress is an L7 HTTP router — a single entry point (one LoadBalancer) that routes traffic to multiple services based on URL paths or hostnames. It requires an Ingress Controller (e.g., NGINX) that interprets Ingress rules and routes HTTP requests accordingly. Ingress supports path-based routing, host-based routing, TLS termination, and authentication — things a basic LoadBalancer Service cannot do.

**Q3: What is CoreDNS and how does pod DNS resolution work?**

> *Answer:* CoreDNS is the cluster DNS server running in `kube-system`. Every pod gets `/etc/resolv.conf` injected with the CoreDNS service IP as nameserver and search domains like `<namespace>.svc.cluster.local`. When a pod does `curl http://my-service`, the name resolver appends search domains: tries `my-service.default.svc.cluster.local` → CoreDNS returns the Service's ClusterIP → kube-proxy routes to a backend pod. Cross-namespace DNS requires at minimum `<service>.<namespace>` format.

**Q4: What is a CNI plugin and why is it needed?**

> *Answer:* Kubernetes defines the networking rules (every pod can reach every other pod directly, no NAT) but doesn't implement them itself. CNI (Container Network Interface) plugins implement these rules. They assign IP addresses to pods, set up routes between nodes, and optionally enforce Network Policies. Popular CNIs include Calico (production, supports NetworkPolicy, uses BGP), Flannel (simple, VXLAN overlay), and Cilium (advanced, eBPF-based, best performance). Without a CNI, pods cannot communicate.

**Q5: How do Endpoints relate to Services?**

> *Answer:* Every Service automatically creates and maintains an Endpoints object with the same name. The Endpoint Controller watches for pods matching the Service's label selector that are in Ready state, and lists their IPs and ports in the Endpoints object. kube-proxy watches the Endpoints object and updates iptables/IPVS rules to route traffic accordingly. When a pod fails a readiness probe, it's removed from Endpoints and receives no more traffic — without being killed.

**Q6: What is the difference between `pathType: Prefix` and `pathType: Exact` in Ingress?**

> *Answer:* `pathType: Exact` matches only the exact path specified — `/api` matches only `/api`, not `/api/users`. `pathType: Prefix` matches the path and all sub-paths — `/api` matches `/api`, `/api/`, `/api/users`, `/api/v1/orders`. In production you typically use Prefix for API routes and Exact for specific endpoints like `/healthz` or `/metrics`.

**Q7: What is `externalTrafficPolicy` and what are the tradeoffs?**

> *Answer:* For NodePort and LoadBalancer services, `externalTrafficPolicy: Cluster` (default) routes external traffic to any pod in the cluster, even on a different node. This causes an extra network hop and source IP is NATed (lost). `externalTrafficPolicy: Local` routes only to pods on the receiving node — eliminates the extra hop and preserves the client's source IP, but traffic is unevenly distributed if pods aren't evenly spread across nodes.

**Q8: What is a headless Service and when would you use it?**

> *Answer:* A headless Service has `clusterIP: None`. Instead of creating a virtual IP, DNS queries return all pod IPs directly. This is used with StatefulSets where you need stable, direct pod addressing (e.g., `pod-0.my-service.namespace.svc.cluster.local`). It's also used when the client should discover and connect to individual pods (like database replication, Kafka, Cassandra), rather than going through load-balanced routing.

---

## 📌 CKA Exam Notes — Chapter 3

```
╔════════════════════════════════════════════════════════════════════╗
║                  CKA EXAM FOCUS — CHAPTER 3                       ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: Services & Networking = ~20% of CKA                 ║
║  This is one of the most heavily tested domains.                  ║
║                                                                    ║
║  MUST-KNOW COMMANDS:                                               ║
║  ✅ kubectl expose deployment <n> --type=<type> --port=<p>        ║
║  ✅ kubectl get svc -o wide                                        ║
║  ✅ kubectl describe svc <name>                                    ║
║  ✅ kubectl get endpoints <svc-name>                               ║
║  ✅ kubectl run test --image=busybox --rm -it --restart=Never      ║
║       -- wget -qO- http://<service>                                ║
║  ✅ kubectl get ingress                                             ║
║  ✅ kubectl describe ingress <name>                                ║
║  ✅ kubectl get networkpolicies                                     ║
║                                                                    ║
║  FASTEST WAY TO CREATE SERVICES IN EXAM:                          ║
║  # From deployment:                                                ║
║  k expose deploy web --port=80 --type=NodePort                    ║
║  # Or generate YAML and edit:                                      ║
║  k expose deploy web --port=80 $do > svc.yaml                     ║
║  vim svc.yaml  (edit type, ports, etc.)                           ║
║  k apply -f svc.yaml                                               ║
║                                                                    ║
║  INGRESS YAML — memorize this skeleton:                            ║
║  apiVersion: networking.k8s.io/v1                                 ║
║  kind: Ingress                                                     ║
║  metadata:                                                         ║
║    name: my-ingress                                                ║
║    annotations:                                                    ║
║      kubernetes.io/ingress.class: "nginx"                         ║
║  spec:                                                             ║
║    rules:                                                          ║
║    - host: example.com                                             ║
║      http:                                                         ║
║        paths:                                                      ║
║        - path: /                                                   ║
║          pathType: Prefix                                          ║
║          backend:                                                  ║
║            service:                                                ║
║              name: my-svc                                          ║
║              port:                                                 ║
║                number: 80                                          ║
║                                                                    ║
║  EXAM TRAPS:                                                       ║
║  ⚠️  Service targetPort must match container's containerPort       ║
║  ⚠️  Ingress needs Ingress Controller installed to work            ║
║  ⚠️  NetworkPolicy requires CNI that supports it (Calico/Cilium)  ║
║  ⚠️  Empty Endpoints = service selector doesn't match pod labels  ║
║  ⚠️  Cross-namespace DNS needs full <svc>.<ns> format             ║
║  ⚠️  ExternalName service type: no selector, uses DNS CNAME        ║
║                                                                    ║
║  DEBUG SEQUENCE FOR BROKEN SERVICE (memorize this!):              ║
║  1. kubectl get svc → exists? right type?                         ║
║  2. kubectl get endpoints <svc> → empty? → selector mismatch      ║
║  3. kubectl get pods --show-labels → labels match selector?        ║
║  4. kubectl exec test-pod -- wget http://<pod-ip>:<port>          ║
║     → direct pod works? → service config wrong                    ║
║  5. kubectl exec test-pod -- wget http://<svc>:<port>             ║
║     → service DNS resolves? → DNS or iptables issue               ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Common Mistakes — Chapter 3

| Mistake | Why It Happens | How to Avoid |
|---|---|---|
| Service has empty Endpoints | Selector doesn't match pod labels | Always verify with `kubectl get endpoints <svc>` |
| Ingress doesn't work — 404 | No Ingress Controller installed | Check `kubectl get pods -n ingress-nginx` first |
| Cross-namespace service not found | Using short name in different namespace | Use `<svc>.<namespace>` for cross-namespace access |
| NodePort not accessible externally | Firewall/Security Group blocking port | Open the NodePort range in cloud security groups |
| LoadBalancer stuck Pending | No cloud provider / MetalLB | On-premise: install MetalLB before using LB type |
| NetworkPolicy blocks everything | Applied deny-all without allow rules | Always add explicit allow rules alongside deny-all |
| Wrong `targetPort` | Confusing Service port with container port | `port` = Service listens, `targetPort` = pod listens |
| DNS timeout from pod | CoreDNS down or overloaded | Check `kubectl get pods -n kube-system -l k8s-app=kube-dns` |

---

## Chapter 3 Summary

You now know:

1. **Kubernetes Network Model** — flat pod network, three IP ranges (node/pod/service)
2. **CNI** — the plugin system that implements pod-to-pod networking across nodes
3. **Services** — stable virtual endpoints backed by label selectors and Endpoints objects
4. **ClusterIP** — internal-only service, the foundation of all other types
5. **NodePort** — exposes a port on every node for external access
6. **LoadBalancer** — cloud-provisioned external LB, extends NodePort
7. **Ingress** — L7 HTTP router with path/host-based routing and TLS, requires Ingress Controller
8. **CoreDNS** — cluster DNS, auto-registers services, enables name-based discovery
9. **NetworkPolicies** — firewall rules for pods, requires CNI support
10. **Endpoints** — the dynamic list of pod IPs behind a Service

---

*Next: Chapter 4 — Storage: Volumes, Persistent Volumes, PVCs, Storage Classes, and Dynamic Provisioning*

*"Your apps are reachable. Now let's make sure they don't lose their data when a pod restarts."*
