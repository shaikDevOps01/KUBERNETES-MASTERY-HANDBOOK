# KUBERNETES MASTERY HANDBOOK
# Part 12: CKA Preparation
# Chapter 12: Exam Strategy, Speed Labs, Troubleshooting Scenarios & Mock Exams

---

> **"The CKA is not a test of memory.
>  It is a test of whether you can solve real Kubernetes problems
>  under time pressure. Speed and accuracy together. Nothing else."**

---

## Chapter Introduction

This is it. The final chapter. Everything in this handbook has been building
to this moment — your CKA certification.

The CKA is unlike any exam you have taken before. There are no multiple choice
questions. No memorising definitions. You sit at a remote desktop with a live
Kubernetes cluster and real problems to solve, against a 2-hour clock.

This chapter gives you exactly what you need to pass:

```
╔══════════════════════════════════════════════════════════════════════╗
║                    CKA EXAM AT A GLANCE                              ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  FORMAT:      Performance-based, hands-on, online proctored         ║
║  DURATION:    2 hours                                                ║
║  QUESTIONS:   15–20 tasks                                            ║
║  PASS SCORE:  66%                                                    ║
║  ENVIRONMENT: Real Kubernetes clusters (multiple contexts)          ║
║  OPEN BOOK:   Yes — kubernetes.io/docs allowed                      ║
║  VALIDITY:    3 years                                                ║
║  COST:        $395 USD (includes 1 free retake)                     ║
║  PROVIDER:    Linux Foundation + PSI                                ║
║                                                                      ║
║  EXAM DOMAIN WEIGHTS:                                                ║
║  ┌────────────────────────────────────────────────────────────┐     ║
║  │  Troubleshooting                         30%               │     ║
║  │  Cluster Architecture, Installation, Config  25%          │     ║
║  │  Services & Networking                   20%               │     ║
║  │  Workloads & Scheduling                  15%               │     ║
║  │  Storage                                 10%               │     ║
║  └────────────────────────────────────────────────────────────┘     ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 12.1 Exam Strategy — How to Approach the CKA

### 12.1.1 The First 3 Minutes

```
╔══════════════════════════════════════════════════════════════════════╗
║            THE FIRST 3 MINUTES — DO THIS BEFORE ANYTHING            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. Set up your terminal aliases (10 seconds)                        ║
║  ─────────────────────────────────────────────                       ║
║  alias k=kubectl                                                     ║
║  export do="--dry-run=client -o yaml"                               ║
║  export now="--force --grace-period=0"                              ║
║  source <(kubectl completion bash)                                   ║
║  complete -F __start_kubectl k                                       ║
║                                                                      ║
║  2. Enable vim syntax highlighting (optional)                        ║
║  ─────────────────────────────────────────────                       ║
║  echo "set nu" >> ~/.vimrc                                           ║
║  echo "set et" >> ~/.vimrc    # expandtab                           ║
║  echo "set ts=2" >> ~/.vimrc  # tabstop=2                           ║
║  echo "set sw=2" >> ~/.vimrc  # shiftwidth=2                        ║
║  echo "set sts=2" >> ~/.vimrc # softtabstop=2                       ║
║                                                                      ║
║  3. Note the exam clusters available                                 ║
║  ─────────────────────────────────────────────                       ║
║  kubectl config get-contexts                                         ║
║  # The exam has 6 clusters. Each question tells you which to use.   ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 12.1.2 Question Handling Strategy

```
╔══════════════════════════════════════════════════════════════════════╗
║              QUESTION HANDLING — THE 6-STEP METHOD                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  STEP 1: Read the question TWICE                                     ║
║  Understand exactly what is asked. Miss one word = wrong answer.    ║
║  Look for: namespace, resource name, specific values, constraints.  ║
║                                                                      ║
║  STEP 2: CHECK THE CONTEXT                                           ║
║  kubectl config use-context <context-from-question>                 ║
║  This is the #1 cause of lost marks. NEVER skip this.               ║
║                                                                      ║
║  STEP 3: Note the weight (%)                                         ║
║  High % question: attempt carefully, budget more time.              ║
║  Low % question: if stuck after 3 min, flag and come back.          ║
║                                                                      ║
║  STEP 4: Solve using imperative commands FIRST                       ║
║  kubectl create/run/expose --dry-run=client -o yaml > file.yaml     ║
║  Edit the file. Apply. Faster than writing YAML from scratch.       ║
║                                                                      ║
║  STEP 5: Verify your solution                                        ║
║  kubectl get/describe — confirm the resource exists as required.    ║
║  Test it if possible (curl, exec, logs).                            ║
║                                                                      ║
║  STEP 6: Move on                                                     ║
║  Don't polish. Don't over-engineer. Done = done.                    ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 12.1.3 Time Management

```
╔══════════════════════════════════════════════════════════════════════╗
║              TIME MANAGEMENT — 120 MINUTES                           ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PASS 1 (0–60 min): Quick wins                                       ║
║  ──────────────────────────────                                      ║
║  Do ALL questions you can solve in under 4 minutes each.            ║
║  Flag anything that seems complex or unfamiliar.                    ║
║  Target: Complete 10–12 of 17 questions.                            ║
║                                                                      ║
║  PASS 2 (60–100 min): Flagged medium questions                       ║
║  ─────────────────────────────────────────────                       ║
║  Return to flagged questions. Use docs if needed.                   ║
║  Spend up to 8 minutes on each.                                     ║
║  Target: Complete 3–4 more questions.                               ║
║                                                                      ║
║  PASS 3 (100–115 min): Hard/stuck questions                          ║
║  ─────────────────────────────────────────────                       ║
║  Attempt remaining questions with whatever time is left.            ║
║  Partial credit: even a partially correct answer scores better      ║
║  than nothing — start the resource even if not perfect.             ║
║                                                                      ║
║  PASS 4 (115–120 min): Review + verify                               ║
║  ─────────────────────────────────────────────                       ║
║  Quick check: kubectl get -n <ns> the key resources from            ║
║  your 3-4 highest-weight questions.                                 ║
║                                                                      ║
║  TIME TARGETS:                                                       ║
║  Easy questions (create pod, scale deploy):  2–3 min each           ║
║  Medium questions (RBAC, NetworkPolicy):     4–6 min each           ║
║  Hard questions (etcd restore, upgrade):     8–12 min each          ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 12.1.4 Using Kubernetes Documentation Effectively

```
╔══════════════════════════════════════════════════════════════════════╗
║           USING DOCS DURING THE EXAM                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ALLOWED SITES:                                                      ║
║  https://kubernetes.io/docs                                         ║
║  https://kubernetes.io/blog                                         ║
║  https://github.com/kubernetes                                      ║
║                                                                      ║
║  BOOKMARK THESE PAGES BEFORE THE EXAM:                              ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │  kubectl Cheat Sheet    k8s.io/docs/reference/kubectl/cheatsheet│ ║
║  │  kubeadm init           k8s.io/docs/reference/setup-tools/kubeadm│ ║
║  │  etcd backup/restore    k8s.io/docs/tasks/administer-cluster/  │ ║
║  │  Cluster upgrade        k8s.io/docs/tasks/administer-cluster/  │ ║
║  │  RBAC                   k8s.io/docs/reference/access-authn-authz│ ║
║  │  Network Policies       k8s.io/docs/concepts/services-networking│ ║
║  │  Storage                k8s.io/docs/concepts/storage/          │ ║
║  │  Ingress                k8s.io/docs/concepts/services-networking│ ║
║  └────────────────────────────────────────────────────────────────┘ ║
║                                                                      ║
║  DOC STRATEGY:                                                       ║
║  Use docs to COPY YAML EXAMPLES, not to read/learn.                 ║
║  Know which page has what — copy, modify, apply.                   ║
║  If you can do it from memory: don't open docs (save time).        ║
║  If you're unsure of YAML structure: open docs, find example,      ║
║  copy to terminal, modify, apply. ~60 seconds.                     ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 12.2 Most Important Topics by Domain

### 12.2.1 Troubleshooting (30% — MOST IMPORTANT)

```
╔══════════════════════════════════════════════════════════════════════╗
║     TROUBLESHOOTING — THE UNIVERSAL DEBUG SEQUENCE                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  For ANY broken resource, always start here:                        ║
║                                                                      ║
║  kubectl config current-context       ← Am I on the right cluster? ║
║  kubectl get <resource> -n <ns>       ← Does it exist?              ║
║  kubectl describe <resource> <name>   ← What do Events say?         ║
║  kubectl logs <pod> --previous        ← What did it say before die? ║
║  kubectl get events --sort-by=.lastTimestamp | tail -20             ║
║                                                                      ║
║  BROKEN NODE:                                                        ║
║  ssh <node>                                                          ║
║  systemctl status kubelet             ← Is kubelet running?          ║
║  journalctl -u kubelet -f             ← Kubelet error messages       ║
║  systemctl start kubelet              ← Start if stopped             ║
║                                                                      ║
║  POD WON'T START:                                                    ║
║  kubectl describe pod <name>          ← Events section               ║
║  Pending   → scheduling issue (resources, affinity, taints)         ║
║  ImagePull → wrong image name or no pull secret                     ║
║  CrashLoop → app error, check logs --previous                       ║
║  Init:0/1  → init container failing                                 ║
║                                                                      ║
║  SERVICE NOT WORKING:                                                ║
║  kubectl get endpoints <svc>          ← Empty? Selector mismatch    ║
║  kubectl get pods --show-labels       ← Do labels match selector?   ║
║  kubectl exec test -- curl <svc>:<port>  ← Can pods reach service?  ║
║                                                                      ║
║  NODE NOT READY:                                                     ║
║  kubectl describe node <name>         ← Conditions section           ║
║  ssh <node>; systemctl status kubelet ← kubelet health              ║
║  journalctl -u kubelet | tail -20     ← Error messages              ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 12.2.2 Top 20 Exam Tasks by Frequency

```
RANK  TASK                              CHAPTER  TIME ESTIMATE
────  ────────────────────────────────  ───────  ─────────────
 1    etcd backup                       Ch 10    3 min
 2    Cluster upgrade (CP + workers)    Ch 10    10 min
 3    Create Pod with specific config   Ch 2     2 min
 4    Create/modify Deployment          Ch 2     2 min
 5    Create Service (ClusterIP/NP)     Ch 3     2 min
 6    Create Ingress rules              Ch 3     4 min
 7    Create RBAC (Role + Binding)      Ch 8     3 min
 8    ServiceAccount + RBAC             Ch 8     4 min
 9    PersistentVolume + PVC            Ch 4     4 min
10    NetworkPolicy                     Ch 3/8   4 min
11    Fix a broken cluster/node         Ch 10    6 min
12    Secrets and ConfigMaps            Ch 5     3 min
13    StatefulSet creation              Ch 6     4 min
14    DaemonSet creation                Ch 6     3 min
15    Taint/Toleration + NodeAffinity   Ch 7     3 min
16    Resource requests and limits      Ch 7     2 min
17    kubeadm init (new cluster)        Ch 10    8 min
18    Scale deployment + check HPA      Ch 2/9   2 min
19    Security context on pod           Ch 8     3 min
20    Job and CronJob creation          Ch 6     3 min
```

---

## 12.3 Speed Labs — Under 3 Minutes Each

Master these until they are automatic. These are the bread-and-butter exam tasks.

### Lab S1: Create a Pod with Every Option

```bash
# Create pod with image, labels, env var, resource limits, port
k run web --image=nginx:alpine \
  --labels="app=web,env=prod" \
  --env="LOG_LEVEL=INFO" \
  --requests='cpu=100m,memory=64Mi' \
  --limits='cpu=500m,memory=256Mi' \
  --port=80 \
  --restart=Never

# Create pod in specific namespace
k run db --image=redis:alpine --namespace=database

# Create pod with serviceAccount
k run api --image=my-app:v1 \
  --serviceaccount=api-sa \
  --namespace=production

# GENERATE YAML (your most used exam command)
k run web $do --image=nginx:alpine > pod.yaml
# Edit pod.yaml to add anything needed
vim pod.yaml
k apply -f pod.yaml
k get pod web    # Verify
```

### Lab S2: Create and Modify a Deployment

```bash
# Create deployment
k create deploy web --image=nginx:alpine --replicas=3

# Scale
k scale deploy web --replicas=5
k get pods -l app=web    # Verify 5 pods

# Update image (rolling update)
k set image deploy/web nginx=nginx:1.25-alpine
k rollout status deploy/web

# Rollback
k rollout undo deploy/web
k rollout history deploy/web

# Generate YAML
k create deploy web --image=nginx --replicas=3 $do > deploy.yaml
```

### Lab S3: Create All Service Types

```bash
# ClusterIP (default)
k expose deploy web --port=80 --target-port=80 --name=web-svc

# NodePort
k expose deploy web --type=NodePort --port=80 --name=web-np

# Get NodePort assigned
k get svc web-np -o jsonpath='{.spec.ports[0].nodePort}'

# Generate and customise
k expose deploy web --port=80 $do > svc.yaml
vim svc.yaml    # Edit type, ports, etc.
k apply -f svc.yaml
k get endpoints web-svc    # Verify not empty
```

### Lab S4: RBAC in 60 Seconds

```bash
# Role + RoleBinding for a user
k create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  --namespace=default

k create rolebinding pod-reader-binding \
  --role=pod-reader \
  --user=alice \
  --namespace=default

# Verify
k auth can-i get pods --as alice    # yes
k auth can-i delete pods --as alice # no

# Role + RoleBinding for ServiceAccount
k create sa my-sa
k create role sa-role --verb=get,list --resource=pods
k create rolebinding sa-binding \
  --role=sa-role \
  --serviceaccount=default:my-sa

# ClusterRole + ClusterRoleBinding
k create clusterrole node-reader --verb=get,list,watch --resource=nodes
k create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader \
  --user=bob
```

### Lab S5: etcd Backup in 60 Seconds

```bash
# Always the same command pattern on kubeadm clusters:
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db \
  --write-out=table

# NOTE: Find cert paths if different from default:
cat /etc/kubernetes/manifests/etcd.yaml | grep -E "cert|ca|key|endpoint"
```

### Lab S6: PV + PVC + Pod

```bash
# Create PV
cat <<'EOF' | k apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv
spec:
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/task-pv
EOF

# Create PVC
cat <<'EOF' | k apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
EOF

k get pv,pvc    # Verify bound

# Create pod using PVC
k run task-pod --image=nginx $do > pod.yaml
# Add volumeMounts and volumes to pod.yaml:
cat >> pod.yaml << 'EOF'
# Edit to add:
#    volumeMounts:
#    - name: task-storage
#      mountPath: /data
#  volumes:
#  - name: task-storage
#    persistentVolumeClaim:
#      claimName: task-pvc
EOF
```

### Lab S7: NetworkPolicy

```bash
# Allow only specific pods to reach backend
cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
EOF

# Deny all ingress
cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
```

### Lab S8: Security Context

```bash
# Pod with specific UID + no privilege escalation
cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
EOF

# Verify
k exec secure-pod -- id
# uid=1000 gid=3000 groups=3000,2000
```

---

## 12.4 Full Troubleshooting Scenarios

### Scenario 1: Broken Worker Node

```
TASK: Worker node worker-1 is in NotReady state. Fix it.
```

```bash
# Step 1: Identify the problem
kubectl get nodes
# worker-1   NotReady   ...

kubectl describe node worker-1 | grep -A10 Conditions
# DiskPressure=False, MemoryPressure=False
# NetworkUnavailable=False, Ready=False ← kubelet not communicating

# Step 2: SSH to the node
ssh worker-1

# Step 3: Check kubelet
systemctl status kubelet
# Active: failed (Result: exit-code)

# Step 4: Check why it failed
journalctl -u kubelet | tail -20
# Error: "failed to load kubelet config file /var/lib/kubelet/config.yaml"
# OR: "runtime_service.go:400" "connect: connection refused" (containerd down)

# Step 5: Fix based on error
# If config missing:
kubeadm init phase kubelet-start --config /etc/kubernetes/kubelet-config.yaml

# If containerd down:
systemctl start containerd
systemctl enable containerd

# Start kubelet
systemctl start kubelet
systemctl enable kubelet

# Step 6: Back on control plane — verify
kubectl get nodes | grep worker-1
# worker-1   Ready   ...  ✅
```

### Scenario 2: Service Not Routing Traffic

```
TASK: The service "payment-svc" in namespace "production" is not
      routing traffic to pods. Fix it.
```

```bash
# Step 1: Check service
kubectl get svc payment-svc -n production
kubectl describe svc payment-svc -n production
# Look at: Selector, Endpoints

# Step 2: Check endpoints
kubectl get endpoints payment-svc -n production
# NAME          ENDPOINTS   AGE
# payment-svc   <none>      5m   ← EMPTY! Selector mismatch

# Step 3: Find what labels the pods actually have
kubectl get pods -n production --show-labels
# payment-pod-xyz   Running   app=payment,tier=backend

# Step 4: Check service selector
kubectl get svc payment-svc -n production -o yaml | grep selector
# selector:
#   app: payments   ← WRONG! Pod label is "payment" not "payments"

# Step 5: Fix the selector
kubectl patch svc payment-svc -n production \
  -p '{"spec":{"selector":{"app":"payment"}}}'

# Step 6: Verify
kubectl get endpoints payment-svc -n production
# NAME          ENDPOINTS              AGE
# payment-svc   10.244.1.5:8080,...   5m   ✅
```

### Scenario 3: Fix a Broken Static Pod

```
TASK: The kube-scheduler is not working. Investigate and fix.
```

```bash
# Step 1: Check scheduler pod
kubectl get pods -n kube-system | grep scheduler
# kube-scheduler-cp   0/1   CrashLoopBackOff   5   2m

# Step 2: Check logs
kubectl logs kube-scheduler-cp -n kube-system
# Error: "stat /etc/kubernetes/scheduler.conf: no such file or directory"

# Step 3: Check the static pod manifest
cat /etc/kubernetes/manifests/kube-scheduler.yaml | grep kubeconfig
# --kubeconfig=/etc/kubernetes/scheduuler.conf   ← TYPO! scheduuler

# Step 4: Fix the manifest
sed -i 's/scheduuler.conf/scheduler.conf/' \
  /etc/kubernetes/manifests/kube-scheduler.yaml

# Step 5: Wait for kubelet to restart the pod
sleep 20
kubectl get pods -n kube-system | grep scheduler
# kube-scheduler-cp   1/1   Running   0   30s ✅
```

### Scenario 4: etcd Restore

```
TASK: Restore the cluster using etcd snapshot at /opt/etcd-backup.db
```

```bash
# Step 1: Stop control plane
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
sleep 30

# Step 2: Restore
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restore \
  --name=$(hostname) \
  --initial-cluster=$(hostname)=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# Step 3: Update etcd manifest to use new data dir
sed 's|/var/lib/etcd|/var/lib/etcd-restore|g' \
  /tmp/etcd.yaml > /etc/kubernetes/manifests/etcd.yaml
sleep 20

# Step 4: Restore other manifests
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
sleep 60

# Step 5: Verify
kubectl get nodes
kubectl get pods -A
```

### Scenario 5: RBAC — Fix Forbidden Error

```
TASK: User "dev-user" gets Forbidden when trying to list pods
      in namespace "development". Fix this.
```

```bash
# Step 1: Confirm the error
kubectl get pods -n development --as dev-user
# Error: pods is forbidden: User "dev-user" cannot list resource "pods"

# Step 2: Check existing RBAC
kubectl get rolebindings -n development -o yaml | grep -B5 "dev-user"
# No rolebinding found for dev-user

kubectl get clusterrolebindings -o yaml | grep -B5 "dev-user"
# No clusterrolebinding either

# Step 3: Create the required access
# Option A: Using built-in view ClusterRole
kubectl create rolebinding dev-user-view \
  --clusterrole=view \
  --user=dev-user \
  --namespace=development

# Option B: If task specifies exact permissions
kubectl create role pod-reader -n development \
  --verb=get,list,watch \
  --resource=pods

kubectl create rolebinding dev-user-binding -n development \
  --role=pod-reader \
  --user=dev-user

# Step 4: Verify
kubectl auth can-i list pods -n development --as dev-user
# yes ✅
kubectl get pods -n development --as dev-user
# (lists pods) ✅
```

---

## 12.5 Mock Exam 1

**Time: 120 minutes | Pass score: 66% | 17 questions**

> Set your timer. Switch context before each question. Verify each answer.

---

**Q1 [3%] — Context: k8s-cluster-1**
Create a pod named `web-server` in namespace `default` using image `nginx:1.21-alpine`.
The pod should have a label `tier=frontend`. Make sure it is running.

```bash
k config use-context k8s-cluster-1
k run web-server --image=nginx:1.21-alpine --labels=tier=frontend
k get pod web-server --show-labels
```

---

**Q2 [5%] — Context: k8s-cluster-1**
Create a Deployment named `backend-deploy` with image `python:3.9-slim`, 3 replicas,
in namespace `backend`. The containers must run as user 1001 (non-root).
Expose port 5000.

```bash
k config use-context k8s-cluster-1
k create ns backend
k create deploy backend-deploy \
  --image=python:3.9-slim \
  --replicas=3 \
  --namespace=backend \
  $do > deploy.yaml

# Edit to add port + securityContext:
vim deploy.yaml
# Add under spec.template.spec.containers[0]:
#   ports:
#   - containerPort: 5000
#   securityContext:
#     runAsUser: 1001
#     runAsNonRoot: true

k apply -f deploy.yaml
k get pods -n backend
```

---

**Q3 [4%] — Context: k8s-cluster-1**
Scale the `backend-deploy` deployment in namespace `backend` to 5 replicas.
Verify all 5 pods are running.

```bash
k scale deploy backend-deploy -n backend --replicas=5
k get pods -n backend -w   # Wait for all Running
k get deploy backend-deploy -n backend
# READY should show 5/5
```

---

**Q4 [5%] — Context: k8s-cluster-1**
Expose the `backend-deploy` deployment in namespace `backend` with a ClusterIP
service named `backend-svc` on port 80, forwarding to container port 5000.

```bash
k expose deploy backend-deploy \
  -n backend \
  --name=backend-svc \
  --port=80 \
  --target-port=5000 \
  --type=ClusterIP

k get svc backend-svc -n backend
k get endpoints backend-svc -n backend
# Endpoints should not be empty!
```

---

**Q5 [5%] — Context: k8s-cluster-2**
Create a PersistentVolume named `log-pv` with:
- Capacity: 2Gi
- Access mode: ReadWriteOnce
- StorageClass: manual
- hostPath: /data/logs
- Reclaim policy: Retain
Then create a PVC named `log-pvc` that binds to it.

```bash
k config use-context k8s-cluster-2

cat <<'EOF' | k apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: log-pv
spec:
  capacity:
    storage: 2Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/logs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: log-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: manual
  resources:
    requests:
      storage: 2Gi
EOF

k get pv log-pv
k get pvc log-pvc
# Both should show STATUS: Bound
```

---

**Q6 [7%] — Context: k8s-cluster-2**
Create a ConfigMap named `app-env` in namespace `config-lab` with:
- `APP_ENV=production`
- `LOG_LEVEL=INFO`
- `MAX_CONNECTIONS=100`

Then create a Pod named `config-pod` that uses this ConfigMap to set
all keys as environment variables. Verify the env vars are present.

```bash
k create ns config-lab
k create cm app-env \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=MAX_CONNECTIONS=100 \
  -n config-lab

cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
  namespace: config-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-env
EOF

k logs config-pod -n config-lab | grep -E "APP_ENV|LOG_LEVEL|MAX_CONNECTIONS"
```

---

**Q7 [7%] — Context: k8s-cluster-2**
Create a ServiceAccount `ci-deployer` in namespace `deployments`.
Create a Role that allows creating and updating Deployments and Services.
Bind the role to the ServiceAccount.
Verify the ServiceAccount has the expected permissions.

```bash
k create ns deployments
k create sa ci-deployer -n deployments

k create role deploy-role -n deployments \
  --verb=create,update,patch,get,list \
  --resource=deployments,services

k create rolebinding ci-deployer-binding -n deployments \
  --role=deploy-role \
  --serviceaccount=deployments:ci-deployer

k auth can-i create deployments \
  --as system:serviceaccount:deployments:ci-deployer \
  -n deployments
# yes ✅

k auth can-i delete deployments \
  --as system:serviceaccount:deployments:ci-deployer \
  -n deployments
# no ✅
```

---

**Q8 [8%] — Context: k8s-cluster-3**
A pod named `broken-pod` in namespace `troubleshoot` is not starting.
Investigate and fix the issue.

```bash
k config use-context k8s-cluster-3
k get pod broken-pod -n troubleshoot
k describe pod broken-pod -n troubleshoot
# Look at Events section...
# Example: "Error: secret "app-secret" not found"

# Fix based on finding:
# If missing secret:
k create secret generic app-secret \
  --from-literal=key=value \
  -n troubleshoot

# If wrong image name:
k set image pod/broken-pod <container>=correct:image -n troubleshoot
# OR edit the pod:
k edit pod broken-pod -n troubleshoot

k get pod broken-pod -n troubleshoot
# STATUS: Running ✅
```

---

**Q9 [6%] — Context: k8s-cluster-3**
Create a NetworkPolicy in namespace `secure-ns` that:
- Applies to pods with label `role=database`
- Allows ingress ONLY from pods with label `role=backend`
- On port 5432

```bash
cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      role: database
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - port: 5432
      protocol: TCP
EOF

k describe netpol db-policy -n secure-ns
```

---

**Q10 [8%] — Context: k8s-cluster-3**
Node `worker-2` is showing NotReady status. Investigate and fix the issue.

```bash
k get nodes
k describe node worker-2 | tail -30

ssh worker-2
  systemctl status kubelet
  # Active: inactive (dead)
  systemctl start kubelet
  systemctl enable kubelet
  exit

k get nodes | grep worker-2
# worker-2   Ready   ... ✅
```

---

**Q11 [10%] — Context: k8s-cluster-4**
Perform an etcd backup to `/opt/etcd-snapshot.db`.
Verify the snapshot was saved successfully.

```bash
k config use-context k8s-cluster-4

# Find etcd cert locations
cat /etc/kubernetes/manifests/etcd.yaml | \
  grep -E "cert-file|key-file|trusted-ca-file|listen-client"

ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-snapshot.db \
  --write-out=table
# Shows hash, revision, total keys, total size ✅
```

---

**Q12 [12%] — Context: k8s-cluster-4**
Upgrade the cluster from v1.28 to v1.29.
Upgrade control plane first, then worker nodes.

```bash
# (See Chapter 10, Section 10.4 for full procedure)
# Key steps:
# 1. Backup etcd: etcdctl snapshot save ...
# 2. apt install kubeadm=1.29.0-1.1 → kubeadm upgrade apply v1.29.0
# 3. apt install kubelet kubectl → systemctl restart kubelet
# 4. For each worker: cordon → drain → ssh → apt upgrade → uncordon
k get nodes   # Verify all v1.29.0
```

---

**Q13 [5%] — Context: k8s-cluster-1**
Create a pod with:
- Name: `multi-container-pod`
- Container 1: name `main`, image `nginx:alpine`
- Container 2: name `sidecar`, image `busybox`, command: `sleep 3600`
- Both containers should share a volume at `/shared`

```bash
cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: main
    image: nginx:alpine
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  - name: sidecar
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

k get pod multi-container-pod
k exec multi-container-pod -c main -- ls /shared
k exec multi-container-pod -c sidecar -- ls /shared
```

---

**Q14 [5%] — Context: k8s-cluster-2**
Create a CronJob named `hourly-cleanup` that runs every hour,
using image `busybox`, running the command: `echo "cleanup done"`.
Keep 3 successful job histories and 1 failed.

```bash
cat <<'EOF' | k apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hourly-cleanup
spec:
  schedule: "0 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: busybox
            command: ["sh", "-c", "echo cleanup done"]
EOF

k get cj hourly-cleanup

# Manually trigger to test
k create job test-cleanup --from=cronjob/hourly-cleanup
k get jobs
k logs job/test-cleanup
```

---

**Q15 [5%] — Context: k8s-cluster-1**
Apply a taint to node `worker-3`: `env=production:NoSchedule`.
Create a pod that tolerates this taint and is scheduled on `worker-3`.

```bash
k taint node worker-3 env=production:NoSchedule
k describe node worker-3 | grep Taint

cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  tolerations:
  - key: env
    operator: Equal
    value: production
    effect: NoSchedule
  nodeSelector:
    kubernetes.io/hostname: worker-3
  containers:
  - name: app
    image: nginx:alpine
EOF

k get pod toleration-pod -o wide
# NODE column should show worker-3 ✅
```

---

**Q16 [5%] — Context: k8s-cluster-3**
Create a StatefulSet named `cache` in namespace `stateful-lab`
with 3 replicas using image `redis:alpine`.
Use a headless service named `cache-headless`.
Each pod should have a 1Gi PVC for `/data`.

```bash
k create ns stateful-lab

cat <<'EOF' | k apply -f -
apiVersion: v1
kind: Service
metadata:
  name: cache-headless
  namespace: stateful-lab
spec:
  clusterIP: None
  selector:
    app: cache
  ports:
  - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cache
  namespace: stateful-lab
spec:
  serviceName: cache-headless
  replicas: 3
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
EOF

k get sts cache -n stateful-lab
k get pods -n stateful-lab -w   # Watch ordered startup
k get pvc -n stateful-lab       # 3 PVCs: data-cache-0,1,2
```

---

**Q17 [5%] — Context: k8s-cluster-1**
Create an Ingress named `app-ingress` in namespace `ingress-lab`:
- Route `myapp.example.com/api` → Service `api-svc` port 8080
- Route `myapp.example.com/web` → Service `web-svc` port 80

```bash
k create ns ingress-lab

cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: ingress-lab
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
EOF

k describe ingress app-ingress -n ingress-lab
```

---

## 12.6 Final Pre-Exam Checklist

```
╔══════════════════════════════════════════════════════════════════════╗
║              24 HOURS BEFORE THE EXAM                                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  TECHNICAL:                                                          ║
║  ✅ etcd backup command memorised letter-perfect                    ║
║  ✅ Cluster upgrade sequence clear (backup → CP → workers)          ║
║  ✅ RBAC: role + binding + SA pattern fluent                        ║
║  ✅ PV + PVC storageClassName must match — confirmed                ║
║  ✅ Service selector must match pod labels — confirmed              ║
║  ✅ Job restartPolicy = OnFailure, not Always                       ║
║  ✅ StatefulSet needs headless service                              ║
║  ✅ `k auth can-i` for permission verification                      ║
║                                                                      ║
║  ALIASES (paste into terminal at exam start):                        ║
║  alias k=kubectl                                                     ║
║  export do="--dry-run=client -o yaml"                               ║
║  export now="--force --grace-period=0"                              ║
║  source <(kubectl completion bash)                                   ║
║  complete -F __start_kubectl k                                       ║
║                                                                      ║
║  LOGISTICS:                                                          ║
║  ✅ Valid government ID ready                                        ║
║  ✅ Webcam, microphone, clean desk confirmed                        ║
║  ✅ Stable internet connection tested                               ║
║  ✅ PSI browser installed and tested                                ║
║  ✅ Quiet room, 2 hours uninterrupted                               ║
║                                                                      ║
║  MINDSET:                                                            ║
║  ✅ 66% pass rate = you don't need to be perfect                    ║
║  ✅ Flag hard questions, return with fresh eyes                     ║
║  ✅ Always switch context before each question                      ║
║  ✅ Always verify your answer after completing it                   ║
║  ✅ Trust your preparation — you have done the work                 ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 12.7 The Complete kubectl Reference Card

```bash
# ══════════════════════════════════════════════════════════════════════
# CONTEXT AND NAMESPACE
# ══════════════════════════════════════════════════════════════════════
k config current-context
k config use-context <ctx>
k config set-context --current --namespace=<ns>
k config get-contexts

# ══════════════════════════════════════════════════════════════════════
# PODS
# ══════════════════════════════════════════════════════════════════════
k run <n> --image=<img> [--labels=k=v] [--env=K=V] [--port=N] [--restart=Never]
k get pods [-n <ns>] [-A] [-o wide] [--show-labels] [-w] [-l key=val]
k describe pod <n>
k logs <n> [-f] [--previous] [--tail=N] [-c <container>]
k exec -it <n> -- bash
k delete pod <n> [$now]
k get pod <n> -o jsonpath='{.status.podIP}'

# ══════════════════════════════════════════════════════════════════════
# DEPLOYMENTS
# ══════════════════════════════════════════════════════════════════════
k create deploy <n> --image=<img> --replicas=N
k scale deploy <n> --replicas=N
k set image deploy/<n> <container>=<image>
k rollout status deploy/<n>
k rollout undo deploy/<n> [--to-revision=N]
k rollout history deploy/<n>
k rollout restart deploy/<n>

# ══════════════════════════════════════════════════════════════════════
# SERVICES
# ══════════════════════════════════════════════════════════════════════
k expose deploy <n> --port=80 [--target-port=8080] [--type=NodePort]
k get svc [-o wide]
k get endpoints <svc>

# ══════════════════════════════════════════════════════════════════════
# STORAGE
# ══════════════════════════════════════════════════════════════════════
k get pv,pvc,sc
k describe pvc <n>
k patch pvc <n> -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# ══════════════════════════════════════════════════════════════════════
# CONFIG
# ══════════════════════════════════════════════════════════════════════
k create cm <n> --from-literal=k=v [--from-file=<f>]
k create secret generic <n> --from-literal=k=v
k get secret <n> -o jsonpath='{.data.<key>}' | base64 -d

# ══════════════════════════════════════════════════════════════════════
# RBAC
# ══════════════════════════════════════════════════════════════════════
k create role <n> --verb=get,list --resource=pods
k create clusterrole <n> --verb=get,list --resource=nodes
k create rolebinding <n> --role=<r> --user=<u> [-n <ns>]
k create rolebinding <n> --role=<r> --serviceaccount=<ns>:<sa>
k create clusterrolebinding <n> --clusterrole=<r> --user=<u>
k auth can-i <verb> <resource> [--as <user>] [-n <ns>]
k auth can-i --list --as <user>
k auth whoami

# ══════════════════════════════════════════════════════════════════════
# NODES
# ══════════════════════════════════════════════════════════════════════
k get nodes [-o wide] [--show-labels]
k describe node <n>
k top nodes
k top pods [-A] [--sort-by=cpu]
k cordon <n>
k drain <n> --ignore-daemonsets [--delete-emptydir-data]
k uncordon <n>
k taint node <n> key=val:Effect
k taint node <n> key=val:Effect-   # Remove taint
k label node <n> key=val

# ══════════════════════════════════════════════════════════════════════
# etcd
# ══════════════════════════════════════════════════════════════════════
ETCDCTL_API=3 etcdctl snapshot save <file> \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl snapshot status <file> --write-out=table

# ══════════════════════════════════════════════════════════════════════
# DEBUGGING
# ══════════════════════════════════════════════════════════════════════
k get events -A --sort-by=.lastTimestamp
k describe <resource> <name>
k get <resource> <name> -o yaml
k explain <resource>[.field]
k run debug --image=nicolaka/netshoot --rm -it --restart=Never -- bash
systemctl status kubelet
journalctl -u kubelet -f
crictl ps
crictl logs <container-id>
```

---

## Chapter 12 Summary — The CKA in One Page

```
╔══════════════════════════════════════════════════════════════════════╗
║           THE CKA IN ONE PAGE — FINAL REVISION                       ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  BEFORE EACH QUESTION:  k config use-context <from-question>        ║
║  AFTER EACH ANSWER:     k get <resource> → confirm it exists        ║
║                                                                      ║
║  HARD RULES:                                                         ║
║  ① Context switch    EVERY SINGLE QUESTION. No exceptions.          ║
║  ② Imperative first  k create/run/expose $do > file.yaml            ║
║  ③ Verify always     k get / k describe / k auth can-i              ║
║  ④ Flag and move     >3 min stuck → flag, return on pass 2          ║
║  ⑤ Backup etcd       before ANY cluster modification                ║
║                                                                      ║
║  YAML PATTERNS TO MEMORISE:                                          ║
║  PV/PVC:     storageClassName must match exactly (case-sensitive)   ║
║  Job:        restartPolicy: OnFailure (never Always)                ║
║  StatefulSet: serviceName: headless-service-name                    ║
║  Ingress:    apiVersion: networking.k8s.io/v1                       ║
║  NetPolicy:  podSelector + policyTypes + ingress/egress rules       ║
║  Toleration: key + operator + value + effect (all 4 fields)        ║
║  SA binding: --serviceaccount=namespace:sa-name (two parts!)        ║
║                                                                      ║
║  TROUBLESHOOT SEQUENCE:                                              ║
║  get → describe → logs --previous → events → ssh → journalctl      ║
║                                                                      ║
║  etcd BACKUP (commit to memory):                                     ║
║  ETCDCTL_API=3 etcdctl snapshot save /path/backup.db \             ║
║    --endpoints=https://127.0.0.1:2379 \                             ║
║    --cacert=/etc/kubernetes/pki/etcd/ca.crt \                       ║
║    --cert=/etc/kubernetes/pki/etcd/server.crt \                     ║
║    --key=/etc/kubernetes/pki/etcd/server.key                        ║
║                                                                      ║
║  YOU ARE READY. NOW GO PASS THE CKA. 🎯                             ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

# HANDBOOK COMPLETE

**Congratulations.** You have completed the full Kubernetes Mastery Handbook.

12 Parts. 12 Chapters. From zero — to CKA-ready Administrator.

```
╔══════════════════════════════════════════════════════════════════════╗
║           WHAT YOU HAVE MASTERED                                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Part 1:  Kubernetes architecture, control plane, worker nodes      ║
║  Part 2:  Pods, Deployments, ReplicaSets, Namespaces, Labels        ║
║  Part 3:  Services, Ingress, DNS, CNI, NetworkPolicies              ║
║  Part 4:  Volumes, PVs, PVCs, StorageClasses, dynamic provisioning  ║
║  Part 5:  ConfigMaps, Secrets, Downward API, Projected Volumes      ║
║  Part 6:  DaemonSets, StatefulSets, Jobs, CronJobs                  ║
║  Part 7:  Scheduling — affinity, taints, resources, QoS, priority  ║
║  Part 8:  RBAC, ServiceAccounts, SecurityContexts, PSS, Admission  ║
║  Part 9:  Metrics Server, Prometheus, Grafana, Logging, Loki        ║
║  Part 10: kubeadm, cluster upgrade, etcd backup/restore, certs      ║
║  Part 11: HA, Helm, Kustomize, GitOps, ArgoCD, best practices      ║
║  Part 12: CKA strategy, speed labs, troubleshooting, mock exams    ║
║                                                                      ║
║  WHAT TO DO NEXT:                                                    ║
║  1. Practice the speed labs until each takes under 3 minutes        ║
║  2. Do the mock exam under real time pressure (120 min, no pauses)  ║
║  3. Take killer.sh practice exam (included with CKA purchase)       ║
║  4. Book the exam — you are ready                                   ║
║                                                                      ║
║  AFTER YOU PASS:                                                     ║
║  → Open a PR and add your name to the Wall of Fame                 ║
║  → Share this handbook with the next engineer who needs it          ║
║  → Start the CKAD or CKS if you want more                          ║
║                                                                      ║
║  Published by NEXTIQZ Technologies                                   ║
║  github.com/NEXTIQZ/kubernetes-mastery-handbook                     ║
╚══════════════════════════════════════════════════════════════════════╝
```
