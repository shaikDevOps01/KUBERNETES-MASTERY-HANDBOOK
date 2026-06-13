# KUBERNETES MASTERY HANDBOOK
# Part 8: Security
# Chapter 8: RBAC, Service Accounts, Network Policies, Security Contexts & Admission Controllers

---

> **"In Kubernetes, the question is never 'Is it running?'
>  The question is: 'WHO can run it, WHERE can it run,
>  and WHAT can it do once it's running?'"**

---

## Chapter Introduction

Security in Kubernetes operates across four distinct layers. Miss any one and
your cluster is vulnerable — regardless of how perfectly everything else is configured.

```
╔══════════════════════════════════════════════════════════════════════╗
║              THE FOUR LAYERS OF KUBERNETES SECURITY                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  LAYER 1: AUTHENTICATION — "Who are you?"                            ║
║  ─────────────────────────────────────────                           ║
║  Certificates (X.509), Service Account Tokens, OIDC, Bootstrap      ║
║  Tokens, Static Token Files, Webhook Token Authentication           ║
║  → Covered in: Cluster Administration (Chapter 10)                  ║
║                                                                      ║
║  LAYER 2: AUTHORISATION — "What are you allowed to do?"             ║
║  ─────────────────────────────────────────────────────              ║
║  RBAC (Role-Based Access Control) — the primary mechanism           ║
║  → WHO (Subject) can do WHAT (Verb) on WHICH (Resource)             ║
║  → Covered in: Section 8.1 — 8.3 (this chapter)                    ║
║                                                                      ║
║  LAYER 3: ADMISSION CONTROL — "Is the request allowed by policy?"   ║
║  ─────────────────────────────────────────────────────────────      ║
║  Validates and mutates requests AFTER auth, BEFORE persistence      ║
║  Pod Security Standards, OPA/Gatekeeper, Kyverno                   ║
║  → Covered in: Section 8.7 (this chapter)                           ║
║                                                                      ║
║  LAYER 4: RUNTIME SECURITY — "What can the pod DO on the node?"     ║
║  ───────────────────────────────────────────────────────────────    ║
║  Security Contexts, Capabilities, seccomp, AppArmor                ║
║  Network Policies (what can pods COMMUNICATE with?)                 ║
║  → Covered in: Sections 8.4 — 8.6 (this chapter)                   ║
║                                                                      ║
║  REQUEST LIFECYCLE:                                                  ║
║  kubectl apply → API Server → Authn → Authz → Admission → etcd     ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 8.1 RBAC — Role-Based Access Control

### 8.1.1 What Is RBAC?

#### In Plain English

RBAC is the **permission system** of Kubernetes. It works exactly like
permissions in a large company:

- A **Role** is a **job description** — "A DevOps Engineer can deploy apps,
  view logs, and restart pods in the staging namespace."
- A **RoleBinding** is the **HR letter** — "Alice is a DevOps Engineer."
- A **ClusterRole** is a **company-wide job description** — same across all departments.
- A **ClusterRoleBinding** is a **company-wide appointment** — applies everywhere.

#### In Technical Language

RBAC uses the `rbac.authorization.k8s.io` API group to drive authorisation
decisions. It is composed of four objects:

```
╔══════════════════════════════════════════════════════════════════════╗
║                   RBAC — FOUR OBJECTS                                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ROLE (namespaced)                                                   ║
║  Defines WHAT actions are allowed on WHICH resources                ║
║  Scope: ONE namespace only                                          ║
║                                                                      ║
║  CLUSTERROLE (cluster-wide)                                          ║
║  Same as Role but applies across ALL namespaces                     ║
║  Also used for: non-namespaced resources (nodes, PVs, namespaces)   ║
║                                                                      ║
║  ROLEBINDING (namespaced)                                            ║
║  Binds a Role OR ClusterRole to a subject in ONE namespace          ║
║  Subjects: User, Group, ServiceAccount                              ║
║                                                                      ║
║  CLUSTERROLEBINDING (cluster-wide)                                   ║
║  Binds a ClusterRole to a subject across ALL namespaces             ║
║                                                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  RBAC MATRIX — which combinations are valid:                         ║
║                                                                      ║
║  ┌─────────────────┬────────────────────┬────────────────────────┐  ║
║  │  Binding Type   │  Role Type         │  Scope of Access       │  ║
║  ├─────────────────┼────────────────────┼────────────────────────┤  ║
║  │  RoleBinding    │  Role              │  One namespace         │  ║
║  │  RoleBinding    │  ClusterRole       │  One namespace only    │  ║
║  │  ClusterRoleBinding│ ClusterRole     │  ALL namespaces        │  ║
║  │  ClusterRoleBinding│ Role            │  ❌ Not valid          │  ║
║  └─────────────────┴────────────────────┴────────────────────────┘  ║
║                                                                      ║
║  KEY INSIGHT: A ClusterRole + RoleBinding = scoped to one namespace ║
║  This is useful for re-using a common role definition across NSes   ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 8.1.2 RBAC Verbs and Resources

```
╔══════════════════════════════════════════════════════════════════════╗
║                  RBAC VERBS (Actions)                                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  VERB          kubectl EQUIVALENT         HTTP METHOD                ║
║  ─────────     ──────────────────         ───────────                ║
║  get           kubectl get <resource>     GET                        ║
║  list          kubectl get <resources>    GET (collection)           ║
║  watch         kubectl get -w             GET (watch)                ║
║  create        kubectl create/apply       POST                       ║
║  update        kubectl apply (replace)    PUT                        ║
║  patch         kubectl patch              PATCH                      ║
║  delete        kubectl delete             DELETE                     ║
║  deletecollection  kubectl delete --all  DELETE (collection)        ║
║                                                                      ║
║  SPECIAL VERBS (non-resource URLs):                                  ║
║  use           Use a PodSecurityPolicy / StorageClass                ║
║  bind          Bind a Role/ClusterRole to a subject                 ║
║  escalate      Update a Role to grant more permissions               ║
║  impersonate   Act as another user/group                             ║
║  exec          kubectl exec into a pod                              ║
║  portforward   kubectl port-forward                                  ║
║  proxy         kubectl proxy                                         ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 8.1.3 Role and ClusterRole YAML

```yaml
# ══════════════════════════════════════════════════════════════════════
# ROLE: Namespaced — grants access within ONE namespace
# ══════════════════════════════════════════════════════════════════════
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production              # Scoped to production namespace
rules:
- apiGroups: [""]                    # "" = core API group (pods, services, etc.)
  resources: ["pods", "pods/log"]    # What resources
  verbs: ["get", "list", "watch"]    # What actions

- apiGroups: ["apps"]                # apps API group (deployments, etc.)
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]

---
# DEVELOPER ROLE: Can deploy apps but not delete or access secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: staging
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods/portforward"]
  verbs: ["create"]

---
# ══════════════════════════════════════════════════════════════════════
# CLUSTERROLE: Cluster-wide — works across all namespaces
# ══════════════════════════════════════════════════════════════════════
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]               # Non-namespaced resource
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["persistentvolumes"]   # Also cluster-scoped
  verbs: ["get", "list", "watch"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]

---
# CLUSTER ADMIN (be careful with this!)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: full-admin
rules:
- apiGroups: ["*"]                   # ALL API groups
  resources: ["*"]                   # ALL resources
  verbs: ["*"]                       # ALL verbs (use sparingly!)

---
# RESTRICTIVE: Read-only everything in a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-viewer
rules:
- apiGroups: ["", "apps", "batch", "extensions", "networking.k8s.io"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### 8.1.4 RoleBinding and ClusterRoleBinding YAML

```yaml
# ══════════════════════════════════════════════════════════════════════
# ROLEBINDING: Binds a Role to a subject in ONE namespace
# ══════════════════════════════════════════════════════════════════════
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: staging
subjects:
- kind: User                        # User | Group | ServiceAccount
  name: alice                       # Username (from certificate CN or OIDC)
  apiGroup: rbac.authorization.k8s.io
roleRef:                            # What role to bind (immutable after creation)
  kind: Role                        # Role | ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io

---
# Binding a GROUP to a role (all users in group get access)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devops-team-binding
  namespace: production
subjects:
- kind: Group
  name: devops-team                 # Group name (from certificate O field or OIDC)
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole                 # Reuse ClusterRole in this single namespace
  name: developer
  apiGroup: rbac.authorization.k8s.io

---
# Binding a ServiceAccount to a role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: my-app-sa                   # ServiceAccount name
  namespace: production             # ServiceAccount namespace (required)
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ══════════════════════════════════════════════════════════════════════
# CLUSTERROLEBINDING: Cluster-wide access
# ══════════════════════════════════════════════════════════════════════
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-team-cluster-admin
subjects:
- kind: Group
  name: cluster-operators
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: bob-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: full-admin
  apiGroup: rbac.authorization.k8s.io
```

### 8.1.5 RBAC Commands

```bash
# ── CREATE RBAC OBJECTS ───────────────────────────────────────────────
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  --namespace=production

kubectl create clusterrole node-viewer \
  --verb=get,list,watch \
  --resource=nodes,persistentvolumes

kubectl create rolebinding alice-binding \
  --role=pod-reader \
  --user=alice \
  --namespace=production

kubectl create rolebinding sa-binding \
  --role=pod-reader \
  --serviceaccount=production:my-app-sa \
  --namespace=production

kubectl create clusterrolebinding ops-admin \
  --clusterrole=cluster-admin \
  --user=bob \
  --group=ops-team

# ── INSPECT RBAC ──────────────────────────────────────────────────────
kubectl get roles -A                           # All roles across namespaces
kubectl get clusterroles                       # All ClusterRoles
kubectl get rolebindings -A                    # All RoleBindings
kubectl get clusterrolebindings                # All ClusterRoleBindings
kubectl describe role pod-reader -n production # What does this role allow?
kubectl describe rolebinding alice-binding -n production

# ── CHECK PERMISSIONS ─────────────────────────────────────────────────
# What can I do?
kubectl auth can-i create pods
kubectl auth can-i delete deployments -n production
kubectl auth can-i list secrets -n kube-system

# What can a specific user do?
kubectl auth can-i create pods --as alice
kubectl auth can-i delete nodes --as alice
kubectl auth can-i create deployments --as alice --namespace=staging

# What can a ServiceAccount do?
kubectl auth can-i list pods \
  --as system:serviceaccount:production:my-app-sa

# List ALL permissions for current user
kubectl auth can-i --list
kubectl auth can-i --list --namespace=production

# Who am I?
kubectl auth whoami

# ── GENERATE YAML WITHOUT CREATING ───────────────────────────────────
kubectl create role pod-reader \
  --verb=get,list,watch --resource=pods \
  --dry-run=client -o yaml
```

---

## 8.2 Service Accounts — Pod Identity

### 8.2.1 What Is a Service Account?

#### In Plain English

A Service Account is the **identity badge** for a pod. When your pod needs to
talk to the Kubernetes API (e.g., a monitoring tool reading pod metrics, or an
operator creating custom resources), it authenticates using its Service Account.
Just like a human user authenticates with a username/password or certificate,
a pod authenticates with its Service Account token.

#### In Technical Language

A **ServiceAccount** is a namespaced Kubernetes object that provides identity
for processes running in pods. When a pod is created without a specified
ServiceAccount, it is automatically assigned the `default` ServiceAccount of
its namespace. The token is automatically mounted into the pod at
`/var/run/secrets/kubernetes.io/serviceaccount/token`.

```
╔══════════════════════════════════════════════════════════════════════╗
║               SERVICE ACCOUNT — HOW IT WORKS                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. ServiceAccount created in namespace                              ║
║  2. Pod created with serviceAccountName: my-app-sa                  ║
║  3. Kubernetes automatically:                                        ║
║     a. Creates a short-lived token (K8s 1.22+, via TokenRequest API)║
║     b. Mounts token at /var/run/secrets/kubernetes.io/serviceaccount/║
║                                                                      ║
║  INSIDE THE POD:                                                     ║
║  /var/run/secrets/kubernetes.io/serviceaccount/                      ║
║  ├── token          ← JWT token for API authentication              ║
║  ├── ca.crt         ← CA cert to verify API server TLS              ║
║  └── namespace      ← Current namespace name                        ║
║                                                                      ║
║  4. Pod calls Kubernetes API:                                        ║
║     curl -H "Authorization: Bearer $(cat /var/run/secrets/          ║
║       kubernetes.io/serviceaccount/token)"                           ║
║     https://kubernetes.default.svc/api/v1/pods                      ║
║                                                                      ║
║  5. API Server validates token → checks RBAC for the SA             ║
║     → allows or denies the request                                  ║
║                                                                      ║
║  SECURITY BEST PRACTICES:                                            ║
║  • Use dedicated SA per application (not default)                   ║
║  • Grant MINIMAL permissions via RBAC (least privilege)             ║
║  • Set automountServiceAccountToken: false when not needed          ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 8.2.2 Service Account YAML and Commands

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  labels:
    app: my-app
automountServiceAccountToken: true   # true (default) | false

---
# DISABLE auto-mount for pods that don't need API access
apiVersion: v1
kind: ServiceAccount
metadata:
  name: restricted-sa
  namespace: production
automountServiceAccountToken: false   # More secure — no token in pod

---
# Pod using a specific ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: api-aware-pod
spec:
  serviceAccountName: my-app-sa      # Which SA to use
  automountServiceAccountToken: true # Can also control at pod level
  containers:
  - name: app
    image: my-app:v1

---
# Complete RBAC + ServiceAccount example:
# App that can list and get pods in its own namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-inspector-sa
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-inspector-role
  namespace: monitoring
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/status"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-inspector-binding
  namespace: monitoring
subjects:
- kind: ServiceAccount
  name: pod-inspector-sa
  namespace: monitoring
roleRef:
  kind: Role
  name: pod-inspector-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-inspector
  namespace: monitoring
spec:
  serviceAccountName: pod-inspector-sa
  containers:
  - name: inspector
    image: bitnami/kubectl:latest
    command: ["sh", "-c"]
    args: ["kubectl get pods -n monitoring && sleep 3600"]
```

```bash
# ServiceAccount commands
kubectl get serviceaccounts             # List SAs in current namespace
kubectl get sa -A                       # All namespaces
kubectl describe sa my-app-sa -n production

# Create SA
kubectl create serviceaccount my-app-sa --namespace production

# Create and immediately get a token (for testing)
kubectl create token my-app-sa --namespace production
kubectl create token my-app-sa --duration=1h

# Inspect the token mounted in a pod
kubectl exec my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
kubectl exec my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/namespace

# List SA tokens
kubectl get secrets -n production | grep my-app-sa

# Check what a SA can do
kubectl auth can-i list pods \
  --as system:serviceaccount:production:my-app-sa \
  --namespace production
```

---

## 8.3 RBAC — Real-World Patterns

### 8.3.1 Least Privilege Patterns

```
╔══════════════════════════════════════════════════════════════════════╗
║              PRODUCTION RBAC PATTERNS                                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PATTERN 1: Read-only Access for Developers                         ║
║  ClusterRole: view (built-in)                                        ║
║  RoleBinding: devs → view (in staging namespace only)               ║
║                                                                      ║
║  PATTERN 2: Deploy-only Access (CI/CD pipeline)                     ║
║  Custom Role: create/update deployments, configmaps                 ║
║  No access to: secrets, RBAC, nodes, PVs                           ║
║                                                                      ║
║  PATTERN 3: Namespace Admin                                          ║
║  ClusterRole: admin (built-in)                                       ║
║  RoleBinding: team-lead → admin (in team namespace only)            ║
║                                                                      ║
║  PATTERN 4: Cross-namespace Read                                     ║
║  ClusterRole: pod-reader                                             ║
║  ClusterRoleBinding: monitoring-sa → pod-reader (cluster-wide)      ║
║                                                                      ║
║  BUILT-IN CLUSTERROLES (use these before creating custom ones):     ║
║  ┌───────────────────────────────────────────────────────────────┐  ║
║  │  cluster-admin  Full access to everything. Use sparingly.     │  ║
║  │  admin          Full access within a namespace.               │  ║
║  │  edit           Read-write within a namespace (no RBAC).      │  ║
║  │  view           Read-only within a namespace.                 │  ║
║  └───────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝
```

```yaml
# ci-cd-role.yaml — for a CI/CD pipeline service account
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer
  namespace: production
rules:
# Can deploy applications
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# Can manage config
- apiGroups: [""]
  resources: ["configmaps", "services"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# Can read pods and their logs (for health checks)
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/status"]
  verbs: ["get", "list", "watch"]
# Can manage Jobs (for DB migrations)
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "delete"]
# Explicitly NO access to:
# - secrets (handled by external vault)
# - RBAC objects (cannot escalate own permissions)
# - nodes, PVs (infrastructure only)
```

---

## 8.4 Network Policies — Pod-Level Firewall

Network Policies were introduced in Chapter 3 as part of networking.
Here we cover them in depth from a security perspective.

### 8.4.1 Default Deny — Security Baseline

```yaml
# security-network-policies.yaml

# STEP 1: Apply default-deny to ALL pods in namespace
# This is the security baseline — deny everything, then allow what's needed
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}           # Applies to ALL pods
  policyTypes:
  - Ingress
  - Egress

---
# STEP 2: Allow pods to query DNS (always needed)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP

---
# STEP 3: Allow frontend to call backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
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
# STEP 4: Allow backend to reach database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: production
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

---
# STEP 5: Allow monitoring namespace to scrape all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-ingress
  namespace: production
spec:
  podSelector: {}            # All pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - port: 9090             # Prometheus metrics port

---
# STEP 6: Allow pods to reach external internet (egress)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      external-access: allowed        # Only labeled pods get internet
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0              # Allow all IPs
        except:
        - 10.0.0.0/8                 # But NOT internal cluster IPs
        - 172.16.0.0/12
        - 192.168.0.0/16
```

---

## 8.5 Security Contexts — Container Runtime Security

### 8.5.1 What Is a Security Context?

#### In Plain English

A Security Context defines the **operating system-level permissions** for a
pod or container. It answers: Can the container run as root? Can it write to
the host filesystem? Can it use privileged system calls? Think of it as
the **terms and conditions** the container must follow when it runs.

#### In Technical Language

A `securityContext` in Kubernetes maps to Linux security features:
user/group IDs, capabilities, SELinux/AppArmor profiles, seccomp filters,
read-only root filesystems, and privilege escalation controls.

```
╔══════════════════════════════════════════════════════════════════════╗
║           SECURITY CONTEXT — POD vs CONTAINER LEVEL                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  spec.securityContext (POD-LEVEL)                                    ║
║  Applies to ALL containers in the pod:                              ║
║  • runAsUser / runAsGroup — UID/GID for all containers              ║
║  • fsGroup — group ownership of mounted volumes                     ║
║  • runAsNonRoot — reject if container wants root                    ║
║  • sysctls — kernel parameters                                      ║
║                                                                      ║
║  spec.containers[].securityContext (CONTAINER-LEVEL)                 ║
║  Overrides or extends pod-level for specific container:             ║
║  • runAsUser / runAsGroup — override pod-level UID/GID              ║
║  • allowPrivilegeEscalation — can su/sudo? (default: true!)        ║
║  • privileged — full host access (like root on the node)            ║
║  • readOnlyRootFilesystem — container filesystem is read-only       ║
║  • capabilities — add/drop Linux capabilities                       ║
║  • seccompProfile — system call filtering                           ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 8.5.2 Security Context YAML — Production Hardened

```yaml
# security-context-hardened.yaml
# Production-hardened pod with minimal attack surface
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  # ── POD-LEVEL SECURITY CONTEXT ──────────────────────────────────
  securityContext:
    runAsNonRoot: true           # Reject if image wants to run as root (UID 0)
    runAsUser: 1000              # Run ALL containers as UID 1000
    runAsGroup: 3000             # Run ALL containers as GID 3000
    fsGroup: 2000                # Volumes mounted with group 2000
                                 # Allows app to write to mounted volumes
    fsGroupChangePolicy: OnRootMismatch   # Only fix permissions if needed
    seccompProfile:
      type: RuntimeDefault       # Use container runtime's default seccomp profile
                                 # Blocks ~150 dangerous system calls
    supplementalGroups: [4000]   # Additional groups for volume access
    sysctls:                     # Kernel parameters (use carefully)
    - name: net.core.somaxconn
      value: "1024"

  containers:
  - name: app
    image: my-app:v1

    # ── CONTAINER-LEVEL SECURITY CONTEXT ────────────────────────────
    securityContext:
      allowPrivilegeEscalation: false   # Cannot escalate to root (su, sudo)
                                         # ALWAYS set this to false!
      privileged: false                  # No full host access
      readOnlyRootFilesystem: true       # Container cannot write to its own FS
                                         # Forces use of explicit volumes for writes
      runAsUser: 1000                    # Override pod-level if needed
      runAsNonRoot: true

      # ── LINUX CAPABILITIES ──────────────────────────────────────
      # Capabilities are fine-grained root permissions
      # Drop ALL, then add only what's specifically needed
      capabilities:
        drop:
        - ALL                    # Drop ALL capabilities (most secure baseline)
        add:
        - NET_BIND_SERVICE       # Allow binding to port < 1024 (e.g., port 80)
        # Common capabilities and what they allow:
        # NET_BIND_SERVICE — bind ports below 1024
        # NET_RAW          — raw socket access (ping) — usually drop this
        # SYS_TIME         — set system time — almost never needed
        # SYS_ADMIN        — most dangerous — avoid at all costs

      # ── SECCOMP PROFILE ───────────────────────────────────────────
      seccompProfile:
        type: RuntimeDefault     # RuntimeDefault | Localhost | Unconfined

    # Writable directories must be explicit volumes
    # (because readOnlyRootFilesystem: true)
    volumeMounts:
    - name: tmp-dir
      mountPath: /tmp
    - name: app-logs
      mountPath: /app/logs

  volumes:
  - name: tmp-dir
    emptyDir: {}                 # Writable temp directory
  - name: app-logs
    emptyDir: {}                 # Writable log directory

---
# SECURITY CONTEXT COMPARISON TABLE (in YAML comment form):
# ┌──────────────────────────────┬─────────────┬──────────────────────┐
# │  Setting                     │  Default    │  Secure Value        │
# ├──────────────────────────────┼─────────────┼──────────────────────┤
# │  runAsNonRoot                │  false      │  true                │
# │  allowPrivilegeEscalation    │  true  ⚠️  │  false               │
# │  privileged                  │  false      │  false               │
# │  readOnlyRootFilesystem      │  false      │  true                │
# │  capabilities.drop           │  none       │  ["ALL"]             │
# │  seccompProfile              │  Unconfined │  RuntimeDefault      │
# └──────────────────────────────┴─────────────┴──────────────────────┘
```

### 8.5.3 Linux Capabilities Deep Dive

```
╔══════════════════════════════════════════════════════════════════════╗
║              LINUX CAPABILITIES — KEY ONES TO KNOW                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Default capabilities given to containers (Docker/containerd):      ║
║  CHOWN, DAC_OVERRIDE, FSETID, FOWNER, MKNOD, NET_RAW, SETGID,      ║
║  SETUID, SETFCAP, SETPCAP, NET_BIND_SERVICE, SYS_CHROOT,           ║
║  KILL, AUDIT_WRITE                                                   ║
║                                                                      ║
║  CAPABILITY       ALLOWS                          RISK LEVEL         ║
║  ─────────────    ─────────────────────────────   ──────────         ║
║  NET_RAW          Raw/packet sockets (ping)        Medium            ║
║                   → Can sniff network traffic                        ║
║                                                                      ║
║  NET_BIND_SERVICE Bind ports < 1024               Low               ║
║                   → Needed for web servers on port 80               ║
║                                                                      ║
║  SYS_ADMIN        Most dangerous capability        CRITICAL          ║
║                   → Equivalent to root on the node                  ║
║                   → Can mount filesystems, load kernel modules      ║
║                                                                      ║
║  SYS_PTRACE       Trace other processes            HIGH              ║
║                   → Can read memory of any process                  ║
║                                                                      ║
║  CAP_SETUID       Change UID to any user           HIGH              ║
║  CAP_SETGID       Change GID to any group          HIGH              ║
║                                                                      ║
║  RECOMMENDATION:                                                     ║
║  capabilities:                                                       ║
║    drop: [ALL]                # Drop everything first               ║
║    add: [NET_BIND_SERVICE]    # Add back only what is truly needed  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 8.6 Pod Security Standards (PSS) — Namespace-Level Security

### 8.6.1 What Are Pod Security Standards?

Pod Security Standards replaced the deprecated PodSecurityPolicy (PSP) in
Kubernetes 1.25. They define three levels of security that can be applied
to namespaces:

```
╔══════════════════════════════════════════════════════════════════════╗
║              POD SECURITY STANDARDS — THREE LEVELS                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PRIVILEGED — No restrictions                                        ║
║  ─────────────────────────────                                       ║
║  Allows everything: privileged containers, host networking,          ║
║  host PID namespace, all capabilities.                              ║
║  Use: kube-system namespace, trusted cluster infrastructure         ║
║                                                                      ║
║  BASELINE — Minimally restrictive                                    ║
║  ─────────────────────────────────                                   ║
║  Prevents known privilege escalations:                              ║
║  ✅ Allows: most workloads                                           ║
║  ❌ Blocks: privileged containers, host network/PID/IPC,            ║
║             dangerous capabilities (SYS_ADMIN, NET_RAW, etc.)       ║
║  Use: General-purpose workloads, default for most namespaces        ║
║                                                                      ║
║  RESTRICTED — Hardened, security-best-practice                       ║
║  ────────────────────────────────────────────────                    ║
║  All Baseline restrictions PLUS:                                     ║
║  ❌ Must run as non-root                                             ║
║  ❌ Must drop ALL capabilities                                       ║
║  ❌ Must have allowPrivilegeEscalation: false                       ║
║  ❌ seccompProfile must be RuntimeDefault or Localhost               ║
║  Use: Security-sensitive workloads, regulated environments          ║
║                                                                      ║
║  THREE ENFORCEMENT MODES:                                            ║
║  enforce — Reject pods that violate the policy                      ║
║  audit   — Allow pods but log violations (good for migration)       ║
║  warn    — Allow pods but warn the user in kubectl output           ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 8.6.2 Applying Pod Security Standards to Namespaces

```yaml
# pod-security-standards.yaml
# Apply via namespace labels

apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # ENFORCE restricted — pods violating policy are rejected
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

    # AUDIT baseline — log violations but allow
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/audit-version: latest

    # WARN baseline — warn user in kubectl output
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: latest

---
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    # Baseline enforcement for dev (less strict)
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted    # Warn to help developers improve

---
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    # System namespace needs privileged access
    pod-security.kubernetes.io/enforce: privileged
```

```bash
# Apply PSS to existing namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted

# Test what would happen (dry-run)
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  --dry-run=server

# Check existing pods in namespace against new policy
kubectl -n production apply \
  --dry-run=server -f existing-deployment.yaml
```

---

## 8.7 Admission Controllers

### 8.7.1 What Are Admission Controllers?

```
╔══════════════════════════════════════════════════════════════════════╗
║              ADMISSION CONTROLLER PIPELINE                           ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  API Request                                                         ║
║      │                                                               ║
║      ▼                                                               ║
║  Authentication → Authorization → ADMISSION CONTROL → etcd         ║
║                                        │                             ║
║                               ┌────────┴────────┐                   ║
║                               │                 │                   ║
║                    MUTATING WEBHOOKS   VALIDATING WEBHOOKS          ║
║                    (change the object) (accept or reject)           ║
║                               │                 │                   ║
║                    e.g.:       │      e.g.:      │                   ║
║                    • Auto-inject│    • Require   │                   ║
║                      sidecars  │      resource  │                   ║
║                    • Add labels│      limits    │                   ║
║                    • Set default│    • Reject   │                   ║
║                      security  │      root      │                   ║
║                      context   │      containers│                   ║
║                                                                      ║
║  BUILT-IN ADMISSION CONTROLLERS (always-on examples):               ║
║  NamespaceLifecycle   — reject ops in terminating namespaces        ║
║  LimitRanger          — enforce LimitRange defaults                 ║
║  ServiceAccount       — auto-mount SA token                         ║
║  ResourceQuota        — enforce namespace quotas                    ║
║  PodSecurity          — enforce Pod Security Standards              ║
║  DefaultStorageClass  — add default SC to unset PVCs                ║
║                                                                      ║
║  EXTERNAL POLICY ENGINES:                                            ║
║  OPA/Gatekeeper — Rego policy language, most mature                ║
║  Kyverno         — Kubernetes-native YAML policies, simpler         ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 8.7.2 Kyverno — Kubernetes-Native Policy Engine

```yaml
# kyverno-policy-require-resources.yaml
# Require all pods to have resource requests and limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-pod-resources
spec:
  validationFailureAction: Enforce    # Enforce | Audit
  rules:
  - name: check-container-resources
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "CPU and memory resource requests and limits are required."
      pattern:
        spec:
          containers:
          - resources:
              requests:
                memory: "?*"         # Must be set (any value)
                cpu: "?*"
              limits:
                memory: "?*"
                cpu: "?*"

---
# Require labels on all pods
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-pod-labels
spec:
  validationFailureAction: Audit
  rules:
  - name: check-labels
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Pod must have 'app' and 'env' labels."
      pattern:
        metadata:
          labels:
            app: "?*"
            env: "?*"

---
# Auto-add label to all pods (mutating policy)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-managed-by-label
spec:
  rules:
  - name: add-label
    match:
      any:
      - resources:
          kinds: ["Pod"]
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            managed-by: kyverno
```

---

## 8.8 User Authentication — Certificate-Based

```bash
# Create a user certificate for 'alice' (cluster admin task)

# Step 1: Generate private key
openssl genrsa -out alice.key 2048

# Step 2: Create Certificate Signing Request
openssl req -new -key alice.key \
  -subj "/CN=alice/O=developers" \   # CN=username, O=group
  -out alice.csr

# Step 3: Create Kubernetes CSR object
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: alice-csr
spec:
  request: $(cat alice.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400          # 24 hours
  usages:
  - client auth
EOF

# Step 4: Approve the CSR (cluster admin action)
kubectl certificate approve alice-csr
kubectl get csr alice-csr            # Check STATUS: Approved, Issued

# Step 5: Extract the signed certificate
kubectl get csr alice-csr \
  -o jsonpath='{.status.certificate}' | base64 -d > alice.crt

# Step 6: Add to kubeconfig
kubectl config set-credentials alice \
  --client-certificate=alice.crt \
  --client-key=alice.key

kubectl config set-context alice-context \
  --cluster=my-cluster \
  --user=alice

# Step 7: Grant RBAC permissions
kubectl create rolebinding alice-dev \
  --clusterrole=edit \
  --user=alice \
  --namespace=staging

# Step 8: Test as alice
kubectl auth can-i create pods --as alice --namespace staging
```

---

## 8.9 Secrets Encryption at Rest

```yaml
# encryption-config.yaml
# Configure API server to encrypt Secrets in etcd
# Apply to: /etc/kubernetes/encryption-config.yaml on control plane

apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets                         # Encrypt Secret objects
  providers:
  - aescbc:                         # AES-CBC encryption (recommended)
      keys:
      - name: key1
        secret: <32-byte-base64-key>  # openssl rand -base64 32
  - identity: {}                    # Fallback: no encryption (for existing secrets)
                                    # Remove this after encrypting all secrets

# Also supports:
# - aesgcm (AES-GCM, faster)
# - secretbox (XSalsa20 + Poly1305)
# - kms (AWS KMS, GCP KMS — production recommendation)
```

```bash
# Generate encryption key
openssl rand -base64 32

# After applying encryption config, encrypt all existing secrets:
kubectl get secrets -A -o json | kubectl replace -f -

# Verify a secret is encrypted in etcd
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | hexdump -C | head
# If encrypted: shows "k8s:enc:aescbc:v1:key1:..." prefix
```

---

## Chapter 8: Hands-On Labs

### Lab 8.1 — RBAC: Create User with Limited Permissions

```bash
# Step 1: Create a namespace and a test role
kubectl create namespace lab-rbac

kubectl create role pod-viewer \
  --verb=get,list,watch \
  --resource=pods,pods/log \
  --namespace=lab-rbac

kubectl create rolebinding test-user-binding \
  --role=pod-viewer \
  --user=test-user \
  --namespace=lab-rbac

# Step 2: Verify permissions using --as flag
kubectl auth can-i get pods \
  --as test-user --namespace lab-rbac       # yes
kubectl auth can-i delete pods \
  --as test-user --namespace lab-rbac       # no
kubectl auth can-i get secrets \
  --as test-user --namespace lab-rbac       # no
kubectl auth can-i get pods \
  --as test-user --namespace default        # no (role is namespace-scoped)

# Step 3: List ALL permissions for test-user
kubectl auth can-i --list --as test-user --namespace lab-rbac

# Step 4: Add deployment access
kubectl patch role pod-viewer \
  --namespace lab-rbac \
  --type='json' \
  -p='[{"op":"add","path":"/rules/-","value":
    {"apiGroups":["apps"],"resources":["deployments"],
     "verbs":["get","list","watch"]}}]'

kubectl auth can-i get deployments \
  --as test-user --namespace lab-rbac       # yes now!
```

### Lab 8.2 — Service Account with RBAC

```bash
# Step 1: Create SA, Role, and RoleBinding
kubectl create namespace sa-lab

kubectl create serviceaccount metrics-reader \
  --namespace sa-lab

kubectl create role metrics-role \
  --verb=get,list,watch \
  --resource=pods,services,endpoints \
  --namespace sa-lab

kubectl create rolebinding metrics-binding \
  --role=metrics-role \
  --serviceaccount=sa-lab:metrics-reader \
  --namespace sa-lab

# Step 2: Deploy a pod using this SA
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: metrics-pod
  namespace: sa-lab
spec:
  serviceAccountName: metrics-reader
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=ready pod/metrics-pod -n sa-lab --timeout=60s

# Step 3: Test API access from inside the pod
kubectl exec -n sa-lab metrics-pod -- \
  kubectl get pods -n sa-lab              # Should work (has permission)

kubectl exec -n sa-lab metrics-pod -- \
  kubectl get secrets -n sa-lab          # Should fail (no permission)

kubectl exec -n sa-lab metrics-pod -- \
  kubectl get pods -n default            # Should fail (different namespace)

# Step 4: Inspect the mounted token
kubectl exec -n sa-lab metrics-pod -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
# Output: sa-lab
```

### Lab 8.3 — Security Context Enforcement

```bash
# Test 1: Pod running as root (BAD)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl exec root-pod -- id
# uid=0(root) gid=0(root) — running as root!

# Test 2: Same pod with security context (GOOD)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: false   # nginx needs to write
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
EOF

kubectl get pod secure-pod     # May fail if nginx needs root
kubectl exec secure-pod -- id  # Should show uid=1000

# Test 3: Try to run privileged pod in restricted namespace
kubectl label namespace default \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true
EOF
# Should be REJECTED with: Error: pods "privileged-pod" is forbidden

# Reset namespace label
kubectl label namespace default \
  pod-security.kubernetes.io/enforce-
```

### Lab 8.4 — Network Policy: Allow and Block

```bash
kubectl create namespace netpol-security-lab

# Deploy two apps
kubectl run backend --image=nginx \
  --labels="app=backend" \
  --namespace=netpol-security-lab
kubectl run frontend --image=nginx \
  --labels="app=frontend" \
  --namespace=netpol-security-lab
kubectl run intruder --image=nginx \
  --labels="app=intruder" \
  --namespace=netpol-security-lab

kubectl expose pod backend \
  --port=80 --name=backend-svc \
  --namespace=netpol-security-lab

# Without policy — everyone can reach backend
kubectl exec -n netpol-security-lab frontend -- \
  wget -qO- --timeout=3 http://backend-svc  # Works
kubectl exec -n netpol-security-lab intruder -- \
  wget -qO- --timeout=3 http://backend-svc  # Also works (bad!)

# Apply deny-all + allow only frontend
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: netpol-security-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 80
EOF

# Test with NetworkPolicy applied
kubectl exec -n netpol-security-lab frontend -- \
  wget -qO- --timeout=3 http://backend-svc  # Still works (allowed)
kubectl exec -n netpol-security-lab intruder -- \
  wget -qO- --timeout=3 http://backend-svc  # BLOCKED! (timeout)
```

---

## Chapter 8: Troubleshooting Guide

### Issue 1: `Forbidden` — RBAC permission denied

```bash
kubectl get pods -n production
# Error from server (Forbidden): pods is forbidden:
# User "alice" cannot list resource "pods" in namespace "production"

# DIAGNOSIS:
# Step 1: Check who you are
kubectl auth whoami

# Step 2: Check what you can do
kubectl auth can-i list pods -n production

# Step 3: Check existing bindings
kubectl get rolebindings -n production -o yaml | grep -A5 subjects
kubectl get clusterrolebindings -o yaml | grep -A5 "name: alice"

# Step 4: Find all bindings for a user
kubectl get rolebindings,clusterrolebindings -A \
  -o custom-columns='KIND:kind,NAMESPACE:metadata.namespace,
  NAME:metadata.name,SERVICE_ACCOUNTS:subjects[*].name' | \
  grep alice

# Fix: Create the missing binding
kubectl create rolebinding fix-alice \
  --clusterrole=view \
  --user=alice \
  --namespace=production
```

### Issue 2: Pod rejected by Pod Security Standards

```bash
kubectl apply -f my-pod.yaml
# Error: pods "my-pod" is forbidden:
# violates PodSecurity "restricted:latest":
# allowPrivilegeEscalation != false

# DIAGNOSIS: Check namespace policy
kubectl get namespace production \
  -o jsonpath='{.metadata.labels}'

# What exactly is failing:
kubectl apply -f my-pod.yaml --dry-run=server
# This shows the full policy violation message

# Fix: Add required security context
# For restricted level, you need:
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

### Issue 3: ServiceAccount cannot access API

```bash
kubectl exec my-pod -- kubectl get pods
# Error: pods is forbidden: User "system:serviceaccount:default:default"
# cannot list resource "pods" in namespace "default"

# DIAGNOSIS:
# Step 1: What SA does the pod use?
kubectl get pod my-pod -o jsonpath='{.spec.serviceAccountName}'
# Output: default (the default SA — no special permissions)

# Step 2: Check what the SA can do
kubectl auth can-i list pods \
  --as system:serviceaccount:default:default

# Fix: Create a dedicated SA with correct RBAC
kubectl create serviceaccount pod-reader-sa
kubectl create rolebinding pod-reader-binding \
  --clusterrole=view \
  --serviceaccount=default:pod-reader-sa

# Update the deployment to use new SA
kubectl patch deployment my-app \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"pod-reader-sa"}}}}'
```

### Issue 4: Network Policy blocking unexpected traffic

```bash
# Debug: Run a netshoot pod from the SAME namespace
kubectl run debug --image=nicolaka/netshoot \
  --labels="app=debug" \
  --namespace=production -it --rm --restart=Never -- bash

  # Test connectivity
  curl -v http://backend-svc:8080      # Can debug pod reach backend?
  nslookup backend-svc                 # Does DNS resolve?

# List all NetworkPolicies in namespace
kubectl get networkpolicies -n production
kubectl describe networkpolicy my-policy -n production

# Check if the pod has labels that match policy
kubectl get pod my-pod --show-labels
# Compare against NetworkPolicy podSelector

# Temporarily remove policy to confirm it's the cause
kubectl delete networkpolicy my-policy -n production
# If traffic works now → policy was the blocker
```

### Issue 5: Container keeps getting OOMKilled or segfault due to seccomp

```bash
kubectl describe pod my-pod | grep -A3 "Last State"
# Reason: OOMKilled or Error

# Check if seccomp is blocking system calls
kubectl get pod my-pod -o yaml | grep seccomp

# Temporarily set to Unconfined (NOT for production)
seccompProfile:
  type: Unconfined
# If this fixes it → your app uses non-standard syscalls
# Long-term: create custom seccomp profile allowing those calls
```

---

## Chapter 8: Interview Questions

**Q1: What are the four components of Kubernetes RBAC?**

> RBAC has four objects: (1) **Role** — defines allowed actions on resources, namespaced. (2) **ClusterRole** — same as Role but cluster-wide; also used for non-namespaced resources like nodes and PVs. (3) **RoleBinding** — binds a Role or ClusterRole to subjects (User, Group, ServiceAccount) in one namespace. (4) **ClusterRoleBinding** — binds a ClusterRole to subjects across all namespaces. A ClusterRole + RoleBinding is valid and scopes the cluster role to a single namespace.

**Q2: What is the difference between a Role and a ClusterRole?**

> A Role is namespaced — it only grants access to resources within one specific namespace. A ClusterRole is cluster-scoped — it grants access across all namespaces and can also govern non-namespaced resources like nodes, PersistentVolumes, and StorageClasses. ClusterRoles can be bound namespace-specifically via a RoleBinding or cluster-wide via a ClusterRoleBinding. You should use ClusterRoles when you want to reuse the same role definition across multiple namespaces, but bind it per-namespace via RoleBindings.

**Q3: What is a ServiceAccount and how is it different from a User?**

> A ServiceAccount is a namespaced Kubernetes object that provides an identity for pods and processes running inside the cluster. It is automatically assigned a JWT token mounted into pods at a well-known path. A User (in Kubernetes) is an external identity — a person or system outside the cluster, authenticated via certificates, OIDC tokens, or bearer tokens. Users are not Kubernetes objects — they exist only in the authentication layer. ServiceAccounts have first-class RBAC support, auto-created tokens, and are specific to a namespace.

**Q4: What is `allowPrivilegeEscalation` and why should it always be false?**

> `allowPrivilegeEscalation` controls whether a process inside a container can gain more privileges than its parent process — essentially whether it can use `su`, `sudo`, or SUID binaries to become root. The default is `true`, meaning containers CAN escalate to root even if they started as a non-root user. This is a significant security risk. Setting `allowPrivilegeEscalation: false` prevents this entirely. In a `restricted` Pod Security Standard, this must be false.

**Q5: Explain the three Pod Security Standard levels.**

> **Privileged** — no restrictions, allows everything including host network, privileged containers, and all capabilities. For system namespaces only. **Baseline** — prevents known privilege escalations: blocks privileged containers, dangerous capabilities (SYS_ADMIN), host PID/network/IPC sharing. Most workloads should meet this. **Restricted** — implements security best practices: requires non-root, drops all capabilities, requires `allowPrivilegeEscalation: false`, and requires a seccomp profile. For security-critical workloads.

**Q6: How do you check what permissions a user or ServiceAccount has?**

> Use `kubectl auth can-i`: `kubectl auth can-i list pods --as alice --namespace staging`. For ServiceAccounts: `kubectl auth can-i list pods --as system:serviceaccount:namespace:sa-name`. To list ALL permissions: `kubectl auth can-i --list --as alice --namespace staging`. To find all RoleBindings for a subject: `kubectl get rolebindings,clusterrolebindings -A -o yaml | grep -B5 "name: alice"`.

**Q7: What is the difference between `readOnlyRootFilesystem` and volume mounts?**

> `readOnlyRootFilesystem: true` makes the entire container filesystem read-only — the container cannot write anywhere in its own filesystem layer. This prevents attackers from writing malicious files, modifying binaries, or creating backdoors. Applications that need to write files (temp files, logs, caches) must use explicit `volumeMounts` with `emptyDir` or PVC volumes — these are mounted into the container separately and remain writable regardless of the root filesystem setting. This is considered a security best practice.

**Q8: What is an Admission Controller and how does it differ from RBAC?**

> RBAC answers "Is this user/SA allowed to perform this action?" — it is a binary authorisation decision before the request is processed. Admission Controllers run AFTER successful auth, and examine the actual content of the request to validate or mutate it. They can: enforce policies (all pods must have resource limits), mutate objects (auto-inject sidecars), or perform complex validation that RBAC cannot (reject pods requesting privileged access even from authorised users). RBAC is who can do what; Admission Controllers are what is policy-compliant.

---

## CKA Exam Notes — Chapter 8

```
╔════════════════════════════════════════════════════════════════════╗
║                  CKA EXAM FOCUS — CHAPTER 8                       ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: Security = ~15% of CKA exam                         ║
║  RBAC is the most tested topic in this chapter.                   ║
║                                                                    ║
║  FASTEST RBAC COMMANDS (memorize these):                           ║
║  k create role <n> --verb=get,list --resource=pods -n <ns>        ║
║  k create clusterrole <n> --verb=get,list --resource=nodes        ║
║  k create rolebinding <n> --role=<r> --user=<u> -n <ns>          ║
║  k create rolebinding <n> --role=<r>                               ║
║     --serviceaccount=<ns>:<sa> -n <ns>                            ║
║  k create clusterrolebinding <n> --clusterrole=<r> --user=<u>    ║
║                                                                    ║
║  AUTH CHECKING (critical exam skill):                              ║
║  k auth can-i <verb> <resource> --as <user> -n <ns>               ║
║  k auth can-i --list --as <user> -n <ns>                          ║
║  k auth whoami                                                     ║
║                                                                    ║
║  SERVICEACCOUNT EXAM PATTERN:                                      ║
║  1. k create sa my-sa -n my-ns                                    ║
║  2. k create role my-role --verb=... --resource=... -n my-ns      ║
║  3. k create rolebinding my-rb --role=my-role                      ║
║        --serviceaccount=my-ns:my-sa -n my-ns                      ║
║  4. In pod spec: serviceAccountName: my-sa                        ║
║                                                                    ║
║  SECURITY CONTEXT EXAM PATTERN:                                    ║
║  spec:                                                             ║
║    securityContext:                                                ║
║      runAsUser: 1000                                               ║
║      runAsGroup: 3000                                              ║
║      fsGroup: 2000                                                 ║
║    containers:                                                     ║
║    - name: app                                                     ║
║      securityContext:                                              ║
║        allowPrivilegeEscalation: false                             ║
║        capabilities:                                               ║
║          drop: ["ALL"]                                             ║
║                                                                    ║
║  EXAM TRAPS:                                                       ║
║  RoleBinding references: roleRef is IMMUTABLE after creation      ║
║  SA format in rolebinding: namespace:sa-name (two parts!)         ║
║  ClusterRoleBinding + Role = INVALID combination                  ║
║  runAsUser is UID (number), not username string                    ║
║  kubectl auth can-i uses current context — specify --as to test   ║
║                                                                    ║
║  BUILT-IN CLUSTERROLES (use in exam instead of creating custom):  ║
║  cluster-admin  admin  edit  view                                  ║
║  k create rolebinding dev-bind --clusterrole=edit                 ║
║     --user=alice -n staging   ← Scopes edit role to one namespace ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Common Mistakes — Chapter 8

| Mistake | Why It Happens | How to Avoid |
|---|---|---|
| Wrong SA format in RoleBinding | Should be `namespace:name` | Always `--serviceaccount=namespace:sa-name` |
| ClusterRoleBinding + Role | Trying to bind a Role cluster-wide | ClusterRoleBinding can only ref ClusterRole |
| Forgetting `apiGroups` in Role rules | Core resources need `""`, apps need `"apps"` | Always check `kubectl api-resources` for apiGroup |
| `runAsUser` as string not int | `runAsUser: "1000"` fails | Must be integer: `runAsUser: 1000` |
| No `automountServiceAccountToken: false` | Default SA token always mounted | Disable for pods that don't need API access |
| Applying default-deny without DNS allow | DNS stops working → all name resolution fails | Always add DNS egress rule alongside default-deny |
| `readOnlyRootFilesystem` without emptyDir | App crashes writing to /tmp | Add emptyDir volumes for any writable paths |
| Testing NetworkPolicy on Flannel | Flannel doesn't support NetworkPolicy | Use Calico or Cilium for NetworkPolicy testing |

---

## Chapter 8 Summary

1. **Four security layers** — Authentication → Authorisation (RBAC) → Admission → Runtime
2. **RBAC objects** — Role (namespaced), ClusterRole (cluster-wide), RoleBinding, ClusterRoleBinding
3. **RBAC rules** — apiGroups + resources + verbs; built-in roles: cluster-admin, admin, edit, view
4. **ServiceAccounts** — pod identity; token auto-mounted; use dedicated SA per app; least privilege
5. **auth can-i** — essential command for permission verification; use `--as` to test other users
6. **Network Policies** — default-deny baseline; always allow DNS; layer in explicit allow rules
7. **Security Contexts** — pod-level (user/group/fsGroup) + container-level (capabilities, privilege)
8. **allowPrivilegeEscalation: false** — always set this; prevents su/sudo escalation
9. **capabilities** — drop ALL first, add back only what is required
10. **readOnlyRootFilesystem** — prevents filesystem tampering; use emptyDir for writable paths
11. **Pod Security Standards** — Privileged / Baseline / Restricted applied at namespace level
12. **Admission Controllers** — validate/mutate requests post-auth; Kyverno / OPA for policy as code
13. **Secrets encryption at rest** — requires explicit EncryptionConfiguration; use KMS for production

---

*Next: Chapter 9 — Monitoring & Logging: Metrics Server, Prometheus, Grafana & Logging Architecture*

*"Your cluster is secured. Now let's make sure you can SEE what's happening inside it —
 before your users tell you something is broken."*
