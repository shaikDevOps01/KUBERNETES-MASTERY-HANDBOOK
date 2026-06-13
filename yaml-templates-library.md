# KUBERNETES MASTERY HANDBOOK
# Supplementary: YAML Templates Library

---

> **Copy. Paste. Modify. Apply. These are production-tested templates.**
> Replace ALL values in `<angle-brackets>` with your own values.

---

## 01 — Pod (Minimal)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
  namespace: <namespace>
  labels:
    app: <app-name>
spec:
  containers:
  - name: <container-name>
    image: <image>:<tag>
    ports:
    - containerPort: <port>
```

## 02 — Pod (Production Hardened)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
  namespace: <namespace>
  labels:
    app: <app-name>
    env: production
spec:
  serviceAccountName: <sa-name>
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: <container-name>
    image: <image>:<tag>
    ports:
    - containerPort: <port>
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: [ALL]
    livenessProbe:
      httpGet:
        path: /healthz
        port: <port>
      initialDelaySeconds: 15
      periodSeconds: 20
    readinessProbe:
      httpGet:
        path: /ready
        port: <port>
      initialDelaySeconds: 5
      periodSeconds: 10
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
  terminationGracePeriodSeconds: 30
```

## 03 — Multi-Container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
spec:
  containers:
  - name: main
    image: <main-image>:<tag>
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  - name: sidecar
    image: <sidecar-image>:<tag>
    command: ["<command>"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  volumes:
  - name: shared-data
    emptyDir: {}
```

## 04 — Deployment (Minimal)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <deployment-name>
  namespace: <namespace>
spec:
  replicas: <count>
  selector:
    matchLabels:
      app: <app-name>
  template:
    metadata:
      labels:
        app: <app-name>
    spec:
      containers:
      - name: <container-name>
        image: <image>:<tag>
        ports:
        - containerPort: <port>
```

## 05 — Deployment (Production)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <deployment-name>
  namespace: <namespace>
  labels:
    app: <app-name>
  annotations:
    kubernetes.io/change-cause: "<describe this change>"
spec:
  replicas: <count>
  selector:
    matchLabels:
      app: <app-name>
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 600
  template:
    metadata:
      labels:
        app: <app-name>
        version: "<version>"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "<metrics-port>"
    spec:
      serviceAccountName: <sa-name>
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: <app-name>
      containers:
      - name: <container-name>
        image: <image>:<tag>
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: <port>
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: [ALL]
        readinessProbe:
          httpGet:
            path: /ready
            port: <port>
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /live
            port: <port>
          initialDelaySeconds: 20
          periodSeconds: 20
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
      terminationGracePeriodSeconds: 60
```

## 06 — Service: ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
  namespace: <namespace>
spec:
  type: ClusterIP
  selector:
    app: <app-name>
  ports:
  - name: http
    protocol: TCP
    port: <service-port>
    targetPort: <container-port>
```

## 07 — Service: NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
  namespace: <namespace>
spec:
  type: NodePort
  selector:
    app: <app-name>
  ports:
  - name: http
    protocol: TCP
    port: <service-port>
    targetPort: <container-port>
    nodePort: <30000-32767>
```

## 08 — Service: LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
  namespace: <namespace>
spec:
  type: LoadBalancer
  selector:
    app: <app-name>
  ports:
  - name: http
    port: 80
    targetPort: <container-port>
  - name: https
    port: 443
    targetPort: <container-port>
```

## 09 — Service: Headless (for StatefulSets)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
  namespace: <namespace>
spec:
  clusterIP: None
  selector:
    app: <app-name>
  ports:
  - port: <port>
    targetPort: <port>
```

## 10 — Ingress: Path-Based

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <ingress-name>
  namespace: <namespace>
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: <hostname.example.com>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <frontend-svc>
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: <backend-svc>
            port:
              number: 8080
```

## 11 — Ingress: TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <ingress-name>
  namespace: <namespace>
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - <hostname.example.com>
    secretName: <tls-secret-name>
  rules:
  - host: <hostname.example.com>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <service-name>
            port:
              number: 80
```

## 12 — ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <configmap-name>
  namespace: <namespace>
data:
  DATABASE_HOST: "<db-host>"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "INFO"
  config.properties: |
    server.port=8080
    spring.datasource.url=jdbc:postgresql://${DATABASE_HOST}:${DATABASE_PORT}/mydb
    logging.level.root=${LOG_LEVEL}
```

## 13 — Secret: Opaque

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
  namespace: <namespace>
type: Opaque
# Use 'data' for base64-encoded values:
data:
  DB_PASSWORD: <base64-encoded-value>
# OR use 'stringData' for plain text (auto-encoded):
stringData:
  DB_PASSWORD: "<plain-text-password>"
  API_KEY: "<plain-text-api-key>"
```

## 14 — Secret: TLS

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <tls-secret-name>
  namespace: <namespace>
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

## 15 — PersistentVolume: hostPath

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <pv-name>
spec:
  capacity:
    storage: <size>Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/<path>
    type: DirectoryOrCreate
```

## 16 — PersistentVolume: NFS

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <pv-name>
spec:
  capacity:
    storage: <size>Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-shared
  nfs:
    server: <nfs-server-ip>
    path: /exports/<path>
```

## 17 — PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <pvc-name>
  namespace: <namespace>
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: <storage-class-name>
  resources:
    requests:
      storage: <size>Gi
```

## 18 — StorageClass: AWS EBS gp3

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
```

## 19 — Namespace with ResourceQuota

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace>
  labels:
    env: <environment>
    pod-security.kubernetes.io/enforce: baseline
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: <namespace>-quota
  namespace: <namespace>
spec:
  hard:
    pods: "50"
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: <namespace>-limits
  namespace: <namespace>
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 64Mi
```

## 20 — RBAC: Role + RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: <role-name>
  namespace: <namespace>
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: <rolebinding-name>
  namespace: <namespace>
subjects:
- kind: User
  name: <username>
  apiGroup: rbac.authorization.k8s.io
# OR for ServiceAccount:
# - kind: ServiceAccount
#   name: <sa-name>
#   namespace: <namespace>
roleRef:
  kind: Role
  name: <role-name>
  apiGroup: rbac.authorization.k8s.io
```

## 21 — RBAC: ClusterRole + ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <clusterrole-name>
rules:
- apiGroups: [""]
  resources: ["nodes", "persistentvolumes", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: <clusterrolebinding-name>
subjects:
- kind: User
  name: <username>
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: <clusterrole-name>
  apiGroup: rbac.authorization.k8s.io
```

## 22 — ServiceAccount with RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <sa-name>
  namespace: <namespace>
automountServiceAccountToken: true
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: <sa-name>-role
  namespace: <namespace>
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: <sa-name>-binding
  namespace: <namespace>
subjects:
- kind: ServiceAccount
  name: <sa-name>
  namespace: <namespace>
roleRef:
  kind: Role
  name: <sa-name>-role
  apiGroup: rbac.authorization.k8s.io
```

## 23 — NetworkPolicy: Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: <namespace>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## 24 — NetworkPolicy: Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: <policy-name>
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: <target-app>
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: <allowed-source-app>
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: <allowed-namespace>
    ports:
    - port: <port>
      protocol: TCP
  egress:
  - to: []
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

## 25 — DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: <daemonset-name>
  namespace: <namespace>
spec:
  selector:
    matchLabels:
      app: <app-name>
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: <app-name>
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: <container-name>
        image: <image>:<tag>
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

## 26 — StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <headless-svc-name>
  namespace: <namespace>
spec:
  clusterIP: None
  selector:
    app: <app-name>
  ports:
  - port: <port>
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: <statefulset-name>
  namespace: <namespace>
spec:
  serviceName: <headless-svc-name>
  replicas: <count>
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: <app-name>
  template:
    metadata:
      labels:
        app: <app-name>
    spec:
      containers:
      - name: <container-name>
        image: <image>:<tag>
        ports:
        - containerPort: <port>
        volumeMounts:
        - name: data
          mountPath: /data
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: <storage-class>
      resources:
        requests:
          storage: <size>Gi
```

## 27 — Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: <job-name>
  namespace: <namespace>
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: <container-name>
        image: <image>:<tag>
        command: ["<command>"]
        args: ["<arg1>", "<arg2>"]
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
```

## 28 — CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: <cronjob-name>
  namespace: <namespace>
spec:
  schedule: "<cron-expression>"
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 300
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: <container-name>
            image: <image>:<tag>
            command: ["sh", "-c"]
            args: ["<your-command-here>"]
```

## 29 — HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: <hpa-name>
  namespace: <namespace>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <deployment-name>
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
```

## 30 — PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: <pdb-name>
  namespace: <namespace>
spec:
  minAvailable: 2
  # OR: maxUnavailable: 1
  selector:
    matchLabels:
      app: <app-name>
```

## 31 — Node Affinity + Toleration (Combined)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
spec:
  tolerations:
  - key: "<taint-key>"
    operator: Equal
    value: "<taint-value>"
    effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: <label-key>
            operator: In
            values: [<value1>, <value2>]
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: <app-name>
          topologyKey: kubernetes.io/hostname
  containers:
  - name: <container-name>
    image: <image>:<tag>
```

## 32 — ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: <https://github.com/org/repo.git>
    targetRevision: main
    path: <path/in/repo>
  destination:
    server: https://kubernetes.default.svc
    namespace: <target-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

*This templates library is part of the Kubernetes Mastery Handbook*
*Published by Shaik Dasthagiri-DevOps Engineer - kubernetes-mastery-handbook*
