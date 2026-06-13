# KUBERNETES MASTERY HANDBOOK
# Supplementary: Complete kubectl Reference Card

---

> **Print this. Tape it to your monitor. Use it every day.**

---

## SECTION 1 — Setup and Context

```bash
# ── INSTALLATION ──────────────────────────────────────────────────────
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl
kubectl version --client

# ── ALIASES (paste at start of EVERY exam / terminal session) ─────────
alias k=kubectl
alias kgp='kubectl get pods'
alias kgd='kubectl get deployments'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kga='kubectl get all'
alias kdp='kubectl describe pod'
alias klog='kubectl logs'
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
source <(kubectl completion bash)
complete -F __start_kubectl k

# ── VIM SETUP FOR YAML ────────────────────────────────────────────────
cat >> ~/.vimrc << 'EOF'
set nu
set et
set ts=2
set sw=2
set sts=2
set ai
EOF

# ── CONTEXT MANAGEMENT ────────────────────────────────────────────────
kubectl config get-contexts              # List all contexts
kubectl config current-context           # Which cluster am I on?
kubectl config use-context <name>        # Switch context
kubectl config set-context --current \
  --namespace=<namespace>               # Set default namespace
kubectl config view                      # Show full kubeconfig
kubectl config view --minify            # Only current context
kubectl config view -o \
  jsonpath='{.clusters[0].cluster.server}' # Get API server URL

# ── ADD CLUSTER TO KUBECONFIG ─────────────────────────────────────────
kubectl config set-cluster my-cluster \
  --server=https://192.168.1.10:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt

kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/pki/admin.crt \
  --client-key=/etc/kubernetes/pki/admin.key

kubectl config set-context my-ctx \
  --cluster=my-cluster \
  --user=admin \
  --namespace=default

kubectl config use-context my-ctx
```

---

## SECTION 2 — Cluster Information

```bash
kubectl cluster-info                     # API server and CoreDNS endpoints
kubectl cluster-info dump                # Detailed cluster state dump
kubectl get nodes                        # List all nodes
kubectl get nodes -o wide               # With IP, OS, runtime info
kubectl get nodes --show-labels         # With all labels
kubectl describe node <name>            # Full node details
kubectl top nodes                        # CPU/memory usage (needs metrics-server)
kubectl get componentstatuses           # Control plane component health
kubectl api-resources                   # All resource types + API groups
kubectl api-resources --namespaced=true # Only namespace-scoped resources
kubectl api-versions                     # All API versions available
kubectl explain pod                      # What is a pod? (built-in docs)
kubectl explain pod.spec                 # What fields does spec have?
kubectl explain pod.spec.containers.resources
kubectl version                          # Client + server versions
kubectl version --short                  # Compact version output
```

---

## SECTION 3 — Namespaces

```bash
kubectl get namespaces                   # List all namespaces
kubectl get ns                           # Shorthand
kubectl create namespace <name>          # Create namespace
kubectl create ns <name>                 # Shorthand
kubectl delete namespace <name>          # DELETE namespace (deletes all resources!)
kubectl apply -f namespace.yaml          # Declarative creation
kubectl get all -n <namespace>           # All resources in namespace
kubectl get pods -n kube-system         # System pods
kubectl get pods -A                      # All namespaces (shorthand for --all-namespaces)
kubectl config set-context --current \
  --namespace=<ns>                       # Avoid typing -n every time
```

---

## SECTION 4 — Pods

```bash
# ── CREATE ────────────────────────────────────────────────────────────
kubectl run <name> --image=<img>
kubectl run <name> --image=<img> --restart=Never    # Single pod (not deployment)
kubectl run <name> --image=<img> --port=80
kubectl run <name> --image=<img> --labels="app=web,env=prod"
kubectl run <name> --image=<img> --env="KEY=VALUE"
kubectl run <name> --image=<img> \
  --requests='cpu=100m,memory=64Mi' \
  --limits='cpu=500m,memory=256Mi'
kubectl run <name> --image=<img> --serviceaccount=<sa>
kubectl run <name> --image=<img> --command -- sleep 3600
kubectl run <name> --image=<img> $do > pod.yaml     # Generate YAML

# Run and immediately exec in (ephemeral debug pod)
kubectl run debug --image=nicolaka/netshoot \
  --rm -it --restart=Never -- bash
kubectl run test --image=busybox \
  --rm -it --restart=Never -- sh

# ── READ ──────────────────────────────────────────────────────────────
kubectl get pods                          # Current namespace
kubectl get pods -n <ns>                  # Specific namespace
kubectl get pods -A                       # All namespaces
kubectl get pods -o wide                  # Node, IP, nominated node
kubectl get pods --show-labels            # All labels
kubectl get pods -l app=web              # Filter by label
kubectl get pods -l "app=web,env=prod"   # Multiple labels (AND)
kubectl get pods -l "env in (prod,staging)" # Set-based
kubectl get pods -w                       # Watch for changes
kubectl get pods -o yaml                  # Full YAML output
kubectl get pods -o json                  # Full JSON output
kubectl get pod <name> -o jsonpath='{.status.podIP}'
kubectl get pod <name> -o jsonpath='{.spec.nodeName}'
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get pod <name> -o jsonpath='{.status.qosClass}'

kubectl describe pod <name>               # Full details including Events
kubectl describe pod <name> -n <ns>

# ── LOGS ──────────────────────────────────────────────────────────────
kubectl logs <name>                       # Current logs
kubectl logs <name> -f                   # Follow/stream logs
kubectl logs <name> --previous           # Logs from CRASHED container
kubectl logs <name> --tail=100           # Last 100 lines
kubectl logs <name> --since=1h           # Last 1 hour
kubectl logs <name> --since-time="2024-01-15T10:00:00Z"
kubectl logs <name> -c <container>       # Specific container in pod
kubectl logs <name> --all-containers     # All containers in pod
kubectl logs -l app=web --prefix         # All pods matching label
kubectl logs -l app=web --all-containers --prefix --tail=20

# ── EXEC ──────────────────────────────────────────────────────────────
kubectl exec <name> -- <command>          # Run command in pod
kubectl exec <name> -- ls /app
kubectl exec <name> -- env
kubectl exec <name> -- cat /etc/config/key
kubectl exec -it <name> -- bash          # Interactive bash
kubectl exec -it <name> -- sh            # Interactive sh (alpine)
kubectl exec -it <name> -c <container> -- bash  # Specific container

# ── PORT FORWARD ──────────────────────────────────────────────────────
kubectl port-forward pod/<name> 8080:80  # localhost:8080 → pod:80
kubectl port-forward pod/<name> 8080:80 & # Background
kubectl port-forward svc/<name> 8080:80  # Via service

# ── UPDATE ────────────────────────────────────────────────────────────
kubectl label pod <name> env=prod         # Add label
kubectl label pod <name> env=staging --overwrite
kubectl label pod <name> env-             # Remove label (trailing dash)
kubectl annotate pod <name> desc="my pod"
kubectl annotate pod <name> desc-         # Remove annotation
kubectl edit pod <name>                   # Edit in $EDITOR

# ── DELETE ────────────────────────────────────────────────────────────
kubectl delete pod <name>                  # Graceful (30s grace period)
kubectl delete pod <name> $now             # Immediate deletion
kubectl delete pod <name1> <name2>         # Multiple pods
kubectl delete pods --all                  # All pods in namespace
kubectl delete pods -l app=web             # By label selector
kubectl delete pods --field-selector status.phase=Succeeded  # Completed pods
```

---

## SECTION 5 — Deployments

```bash
# ── CREATE ────────────────────────────────────────────────────────────
kubectl create deployment <name> --image=<img>
kubectl create deployment <name> --image=<img> --replicas=3
kubectl create deployment <name> --image=<img> --port=80
kubectl create deployment <name> --image=<img> --replicas=3 \
  $do > deploy.yaml                       # Generate YAML

# ── READ ──────────────────────────────────────────────────────────────
kubectl get deployments                   # List deployments
kubectl get deploy                        # Shorthand
kubectl get deploy -A                     # All namespaces
kubectl get deploy -o wide               # More detail
kubectl describe deploy <name>            # Full details + events
kubectl get deploy <name> -o yaml        # Full YAML

# ── SCALE ─────────────────────────────────────────────────────────────
kubectl scale deploy <name> --replicas=5
kubectl scale deploy <name> --replicas=0  # Stop all pods (keep deployment)
kubectl autoscale deploy <name> \
  --min=2 --max=10 --cpu-percent=70       # Create HPA

# ── UPDATE ────────────────────────────────────────────────────────────
kubectl set image deploy/<name> <container>=<newimage>
kubectl set image deploy/<name> nginx=nginx:1.25
kubectl set resources deploy/<name> \
  --limits=cpu=500m,memory=256Mi \
  --requests=cpu=100m,memory=128Mi
kubectl set env deploy/<name> KEY=VALUE
kubectl edit deploy <name>               # Edit in $EDITOR
kubectl apply -f deploy.yaml             # Apply updated YAML

# ── ROLLOUT ───────────────────────────────────────────────────────────
kubectl rollout status deploy/<name>     # Watch rollout progress
kubectl rollout history deploy/<name>    # View revision history
kubectl rollout history deploy/<name> --revision=2  # Specific revision details
kubectl rollout undo deploy/<name>       # Rollback to previous
kubectl rollout undo deploy/<name> --to-revision=1  # Rollback to revision 1
kubectl rollout pause deploy/<name>      # Pause rollout
kubectl rollout resume deploy/<name>     # Resume paused rollout
kubectl rollout restart deploy/<name>    # Rolling restart all pods

# ── DELETE ────────────────────────────────────────────────────────────
kubectl delete deploy <name>
kubectl delete deploy --all
```

---

## SECTION 6 — Services

```bash
# ── CREATE ────────────────────────────────────────────────────────────
# From a deployment (most common)
kubectl expose deploy <name> --port=80
kubectl expose deploy <name> --port=80 --target-port=8080
kubectl expose deploy <name> --type=NodePort --port=80
kubectl expose deploy <name> --type=LoadBalancer --port=80
kubectl expose deploy <name> --port=80 \
  --name=custom-svc-name                  # Custom service name
kubectl expose pod <name> --port=80      # From a pod
kubectl expose deploy <name> --port=80 $do > svc.yaml  # Generate YAML

# ── READ ──────────────────────────────────────────────────────────────
kubectl get services                      # List services
kubectl get svc                           # Shorthand
kubectl get svc -A                        # All namespaces
kubectl get svc -o wide                  # With selector
kubectl describe svc <name>              # Full details + endpoints
kubectl get endpoints <name>             # Pod IPs backing the service

# ── TEST CONNECTIVITY ─────────────────────────────────────────────────
# From inside cluster
kubectl run test --image=busybox --rm -it --restart=Never -- \
  wget -qO- http://<service-name>:<port>
kubectl run test --image=busybox --rm -it --restart=Never -- \
  wget -qO- http://<service-name>.<namespace>:<port>

# ── NODE PORT ─────────────────────────────────────────────────────────
kubectl get svc <name> -o jsonpath='{.spec.ports[0].nodePort}'
curl http://$(kubectl get node -o jsonpath='{.items[0].status.addresses[0].address}'):<nodeport>
```

---

## SECTION 7 — Ingress

```bash
kubectl get ingress                       # List ingresses
kubectl get ing                           # Shorthand
kubectl get ing -A                        # All namespaces
kubectl describe ing <name>              # Rules + events
kubectl delete ing <name>

# Quick ingress YAML generation (no imperative create — use yaml directly)
# See YAML templates file for skeleton
```

---

## SECTION 8 — ConfigMaps

```bash
# ── CREATE ────────────────────────────────────────────────────────────
kubectl create configmap <name> \
  --from-literal=KEY=VALUE \
  --from-literal=KEY2=VALUE2
kubectl create cm <name> --from-literal=KEY=VALUE  # Shorthand
kubectl create cm <name> --from-file=config.properties
kubectl create cm <name> --from-file=mykey=config.properties  # Custom key name
kubectl create cm <name> --from-env-file=app.env
kubectl create cm <name> --from-file=./config-dir/   # All files in directory
kubectl create cm <name> --from-literal=KEY=VALUE $do > cm.yaml

# ── READ ──────────────────────────────────────────────────────────────
kubectl get configmaps                    # List
kubectl get cm                            # Shorthand
kubectl describe cm <name>               # Full details with values
kubectl get cm <name> -o yaml            # Full YAML
kubectl get cm <name> -o jsonpath='{.data.KEY}'  # Specific value

# ── UPDATE ────────────────────────────────────────────────────────────
kubectl edit cm <name>                    # Edit in place
# Patch specific key:
kubectl patch cm <name> \
  -p '{"data":{"LOG_LEVEL":"DEBUG"}}'
# Replace entire configmap:
kubectl create cm <name> --from-literal=KEY=NEWVALUE \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## SECTION 9 — Secrets

```bash
# ── CREATE ────────────────────────────────────────────────────────────
kubectl create secret generic <name> \
  --from-literal=KEY=VALUE
kubectl create secret generic <name> \
  --from-file=ssh-key=~/.ssh/id_rsa
kubectl create secret tls <name> \
  --cert=tls.crt --key=tls.key
kubectl create secret docker-registry <name> \
  --docker-server=registry.io \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com

# ── READ ──────────────────────────────────────────────────────────────
kubectl get secrets
kubectl describe secret <name>           # Shows keys but not values
kubectl get secret <name> -o yaml        # Shows base64-encoded values

# DECODE a secret value
kubectl get secret <name> \
  -o jsonpath='{.data.KEY}' | base64 -d
kubectl get secret <name> \
  -o go-template='{{.data.KEY | base64decode}}'
kubectl get secret <name> \
  -o jsonpath='{.data}' | python3 -c \
  "import sys,json,base64; \
   [print(f'{k}: {base64.b64decode(v).decode()}') \
    for k,v in json.load(sys.stdin).items()]"

# ENCODE for use in YAML
echo -n "mysecret" | base64             # Encode (ALWAYS use -n!)
echo "bXlzZWNyZXQ=" | base64 -d        # Decode
```

---

## SECTION 10 — Storage (PV, PVC, StorageClass)

```bash
# ── PersistentVolumes ─────────────────────────────────────────────────
kubectl get pv                            # List (cluster-wide, not namespaced)
kubectl get pv -o wide                   # With extra info
kubectl describe pv <name>
kubectl delete pv <name>
kubectl get pv --sort-by=.spec.capacity.storage  # Sort by size

# ── PersistentVolumeClaims ────────────────────────────────────────────
kubectl get pvc                           # Current namespace
kubectl get pvc -A                        # All namespaces
kubectl get pvc -A --field-selector \
  status.phase=Pending                    # Find unbound PVCs
kubectl describe pvc <name>              # Check Events for binding issues
kubectl delete pvc <name>
# Resize a PVC:
kubectl patch pvc <name> \
  -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# ── StorageClasses ────────────────────────────────────────────────────
kubectl get storageclass
kubectl get sc                            # Shorthand
kubectl describe sc <name>
kubectl get sc -o wide                   # With provisioner info

# ── See everything storage-related at once ────────────────────────────
kubectl get pv,pvc,sc
kubectl get pv,pvc -A
```

---

## SECTION 11 — RBAC

```bash
# ── ROLES (namespaced) ────────────────────────────────────────────────
kubectl create role <name> \
  --verb=get,list,watch \
  --resource=pods
kubectl create role <name> \
  --verb=get,list,watch \
  --resource=pods,pods/log,services
kubectl create role <name> \
  --verb=get,list,watch \
  --resource=deployments \
  --namespace=production
kubectl get roles -A
kubectl describe role <name> -n <ns>

# ── CLUSTERROLES (cluster-wide) ───────────────────────────────────────
kubectl create clusterrole <name> \
  --verb=get,list,watch \
  --resource=nodes,persistentvolumes
kubectl get clusterroles
kubectl describe clusterrole <name>

# ── ROLEBINDINGS (namespaced) ─────────────────────────────────────────
kubectl create rolebinding <name> \
  --role=<role> \
  --user=<username>
kubectl create rolebinding <name> \
  --role=<role> \
  --user=<username> \
  --namespace=<ns>
kubectl create rolebinding <name> \
  --clusterrole=view \
  --user=<username> \
  --namespace=<ns>          # ClusterRole scoped to one namespace via RoleBinding
kubectl create rolebinding <name> \
  --role=<role> \
  --serviceaccount=<namespace>:<sa-name>  # For ServiceAccount
kubectl create rolebinding <name> \
  --role=<role> \
  --group=<groupname>
kubectl get rolebindings -A
kubectl describe rolebinding <name> -n <ns>

# ── CLUSTERROLEBINDINGS (cluster-wide) ───────────────────────────────
kubectl create clusterrolebinding <name> \
  --clusterrole=<clusterrole> \
  --user=<username>
kubectl create clusterrolebinding <name> \
  --clusterrole=cluster-admin \
  --user=<username>            # Full admin access — use carefully!
kubectl create clusterrolebinding <name> \
  --clusterrole=<clusterrole> \
  --serviceaccount=<ns>:<sa>

# ── PERMISSION CHECKS ─────────────────────────────────────────────────
kubectl auth can-i create pods                          # Current user
kubectl auth can-i create pods -n production            # In namespace
kubectl auth can-i create pods --as alice               # As another user
kubectl auth can-i create pods --as alice -n production # User + namespace
kubectl auth can-i list pods \
  --as system:serviceaccount:ns:sa-name                # As ServiceAccount
kubectl auth can-i --list                               # All current user permissions
kubectl auth can-i --list --as alice -n staging        # All permissions for alice
kubectl auth whoami                                     # Who am I?
```

---

## SECTION 12 — Service Accounts

```bash
kubectl get serviceaccounts               # List (current namespace)
kubectl get sa                            # Shorthand
kubectl get sa -A                         # All namespaces
kubectl create serviceaccount <name>
kubectl create sa <name> -n <namespace>
kubectl describe sa <name>

# Create token for a ServiceAccount (testing)
kubectl create token <sa-name>
kubectl create token <sa-name> --duration=1h

# Check token mounted in pod
kubectl exec <pod> -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token
kubectl exec <pod> -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

---

## SECTION 13 — Nodes

```bash
# ── READ ──────────────────────────────────────────────────────────────
kubectl get nodes
kubectl get nodes -o wide               # IP, OS, runtime, kernel
kubectl get nodes --show-labels
kubectl describe node <name>            # Taints, conditions, capacity, events
kubectl top nodes                        # CPU/memory usage
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory

# ── LABELS AND TAINTS ─────────────────────────────────────────────────
kubectl label node <name> disk=ssd
kubectl label node <name> disk=hdd --overwrite
kubectl label node <name> disk-          # Remove label
kubectl get nodes -l disk=ssd           # Filter by label

kubectl taint node <name> key=value:NoSchedule
kubectl taint node <name> key=value:PreferNoSchedule
kubectl taint node <name> key=value:NoExecute
kubectl taint node <name> key=value:NoSchedule-   # Remove taint
kubectl describe node <name> | grep Taints

# ── MAINTENANCE ───────────────────────────────────────────────────────
kubectl cordon <name>                   # Mark unschedulable (no new pods)
kubectl drain <name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s
kubectl uncordon <name>                 # Re-enable scheduling
```

---

## SECTION 14 — Jobs and CronJobs

```bash
# ── JOBS ──────────────────────────────────────────────────────────────
kubectl create job <name> --image=<img> -- <command>
kubectl create job <name> --image=busybox -- sh -c "echo done"
kubectl create job <name> --from=cronjob/<cj-name>  # Manually trigger CronJob
kubectl get jobs
kubectl get jobs -A
kubectl describe job <name>
kubectl logs job/<name>
kubectl delete job <name>
kubectl delete jobs --field-selector \
  status.successful=1                   # Delete completed jobs

# ── CRONJOBS ──────────────────────────────────────────────────────────
kubectl get cronjobs
kubectl get cj                          # Shorthand
kubectl describe cj <name>
# Suspend a CronJob:
kubectl patch cj <name> \
  -p '{"spec":{"suspend":true}}'
# Resume:
kubectl patch cj <name> \
  -p '{"spec":{"suspend":false}}'
kubectl delete cj <name>
```

---

## SECTION 15 — Workload Controllers

```bash
# ── REPLICASETS ───────────────────────────────────────────────────────
kubectl get replicasets
kubectl get rs
kubectl scale rs <name> --replicas=5
kubectl describe rs <name>

# ── STATEFULSETS ──────────────────────────────────────────────────────
kubectl get statefulsets
kubectl get sts
kubectl get sts -A
kubectl describe sts <name>
kubectl scale sts <name> --replicas=5
kubectl rollout status sts/<name>

# ── DAEMONSETS ────────────────────────────────────────────────────────
kubectl get daemonsets
kubectl get ds
kubectl get ds -n kube-system
kubectl describe ds <name>
kubectl rollout status ds/<name>
kubectl rollout history ds/<name>
```

---

## SECTION 16 — Monitoring and Metrics

```bash
# ── RESOURCE USAGE ────────────────────────────────────────────────────
kubectl top nodes
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory
kubectl top pods
kubectl top pods -A
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
kubectl top pods --containers          # Per-container breakdown
kubectl top pods -l app=web

# ── HPA ───────────────────────────────────────────────────────────────
kubectl get hpa
kubectl get hpa -A
kubectl describe hpa <name>
kubectl autoscale deploy <name> \
  --min=2 --max=10 --cpu-percent=70
kubectl delete hpa <name>
```

---

## SECTION 17 — Debugging and Troubleshooting

```bash
# ── EVENTS ────────────────────────────────────────────────────────────
kubectl get events                      # Current namespace
kubectl get events -A                   # All namespaces
kubectl get events --sort-by=.lastTimestamp
kubectl get events --sort-by=.lastTimestamp | tail -20
kubectl get events -n kube-system --sort-by='.lastTimestamp'
kubectl get events --field-selector \
  reason=FailedScheduling               # Scheduling failures
kubectl get events --field-selector \
  involvedObject.name=my-pod           # Events for specific pod
kubectl get events --field-selector \
  reason=Evicted                        # Evicted pods

# ── GENERIC DEBUG ─────────────────────────────────────────────────────
kubectl get all -n <namespace>          # Everything in namespace
kubectl get all -A                      # Everything everywhere
kubectl describe <resource> <name>      # ALWAYS read the Events section
kubectl get <resource> <name> -o yaml   # Full object spec
kubectl diff -f file.yaml               # What would change?

# ── API SERVER CHECKS ─────────────────────────────────────────────────
kubectl get --raw /healthz
kubectl get --raw /readyz
kubectl get --raw /version | python3 -m json.tool

# ── NETWORK DEBUGGING POD ────────────────────────────────────────────
kubectl run netdebug \
  --image=nicolaka/netshoot \
  --rm -it --restart=Never -- bash
# Inside: nslookup, dig, curl, wget, ping, traceroute, tcpdump, ss, ip

# ── STATIC POD DEBUGGING (control plane) ─────────────────────────────
ls /etc/kubernetes/manifests/          # Static pod definitions
crictl ps                               # Running containers (without kubelet)
crictl ps -a                            # All containers including stopped
crictl logs <container-id>             # Container logs without kubectl
crictl inspect <container-id>          # Container details

# ── KUBELET ───────────────────────────────────────────────────────────
systemctl status kubelet
systemctl start kubelet
systemctl restart kubelet
systemctl enable kubelet
journalctl -u kubelet -f               # Live kubelet logs
journalctl -u kubelet | tail -50       # Last 50 lines
journalctl -u kubelet --since "10 min ago"
```

---

## SECTION 18 — etcd Operations

```bash
# ALWAYS SET THIS FIRST:
export ETCDCTL_API=3

# Common TLS flags (save as alias)
export ETCD_FLAGS="--endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

# ── HEALTH AND STATUS ─────────────────────────────────────────────────
etcdctl $ETCD_FLAGS endpoint health
etcdctl $ETCD_FLAGS endpoint status --write-out=table
etcdctl $ETCD_FLAGS member list
etcdctl $ETCD_FLAGS version

# ── BACKUP ────────────────────────────────────────────────────────────
etcdctl $ETCD_FLAGS snapshot save /opt/etcd-backup.db
etcdctl snapshot status /opt/etcd-backup.db --write-out=table

# ── RESTORE ───────────────────────────────────────────────────────────
etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restore \
  --name=$(hostname) \
  --initial-cluster=$(hostname)=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# ── READING KEYS (debugging) ──────────────────────────────────────────
etcdctl $ETCD_FLAGS get /registry/namespaces/default
etcdctl $ETCD_FLAGS get /registry/pods/ --prefix --keys-only
etcdctl $ETCD_FLAGS get /registry/secrets/ --prefix --keys-only
```

---

## SECTION 19 — Certificates

```bash
# ── CHECK EXPIRY ──────────────────────────────────────────────────────
kubeadm certs check-expiration

# ── RENEW ─────────────────────────────────────────────────────────────
kubeadm certs renew all
kubeadm certs renew apiserver
kubeadm certs renew admin.conf
cp /etc/kubernetes/admin.conf ~/.kube/config  # Update kubeconfig after renew
systemctl restart kubelet

# ── INSPECT MANUALLY ──────────────────────────────────────────────────
openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -dates -subject -issuer

# ── CSR FOR NEW USER ──────────────────────────────────────────────────
openssl genrsa -out user.key 2048
openssl req -new -key user.key \
  -subj "/CN=username/O=groupname" \
  -out user.csr
# Create CertificateSigningRequest object → approve → extract cert
```

---

## SECTION 20 — Output Formats and Filtering

```bash
# ── OUTPUT FORMATS ────────────────────────────────────────────────────
kubectl get pods -o wide              # Extra columns
kubectl get pods -o yaml              # Full YAML
kubectl get pods -o json              # Full JSON
kubectl get pods -o name              # Just resource names
kubectl get pods -o \
  custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# ── JSONPATH ──────────────────────────────────────────────────────────
kubectl get pod <name> -o jsonpath='{.status.podIP}'
kubectl get pod <name> -o jsonpath='{.spec.nodeName}'
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].image}'
kubectl get nodes -o \
  jsonpath='{.items[*].status.addresses[0].address}'
kubectl get pods -o \
  jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# ── SORTING ───────────────────────────────────────────────────────────
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.phase
kubectl get pv --sort-by=.spec.capacity.storage

# ── FIELD SELECTORS ───────────────────────────────────────────────────
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector status.phase!=Running
kubectl get pods --field-selector \
  spec.nodeName=worker-1,status.phase=Running
kubectl get pods --field-selector \
  metadata.namespace=production
kubectl get pvc --field-selector status.phase=Pending
```

---

## SECTION 21 — Helm (Essential Commands)

```bash
# ── REPOS ─────────────────────────────────────────────────────────────
helm repo add <name> <url>
helm repo update
helm repo list
helm repo remove <name>
helm search repo <keyword>
helm search repo <chart> --versions     # All available versions
helm show values <chart>                # Default values

# ── INSTALL / UPGRADE ─────────────────────────────────────────────────
helm install <release> <chart>
helm install <release> <chart> -n <ns> --create-namespace
helm install <release> <chart> --set key=value
helm install <release> <chart> -f values.yaml
helm install <release> <chart> --dry-run --debug
helm upgrade <release> <chart>
helm upgrade <release> <chart> --reuse-values
helm upgrade --install <release> <chart>   # Idempotent

# ── MANAGE RELEASES ───────────────────────────────────────────────────
helm list
helm list -A
helm status <release>
helm history <release>
helm get values <release>
helm get manifest <release>
helm rollback <release> <revision>
helm uninstall <release>
helm uninstall <release> --keep-history

# ── CHART DEVELOPMENT ─────────────────────────────────────────────────
helm create <chart-name>
helm lint <chart-dir>
helm template <release> <chart>         # Preview rendered YAML
helm package <chart-dir>
```

---

## SECTION 22 — Kustomize

```bash
kubectl apply -k <directory>            # Apply kustomized config
kubectl kustomize <directory>           # Preview without applying
kubectl diff -k <directory>             # What would change?
kubectl delete -k <directory>           # Delete kustomized resources
```

---

*This reference card is part of the Kubernetes Mastery Handbook*
*Published by Shaik Dasthagiri — DevOps Engineer*Bengaluru.india
