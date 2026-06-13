# KUBERNETES MASTERY HANDBOOK
# Part 6: Workload Management
# Chapter 6: DaemonSets, StatefulSets, Jobs & CronJobs

---

> **"A Deployment asks: how many copies?
>  A DaemonSet asks: which nodes?
>  A StatefulSet asks: in what order, with what identity?
>  A Job asks: did it finish successfully?"**

---

## Chapter Introduction

So far you have used Deployments for everything. But not every workload is
stateless and interchangeable. Kubernetes provides four specialised workload
controllers — each designed for a specific class of application:

```
╔══════════════════════════════════════════════════════════════════════╗
║               THE WORKLOAD SPECTRUM                                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  STATELESS (Deployment)                                              ║
║  ─────────────────────                                               ║
║  Pods are interchangeable. Any pod = any other pod.                 ║
║  Scale up: new pods are identical. Order doesn't matter.            ║
║  Example: web servers, REST APIs, microservices                     ║
║                                                                      ║
║  NODE-BOUND (DaemonSet)                                              ║
║  ──────────────────────                                              ║
║  Exactly ONE pod per node. Follows nodes as they join/leave.        ║
║  Example: log collectors, metric agents, CNI plugins, proxies       ║
║                                                                      ║
║  STATEFUL (StatefulSet)                                              ║
║  ──────────────────────                                              ║
║  Pods have STABLE IDENTITY: fixed name, fixed DNS, fixed storage.   ║
║  Order matters: start in order 0,1,2 — stop in reverse 2,1,0.      ║
║  Example: databases, Kafka, Zookeeper, Redis Cluster, Cassandra     ║
║                                                                      ║
║  BATCH / ONE-OFF (Job)                                               ║
║  ─────────────────────                                               ║
║  Runs to COMPLETION. Success = exit code 0. Not forever.            ║
║  Example: database migrations, batch reports, data pipelines        ║
║                                                                      ║
║  SCHEDULED BATCH (CronJob)                                           ║
║  ─────────────────────────                                           ║
║  Creates Jobs on a cron schedule.                                   ║
║  Example: nightly DB backup, hourly report, daily cleanup           ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 6.1 DaemonSets — One Pod Per Node

### 6.1.1 What Is a DaemonSet?

#### In Plain English

A DaemonSet is a **tax collector** — it visits every house (node) in the city (cluster)
without exception. As new houses are built (nodes added), the tax collector visits them
automatically. When a house is demolished (node removed), the collector leaves.
You cannot have two collectors in one house, and you cannot have a house with no collector.

#### In Technical Language

A **DaemonSet** ensures that a copy of a pod runs on every (or a subset of) nodes
in the cluster. As nodes are added to the cluster, pods are automatically added to them.
As nodes are removed, those pods are garbage collected.

```
╔══════════════════════════════════════════════════════════════════════╗
║                  DAEMONSET — ONE POD PER NODE                        ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Cluster with 4 nodes:                                               ║
║                                                                      ║
║  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  ║
║  │  Node-1     │ │  Node-2     │ │  Node-3     │ │  Node-4     │  ║
║  │  ┌────────┐ │ │  ┌────────┐ │ │  ┌────────┐ │ │  ┌────────┐ │  ║
║  │  │fluentd │ │ │  │fluentd │ │ │  │fluentd │ │ │  │fluentd │ │  ║
║  │  │ pod    │ │ │  │ pod    │ │ │  │ pod    │ │ │  │ pod    │ │  ║
║  │  └────────┘ │ │  └────────┘ │ │  └────────┘ │ │  └────────┘ │  ║
║  │  (app pods) │ │  (app pods) │ │  (app pods) │ │  (app pods) │  ║
║  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘  ║
║                                                                      ║
║  Node-5 ADDED to cluster:                                           ║
║  ┌─────────────┐                                                    ║
║  │  Node-5     │  ← NEW NODE                                        ║
║  │  ┌────────┐ │                                                    ║
║  │  │fluentd │ │  ← DaemonSet pod AUTO-CREATED here                ║
║  │  │ pod    │ │                                                    ║
║  │  └────────┘ │                                                    ║
║  └─────────────┘                                                    ║
║                                                                      ║
║  REAL-WORLD DAEMONSET USE CASES:                                    ║
║  • Log shipping (Fluentd, Fluent Bit, Logstash)                     ║
║  • Metrics collection (Prometheus Node Exporter, Datadog Agent)     ║
║  • Storage daemons (GlusterFS, Ceph, Longhorn)                      ║
║  • Network plugins (Calico node, Cilium agent, kube-proxy)          ║
║  • Security agents (Falco, Sysdig, NeuVector)                       ║
║  • GPU drivers / device plugins                                     ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 6.1.2 DaemonSet YAML

```yaml
# daemonset-fluentd.yaml
# Real-world: Fluentd log collector on every node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-log-collector
  namespace: kube-system
  labels:
    app: fluentd
    component: logging
spec:
  selector:
    matchLabels:
      app: fluentd

  # DaemonSet update strategy
  updateStrategy:
    type: RollingUpdate         # RollingUpdate | OnDelete
    rollingUpdate:
      maxUnavailable: 1         # Update 1 node at a time

  template:
    metadata:
      labels:
        app: fluentd
    spec:
      # DaemonSets often need to run on control-plane nodes too
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule

      # Run before user workloads (higher priority)
      priorityClassName: system-node-critical

      # Access node filesystem for log collection
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluentd-config
        configMap:
          name: fluentd-config

      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8-1
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch-svc.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        resources:
          limits:
            memory: 200Mi
            cpu: 100m
          requests:
            memory: 100Mi
            cpu: 50m
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc/

---
# TARGETED DAEMONSET: Run only on specific nodes (using nodeSelector)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-driver-daemon
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: gpu-driver
  template:
    metadata:
      labels:
        app: gpu-driver
    spec:
      nodeSelector:
        accelerator: nvidia-gpu    # Only on nodes with GPU label
      containers:
      - name: gpu-driver-installer
        image: nvidia/driver:latest
        securityContext:
          privileged: true
```

### 6.1.3 DaemonSet Commands

```bash
# List DaemonSets
kubectl get daemonsets -n kube-system      # System DaemonSets
kubectl get ds                             # Shorthand
kubectl get ds -A                          # All namespaces
kubectl describe ds fluentd-log-collector  # Full details

# Check how many pods are deployed vs desired
kubectl get ds fluentd-log-collector
# NAME                    DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE  NODE SELECTOR
# fluentd-log-collector   4        4        4      4           4          <none>

# Get pods created by DaemonSet
kubectl get pods -l app=fluentd -o wide    # Show which node each pod runs on

# Roll out DaemonSet update
kubectl set image ds/fluentd-log-collector \
  fluentd=fluent/fluentd-kubernetes-daemonset:v1.17

kubectl rollout status ds/fluentd-log-collector
kubectl rollout history ds/fluentd-log-collector
kubectl rollout undo ds/fluentd-log-collector

# Simulate running DaemonSet pod on a specific node
kubectl get pod -l app=fluentd -o wide | grep <node-name>
```

---

## 6.2 StatefulSets — Identity for Stateful Applications

### 6.2.1 What Is a StatefulSet?

#### In Plain English

A StatefulSet is like a **military squad with ranks**. Each soldier has a fixed
rank and number: Private-0, Corporal-1, Sergeant-2. The squad always deploys in
order (0, then 1, then 2). They retreat in reverse order (2, then 1, then 0).
Each soldier always returns to their own bunk (storage). If Private-0 falls,
a replacement is created with the SAME name and rank — not a random new recruit.

#### In Technical Language

A **StatefulSet** manages the deployment and scaling of a set of pods while
providing guarantees about the ordering and uniqueness of these pods. Unlike
Deployments, StatefulSets maintain a sticky identity for each of their pods
through three guarantees:

```
╔══════════════════════════════════════════════════════════════════════╗
║             STATEFULSET THREE CORE GUARANTEES                        ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  GUARANTEE 1: STABLE, UNIQUE NETWORK IDENTITY                        ║
║  ─────────────────────────────────────────────                       ║
║  Pod names are PREDICTABLE: <statefulset-name>-<ordinal>            ║
║  mysql-0, mysql-1, mysql-2   (not random suffixes like Deployments) ║
║                                                                      ║
║  Each pod gets a stable DNS hostname (via Headless Service):        ║
║  mysql-0.mysql-headless.production.svc.cluster.local               ║
║  mysql-1.mysql-headless.production.svc.cluster.local               ║
║  mysql-2.mysql-headless.production.svc.cluster.local               ║
║                                                                      ║
║  Pod mysql-0 is ALWAYS mysql-0, even after restart. The hostname    ║
║  is stable — other pods can hard-reference mysql-0 as the primary.  ║
║                                                                      ║
║  GUARANTEE 2: STABLE, PERSISTENT STORAGE                             ║
║  ────────────────────────────────────────                            ║
║  Each pod gets its OWN PVC via volumeClaimTemplates.                ║
║  mysql-0 → pvc: data-mysql-0  (exclusive, never reassigned)         ║
║  mysql-1 → pvc: data-mysql-1  (exclusive, never reassigned)         ║
║                                                                      ║
║  Even if mysql-0 is rescheduled to a different node, it ALWAYS      ║
║  remounts its own data-mysql-0 PVC. Data follows the pod identity.  ║
║                                                                      ║
║  GUARANTEE 3: ORDERED, GRACEFUL DEPLOYMENT AND SCALING              ║
║  ─────────────────────────────────────────────────────              ║
║  SCALE UP:   0 starts → 0 Ready → 1 starts → 1 Ready → 2 starts    ║
║  SCALE DOWN: 2 stops → 2 deleted → 1 stops → 1 deleted             ║
║  Kubernetes waits for each pod to be Running+Ready before next.     ║
║  This is critical for databases that require a primary to exist     ║
║  before replicas can join the cluster.                              ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 6.2.2 StatefulSet Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════╗
║              STATEFULSET — FULL ARCHITECTURE (MySQL HA)              ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  StatefulSet: mysql  (replicas: 3)                                   ║
║  Headless Service: mysql-headless                                    ║
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  mysql-0  (PRIMARY)                                         │    ║
║  │  DNS: mysql-0.mysql-headless.prod.svc.cluster.local        │    ║
║  │  PVC: data-mysql-0  (10Gi, RWO) ← exclusive storage        │    ║
║  │  Role: Primary (accepts writes)                             │    ║
║  └────────────────────────┬────────────────────────────────────┘    ║
║                           │ replicates to                           ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  mysql-1  (REPLICA)                                         │    ║
║  │  DNS: mysql-1.mysql-headless.prod.svc.cluster.local        │    ║
║  │  PVC: data-mysql-1  (10Gi, RWO) ← own storage              │    ║
║  │  Replicates from: mysql-0.mysql-headless (stable hostname!) │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  mysql-2  (REPLICA)                                         │    ║
║  │  DNS: mysql-2.mysql-headless.prod.svc.cluster.local        │    ║
║  │  PVC: data-mysql-2  (10Gi, RWO) ← own storage              │    ║
║  │  Replicates from: mysql-1.mysql-headless (chained)          │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  Services:                                                           ║
║  mysql-headless (ClusterIP:None) → direct pod DNS (for replication) ║
║  mysql-write    (ClusterIP)      → routes to mysql-0 only (primary) ║
║  mysql-read     (ClusterIP)      → routes to mysql-1, mysql-2       ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 6.2.3 StatefulSet YAML — Production Redis Cluster

```yaml
# statefulset-redis.yaml
# A 3-replica Redis StatefulSet with persistent storage

---
# HEADLESS SERVICE — required for stable pod DNS
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: production
  labels:
    app: redis
spec:
  clusterIP: None             # Headless — DNS returns pod IPs directly
  selector:
    app: redis
  ports:
  - name: redis
    port: 6379
    targetPort: 6379

---
# REGULAR SERVICE — for client access (load balanced)
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379

---
# STATEFULSET
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: production
spec:
  serviceName: redis-headless   # MUST reference the headless service
  replicas: 3

  selector:
    matchLabels:
      app: redis

  # Update strategy for StatefulSets
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0            # Update all pods (0 = all replicas updated)
                              # partition: 2 = only pods 2+ updated (canary)

  # StatefulSet pod management policy
  podManagementPolicy: OrderedReady  # OrderedReady | Parallel
  # OrderedReady: sequential (0 before 1 before 2) — default
  # Parallel: start all simultaneously (faster but no ordering guarantee)

  template:
    metadata:
      labels:
        app: redis
    spec:
      # Init container for configuration
      initContainers:
      - name: config-init
        image: redis:7-alpine
        command: ["sh", "-c"]
        args:
        - |
          # Determine if this is primary (index 0) or replica
          ORDINAL=${HOSTNAME##*-}    # Extract number from redis-0, redis-1, etc.
          if [ "$ORDINAL" = "0" ]; then
            cp /etc/redis-config/primary.conf /etc/redis/redis.conf
          else
            cp /etc/redis-config/replica.conf /etc/redis/redis.conf
            # Point replica to primary using stable DNS
            echo "replicaof redis-0.redis-headless.production.svc.cluster.local 6379" \
              >> /etc/redis/redis.conf
          fi
        volumeMounts:
        - name: redis-config
          mountPath: /etc/redis-config
        - name: redis-conf
          mountPath: /etc/redis

      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server", "/etc/redis/redis.conf"]
        ports:
        - containerPort: 6379
          name: redis
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: data
          mountPath: /data           # Redis persistent data
        - name: redis-conf
          mountPath: /etc/redis
        livenessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 5

      volumes:
      - name: redis-config
        configMap:
          name: redis-config
      - name: redis-conf
        emptyDir: {}

  # VolumeClaimTemplates: Each pod gets its OWN PVC automatically
  volumeClaimTemplates:
  - metadata:
      name: data                 # PVC name = data-redis-0, data-redis-1, data-redis-2
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3-ssd
      resources:
        requests:
          storage: 10Gi
```

### 6.2.4 StatefulSet Scaling and Lifecycle

```bash
# List StatefulSets
kubectl get statefulsets                   # Shorthand: kubectl get sts
kubectl get sts -A
kubectl describe sts redis

# Watch pods come up in ORDER
kubectl get pods -l app=redis -w
# redis-0   0/1   ContainerCreating  ...
# redis-0   1/1   Running            ...  ← 0 is Running before 1 starts
# redis-1   0/1   ContainerCreating  ...
# redis-1   1/1   Running            ...  ← 1 is Running before 2 starts
# redis-2   0/1   ContainerCreating  ...
# redis-2   1/1   Running            ...

# Scale up StatefulSet
kubectl scale sts redis --replicas=5
# Adds redis-3 then redis-4 in order

# Scale down (reverses order: 4 → 3 removed)
kubectl scale sts redis --replicas=3

# Rolling update with partition (canary for StatefulSets)
# Only update pods with ordinal >= partition value
kubectl patch sts redis -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'
kubectl set image sts/redis redis=redis:7.2-alpine
# Only redis-2 updated — redis-0 and redis-1 keep old version
# Verify new version works, then lower partition to 0 for full rollout

# Delete StatefulSet WITHOUT deleting pods (orphan)
kubectl delete sts redis --cascade=orphan

# Access a specific pod directly (via stable DNS)
kubectl exec -it redis-0 -- redis-cli
kubectl exec -it redis-1 -- redis-cli -h redis-0.redis-headless

# Check PVCs created by StatefulSet
kubectl get pvc -l app=redis
# NAME         STATUS  VOLUME        CAPACITY  ACCESS MODES
# data-redis-0 Bound   pvc-aaa...    10Gi      RWO
# data-redis-1 Bound   pvc-bbb...    10Gi      RWO
# data-redis-2 Bound   pvc-ccc...    10Gi      RWO
```

---

## 6.3 Jobs — Run-to-Completion Workloads

### 6.3.1 What Is a Job?

#### In Plain English

A Job is a **one-time contractor** you hire for a specific task. You say:
"I need this data migration done. Run it, confirm it succeeded (exit code 0),
and don't start it again." If the contractor fails (exit code ≠ 0), you hire
another one automatically. Once the job is done, you pay them off and they leave.

#### In Technical Language

A **Job** creates one or more pods and tracks the successful completion of a
specified number of them. When a specified number of successful completions is
reached, the Job is complete. Deleting a Job cleans up its pods.

```
╔══════════════════════════════════════════════════════════════════════╗
║                   JOB EXECUTION PATTERNS                             ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PATTERN 1: Single Job (completions:1, parallelism:1)               ║
║  ─────────────────────────────────────────────────────              ║
║  Run ONE task, confirm success.                                      ║
║  Use: DB migration, one-time setup, manual data fix                 ║
║  [Pod] → succeeds → Job Complete ✅                                 ║
║                                                                      ║
║  PATTERN 2: Multiple completions (completions:5, parallelism:1)     ║
║  ──────────────────────────────────────────────────────────────     ║
║  Run the same task 5 times sequentially.                            ║
║  Use: Sequential processing, ordered batch                          ║
║  [Pod1]→✅ [Pod2]→✅ [Pod3]→✅ [Pod4]→✅ [Pod5]→✅ → Complete     ║
║                                                                      ║
║  PATTERN 3: Parallel Jobs (completions:5, parallelism:3)            ║
║  ────────────────────────────────────────────────────────           ║
║  Run up to 3 pods simultaneously until 5 total succeed.             ║
║  Use: Parallel data processing, parallel testing                    ║
║  [Pod1,Pod2,Pod3] running → Pod1✅ → start Pod4                    ║
║  [Pod2,Pod3,Pod4] running → Pod2✅ → start Pod5                    ║
║  ... until 5 total complete                                         ║
║                                                                      ║
║  PATTERN 4: Indexed Job (completions:3, parallelism:3,              ║
║             completionMode: Indexed)                                 ║
║  ──────────────────────────────────────────────────────             ║
║  Each pod gets a unique index (0,1,2) in JOB_COMPLETION_INDEX env. ║
║  Use: Processing slices of a dataset (pod-0 handles records 0-999, ║
║  pod-1 handles 1000-1999, etc.)                                     ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 6.3.2 Job YAML — All Patterns

```yaml
# PATTERN 1: Single run-to-completion Job
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: production
spec:
  completions: 1              # Number of successful completions needed
  parallelism: 1              # Max pods running in parallel
  backoffLimit: 3             # Retry up to 3 times on failure
  activeDeadlineSeconds: 300  # Kill job after 5 minutes (timeout)
  ttlSecondsAfterFinished: 86400  # Auto-delete job after 24h
  template:
    metadata:
      labels:
        job: db-migration
    spec:
      restartPolicy: OnFailure  # OnFailure | Never (NOT Always)
      # OnFailure: restart container in same pod
      # Never: create new pod on failure (preserves crash logs)
      containers:
      - name: migration
        image: my-app:v2.0
        command: ["python", "manage.py", "migrate"]
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        resources:
          requests:
            cpu: 200m
            memory: 256Mi

---
# PATTERN 3: Parallel Job with work queue
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-image-processor
spec:
  completions: 20             # Process 20 images total
  parallelism: 5              # 5 workers at a time
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: image-processor
        image: my-processor:v1
        command: ["python", "process.py"]
        env:
        - name: QUEUE_URL
          value: "sqs://my-image-queue"

---
# PATTERN 4: Indexed Job (each pod knows its slice)
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-data-processor
spec:
  completions: 6
  parallelism: 3
  completionMode: Indexed      # Each pod gets unique JOB_COMPLETION_INDEX
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: processor
        image: my-processor:v1
        command: ["python", "process_shard.py"]
        env:
        - name: TOTAL_SHARDS
          value: "6"
        # JOB_COMPLETION_INDEX automatically set to 0,1,2,3,4,5
        # Each pod processes shard = JOB_COMPLETION_INDEX
```

### 6.3.3 Job Commands

```bash
# Create a Job
kubectl apply -f db-migration.yaml
kubectl create job manual-migration --image=my-app:v2 \
  -- python manage.py migrate

# Watch Job progress
kubectl get jobs
# NAME           COMPLETIONS   DURATION   AGE
# db-migration   0/1           5s         5s
# db-migration   1/1           42s        42s   ← DONE

kubectl get pods -l job-name=db-migration
# NAME                  READY   STATUS      RESTARTS
# db-migration-xk2n9    0/1     Completed   0

# Get logs from completed job pod
kubectl logs job/db-migration
kubectl logs $(kubectl get pods -l job-name=db-migration -o name)

# Describe job for status and events
kubectl describe job db-migration

# Manually run a job again (create a new Job with timestamp)
kubectl create job rerun-migration-$(date +%s) \
  --from=job/db-migration

# Delete job and its pods
kubectl delete job db-migration

# Delete all completed jobs
kubectl delete jobs --field-selector status.successful=1
```

---

## 6.4 CronJobs — Scheduled Automation

### 6.4.1 What Is a CronJob?

#### In Plain English

A CronJob is an **alarm clock that does something when it goes off**. You set it once:
"Every day at 2 AM, back up the database." The alarm fires, runs the job, then waits
for the next scheduled time. If the alarm fires and the last one is still running,
you can configure whether to skip, stack, or replace it.

#### In Technical Language

A **CronJob** creates Jobs on a repeating schedule. It uses standard Unix cron
syntax and manages the lifecycle of the Jobs it creates, including cleanup of
old Job history.

### 6.4.2 Cron Syntax Reference

```
╔══════════════════════════════════════════════════════════════════════╗
║                    CRON SYNTAX REFERENCE                             ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ┌───────── minute        (0-59)                                    ║
║  │ ┌─────── hour          (0-23)                                    ║
║  │ │ ┌───── day of month  (1-31)                                    ║
║  │ │ │ ┌─── month         (1-12 or JAN-DEC)                        ║
║  │ │ │ │ ┌─ day of week   (0-6 or SUN-SAT, 0=Sunday)              ║
║  │ │ │ │ │                                                          ║
║  * * * * *                                                          ║
║                                                                      ║
║  COMMON SCHEDULES:                                                   ║
║  "0 * * * *"      Every hour at :00                                 ║
║  "*/15 * * * *"   Every 15 minutes                                  ║
║  "0 0 * * *"      Midnight every day                                ║
║  "0 2 * * *"      2 AM every day                                    ║
║  "0 0 * * 0"      Midnight every Sunday                             ║
║  "0 0 1 * *"      Midnight on 1st of every month                   ║
║  "0 8-18 * * 1-5" Every hour 8AM-6PM weekdays                      ║
║  "@hourly"        Alias for "0 * * * *"                             ║
║  "@daily"         Alias for "0 0 * * *"                             ║
║  "@weekly"        Alias for "0 0 * * 0"                             ║
║  "@monthly"       Alias for "0 0 1 * *"                             ║
║                                                                      ║
║  TIP: Use https://crontab.guru to validate expressions!             ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 6.4.3 CronJob YAML — Production Examples

```yaml
# cronjob-db-backup.yaml
# Nightly database backup at 2 AM UTC
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-db-backup
  namespace: production
spec:
  schedule: "0 2 * * *"          # 2 AM every day (UTC)
  timeZone: "Asia/Kolkata"       # K8s 1.27+: specify timezone
  concurrencyPolicy: Forbid      # Forbid | Allow | Replace
  # Forbid:  Skip new run if previous is still running
  # Allow:   Run concurrently (default)
  # Replace: Stop old run, start new one

  successfulJobsHistoryLimit: 3  # Keep last 3 successful job records
  failedJobsHistoryLimit: 1      # Keep last 1 failed job record

  startingDeadlineSeconds: 300   # If schedule missed for 5min, skip
  # e.g., if cluster was down when job was supposed to start,
  # skip that missed run after 5 minutes

  jobTemplate:                   # Template for the Job (not Pod directly)
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600  # Kill if takes > 1 hour
      ttlSecondsAfterFinished: 86400
      template:
        metadata:
          labels:
            job-type: db-backup
        spec:
          restartPolicy: OnFailure
          containers:
          - name: db-backup
            image: postgres:14-alpine
            command: ["sh", "-c"]
            args:
            - |
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              BACKUP_FILE="/backup/db_backup_${TIMESTAMP}.sql.gz"
              echo "Starting backup at $(date)"
              pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | \
                gzip > $BACKUP_FILE
              echo "Backup complete: $BACKUP_FILE"
              # Upload to S3
              aws s3 cp $BACKUP_FILE s3://my-backups/postgres/
              echo "Upload complete at $(date)"
            env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_HOST
            - name: DB_USER
              value: "postgres"
            - name: DB_NAME
              value: "production"
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc

---
# HOURLY CLEANUP JOB
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hourly-temp-cleanup
  namespace: production
spec:
  schedule: "0 * * * *"          # Top of every hour
  concurrencyPolicy: Allow        # Multiple cleanup runs OK
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: busybox:latest
            command: ["sh", "-c"]
            args:
            - find /tmp -type f -mtime +1 -delete && echo "Cleanup done"

---
# REPORT GENERATION — Weekdays at 8 AM
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
  namespace: production
spec:
  schedule: "0 8 * * 1-5"        # 8 AM Monday-Friday
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: report-generator
            image: my-reports:v1
            command: ["python", "generate_daily_report.py"]
            env:
            - name: EMAIL_RECIPIENTS
              value: "team@company.com"
```

### 6.4.4 CronJob Commands

```bash
# List CronJobs
kubectl get cronjobs
kubectl get cj                              # Shorthand
kubectl describe cj nightly-db-backup

# Watch jobs being created by CronJob
kubectl get jobs -w                         # Watch for new jobs
kubectl get jobs --selector=job-name=nightly-db-backup

# Manually trigger a CronJob immediately (without waiting for schedule)
kubectl create job manual-backup-$(date +%s) \
  --from=cronjob/nightly-db-backup
# Very useful for testing CronJob before scheduled time!

# Check status of last run
kubectl get cj nightly-db-backup
# NAME                SCHEDULE    SUSPEND  ACTIVE  LAST SCHEDULE  AGE
# nightly-db-backup   0 2 * * *   False    0       8h             30d

# Suspend a CronJob (pause scheduling)
kubectl patch cj nightly-db-backup -p '{"spec":{"suspend":true}}'

# Resume a suspended CronJob
kubectl patch cj nightly-db-backup -p '{"spec":{"suspend":false}}'

# View history of Jobs created by CronJob
kubectl get jobs -l app=db-backup

# Get logs from latest CronJob run
LATEST_POD=$(kubectl get pods \
  --selector=job-name=$(kubectl get jobs \
    --selector=cronjob-name=nightly-db-backup \
    --sort-by=.metadata.creationTimestamp \
    -o jsonpath='{.items[-1].metadata.name}') \
  -o name)
kubectl logs $LATEST_POD
```

---

## 6.5 Complete Production Example — Kafka with StatefulSet

```yaml
# kafka-statefulset.yaml
# Apache Kafka cluster — the quintessential StatefulSet use case

---
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: data
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - name: client
    port: 9092
  - name: inter-broker
    port: 9093

---
apiVersion: v1
kind: Service
metadata:
  name: kafka-svc
  namespace: data
spec:
  type: ClusterIP
  selector:
    app: kafka
  ports:
  - name: client
    port: 9092

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: data
spec:
  serviceName: kafka-headless
  replicas: 3
  podManagementPolicy: Parallel       # Kafka can start in parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0

  selector:
    matchLabels:
      app: kafka

  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: bitnami/kafka:3.6
        ports:
        - containerPort: 9092
          name: client
        - containerPort: 9093
          name: inter-broker
        env:
        # Kafka uses its hostname (kafka-0, kafka-1, kafka-2) as broker ID
        - name: KAFKA_CFG_NODE_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name  # kafka-0, kafka-1, kafka-2
        - name: KAFKA_CFG_ZOOKEEPER_CONNECT
          value: "zookeeper-headless:2181"
        - name: KAFKA_CFG_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(MY_POD_NAME).kafka-headless.data.svc.cluster.local:9092"
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        volumeMounts:
        - name: kafka-data
          mountPath: /bitnami/kafka
        livenessProbe:
          exec:
            command:
            - kafka-broker-api-versions.sh
            - --bootstrap-server=localhost:9092
          initialDelaySeconds: 30
          periodSeconds: 10

  volumeClaimTemplates:
  - metadata:
      name: kafka-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3-ssd
      resources:
        requests:
          storage: 50Gi
```

---

## Chapter 6: Hands-On Labs

### Lab 6.1 — DaemonSet: Node-Local Monitoring Agent

```bash
# Deploy a simple monitoring DaemonSet that reads node metrics
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  namespace: default
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      containers:
      - name: monitor
        image: busybox
        command: ["sh", "-c"]
        args:
        - |
          while true; do
            echo "=== Node: $NODE_NAME | $(date) ==="
            echo "Load: $(cat /proc/loadavg)"
            echo "Memory: $(cat /proc/meminfo | grep MemAvailable)"
            sleep 30
          done
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          limits:
            cpu: 50m
            memory: 32Mi
        volumeMounts:
        - name: proc
          mountPath: /proc
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
EOF

kubectl get ds node-monitor
kubectl get pods -l app=node-monitor -o wide   # One pod per node!
kubectl logs -l app=node-monitor --prefix=true  # Logs from all pods
```

### Lab 6.2 — StatefulSet: Ordered Identity Demonstration

```bash
# Create a StatefulSet that demonstrates ordering
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: demo-headless
spec:
  clusterIP: None
  selector:
    app: demo-sts
  ports:
  - port: 80

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: demo-sts
spec:
  serviceName: demo-headless
  replicas: 3
  selector:
    matchLabels:
      app: demo-sts
  template:
    metadata:
      labels:
        app: demo-sts
    spec:
      containers:
      - name: demo
        image: nginx:alpine
        command: ["sh", "-c"]
        args:
        - |
          echo "Pod $HOSTNAME starting at $(date)" > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 100Mi
EOF

# Watch ordered startup
kubectl get pods -l app=demo-sts -w
# demo-sts-0  Pending → ContainerCreating → Running
# demo-sts-1  (only starts AFTER demo-sts-0 is Running+Ready)
# demo-sts-2  (only starts AFTER demo-sts-1 is Running+Ready)

# Verify stable DNS names
kubectl run dns-test --image=busybox --rm -it --restart=Never -- sh
  nslookup demo-sts-0.demo-headless       # Resolves to pod-0 IP
  nslookup demo-sts-1.demo-headless       # Resolves to pod-1 IP
  nslookup demo-headless                  # Returns ALL pod IPs
  wget -qO- demo-sts-0.demo-headless      # Talk to specific pod!
  exit

# Verify each pod has its OWN PVC
kubectl get pvc -l app=demo-sts
# data-demo-sts-0  Bound  ...  100Mi
# data-demo-sts-1  Bound  ...  100Mi
# data-demo-sts-2  Bound  ...  100Mi

# Scale down and verify ORDER (2 deleted first, then 1, then 0)
kubectl scale sts demo-sts --replicas=1
kubectl get pods -l app=demo-sts -w   # Watch 2 → 1 → scaled to 1
```

### Lab 6.3 — Job: Database Migration Pattern

```bash
# Simulate a database migration job
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: db-schema-migration
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 120
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrator
        image: busybox
        command: ["sh", "-c"]
        args:
        - |
          echo "Starting migration at $(date)"
          echo "Step 1: Connecting to database..."
          sleep 2
          echo "Step 2: Running schema migration..."
          sleep 3
          echo "Step 3: Seeding initial data..."
          sleep 2
          echo "Migration completed successfully at $(date)"
          exit 0    # Change to exit 1 to test failure behavior
EOF

# Watch the job
kubectl get job db-schema-migration -w
kubectl logs job/db-schema-migration -f

# Test failure and retry
kubectl delete job db-schema-migration
# Change exit 0 to exit 1 in the YAML above, then reapply
# Watch backoffLimit kick in — job retries 3 times then fails
```

### Lab 6.4 — CronJob: Scheduled Task

```bash
# Create a CronJob that runs every minute (for quick testing)
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: minute-reporter
spec:
  schedule: "*/1 * * * *"       # Every minute (testing only!)
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: reporter
            image: busybox
            command: ["sh", "-c"]
            args:
            - echo "Report generated at $(date). Cluster is healthy!"
EOF

kubectl get cj minute-reporter
kubectl get jobs -w                          # Watch new jobs appear every minute

# After 3 runs, see history
kubectl get jobs --sort-by=.metadata.creationTimestamp

# Manually trigger
kubectl create job manual-report \
  --from=cronjob/minute-reporter

# Suspend to stop the schedule
kubectl patch cj minute-reporter -p '{"spec":{"suspend":true}}'

# Cleanup
kubectl delete cj minute-reporter
```

---

## Chapter 6: Troubleshooting Guide

### Issue 1: StatefulSet pods stuck in Pending

```bash
kubectl describe pod my-sts-1     # Check Events

# COMMON CAUSE: PVC from volumeClaimTemplate cannot be bound
kubectl get pvc -l app=my-app
# data-my-sts-0  Pending
# → StorageClass not found or PV not available

# Fix: Ensure StorageClass exists
kubectl get sc
# Fix: Ensure PV has capacity (static provisioning)

# COMMON CAUSE: Previous pod not Ready (ordered startup)
kubectl get pods -l app=my-app
# my-sts-0  0/1  CrashLoopBackOff ← Pod 1 won't start until 0 is Ready
kubectl logs my-sts-0   # Fix pod-0 first!
```

### Issue 2: Job keeps restarting / never completes

```bash
kubectl describe job my-job       # Check Events and status
kubectl get pods -l job-name=my-job

# WRONG restartPolicy (Deployments use Always, Jobs cannot!)
# Error: spec.template.spec.restartPolicy: Unsupported value "Always"
# Fix: Set restartPolicy to OnFailure or Never in job template

# Job pod always fails — check logs
kubectl logs $(kubectl get pods -l job-name=my-job -o name | head -1)

# backoffLimit exceeded — job gives up
kubectl describe job my-job | grep -A5 "Events:"
# Fix: Increase backoffLimit or fix the underlying issue

# activeDeadlineSeconds exceeded — job killed by timeout
# Fix: Increase activeDeadlineSeconds or optimise the task
```

### Issue 3: CronJob not creating Jobs

```bash
kubectl describe cj my-cronjob
kubectl get cj my-cronjob
# Check: SUSPEND = True? → Unsuspend with patch

# CronJob missed schedule — check startingDeadlineSeconds
# If cluster was down or CronController was off for > startingDeadlineSeconds,
# the run is skipped

# concurrencyPolicy: Forbid + previous job still running
kubectl get jobs -l cronjob-name=my-cronjob
# If old job is still running, new one won't start
# Fix: Delete stuck old job or change to Allow/Replace

# Wrong timezone — CronJob uses UTC by default
# Schedule "0 9 * * *" fires at 9 AM UTC, not 9 AM IST
# Fix: Use timeZone: "Asia/Kolkata" in spec (K8s 1.27+)
# Or adjust schedule: IST = UTC+5:30, so 9 AM IST = "30 3 * * *"
```

### Issue 4: DaemonSet pod not created on a node

```bash
kubectl get ds my-daemonset       # Check DESIRED vs CURRENT vs READY
kubectl describe ds my-daemonset  # Check Events and node selectors

# Node has a taint that DaemonSet doesn't tolerate
kubectl describe node my-node | grep Taints
# Taint: node-role.kubernetes.io/control-plane:NoSchedule
# Fix: Add matching toleration to DaemonSet spec

# DaemonSet has nodeSelector that doesn't match node labels
kubectl get node my-node --show-labels
# Fix: Add required label to node or remove nodeSelector
kubectl label node my-node disk=ssd

# DaemonSet pod is Pending on the node — resource exhaustion
kubectl describe pod my-ds-pod   # Events: insufficient CPU/memory
# Fix: Reduce DaemonSet pod resource requests
```

---

## Chapter 6: Interview Questions

**Q1: What is a DaemonSet and when would you use it over a Deployment?**

> A DaemonSet ensures exactly one pod runs on every (or selected) node in the cluster. New nodes automatically get the pod; removed nodes lose it. Use a DaemonSet when you need a per-node agent: log collectors (Fluentd, Fluent Bit), metrics exporters (Node Exporter), storage daemons (Ceph, GlusterFS), CNI plugins, or security agents. Use a Deployment when you want a specific number of replicas that can run anywhere.

**Q2: What are the three guarantees of a StatefulSet?**

> (1) Stable, unique network identity — pods are named `<sts-name>-<ordinal>` (e.g., mysql-0, mysql-1) and get stable DNS via a headless service. (2) Stable, persistent storage — each pod gets its own PVC from `volumeClaimTemplates` that is never reassigned; data follows pod identity across restarts. (3) Ordered, graceful deployment and scaling — pods start in order (0, 1, 2), each waiting for the previous to be Running+Ready; they terminate in reverse order (2, 1, 0).

**Q3: What is the difference between a StatefulSet and a Deployment?**

> Deployments manage interchangeable pods — any pod is equivalent to any other. StatefulSets manage pods with unique identities. Key differences: StatefulSet pods have predictable names (not random suffixes), stable DNS hostnames via headless service, individual PVCs per pod that persist across restarts, and ordered startup/shutdown. Deployments are for stateless workloads (web servers, APIs). StatefulSets are for stateful workloads (databases, message brokers, distributed caches).

**Q4: What is a headless service and why does a StatefulSet need one?**

> A headless service has `clusterIP: None`. Instead of creating a single virtual IP, DNS queries return individual pod IPs. StatefulSets require a headless service to give each pod a stable, addressable DNS hostname: `pod-0.my-headless-svc.namespace.svc.cluster.local`. This is how mysql-1 knows to replicate from mysql-0 using the stable hostname regardless of which node mysql-0 runs on or what its current IP is.

**Q5: What is the difference between a Job and a Deployment?**

> A Deployment runs pods continuously and restarts them if they exit. A Job runs pods to completion — it expects pods to exit with code 0 and considers the task done. A Deployment has `restartPolicy: Always`; a Job requires `restartPolicy: OnFailure` or `Never`. Jobs are for finite tasks (migrations, batch processing, reports). Deployments are for long-running services (web servers, APIs).

**Q6: What do `completions`, `parallelism`, and `backoffLimit` mean in a Job?**

> `completions` is the total number of successful pod completions required for the Job to be considered done. `parallelism` is the maximum number of pods that can run simultaneously. `backoffLimit` is the number of times Kubernetes will retry a failed pod before giving up on the Job (marking it Failed). Example: completions=10, parallelism=3 runs 3 pods at a time until 10 total succeed.

**Q7: What is `concurrencyPolicy` in a CronJob?**

> `concurrencyPolicy` controls what happens if a new scheduled run starts while the previous one is still running. `Allow` (default) — let both run concurrently. `Forbid` — skip the new run if old one is still running (prevents overlapping backups). `Replace` — terminate the running job and start the new one (ensures only the latest run is active). For database backups, use Forbid to prevent simultaneous backups corrupting each other.

**Q8: What happens to StatefulSet PVCs when you delete the StatefulSet?**

> By default, deleting a StatefulSet does NOT delete its PVCs. The `volumeClaimTemplates` creates PVCs that have their own lifecycle. When you delete a StatefulSet (even with `--cascade=foreground`), the PVCs remain. This is intentional — it protects data from accidental deletion. To clean up PVCs you must delete them manually: `kubectl delete pvc -l app=my-sts`. This is why StatefulSets with Retain reclaim policy are doubly safe for databases.

---

## CKA Exam Notes — Chapter 6

```
╔════════════════════════════════════════════════════════════════════╗
║                  CKA EXAM FOCUS — CHAPTER 6                       ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: Workloads & Scheduling = ~15% of CKA               ║
║  StatefulSets and Jobs are frequently tested                      ║
║                                                                    ║
║  DAEMONSET — key facts for exam:                                  ║
║  k get ds -A                  ← see all daemonsets               ║
║  k get ds -o wide             ← see desired/current counts       ║
║  Use tolerations to run on control-plane nodes                    ║
║  nodeSelector to run on subset of nodes                           ║
║                                                                    ║
║  STATEFULSET — key facts for exam:                                ║
║  MUST have serviceName matching a headless service                ║
║  restartPolicy cannot be in StatefulSet pod template              ║
║  k scale sts <name> --replicas=N                                  ║
║  k get pods -l app=myapp -w   ← watch ordered startup            ║
║                                                                    ║
║  JOB — YAML skeleton (memorize!):                                 ║
║  apiVersion: batch/v1                                             ║
║  kind: Job                                                        ║
║  metadata:                                                        ║
║    name: my-job                                                   ║
║  spec:                                                            ║
║    completions: 1                                                 ║
║    parallelism: 1                                                 ║
║    backoffLimit: 3                                                ║
║    template:                                                      ║
║      spec:                                                        ║
║        restartPolicy: OnFailure   ← NOT Always!                  ║
║        containers:                                                ║
║        - name: worker                                             ║
║          image: busybox                                           ║
║          command: ["sh", "-c", "echo done"]                       ║
║                                                                   ║
║  CRONJOB — key facts for exam:                                    ║
║  k create job manual --from=cronjob/my-cj  ← manual trigger!    ║
║  k patch cj my-cj -p '{"spec":{"suspend":true}}'  ← pause       ║
║  concurrencyPolicy: Forbid  ← most common exam answer            ║
║                                                                   ║
║  EXAM TRAPS:                                                      ║
║  Job restartPolicy MUST be OnFailure or Never, never Always       ║
║  StatefulSet needs headless service (clusterIP: None)             ║
║  DaemonSets ignore replicas field — it runs on nodes, not a count ║
║  CronJob schedule is UTC by default unless timeZone set           ║
║  Deleting StatefulSet does NOT delete PVCs                        ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Common Mistakes — Chapter 6

| Mistake | Why It Happens | How to Avoid |
|---|---|---|
| Job with `restartPolicy: Always` | Copying from Deployment | Jobs must use `OnFailure` or `Never` |
| StatefulSet without headless service | serviceName field overlooked | Always create headless service first, reference in serviceName |
| Expecting PVCs to auto-delete with StatefulSet | Intuition from Deployments | PVCs from volumeClaimTemplates must be deleted manually |
| CronJob schedule in local time | UTC is default | Add `timeZone` field or convert to UTC manually |
| DaemonSet not running on control-plane | Missing tolerations | Add NoSchedule toleration for control-plane taint |
| No backoffLimit on Job | Default is 6 retries | Set explicit backoffLimit; infinite retries waste resources |
| StatefulSet scale-down data loss | Deleting pod-0 first | Kubernetes always deletes highest-ordinal first; don't force delete |
| Not checking DESIRED vs CURRENT in DaemonSet | Assuming all pods scheduled | Check `kubectl get ds` — DESIRED vs CURRENT can reveal scheduling issues |

---

## Chapter 6 Summary

1. **Workload spectrum** — Deployment (stateless) → DaemonSet (node-bound) → StatefulSet (stateful) → Job/CronJob (batch)
2. **DaemonSets** — one pod per node, auto-created on new nodes, used for agents/collectors
3. **DaemonSet tolerations** — needed to schedule on control-plane/tainted nodes
4. **StatefulSets** — three guarantees: stable identity, stable storage, ordered lifecycle
5. **Headless service** — required by StatefulSets for stable pod DNS; `clusterIP: None`
6. **volumeClaimTemplates** — auto-creates individual PVCs per pod; NOT deleted with StatefulSet
7. **StatefulSet partition** — enables canary updates by updating only high-ordinal pods
8. **Jobs** — run to completion; `restartPolicy: OnFailure/Never`; completions + parallelism + backoffLimit
9. **Job patterns** — single, multiple sequential, parallel, indexed
10. **CronJobs** — create Jobs on cron schedule; concurrencyPolicy controls overlapping runs
11. **Manual trigger** — `kubectl create job manual --from=cronjob/<name>` for immediate testing

---

*Next: Chapter 7 — Scheduling: Node Selectors, Affinity, Taints & Tolerations, Resource Management*

*"Your workloads are running. Now let's learn to control exactly WHERE and HOW
 Kubernetes places them — and what happens when resources run out."*
