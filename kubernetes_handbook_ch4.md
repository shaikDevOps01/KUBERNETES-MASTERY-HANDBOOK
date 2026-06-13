# KUBERNETES MASTERY HANDBOOK
# Part 4: Storage
# Chapter 4: Volumes, Persistent Volumes, PVCs, Storage Classes & Dynamic Provisioning

---

> **"Pods are cattle, not pets. But your data? Your data is sacred.
>  Storage is how Kubernetes separates what is ephemeral from what must survive."**

---

## Chapter Introduction

Everything you have built so far has one critical flaw: if a pod restarts, all its data is gone.

By default, a container's filesystem is ephemeral — it exists only as long as the container exists. The moment Kubernetes kills and restarts a container (which it does constantly — for updates, crashes, node failures), every file written inside the container is wiped.

This is fine for stateless apps like a web server serving static content. But databases, message queues, ML model stores, log collectors — these need data that outlives the pods that produce it.

```
╔══════════════════════════════════════════════════════════════════════╗
║              THE STORAGE PROBLEM — VISUALIZED                        ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  WITHOUT STORAGE:                                                    ║
║  Pod (MySQL) writes 10GB to /var/lib/mysql  →  POD CRASHES          ║
║  New Pod (MySQL): /var/lib/mysql is EMPTY. 10GB GONE. ☠️            ║
║                                                                      ║
║  WITH PERSISTENT STORAGE:                                            ║
║  Pod (MySQL) → /var/lib/mysql ←mount→ [AWS EBS / NFS / Ceph]        ║
║  POD CRASHES → New Pod mounts SAME volume → Data intact ✅          ║
║                                                                      ║
║  CHAPTER ROADMAP:                                                    ║
║  Volumes (ephemeral) → PersistentVolumes (cluster storage)          ║
║  → PVClaims (pod requests storage) → StorageClasses (automation)    ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 4.1 Volumes — Pod-Level Storage

### 4.1.1 What Is a Volume?

#### In Plain English

A Volume is a shared folder inside a pod. Containers in the same pod can all read and write to it. Unlike a container's own filesystem, a volume's lifetime is tied to the pod — not the container. So if a container crashes and restarts, the volume data survives. But when the pod itself is deleted, most volume types are also cleaned up.

#### In Technical Language

A Kubernetes Volume is a directory accessible to containers in a pod. It is defined in the pod spec under `volumes` and mounted via `volumeMounts`. The volume's lifecycle depends on its type.

```
╔══════════════════════════════════════════════════════════════════════╗
║                      VOLUME TYPES OVERVIEW                           ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  EPHEMERAL (tied to pod lifecycle):                                  ║
║    emptyDir     Empty dir created when pod starts                    ║
║                 Deleted when pod is removed                          ║
║                 Survives container restarts within pod               ║
║                 Use: scratch space, cache, shared temp files         ║
║    configMap    Mounts a ConfigMap as files                          ║
║    secret       Mounts a Secret as files (tmpfs, in-memory)          ║
║    downwardAPI  Exposes pod metadata as files                        ║
║                                                                      ║
║  NODE-LEVEL (tied to node):                                          ║
║    hostPath     Mounts a directory from the HOST node                ║
║                 DANGEROUS: breaks pod portability                    ║
║                 Use: node-local agents (DaemonSets), testing         ║
║                                                                      ║
║  PERSISTENT (outlives pods and nodes):                               ║
║    persistentVolumeClaim  References a PersistentVolume              ║
║    nfs                    Network File System mount                  ║
║    csi                    Container Storage Interface (modern)       ║
║                                                                      ║
║  Modern Kubernetes: Use CSI drivers + PVC always.                   ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.1.2 emptyDir — The Simplest Volume

```yaml
# emptydir-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - while true; do date >> /shared/timestamps.log; sleep 5; done
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  - name: reader
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - while true; do cat /shared/timestamps.log; sleep 10; done
    volumeMounts:
    - name: shared-data
      mountPath: /shared        # Same volume, same path

  volumes:
  - name: shared-data
    emptyDir:
      {}                        # Default: stored on node disk
      # medium: Memory          # Use RAM (faster, counts against memory limit)
      # sizeLimit: 1Gi
```

### 4.1.3 hostPath — Node Filesystem Access

```yaml
# hostpath-pod.yaml
# WARNING: hostPath breaks pod portability and is a security risk
apiVersion: v1
kind: Pod
metadata:
  name: node-log-reader
spec:
  containers:
  - name: log-reader
    image: busybox
    command: ["/bin/sh", "-c", "tail -f /node-logs/syslog"]
    volumeMounts:
    - name: node-logs
      mountPath: /node-logs
      readOnly: true
  volumes:
  - name: node-logs
    hostPath:
      path: /var/log
      type: Directory            # Directory | File | DirectoryOrCreate | Socket
```

---

## 4.2 PersistentVolumes (PV) — Cluster Storage Resources

### 4.2.1 What Is a PersistentVolume?

#### In Plain English

A PersistentVolume is like a parking space in a large car park. The parking space exists regardless of whether a car is parked there. It has a specific size and type. When a developer needs storage, they file a request (a PVC) — the system finds a matching space and assigns it.

The parking space (PV) is created by the cluster administrator. The request (PVC) is filed by the developer. This separation of concerns is the whole point.

```
╔══════════════════════════════════════════════════════════════════════╗
║           THE PV/PVC SEPARATION OF CONCERNS                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  CLUSTER ADMIN                                                       ║
║    PV: pv-large-ssd   100Gi  ReadWriteOnce  AWS EBS vol-123abc      ║
║    PV: pv-medium-nfs   50Gi  ReadWriteMany  NFS 10.0.0.5:/data      ║
║    PV: pv-small-hdd    10Gi  ReadWriteOnce  GCE pd-disk-xyz         ║
║                                                                      ║
║  DEVELOPER                                                           ║
║    PVC: "I need 80Gi ReadWriteOnce for my database"                 ║
║                                                                      ║
║  KUBERNETES BINDING                                                  ║
║    PVC (80Gi RWO) → pv-large-ssd (100Gi RWO) MATCHES → BOUND       ║
║                                                                      ║
║  POD references PVC name — not PV name directly:                    ║
║    volume:                                                           ║
║      persistentVolumeClaim:                                          ║
║        claimName: my-db-storage                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.2.2 PersistentVolume Lifecycle

```
╔══════════════════════════════════════════════════════════════════════╗
║                    PV LIFECYCLE & RECLAIM POLICIES                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Available → Bound → Released → (Reclaimed or Deleted)              ║
║                                                                      ║
║  Available:   PV exists, no PVC bound                               ║
║  Bound:       PVC claimed this PV (1:1 exclusive mapping)           ║
║  Released:    PVC deleted, PV freed but not yet available            ║
║                                                                      ║
║  RECLAIM POLICY:                                                     ║
║  Retain  → PV stays, data intact, admin manually reclaims           ║
║            SAFE for production databases                             ║
║  Delete  → PV AND underlying storage deleted automatically           ║
║            IRREVERSIBLE data loss — use for non-critical storage     ║
║  Recycle → DEPRECATED. Use dynamic provisioning instead.            ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.2.3 Access Modes

```
╔══════════════════════════════════════════════════════════════════════╗
║                     ACCESS MODES                                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  RWO  ReadWriteOnce    Read-write by ONE node                       ║
║                        Block storage: AWS EBS, GCE PD               ║
║                                                                      ║
║  ROX  ReadOnlyMany     Read-only by MANY nodes                      ║
║                        Use: static content, config                  ║
║                                                                      ║
║  RWX  ReadWriteMany    Read-write by MANY nodes                     ║
║                        Requires: NFS, CephFS, Azure File, EFS       ║
║                                                                      ║
║  RWOP ReadWriteOncePod Read-write by ONE pod only (K8s 1.22+)       ║
║                                                                      ║
║  WARNING: Access mode is the CLAIM, not a guarantee.                ║
║  AWS EBS cannot physically attach to multiple nodes.                ║
║  Claiming RWX on EBS will fail at mount time.                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.2.4 PersistentVolume YAML — All Common Types

```yaml
# PV TYPE 1: NFS (on-premise, supports RWX)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-data
  labels:
    type: nfs
    tier: production
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  mountOptions:
  - hard
  - nfsvers=4.1
  nfs:
    server: 10.0.0.5
    path: /exports/k8s/data

---
# PV TYPE 2: hostPath (single-node dev/testing only)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath-small
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/pv-storage
    type: DirectoryOrCreate

---
# PV TYPE 3: CSI (modern — AWS EBS via CSI driver)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-aws-ebs
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: gp3-ssd
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0abc1234
    fsType: ext4
    volumeAttributes:
      type: gp3
```

---

## 4.3 PersistentVolumeClaims (PVC) — Requesting Storage

### 4.3.1 PVC Binding Process

```
╔══════════════════════════════════════════════════════════════════════╗
║                   PVC BINDING PROCESS                                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PVC SUBMITTED:                                                      ║
║    storageClassName: manual                                          ║
║    accessModes: [ReadWriteOnce]                                      ║
║    storage: 8Gi                                                      ║
║                                                                      ║
║  KUBERNETES SEARCHES FOR MATCHING PV:                               ║
║    storageClass matches + accessMode compatible +                    ║
║    capacity >= requested + status = Available                        ║
║                                                                      ║
║  Available PVs:                                                      ║
║    pv-1  manual  RWO   5Gi   Available  TOO SMALL                   ║
║    pv-2  manual  RWO  10Gi   Available  MATCH ✅                    ║
║    pv-3  fast    RWO  10Gi   Available  WRONG CLASS                  ║
║    pv-4  manual  RWX  20Gi   Available  WRONG MODE                  ║
║                                                                      ║
║  RESULT: PVC bound to pv-2                                          ║
║    PVC gets 10Gi (PV capacity), even though only 8Gi requested      ║
║    pv-2 status → Bound (exclusively reserved for this PVC)          ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.3.2 PVC YAML

```yaml
# persistentvolumeclaim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-pvc
  namespace: production
  labels:
    app: mysql
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: manual        # Must match PV storageClassName
  resources:
    requests:
      storage: 8Gi
  # Optional: select specific PV by label
  selector:
    matchLabels:
      type: nfs
```

### 4.3.3 Using a PVC in a Pod

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  namespace: production
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: root-password
    ports:
    - containerPort: 3306
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: mysql-data-pvc   # Reference the PVC by name
```

---

## 4.4 StorageClasses — Automating Storage Provisioning

### 4.4.1 Static vs Dynamic Provisioning

```
╔══════════════════════════════════════════════════════════════════════╗
║              STATIC vs DYNAMIC PROVISIONING                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  STATIC PROVISIONING (Manual):                                       ║
║  Admin creates PV → Dev creates PVC → Kubernetes binds              ║
║  Problems: operational burden, storage waste, no guarantee PV exists ║
║                                                                      ║
║  DYNAMIC PROVISIONING (StorageClass):                                ║
║  Admin creates StorageClass ONCE → Dev creates PVC →                ║
║  Kubernetes auto-creates EXACTLY the right PV on demand             ║
║                                                                      ║
║  DYNAMIC PROVISIONING FLOW:                                          ║
║                                                                      ║
║  PVC created                                                         ║
║      │                                                               ║
║      ▼                                                               ║
║  StorageClass found (matching name)                                  ║
║      │                                                               ║
║      ▼                                                               ║
║  Provisioner called (e.g. ebs.csi.aws.com)                          ║
║      │                                                               ║
║      ▼                                                               ║
║  Cloud API creates EBS volume (exact size requested)                 ║
║      │                                                               ║
║      ▼                                                               ║
║  PV auto-created and bound to PVC                                    ║
║      │                                                               ║
║      ▼                                                               ║
║  Pod scheduled → volume attached → mounted → Ready ✅               ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.4.2 StorageClass YAML — Multiple Environments

```yaml
# StorageClass 1: AWS EBS gp3 SSD (production)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # Provision after pod scheduled (same AZ)
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  fsType: ext4
  encrypted: "true"

---
# StorageClass 2: GCP Standard Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-pd
provisioner: pd.csi.storage.gke.io
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: pd-standard              # pd-standard | pd-ssd | pd-extreme

---
# StorageClass 3: NFS (on-premise, supports RWX)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-shared
provisioner: nfs.csi.k8s.io
reclaimPolicy: Retain            # SAFER: keep data after PVC deletion
allowVolumeExpansion: false
volumeBindingMode: Immediate
parameters:
  server: 10.0.0.5
  share: /exports/kubernetes

---
# StorageClass 4: Local SSD (high performance, manual provisioning)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer  # CRITICAL for local storage
reclaimPolicy: Delete
```

### 4.4.3 PVC with Dynamic Provisioning

```yaml
# dynamic-pvc.yaml — No PV pre-creation needed!
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-storage
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp3-ssd     # References StorageClass
  resources:
    requests:
      storage: 50Gi             # Gets EXACTLY 50Gi
```

---

## 4.5 Volume Expansion — Resizing PVCs

```bash
# Resize a PVC (StorageClass must have allowVolumeExpansion: true)
kubectl patch pvc database-storage \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# Watch the resize
kubectl get pvc database-storage -w

# For filesystem resize (ext4/xfs) — restart pod
kubectl rollout restart deployment/my-database

# Confirm new size
kubectl get pvc database-storage
```

---

## 4.6 The Complete Storage Architecture

```
╔══════════════════════════════════════════════════════════════════════╗
║            COMPLETE STORAGE STACK IN PRODUCTION                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  CLOUD PROVIDER (AWS/GCP/Azure)                                      ║
║    EBS Volumes / GCE Disks / Azure Disks / NFS                      ║
║         │ managed by                                                 ║
║  CSI DRIVER (aws-ebs-csi-driver / pd-csi / etc.)                    ║
║    Runs as DaemonSet on cluster                                      ║
║    Handles: create/delete/attach/detach/mount/unmount               ║
║         │ creates                                                    ║
║  STORAGECLASS (gp3-ssd)                                              ║
║    "When someone requests storage, call this provisioner"            ║
║         │ triggers                                                   ║
║  PERSISTENTVOLUME (pvc-abc123)                                       ║
║    Auto-created: 50Gi, EBS vol-xyz, RWO, gp3                        ║
║         │ bound to                                                   ║
║  PERSISTENTVOLUMECLAIM (database-storage, namespace: prod)           ║
║    Developer request: 50Gi, RWO, gp3-ssd                            ║
║         │ mounted by                                                 ║
║  POD (mysql-pod, namespace: prod)                                    ║
║    mountPath: /var/lib/mysql                                         ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 4.7 Volume Snapshots — Backup and Restore

```yaml
# STEP 1: VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
driver: ebs.csi.aws.com
deletionPolicy: Retain

---
# STEP 2: Take a snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-backup-2024-01-15
  namespace: production
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: mysql-data

---
# STEP 3: Restore from snapshot into a new PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-restored
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: mysql-storage
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: mysql-backup-2024-01-15
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## 4.8 Real-World Production Example: MySQL with Persistent Storage

```yaml
# production-mysql-complete.yaml

# StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-storage
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain            # NEVER auto-delete DB data!
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"

---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: mysql-storage
  resources:
    requests:
      storage: 100Gi

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: production
type: Opaque
data:
  root-password: c3VwZXJzZWNyZXQ=    # base64("supersecret")
  database: bXlhcHBkYg==              # base64("myappdb")

---
# MySQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: production
spec:
  replicas: 1
  strategy:
    type: Recreate                    # NOT RollingUpdate — RWO disk can't attach to 2 nodes
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: database
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysql", "-h", "localhost", "-uroot",
                      "-p$(MYSQL_ROOT_PASSWORD)", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-data

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

---

## 4.9 Storage Commands

```bash
# PERSISTENTVOLUMES
kubectl get pv                                         # List all PVs (cluster-wide)
kubectl get pv -o wide                                 # With more info
kubectl describe pv my-pv                              # Full PV details
kubectl get pv --sort-by=.spec.capacity.storage        # Sort by size

# PERSISTENTVOLUMECLAIMS
kubectl get pvc                                        # List PVCs in namespace
kubectl get pvc -A                                     # All namespaces
kubectl describe pvc my-pvc                            # Details incl. bound PV
kubectl delete pvc my-pvc                              # Delete PVC

# STORAGECLASSES
kubectl get storageclass                               # List storage classes
kubectl get sc                                         # Shorthand
kubectl describe sc gp3-ssd

# SEE BINDING STATUS
kubectl get pv,pvc                                     # See both together

# FIND PENDING PVCs
kubectl get pvc -A --field-selector status.phase=Pending

# RESIZE PVC
kubectl patch pvc my-pvc \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# CHECK USAGE INSIDE POD
kubectl exec my-pod -- df -h
kubectl exec my-pod -- mount | grep /data
```

---

## Chapter 4: Hands-On Labs

### Lab 4.1 — emptyDir Shared Volume

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: shared-vol-pod
spec:
  containers:
  - name: producer
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - |
      i=0
      while true; do
        echo "Message $i at $(date)" >> /shared/messages.txt
        i=$((i+1))
        sleep 3
      done
    volumeMounts:
    - name: shared
      mountPath: /shared
  - name: consumer
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - while true; do cat /shared/messages.txt 2>/dev/null; sleep 5; done
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
EOF

kubectl logs shared-vol-pod -c consumer -f

# Kill producer — data survives in emptyDir!
kubectl exec shared-vol-pod -c producer -- kill 1
kubectl get pod shared-vol-pod    # Watch restart count
kubectl exec shared-vol-pod -c consumer -- wc -l /shared/messages.txt
# Lines still there even after producer restart!
```

### Lab 4.2 — Static Provisioning with hostPath

```bash
# Create data directory on minikube node
minikube ssh "sudo mkdir -p /data/pv-storage && sudo chmod 777 /data/pv-storage"

# Create PV
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: lab-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/pv-storage
EOF

kubectl get pv lab-pv   # STATUS: Available

# Create PVC
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
EOF

kubectl get pvc lab-pvc   # STATUS: Bound
kubectl get pv lab-pv     # STATUS: Bound, CLAIM: default/lab-pvc

# Use PVC in a pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-test-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c"]
    args:
    - echo "Data written at $(date)" > /data/test.txt; sleep 3600
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: lab-pvc
EOF

kubectl exec pvc-test-pod -- cat /data/test.txt

# Delete pod — recreate — data persists!
kubectl delete pod pvc-test-pod

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-test-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: lab-pvc
EOF

kubectl exec pvc-test-pod -- cat /data/test.txt  # Still there!
```

### Lab 4.3 — Dynamic Provisioning

```bash
# Check available StorageClasses
kubectl get storageclass    # minikube has "standard" (default)

# PVC with dynamic provisioning — no PV pre-created!
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 200Mi
EOF

kubectl get pvc dynamic-pvc    # STATUS: Bound immediately
kubectl get pv                  # Auto-created PV named pvc-xxxxxxxx

# Use in a pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo 'hello dynamic' > /storage/file.txt; sleep 3600"]
    volumeMounts:
    - mountPath: /storage
      name: vol
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: dynamic-pvc
EOF

kubectl exec dynamic-pod -- cat /storage/file.txt

# Test delete behavior
kubectl delete pod dynamic-pod
kubectl delete pvc dynamic-pvc
kubectl get pv    # Auto-deleted (Delete reclaim policy)
```

---

## Chapter 4: Troubleshooting Guide

### Issue 1: PVC stuck in Pending

```bash
kubectl describe pvc my-pvc    # Check Events section!

# CAUSE 1: No matching PV (static provisioning)
# Event: no persistent volumes available for this claim
# Fix: Create PV matching storageClass + accessMode + size

# CAUSE 2: StorageClass does not exist
kubectl get storageclass       # Is the name correct?
# Fix: Create StorageClass or correct the PVC storageClassName

# CAUSE 3: WaitForFirstConsumer — waiting for pod
# Event: waiting for first consumer to be created before binding
# This is NORMAL — just create the pod that uses this PVC

# CAUSE 4: CSI driver not running
kubectl get pods -n kube-system | grep csi
kubectl logs -n kube-system <csi-driver-pod>
```

### Issue 2: Pod stuck in ContainerCreating

```bash
kubectl describe pod my-pod
# Events: Unable to attach or mount volumes

# CAUSE 1: PVC not bound
kubectl get pvc my-pvc         # STATUS must be Bound

# CAUSE 2: Multi-Attach error (RWO volume on wrong node)
# Event: Multi-Attach error for volume "pvc-xxx"
# Volume still attached to old node
# Fix: Delete the old pod gracefully (not force!)
kubectl delete pod old-pod --grace-period=30

# CAUSE 3: Wrong fsGroup permissions
# Fix: Add securityContext to pod spec:
spec:
  securityContext:
    fsGroup: 2000
```

### Issue 3: PVC wont resize

```bash
# Check allowVolumeExpansion
kubectl get sc my-sc -o yaml | grep allowVolumeExpansion
# Must be: true

# Check PVC is Bound (not Pending)
kubectl get pvc my-pvc

# After patch, restart pod for filesystem resize
kubectl rollout restart deployment/my-app
```

### Issue 4: Data not persisted after pod restart

```bash
# Verify you are using PVC not emptyDir
kubectl get pod my-pod -o yaml | grep -A10 volumes

# Verify mount path matches what the app expects
# MySQL: /var/lib/mysql
# PostgreSQL: /var/lib/postgresql/data
# MongoDB: /data/db

# Confirm volume is actually mounted
kubectl exec my-pod -- mount | grep /var/lib/mysql
```

---

## Chapter 4: Interview Questions

**Q1: What is the difference between a PV and a PVC?**

> A PersistentVolume (PV) is a cluster-level storage resource provisioned by an admin representing actual storage (AWS EBS, NFS, etc.). A PersistentVolumeClaim (PVC) is a namespace-scoped request for storage by a user. Kubernetes binds PVCs to matching PVs. The separation lets admins manage infrastructure while developers consume it without knowing the underlying details.

**Q2: What are the access modes and their limitations?**

> RWO (ReadWriteOnce) — read-write by one node; used by block storage like EBS/GCE PD. ROX (ReadOnlyMany) — read-only from many nodes. RWX (ReadWriteMany) — read-write from many nodes; requires network filesystems like NFS, CephFS. RWOP (ReadWriteOncePod, K8s 1.22+) — single pod only. Key trap: access mode is a claim not a guarantee — claiming RWX on EBS fails at mount time because EBS physically cannot attach to multiple nodes.

**Q3: What is a StorageClass and how does dynamic provisioning work?**

> A StorageClass defines a template for dynamically provisioning PVs. It specifies a provisioner (CSI driver), reclaim policy, and parameters (disk type, IOPS). When a PVC references a StorageClass name, Kubernetes calls the provisioner to auto-create a PV of exactly the requested size. This eliminates manual PV pre-creation. Flow: PVC created → StorageClass found → provisioner called → cloud disk created → PV auto-created → PVC bound → pod mounts.

**Q4: What is volumeBindingMode: WaitForFirstConsumer?**

> By default (Immediate), PVs are provisioned immediately when a PVC is created, in any AZ. With block storage (EBS, GCE PD) which is zone-specific, the PV might provision in us-east-1a while the pod schedules to us-east-1b — causing a mount failure. WaitForFirstConsumer delays provisioning until a pod is scheduled, so the provisioner knows which AZ to use. Critical for production cloud clusters.

**Q5: What is the reclaim policy and which should you use in production?**

> Retain keeps the PV and data in Released state — admin manually reclaims it. Safe for databases. Delete automatically deletes the PV and underlying cloud disk — irreversible data loss. Recycle is deprecated. For databases and critical data, always use Retain. Default for dynamically provisioned volumes is typically Delete — override this for production.

**Q6: What is the difference between emptyDir and a PVC?**

> emptyDir is created when a pod starts and deleted when the pod is removed — it survives container restarts but not pod deletion. A PVC references a PersistentVolume that exists independently of any pod — data survives pod deletion, rescheduling, and node failure (for network storage). Use emptyDir for temporary scratch space; use PVCs for any data that must survive pod lifecycle.

**Q7: Can multiple pods use the same PVC?**

> It depends on the access mode. With RWO, only pods on the same node can access simultaneously — pods on different nodes cannot. For concurrent write access from different nodes, use RWX backed by NFS or CephFS. Multiple pods in the same namespace can reference the same PVC if the access mode permits, but StatefulSets typically give each replica its own PVC via volumeClaimTemplates.

**Q8: How do you expand a PVC?**

> Patch the PVC: `kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'`. Prerequisites: StorageClass must have allowVolumeExpansion: true, the CSI driver must support expansion, and the PVC must be Bound. For ext4/xfs filesystem resize, a pod restart is typically required after patching.

---

## CKA Exam Notes — Chapter 4

```
╔════════════════════════════════════════════════════════════════════╗
║                  CKA EXAM FOCUS — CHAPTER 4                       ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: Storage = ~10% of CKA exam                          ║
║  Commonly tested: PVC creation, PV binding, mounting in pods      ║
║                                                                    ║
║  MUST-KNOW COMMANDS:                                               ║
║  k get pv,pvc,sc                                                   ║
║  k describe pvc <name>           always check Events!             ║
║  k get pvc -A --field-selector status.phase=Pending               ║
║  k patch pvc <n> -p '{"spec":{"resources":{"requests":...}}}'     ║
║                                                                    ║
║  PV YAML SKELETON (memorize):                                      ║
║  apiVersion: v1                                                    ║
║  kind: PersistentVolume                                            ║
║  metadata:                                                         ║
║    name: my-pv                                                     ║
║  spec:                                                             ║
║    capacity:                                                       ║
║      storage: 1Gi                                                  ║
║    accessModes:                                                    ║
║    - ReadWriteOnce                                                 ║
║    persistentVolumeReclaimPolicy: Retain                           ║
║    storageClassName: manual                                        ║
║    hostPath:                                                       ║
║      path: /data/pv                                               ║
║                                                                    ║
║  PVC YAML SKELETON (memorize):                                     ║
║  apiVersion: v1                                                    ║
║  kind: PersistentVolumeClaim                                       ║
║  metadata:                                                         ║
║    name: my-pvc                                                    ║
║  spec:                                                             ║
║    accessModes:                                                    ║
║    - ReadWriteOnce                                                 ║
║    storageClassName: manual                                        ║
║    resources:                                                      ║
║      requests:                                                     ║
║        storage: 500Mi                                              ║
║                                                                    ║
║  EXAM TRAPS:                                                       ║
║  storageClassName in PV MUST EXACTLY match PVC (case-sensitive!)  ║
║  PVC accessModes must be SUBSET of PV accessModes                 ║
║  PVC requested size must be <= PV capacity                        ║
║  PVs are cluster-scoped; PVCs are namespace-scoped                ║
║  Use Recreate strategy (not RollingUpdate) for RWO stateful apps  ║
║  hostPath PVs only reliable on single-node clusters (minikube)    ║
║                                                                    ║
║  BINDING TROUBLESHOOT CHECKLIST:                                   ║
║  1. storageClassName matches? (case-sensitive!)                    ║
║  2. accessModes compatible?                                        ║
║  3. PV capacity >= PVC request?                                    ║
║  4. PV status = Available? (not Bound or Released)                ║
║  5. No label selector mismatch?                                    ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Common Mistakes — Chapter 4

| Mistake | Why It Happens | How to Avoid |
|---|---|---|
| storageClassName mismatch | Typo between PV and PVC | Copy-paste the name; verify with `kubectl get sc` |
| RWO with RollingUpdate | Not considering node attachment | Use `strategy: Recreate` for RWO stateful deployments |
| Reclaim policy Delete for prod DB | Default for dynamic provisioning | Always set `reclaimPolicy: Retain` for databases |
| PVC in wrong namespace | PVC namespaced but PV is not | PVC must be in same namespace as the pod |
| No fsGroup for volume permissions | Files created as root | Set `securityContext.fsGroup` in pod spec |
| emptyDir for data that must persist | Quick fix, wrong tool | Any data surviving pod restart needs a PVC |
| Immediate binding on cloud clusters | Default mode | Use `WaitForFirstConsumer` for all cloud block storage |
| hostPath in multi-node cluster | Works on minikube, fails in prod | hostPath is node-specific; use network storage in production |

---

## Chapter 4 Summary

1. **Volumes** — pod-level storage: emptyDir (ephemeral), hostPath (node), configMap/secret (config)
2. **emptyDir** — shared between containers in pod, wiped on pod deletion
3. **PersistentVolumes (PV)** — cluster-level storage resources representing real infrastructure
4. **PV Lifecycle** — Available → Bound → Released, controlled by reclaimPolicy
5. **Access Modes** — RWO (block/one node), ROX (read-only/many), RWX (shared/many)
6. **PersistentVolumeClaims (PVC)** — namespace-scoped storage requests that bind to PVs
7. **StorageClasses** — recipes for dynamic provisioning; eliminates manual PV management
8. **Dynamic Provisioning** — StorageClass + CSI driver auto-creates PVs on demand
9. **volumeBindingMode: WaitForFirstConsumer** — critical for zone-aware cloud block storage
10. **Volume Snapshots** — point-in-time PVC backups for disaster recovery
11. **Volume Expansion** — online PVC resize with allowVolumeExpansion: true

---

*Next: Chapter 5 — Configuration Management: ConfigMaps, Secrets, and Environment Variables*

*"Your app is running with persistent data. Now let's configure it properly —
 without hard-coding passwords and settings into container images."*
