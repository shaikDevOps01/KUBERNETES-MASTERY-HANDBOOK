# KUBERNETES MASTERY HANDBOOK
# Part 11: Production Kubernetes
# Chapter 11: High Availability, GitOps, Helm & ArgoCD

---

> **"A cluster that anyone can build is impressive.
>  A cluster that runs without human intervention,
>  recovers from failure automatically, and deploys itself
>  from a Git commit — that is production Kubernetes."**

---

## Chapter Introduction

You have learned to build, secure, monitor, and recover a Kubernetes cluster.
This chapter bridges the gap between knowing Kubernetes and running it the way
elite engineering teams do — at companies like Spotify, Airbnb, and Shopify.

```
╔══════════════════════════════════════════════════════════════════════╗
║               THE PRODUCTION KUBERNETES MATURITY CURVE              ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  LEVEL 1: "It works"                                                ║
║  → kubectl apply manually. Single control plane. No backups.        ║
║                                                                      ║
║  LEVEL 2: "It's stable"                                             ║
║  → HA control plane. etcd backups. Resource limits. Probes.         ║
║                                                                      ║
║  LEVEL 3: "It's managed"                                            ║
║  → Helm for packaging. Prometheus + Grafana. Node autoscaler.       ║
║                                                                      ║
║  LEVEL 4: "It deploys itself"    ← This chapter                     ║
║  → GitOps with ArgoCD. Everything as code. Drift detection.         ║
║    Automated rollback. Full audit trail.                             ║
║                                                                      ║
║  LEVEL 5: "It runs itself"                                           ║
║  → VPA + HPA + Cluster Autoscaler. Chaos engineering. SLOs.         ║
║    Multi-cluster with federation.                                    ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 11.1 High Availability — Never a Single Point of Failure

### 11.1.1 HA Architecture Overview

```
╔══════════════════════════════════════════════════════════════════════╗
║              PRODUCTION HA KUBERNETES ARCHITECTURE                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  INTERNET / USERS                                                    ║
║       │                                                              ║
║       ▼                                                              ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │  CDN / WAF (Cloudflare, AWS CloudFront)                        │ ║
║  └──────────────────────────────┬─────────────────────────────────┘ ║
║                                 │                                    ║
║  ┌──────────────────────────────▼─────────────────────────────────┐ ║
║  │  External Load Balancer (AWS ALB / GCP GLB / NLB)              │ ║
║  │  Routes to: Ingress Controllers (in multiple AZs)              │ ║
║  └──────┬──────────────────────────────┬──────────────────────────┘ ║
║         │                              │                             ║
║  ┌──────▼──────┐                ┌──────▼──────┐                     ║
║  │  Ingress-1  │                │  Ingress-2  │  (DaemonSet across  ║
║  │  (AZ-a)     │                │  (AZ-b)     │   dedicated nodes)  ║
║  └──────┬──────┘                └──────┬──────┘                     ║
║         │                              │                             ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │          KUBERNETES WORKER NODES (Multi-AZ)                  │   ║
║  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐  │   ║
║  │  │ Worker-1  │  │ Worker-2  │  │ Worker-3  │  │ Worker-4 │  │   ║
║  │  │  (AZ-a)   │  │  (AZ-b)   │  │  (AZ-a)   │  │ (AZ-b)   │  │   ║
║  │  │ app pods  │  │ app pods  │  │ db replicas│  │db replica│  │   ║
║  │  └───────────┘  └───────────┘  └───────────┘  └──────────┘  │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │          CONTROL PLANE (3 nodes across 3 AZs)                │   ║
║  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │   ║
║  │  │  CP-1 (AZ-a) │  │  CP-2 (AZ-b) │  │  CP-3 (AZ-c) │        │   ║
║  │  │  apiserver   │  │  apiserver   │  │  apiserver   │        │   ║
║  │  │  etcd        │◄─►  etcd        │◄─►  etcd        │        │   ║
║  │  │  scheduler   │  │  scheduler   │  │  scheduler   │        │   ║
║  │  └──────────────┘  └──────────────┘  └──────────────┘        │   ║
║  │                  Load Balancer (VIP): 10.0.0.100:6443          │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 11.1.2 PodDisruptionBudgets — Protecting Availability

```yaml
# pod-disruption-budget.yaml
# Ensure at least 2 replicas of payment-service are always available
# during voluntary disruptions (node drains, rolling updates)

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: production
spec:
  # OPTION A: Minimum available (absolute or percentage)
  minAvailable: 2          # At least 2 pods must be running at all times
  # minAvailable: "80%"    # Or as percentage

  # OPTION B: Maximum unavailable (use one OR the other, not both)
  # maxUnavailable: 1      # At most 1 pod can be unavailable

  selector:
    matchLabels:
      app: payment-service

---
# Check PDB status
# kubectl get pdb -n production
# NAME                  MIN AVAILABLE  MAX UNAVAILABLE  ALLOWED DISRUPTIONS  AGE
# payment-service-pdb   2              N/A              1                    5d
# ALLOWED DISRUPTIONS = current replicas - minAvailable
# If 0 → drain will block until more replicas are added
```

### 11.1.3 Vertical Pod Autoscaler (VPA)

VPA automatically adjusts CPU/memory requests based on historical usage:

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"         # Auto | Recreate | Initial | Off
    # Auto: Updates running pods (may restart them)
    # Initial: Only sets requests on new pods
    # Off: Recommendations only — no automatic changes (safe for production)
  resourcePolicy:
    containerPolicies:
    - containerName: web
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 4Gi
      controlledResources: ["cpu", "memory"]
```

### 11.1.4 Cluster Autoscaler

```yaml
# cluster-autoscaler-aws.yaml
# Automatically adds/removes worker nodes based on pending pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.29.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste       # least-waste | random | most-pods
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
        - --scale-down-utilization-threshold=0.5
        resources:
          requests:
            cpu: 100m
            memory: 300Mi
          limits:
            cpu: 100m
            memory: 300Mi
```

---

## 11.2 Helm — The Kubernetes Package Manager

### 11.2.1 What Is Helm?

#### In Plain English

Helm is like the **App Store for Kubernetes**. Instead of managing 15 separate
YAML files for a complex application like PostgreSQL, you install a single
**Helm chart** that packages everything together. You can customize it with
a values file, upgrade it with one command, and roll it back just as easily.

#### In Technical Language

**Helm** is the package manager for Kubernetes. A **chart** is a collection of
files describing a related set of Kubernetes resources. A **release** is an
instance of a chart running in a cluster. Helm manages the full lifecycle:
install, upgrade, rollback, and uninstall.

```
╔══════════════════════════════════════════════════════════════════════╗
║                     HELM ARCHITECTURE                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  CHART STRUCTURE:                                                    ║
║  my-app/                                                             ║
║  ├── Chart.yaml          # Chart metadata (name, version, description)
║  ├── values.yaml         # Default configuration values              ║
║  ├── templates/          # Kubernetes YAML templates                 ║
║  │   ├── deployment.yaml # Uses {{ .Values.xxx }} syntax             ║
║  │   ├── service.yaml                                                ║
║  │   ├── ingress.yaml                                                ║
║  │   ├── configmap.yaml                                              ║
║  │   ├── _helpers.tpl    # Reusable template snippets                ║
║  │   └── NOTES.txt       # Post-install instructions                 ║
║  ├── charts/             # Dependent charts (subcharts)              ║
║  └── .helmignore                                                     ║
║                                                                      ║
║  RELEASE LIFECYCLE:                                                  ║
║  helm install → helm upgrade → helm rollback → helm uninstall       ║
║                                                                      ║
║  HOW TEMPLATING WORKS:                                               ║
║  values.yaml:           templates/deployment.yaml:                  ║
║  replicaCount: 3   →    replicas: {{ .Values.replicaCount }}        ║
║  image: nginx:1.21 →    image: {{ .Values.image }}                  ║
║  Override: helm install --set replicaCount=5                        ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 11.2.2 Helm Commands

```bash
# ── REPOSITORY MANAGEMENT ────────────────────────────────────────────
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update                        # Refresh repo index
helm repo list                          # Show configured repos
helm search repo nginx                  # Search for nginx charts
helm search repo bitnami/postgres       # Search specific chart
helm show values bitnami/postgresql     # Show all configurable values

# ── INSTALLING CHARTS ────────────────────────────────────────────────
# Basic install
helm install my-nginx bitnami/nginx
# install: helm install <RELEASE_NAME> <CHART>

# Install with custom values
helm install my-postgres bitnami/postgresql \
  --namespace database \
  --create-namespace \
  --set auth.postgresPassword=mypassword \
  --set primary.persistence.size=20Gi \
  --set metrics.enabled=true

# Install with values file
helm install my-postgres bitnami/postgresql \
  --namespace database \
  -f my-postgres-values.yaml

# Dry run (preview without installing)
helm install my-app ./my-chart --dry-run --debug

# ── MANAGING RELEASES ─────────────────────────────────────────────────
helm list                               # List releases in current namespace
helm list -A                            # All namespaces
helm list --deployed                    # Only deployed
helm status my-postgres                 # Detailed status
helm get values my-postgres             # Current values
helm get manifest my-postgres           # Rendered YAML

# ── UPGRADES AND ROLLBACKS ───────────────────────────────────────────
# Upgrade release
helm upgrade my-postgres bitnami/postgresql \
  --set auth.postgresPassword=newpassword \
  --reuse-values              # Keep existing values, only override specified ones

# Upgrade or install (idempotent)
helm upgrade --install my-postgres bitnami/postgresql \
  -f values.yaml \
  --namespace database \
  --create-namespace

# View upgrade history
helm history my-postgres
# REVISION  STATUS     DESCRIPTION
# 1         superseded Initial install
# 2         deployed   Upgrade completed

# Rollback to previous version
helm rollback my-postgres 1            # Roll back to revision 1
helm rollback my-postgres              # Roll back to previous revision

# ── UNINSTALL ────────────────────────────────────────────────────────
helm uninstall my-postgres -n database
helm uninstall my-postgres --keep-history   # Keep release history

# ── CREATE YOUR OWN CHART ────────────────────────────────────────────
helm create my-app                     # Scaffold a new chart
cd my-app
# Edit Chart.yaml, values.yaml, templates/
helm lint my-app/                      # Validate chart
helm package my-app/                   # Create .tgz archive
helm install test-release ./my-app-0.1.0.tgz
```

### 11.2.3 Creating a Production Helm Chart

```yaml
# my-app/Chart.yaml
apiVersion: v2
name: my-app
description: Production web application chart
type: application
version: 1.2.0           # Chart version
appVersion: "2.1.0"      # Application version

dependencies:
- name: postgresql
  version: "12.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled   # Only install if enabled in values

---
# my-app/values.yaml
replicaCount: 3

image:
  repository: myregistry/my-app
  tag: "2.1.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  host: myapp.example.com
  tls: true

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true
  auth:
    postgresPassword: ""          # Set via --set or sealed secret
    database: myapp

env:
  LOG_LEVEL: INFO
  CACHE_TTL: "300"

---
# my-app/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 80
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        env:
        {{- range $key, $val := .Values.env }}
        - name: {{ $key }}
          value: {{ $val | quote }}
        {{- end }}
        - name: DB_HOST
          value: {{ printf "%s-postgresql" (include "my-app.fullname" .) }}

---
# my-app/templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-app.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

---

## 11.3 Kustomize — Template-Free Configuration Management

### 11.3.1 What Is Kustomize?

Kustomize is built into `kubectl` and provides a template-free way to
customize application configuration. Instead of templates, it uses **patches**
and **overlays** layered on top of a base configuration.

```
╔══════════════════════════════════════════════════════════════════════╗
║              KUSTOMIZE — BASE + OVERLAYS PATTERN                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  k8s/                                                                ║
║  ├── base/                  ← Shared config for ALL environments    ║
║  │   ├── kustomization.yaml                                          ║
║  │   ├── deployment.yaml   (replicas: 1, image: nginx)              ║
║  │   ├── service.yaml                                                ║
║  │   └── configmap.yaml                                              ║
║  │                                                                   ║
║  └── overlays/                                                       ║
║      ├── dev/              ← Dev-specific changes                    ║
║      │   ├── kustomization.yaml                                      ║
║      │   └── patch-replicas.yaml  (replicas: 1)                     ║
║      │                                                               ║
║      ├── staging/          ← Staging-specific                        ║
║      │   ├── kustomization.yaml                                      ║
║      │   └── patch-replicas.yaml  (replicas: 2)                     ║
║      │                                                               ║
║      └── production/       ← Production-specific                     ║
║          ├── kustomization.yaml                                      ║
║          ├── patch-replicas.yaml  (replicas: 10)                    ║
║          └── patch-resources.yaml (higher CPU/memory limits)        ║
║                                                                      ║
║  kubectl apply -k overlays/production/   ← Deploy to production     ║
╚══════════════════════════════════════════════════════════════════════╝
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
- configmap.yaml
commonLabels:
  app: my-app
  managed-by: kustomize

---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
namespace: production
namePrefix: prod-            # Prefix all resource names
images:
- name: nginx
  newTag: "1.25-alpine"     # Override image tag
patches:
- path: patch-replicas.yaml
  target:
    kind: Deployment
    name: my-app
- patch: |-                 # Inline patch
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: 1Gi
  target:
    kind: Deployment

---
# overlays/production/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 10
```

```bash
# Preview what will be applied
kubectl kustomize overlays/production/

# Apply kustomized config
kubectl apply -k overlays/production/
kubectl apply -k overlays/dev/

# Diff against current state
kubectl diff -k overlays/production/
```

---

## 11.4 GitOps — Infrastructure as Code at Scale

### 11.4.1 What Is GitOps?

#### In Plain English

GitOps means: **Git is the single source of truth for your cluster**.
Every change to your infrastructure must go through a Git commit.
A GitOps tool watches the repository and automatically applies changes
to the cluster — like a robot that constantly reconciles "what Git says"
with "what the cluster is actually running."

#### In Technical Language

**GitOps** is an operational framework that applies DevOps best practices
(version control, collaboration, CI/CD) to infrastructure automation.
The Git repository is the source of truth; any drift between Git and the
cluster is automatically detected and corrected.

```
╔══════════════════════════════════════════════════════════════════════╗
║              GITOPS PRINCIPLES AND FLOW                              ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  4 PRINCIPLES OF GITOPS:                                            ║
║  1. Declarative  — System state described as code                   ║
║  2. Versioned    — State stored in Git with history                 ║
║  3. Immutable    — Pull requests, not manual kubectl apply          ║
║  4. Continuous   — Automated reconciliation of actual vs desired    ║
║                                                                      ║
║  GITOPS DEPLOYMENT FLOW:                                             ║
║                                                                      ║
║  Developer                                                           ║
║     │  git commit + push                                             ║
║     │  "Update payment-service image to v2.1.1"                     ║
║     ▼                                                                ║
║  GitHub / GitLab                                                     ║
║     │  Pull Request → Code Review → Merge to main                   ║
║     ▼                                                                ║
║  CI Pipeline (GitHub Actions / GitLab CI)                            ║
║     │  Build image → Test → Push to registry                        ║
║     │  Update image tag in Git repo                                 ║
║     ▼                                                                ║
║  ArgoCD / Flux (watches Git repo)                                    ║
║     │  Detects new commit in repo                                   ║
║     │  Compares Git state vs cluster state                          ║
║     │  Applies diff → deploys new version                           ║
║     ▼                                                                ║
║  Kubernetes Cluster                                                  ║
║     └── New pods rolling out automatically ✅                       ║
║                                                                      ║
║  BENEFITS vs TRADITIONAL kubectl apply:                              ║
║  ✅ Full audit trail (who changed what and when → Git history)      ║
║  ✅ Automatic drift correction (someone manually changes cluster?   ║
║     ArgoCD reverts it to match Git)                                  ║
║  ✅ Easy rollback (revert the Git commit)                            ║
║  ✅ No cluster credentials needed by developers                     ║
║  ✅ Disaster recovery: recreate entire cluster from Git repo        ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 11.5 ArgoCD — GitOps Continuous Delivery

### 11.5.1 What Is ArgoCD?

**ArgoCD** is the most popular GitOps tool for Kubernetes. It runs inside
your cluster, continuously monitors a Git repository, and reconciles
the cluster state to match what is declared in Git.

```
╔══════════════════════════════════════════════════════════════════════╗
║                    ARGOCD ARCHITECTURE                               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Git Repository (GitHub/GitLab)                                     ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │  apps/                                                         │ ║
║  │  ├── payment-service/   ← Helm chart or Kustomize overlay     │ ║
║  │  ├── user-service/                                             │ ║
║  │  └── notification-service/                                     │ ║
║  └──────────────────────────────┬─────────────────────────────────┘ ║
║                                 │ Watches every 3 minutes (or webhook)
║  ┌──────────────────────────────▼─────────────────────────────────┐ ║
║  │  ArgoCD (running inside cluster)                               │ ║
║  │                                                                │ ║
║  │  ┌──────────────────┐  ┌──────────────────┐                   │ ║
║  │  │  ArgoCD Server   │  │  Repo Server     │                   │ ║
║  │  │  (API + UI)      │  │  (Clone + render)│                   │ ║
║  │  └──────────────────┘  └──────────────────┘                   │ ║
║  │  ┌──────────────────────────────────────────┐                 │ ║
║  │  │  Application Controller                  │                 │ ║
║  │  │  Compares: Git state vs Cluster state    │                 │ ║
║  │  │  Syncs if: OutOfSync detected            │                 │ ║
║  │  └──────────────────────────────────────────┘                 │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                              │                                       ║
║                    kubectl apply (internal)                          ║
║                              │                                       ║
║  Kubernetes Cluster: Deployment, Service, Ingress deployed ✅       ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 11.5.2 Installing ArgoCD

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s

kubectl get pods -n argocd

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# Open https://localhost:8080
# Username: admin
# Password: (from above command)

# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/local/bin/argocd

# Login via CLI
argocd login localhost:8080 --username admin \
  --password <password> --insecure

# Change admin password
argocd account update-password
```

### 11.5.3 ArgoCD Application — Deploying from Git

```yaml
# argocd-application.yaml
# Tells ArgoCD: "Watch THIS Git repo and keep THIS cluster in sync"

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd             # ArgoCD itself lives in argocd namespace
  finalizers:
  - resources-finalizer.argocd.argoproj.io  # Clean up cluster resources when deleted
spec:
  project: default              # ArgoCD project (for RBAC grouping)

  # SOURCE: Where the desired state lives
  source:
    repoURL: https://github.com/myorg/k8s-config.git
    targetRevision: main        # Git branch, tag, or commit SHA
    path: apps/payment-service  # Path within the repo

    # If using Helm:
    # helm:
    #   valueFiles:
    #   - values-production.yaml
    #   parameters:
    #   - name: image.tag
    #     value: v2.1.1

    # If using Kustomize:
    # kustomize:
    #   namePrefix: prod-

  # DESTINATION: Where to deploy
  destination:
    server: https://kubernetes.default.svc  # In-cluster
    namespace: production

  # SYNC POLICY: How to handle differences
  syncPolicy:
    automated:
      prune: true               # Delete resources removed from Git
      selfHeal: true            # Revert manual cluster changes
      allowEmpty: false         # Don't delete everything if Git is empty
    syncOptions:
    - CreateNamespace=true      # Create namespace if missing
    - PrunePropagationPolicy=foreground
    - ApplyOutOfSyncOnly=true   # Only apply changed resources
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

---
# APP OF APPS PATTERN — Manage multiple apps from one ArgoCD Application
# This is how large teams manage 100s of microservices
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-apps
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/k8s-config.git
    targetRevision: main
    path: argocd/apps           # This directory contains more Application manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 11.5.4 ArgoCD CLI Commands

```bash
# ── APPLICATION MANAGEMENT ────────────────────────────────────────────
argocd app list                        # List all applications
argocd app get payment-service         # Application status
argocd app sync payment-service        # Manually trigger sync
argocd app diff payment-service        # Show what would change
argocd app history payment-service     # Deployment history

# Rollback to previous version
argocd app rollback payment-service 3  # Roll back to revision 3

# ── STATUS CHECKING ───────────────────────────────────────────────────
# STATUS options:
# Synced     → Git and cluster are identical
# OutOfSync  → Git differs from cluster (needs sync)
# Unknown    → Cannot determine status

# HEALTH options:
# Healthy    → All resources are healthy
# Progressing → Resources are being updated
# Degraded   → Some resources are unhealthy
# Missing    → Resources don't exist in cluster

# Force sync (even if already synced)
argocd app sync payment-service --force

# Sync only specific resources
argocd app sync payment-service \
  --resource apps:Deployment:payment-service

# ── KUBECTL-STYLE ACCESS ──────────────────────────────────────────────
# ArgoCD also responds to kubectl
kubectl get applications -n argocd
kubectl describe application payment-service -n argocd
```

---

## 11.6 Production Best Practices

### 11.6.1 The Production Readiness Checklist

```
╔══════════════════════════════════════════════════════════════════════╗
║              PRODUCTION READINESS CHECKLIST                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  AVAILABILITY:                                                       ║
║  ✅ Minimum 2 replicas for all stateless services                   ║
║  ✅ Pod Anti-Affinity: spread replicas across nodes/AZs             ║
║  ✅ TopologySpreadConstraints for zone distribution                 ║
║  ✅ PodDisruptionBudget for critical services                       ║
║  ✅ Rolling update strategy with maxUnavailable: 0                  ║
║  ✅ preStop hook + terminationGracePeriodSeconds for graceful stop  ║
║                                                                      ║
║  HEALTH:                                                             ║
║  ✅ Readiness probe on every container                              ║
║  ✅ Liveness probe on every container                               ║
║  ✅ Startup probe for slow-starting apps                            ║
║                                                                      ║
║  RESOURCES:                                                          ║
║  ✅ CPU and memory requests set on every container                  ║
║  ✅ CPU and memory limits set on every container                    ║
║  ✅ ResourceQuota per namespace                                      ║
║  ✅ LimitRange defaults per namespace                               ║
║                                                                      ║
║  SECURITY:                                                           ║
║  ✅ Dedicated ServiceAccount per application                        ║
║  ✅ RBAC with least privilege                                       ║
║  ✅ runAsNonRoot: true                                               ║
║  ✅ allowPrivilegeEscalation: false                                 ║
║  ✅ readOnlyRootFilesystem: true                                    ║
║  ✅ capabilities: drop: [ALL]                                       ║
║  ✅ Pod Security Standards: baseline or restricted                  ║
║  ✅ Network Policies: default-deny + explicit allow                 ║
║  ✅ Secrets from external manager (Vault/AWS SM), not kubectl       ║
║                                                                      ║
║  OBSERVABILITY:                                                      ║
║  ✅ Prometheus metrics endpoint exposed                             ║
║  ✅ ServiceMonitor for Prometheus scraping                          ║
║  ✅ Structured JSON logs to stdout                                  ║
║  ✅ PrometheusRule: alerts for pod crashes, high latency, errors    ║
║  ✅ Grafana dashboard for service                                   ║
║                                                                      ║
║  OPERATIONS:                                                         ║
║  ✅ All config in Git (GitOps)                                      ║
║  ✅ Helm or Kustomize for templating                                ║
║  ✅ etcd backups automated and tested                               ║
║  ✅ Cluster upgrade plan documented                                 ║
║  ✅ Runbooks for common failure scenarios                           ║
║  ✅ Certificate expiry monitoring                                   ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 11.6.2 The Perfect Production Pod Manifest

```yaml
# production-perfect-pod.yaml
# The gold standard pod spec incorporating all best practices
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
  labels:
    app.kubernetes.io/name: payment-service
    app.kubernetes.io/version: "2.1.1"
    app.kubernetes.io/part-of: payments-platform
  annotations:
    kubernetes.io/change-cause: "Update to v2.1.1 — fix race condition CVE-2024-001"
spec:
  replicas: 3

  selector:
    matchLabels:
      app: payment-service

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0            # Zero-downtime deployment

  revisionHistoryLimit: 5

  template:
    metadata:
      labels:
        app: payment-service
        version: "2.1.1"
        env: production
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"

    spec:
      # Identity
      serviceAccountName: payment-service-sa
      automountServiceAccountToken: false   # Don't need API access

      # Termination
      terminationGracePeriodSeconds: 60

      # Security at pod level
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault

      # Spread across nodes and AZs
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: payment-service
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: payment-service

      # Only run on production nodes
      tolerations:
      - key: environment
        value: production
        effect: NoSchedule
        operator: Equal
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: environment
                operator: In
                values: [production]

      containers:
      - name: payment-service
        image: myregistry/payment-service:2.1.1
        imagePullPolicy: IfNotPresent

        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090

        # Config from external sources
        envFrom:
        - configMapRef:
            name: payment-service-config
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace

        # Resources — Guaranteed QoS
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"

        # Health checks
        startupProbe:
          httpGet:
            path: /startup
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 20
          failureThreshold: 3

        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]

        # Container-level security
        securityContext:
          allowPrivilegeEscalation: false
          privileged: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: [ALL]

        # Writable temp space
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: app-logs
          mountPath: /app/logs

      volumes:
      - name: tmp
        emptyDir: {}
      - name: app-logs
        emptyDir: {}
```

---

## Chapter 11: Hands-On Labs

### Lab 11.1 — Deploy with Helm

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# Add repos
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Deploy nginx with Helm
helm install my-nginx bitnami/nginx \
  --set service.type=NodePort \
  --set replicaCount=2

helm list
helm status my-nginx
kubectl get all -l app.kubernetes.io/instance=my-nginx

# Upgrade: change replica count
helm upgrade my-nginx bitnami/nginx \
  --set replicaCount=3 \
  --reuse-values

helm history my-nginx

# Rollback
helm rollback my-nginx 1
kubectl get pods -l app.kubernetes.io/instance=my-nginx

# Create your own chart
helm create my-app
ls my-app/
helm lint my-app/
helm install test-my-app ./my-app --dry-run --debug
helm install test-my-app ./my-app

# Uninstall
helm uninstall my-nginx test-my-app
```

### Lab 11.2 — Kustomize Overlays

```bash
# Create directory structure
mkdir -p k8s/base k8s/overlays/dev k8s/overlays/prod

# Base deployment
cat > k8s/base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:alpine
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
EOF

cat > k8s/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
EOF

# Dev overlay
cat > k8s/overlays/dev/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
namespace: dev
namePrefix: dev-
EOF

# Prod overlay with patch
cat > k8s/overlays/prod/patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5
EOF

cat > k8s/overlays/prod/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
namespace: prod
namePrefix: prod-
images:
- name: nginx
  newTag: "1.25"
patches:
- path: patch.yaml
EOF

# Preview
kubectl kustomize k8s/overlays/dev/    # Dev config
kubectl kustomize k8s/overlays/prod/   # Prod config — 5 replicas, new image

# Apply
kubectl create namespace dev prod
kubectl apply -k k8s/overlays/dev/
kubectl apply -k k8s/overlays/prod/
kubectl get deploy -A | grep -E "dev|prod"
```

### Lab 11.3 — ArgoCD Setup and First Application

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s

# Get password
ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)
echo "Password: $ARGOCD_PWD"

# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Install CLI and login
curl -sSL -o /tmp/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /tmp/argocd && sudo mv /tmp/argocd /usr/local/bin/argocd

argocd login localhost:8080 --username admin \
  --password "$ARGOCD_PWD" --insecure

# Create an Application from the official ArgoCD example
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook \
  --sync-policy automated \
  --auto-prune

# Check status
argocd app get guestbook
argocd app list

# Watch sync
kubectl get pods -n guestbook -w

# Trigger manual sync
argocd app sync guestbook

# View diff (OutOfSync state)
argocd app diff guestbook

# Open UI: https://localhost:8080
```

---

## Chapter 11: Troubleshooting Guide

### Issue 1: Helm install fails — resource already exists

```bash
helm install my-app ./my-chart
# Error: INSTALLATION FAILED: rendered manifests contain a resource
# that already exists. Use --debug flag to see details.

# CAUSE: Resource was created outside Helm (kubectl apply)
# Fix 1: Delete the conflicting resource
kubectl delete deployment my-app

# Fix 2: Adopt existing resources into Helm release
helm install my-app ./my-chart \
  --set-string "metadata.annotations.meta\.helm\.sh/release-name=my-app"

# Fix 3: Use upgrade --install (handles both create and update)
helm upgrade --install my-app ./my-chart
```

### Issue 2: ArgoCD shows OutOfSync but sync doesn't fix it

```bash
argocd app get my-app     # Shows OutOfSync

# CAUSE: Sync waves or hook resource ordering issue
argocd app sync my-app --force

# CAUSE: Annotation diff (metadata that ArgoCD doesn't manage)
# Check what's different:
argocd app diff my-app

# CAUSE: Resource has server-side defaults not in Git
# Fix: Add --server-side to sync
argocd app sync my-app --server-side

# Check ArgoCD controller logs
kubectl logs -n argocd \
  -l app.kubernetes.io/name=argocd-application-controller -f
```

### Issue 3: Kustomize patch not applying

```bash
kubectl apply -k overlays/production/
# Error: invalid value for field: spec.replicas (expected integer)

# Verify kustomization.yaml syntax
kubectl kustomize overlays/production/ | kubectl apply --dry-run=client -f -

# Check patch target matches resource exactly
# The name in patch MUST match the name in base (case-sensitive)
grep "name:" base/deployment.yaml
grep "name:" overlays/production/patch.yaml

# Verify base resources are referenced
grep "bases\|resources" overlays/production/kustomization.yaml
```

---

## Chapter 11: Interview Questions

**Q1: What is Helm and what problem does it solve?**

> Helm is the package manager for Kubernetes. It solves the problem of managing multiple related Kubernetes manifests for a complex application. Instead of maintaining 15 separate YAML files for something like PostgreSQL, you use a Helm chart that packages everything with configurable values. Helm tracks what's installed (releases), enables upgrades with `helm upgrade`, and supports rollbacks with `helm rollback`. It also enables chart reuse across teams — you can deploy the same application in dev, staging, and prod with different values files.

**Q2: What is GitOps and how does ArgoCD implement it?**

> GitOps is an operational model where Git is the single source of truth for cluster state. Every change goes through a Git commit, providing audit trail and easy rollback. ArgoCD implements GitOps by continuously watching a Git repository and comparing its contents (desired state) with the cluster (actual state). When it detects drift — either from a new Git commit or from manual kubectl changes — it automatically syncs the cluster back to Git. With `selfHeal: true`, manual changes to the cluster are immediately reverted.

**Q3: What is the difference between Helm and Kustomize?**

> Helm uses a Go templating engine — you write YAML templates with `{{ .Values.xxx }}` placeholders and supply values files. It tracks releases, supports rollbacks, and has a large ecosystem of pre-built charts. Kustomize uses a patching approach — you have a base configuration and apply patches/overlays for different environments, without templates. Kustomize is built into kubectl (`kubectl apply -k`) requiring no extra tool. For large ecosystems of existing charts, use Helm. For customising base configs per environment without templates, use Kustomize. Many teams use both together.

**Q4: What is a PodDisruptionBudget and why is it critical for production?**

> A PodDisruptionBudget (PDB) limits how many pods of a deployment can be voluntarily disrupted simultaneously — during node drains, rolling updates, or cluster upgrades. `minAvailable: 2` ensures at least 2 pods are always running. Without a PDB, a `kubectl drain` could take down all replicas of a service at once, causing an outage. PDBs are especially critical for stateful services and any service with SLA requirements. They work in cooperation with `kubectl drain` — if draining would violate the PDB, the drain waits.

**Q5: What is the App of Apps pattern in ArgoCD?**

> The App of Apps pattern is a way to manage many ArgoCD Applications using a single root Application. You create one ArgoCD Application that points to a directory containing multiple other Application manifests. ArgoCD deploys all those Application manifests, which in turn deploy the actual services. This enables bootstrapping an entire cluster from a single Git commit — you only need to create one root Application manually, and everything else deploys automatically. It is the standard pattern for managing large clusters with many microservices.

**Q6: What is the Cluster Autoscaler and how does it work?**

> The Cluster Autoscaler automatically adjusts the number of worker nodes in a cluster. When pods are in Pending state due to insufficient resources, it adds nodes. When nodes are consistently underutilized (below 50% for 10+ minutes), it removes them. It works with cloud provider APIs (AWS Auto Scaling Groups, GCP Instance Groups, Azure VMSS) to provision or terminate VMs. It is separate from HPA — HPA scales pods, Cluster Autoscaler scales nodes. Together they provide full elasticity: HPA scales pods first, then Cluster Autoscaler adds nodes if needed.

---

## CKA Exam Notes — Chapter 11

```
╔════════════════════════════════════════════════════════════════════╗
║                  CKA EXAM FOCUS — CHAPTER 11                      ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: ~5% direct, but practices appear throughout          ║
║  Helm and Kustomize are used in advanced exam scenarios            ║
║                                                                    ║
║  HELM EXAM COMMANDS:                                               ║
║  helm repo add <name> <url>                                        ║
║  helm repo update                                                  ║
║  helm search repo <chart>                                          ║
║  helm install <release> <chart> --namespace <ns>                  ║
║  helm upgrade <release> <chart> --reuse-values                    ║
║  helm upgrade --install <release> <chart>   ← idempotent          ║
║  helm rollback <release> <revision>                               ║
║  helm list -A                                                      ║
║  helm uninstall <release>                                          ║
║                                                                    ║
║  KUSTOMIZE EXAM COMMANDS:                                          ║
║  kubectl apply -k <directory>                                      ║
║  kubectl kustomize <directory>   ← preview output                 ║
║  kubectl diff -k <directory>                                       ║
║                                                                    ║
║  PDB YAML SKELETON:                                                ║
║  apiVersion: policy/v1                                             ║
║  kind: PodDisruptionBudget                                         ║
║  metadata:                                                         ║
║    name: my-pdb                                                    ║
║  spec:                                                             ║
║    minAvailable: 2                                                 ║
║    selector:                                                       ║
║      matchLabels:                                                  ║
║        app: my-app                                                 ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Chapter 11 Summary

1. **HA Architecture** — multi-AZ control plane (3 nodes), multi-AZ workers, external LB for API server
2. **PodDisruptionBudget** — protects service availability during drains and updates
3. **VPA** — auto-adjusts resource requests based on historical usage
4. **Cluster Autoscaler** — auto-scales nodes based on pending pods and utilisation
5. **Helm** — Kubernetes package manager; charts, releases, values, upgrade, rollback
6. **Helm commands** — install, upgrade, rollback, list, history, uninstall
7. **Kustomize** — template-free config management with base + overlays
8. **GitOps principles** — declarative, versioned, immutable, continuously reconciled
9. **ArgoCD** — watches Git, detects drift, auto-syncs; self-healing clusters
10. **ArgoCD Application** — source (Git) + destination (cluster) + syncPolicy
11. **App of Apps** — bootstrap entire cluster from single root Application
12. **Production checklist** — availability, health, resources, security, observability, operations

---

*Next: Chapter 12 — CKA Preparation: Exam Strategy, Speed Labs & Mock Exams*

*"The production cluster is built. Now it's time to prove you can administer one —
 under exam pressure, with a timer running. Chapter 12 gets you across the finish line."*
