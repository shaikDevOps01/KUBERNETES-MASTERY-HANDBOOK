# KUBERNETES MASTERY HANDBOOK
# Part 5: Configuration Management
# Chapter 5: ConfigMaps, Secrets & Environment Variables

---

> **"Never bake configuration into your container image.
>  Your image is immutable. Your configuration is not."**

---

## Chapter Introduction

Imagine you have a web application. It needs a database URL, an API key, a log level,
and a feature flag. The naive approach is to hard-code these values into the application
or the Docker image. This is wrong for three reasons:

```
╔══════════════════════════════════════════════════════════════════════╗
║           WHY YOU NEVER HARD-CODE CONFIGURATION                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PROBLEM 1: Environment Drift                                        ║
║  Dev DB URL:  mysql://dev-server:3306/myapp                         ║
║  Prod DB URL: mysql://prod-rds.aws.com:3306/myapp                   ║
║  Hard-coded = different image per environment = chaos               ║
║                                                                      ║
║  PROBLEM 2: Secret Exposure                                          ║
║  API_KEY=sk-abc123 inside your image                                ║
║  → Image pushed to DockerHub → secret is public                    ║
║  → Every layer of the image history stores it forever               ║
║                                                                      ║
║  PROBLEM 3: No Live Updates                                          ║
║  Want to change log level from INFO to DEBUG?                       ║
║  With hard-coded config: rebuild image → redeploy (10 min)          ║
║  With ConfigMap: update one value → app reloads (seconds)           ║
║                                                                      ║
║  THE TWELVE-FACTOR APP RULE #3:                                      ║
║  "Store config in the environment, not in the code."                ║
║                                                                      ║
║  KUBERNETES SOLUTION:                                                ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  Non-sensitive config  →  ConfigMap                          │   ║
║  │  Sensitive config      →  Secret                             │   ║
║  │  Pod metadata          →  Downward API                       │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 5.1 ConfigMaps — Non-Sensitive Configuration

### 5.1.1 What Is a ConfigMap?

#### In Plain English

A ConfigMap is a **dictionary of settings** you keep separate from your application.
Your app reads from it at runtime. The same app image runs in dev, staging, and prod —
only the ConfigMap changes between environments.

#### In Technical Language

A **ConfigMap** is a Kubernetes API object that stores non-confidential data in
key-value pairs. Pods can consume ConfigMaps as:
- Environment variables
- Command-line arguments
- Configuration files in a volume

ConfigMaps are NOT encrypted. Never store passwords or tokens in a ConfigMap.

```
╔══════════════════════════════════════════════════════════════════════╗
║                    CONFIGMAP ANATOMY                                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ConfigMap: app-config                                               ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  KEY                    VALUE                                │   ║
║  │  ─────────────────────  ────────────────────────────────     │   ║
║  │  DATABASE_HOST          mysql-svc.production.svc.cluster.local│  ║
║  │  DATABASE_PORT          3306                                 │   ║
║  │  LOG_LEVEL              INFO                                 │   ║
║  │  FEATURE_DARK_MODE      true                                 │   ║
║  │  MAX_CONNECTIONS        100                                  │   ║
║  │  app.properties         [entire file content as one value]  │   ║
║  │  nginx.conf             [entire nginx config as one value]  │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  THREE WAYS A POD CAN CONSUME A CONFIGMAP:                          ║
║                                                                      ║
║  1. ENV VAR:     DATABASE_HOST=mysql-svc... (single key)            ║
║  2. envFrom:     ALL keys become env vars at once                   ║
║  3. VOLUME:      Each key becomes a FILE inside the container       ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 5.1.2 Creating ConfigMaps

```bash
# METHOD 1: From literal key-value pairs (quick)
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=mysql-svc \
  --from-literal=DATABASE_PORT=3306 \
  --from-literal=LOG_LEVEL=INFO

# METHOD 2: From a .env file
cat > app.env << EOF
DATABASE_HOST=mysql-svc
DATABASE_PORT=3306
LOG_LEVEL=INFO
FEATURE_DARK_MODE=true
EOF
kubectl create configmap app-config --from-env-file=app.env

# METHOD 3: From a single file (key = filename, value = file content)
kubectl create configmap nginx-config --from-file=nginx.conf
# Creates: key="nginx.conf", value=<content of nginx.conf>

# METHOD 4: From a directory (each file becomes a key)
kubectl create configmap all-configs --from-file=./config-dir/
# Creates one key per file in config-dir/

# METHOD 5: From a file with custom key name
kubectl create configmap app-config \
  --from-file=myconfig=./application.properties

# Generate YAML for any of the above:
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=INFO \
  --dry-run=client -o yaml
```

### 5.1.3 ConfigMap YAML — Declarative Definition

```yaml
# configmap-full.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: myapp
    component: config
data:
  # Simple key-value pairs
  DATABASE_HOST: "mysql-svc.production.svc.cluster.local"
  DATABASE_PORT: "3306"
  LOG_LEVEL: "INFO"
  FEATURE_DARK_MODE: "true"
  MAX_CONNECTIONS: "100"
  CACHE_TTL: "3600"

  # Multi-line value — entire config file stored as one key
  # The pipe (|) preserves newlines
  application.properties: |
    spring.datasource.url=jdbc:mysql://${DATABASE_HOST}:${DATABASE_PORT}/myapp
    spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
    logging.level.root=INFO
    server.port=8080
    spring.cache.ttl=3600

  nginx.conf: |
    server {
        listen 80;
        server_name _;
        location / {
            proxy_pass http://backend-svc:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        location /health {
            return 200 'OK';
        }
    }

  # JSON config stored as a value
  feature-flags.json: |
    {
      "darkMode": true,
      "betaFeatures": false,
      "maxRetries": 3,
      "timeout": 30
    }
```

### 5.1.4 Injecting ConfigMaps — All Three Methods

```yaml
# configmap-injection-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-pod
spec:
  containers:
  - name: app
    image: my-app:v1

    # ══════════════════════════════════════════════════════════════
    # METHOD 1: Single env var from specific ConfigMap key
    # ══════════════════════════════════════════════════════════════
    env:
    - name: DB_HOST                 # Name inside the container
      valueFrom:
        configMapKeyRef:
          name: app-config          # ConfigMap name
          key: DATABASE_HOST        # Key inside ConfigMap
          optional: false           # Fail if key missing (default)
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
          optional: true            # Pod starts even if key is missing

    # ══════════════════════════════════════════════════════════════
    # METHOD 2: ALL keys from ConfigMap become env vars (envFrom)
    # ══════════════════════════════════════════════════════════════
    envFrom:
    - configMapRef:
        name: app-config            # All keys → env vars with same names
        optional: false
      prefix: "APP_"               # Optional: prefix all keys (APP_DATABASE_HOST, etc.)
    - configMapRef:
        name: feature-flags-config  # Multiple ConfigMaps supported
    # Result: DATABASE_HOST, DATABASE_PORT, LOG_LEVEL all become env vars

    # ══════════════════════════════════════════════════════════════
    # METHOD 3: ConfigMap as files in a volume
    # ══════════════════════════════════════════════════════════════
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config         # Each key → a file at this path
      readOnly: true
    - name: nginx-config-volume
      mountPath: /etc/nginx/conf.d   # Mount only specific keys as files
      readOnly: true

  volumes:
  # Mount ALL keys as files
  - name: config-volume
    configMap:
      name: app-config
      defaultMode: 0644              # File permissions

  # Mount SPECIFIC keys only
  - name: nginx-config-volume
    configMap:
      name: app-config
      items:                         # Only mount these keys
      - key: nginx.conf
        path: default.conf           # Filename inside mountPath
      - key: feature-flags.json
        path: features.json
```

### 5.1.5 ConfigMap Volume — Live Reload

One of the most powerful features: when a ConfigMap is mounted as a **volume**,
Kubernetes automatically syncs updates within 60 seconds. The files change in the
container WITHOUT restarting the pod — ideal for configuration that apps watch:

```
╔══════════════════════════════════════════════════════════════════════╗
║              CONFIGMAP VOLUME AUTO-UPDATE BEHAVIOUR                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  METHOD       UPDATE BEHAVIOUR                                       ║
║  ──────────── ─────────────────────────────────────────────────     ║
║  Env var      NEVER updated after pod start                         ║
║               App must restart to pick up new value                 ║
║                                                                      ║
║  envFrom      NEVER updated after pod start                         ║
║               App must restart to pick up new value                 ║
║                                                                      ║
║  Volume       AUTO-UPDATED within ~60s (kubelet sync period)        ║
║  (file mount) Files are updated via atomic symlink swap             ║
║               App must WATCH the file to pick up changes            ║
║               (nginx -s reload, Spring Cloud Config Watcher, etc.)  ║
║                                                                      ║
║  FOR IMMEDIATE UPDATE via env:                                       ║
║  kubectl rollout restart deployment/my-app                          ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 5.2 Secrets — Sensitive Data Management

### 5.2.1 What Is a Secret?

#### In Plain English

A Secret is like a **sealed envelope** that Kubernetes passes to your pod.
The envelope contains sensitive information — passwords, API keys, TLS certificates.
Kubernetes handles it differently from ConfigMaps: it stores it in etcd
(optionally encrypted), delivers it in memory (tmpfs, not written to disk),
and restricts who can read it via RBAC.

#### In Technical Language

A **Secret** is a Kubernetes object that holds sensitive data such as passwords,
OAuth tokens, and SSH keys. Secrets reduce the risk of accidentally exposing
sensitive data in pod specifications or container images. By default, Secrets
are stored in etcd as base64-encoded strings (NOT encrypted — base64 is encoding,
not encryption). For true encryption, you must enable **Encryption at Rest**.

```
╔══════════════════════════════════════════════════════════════════════╗
║               CONFIGMAP vs SECRET — KEY DIFFERENCES                  ║
╠═══════════════════════════════╦══════════════════════════════════════╣
║  ConfigMap                    ║  Secret                              ║
╠═══════════════════════════════╬══════════════════════════════════════╣
║  Plain text values            ║  base64-encoded values               ║
║  Stored in etcd (plain)       ║  Stored in etcd (base64, opt.enc.)  ║
║  Shown in kubectl get cm      ║  Hidden in kubectl get secret        ║
║  Not mounted in tmpfs         ║  Mounted in tmpfs (RAM, not disk)    ║
║  No RBAC default restriction  ║  Can restrict via RBAC               ║
║  Use for: URLs, ports, flags  ║  Use for: passwords, tokens, certs  ║
╚═══════════════════════════════╩══════════════════════════════════════╝

IMPORTANT: base64 is NOT encryption!
  echo -n "mypassword" | base64   → bXlwYXNzd29yZA==
  echo bXlwYXNzd29yZA== | base64 -d → mypassword
Anyone with kubectl access can decode secrets unless etcd encryption is enabled.
```

### 5.2.2 Secret Types

```
╔══════════════════════════════════════════════════════════════════════╗
║                     KUBERNETES SECRET TYPES                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  TYPE                              DESCRIPTION                       ║
║  ─────────────────────────────     ──────────────────────────────── ║
║  Opaque                            Arbitrary user-defined data       ║
║  (default)                         Most common — passwords, tokens   ║
║                                                                      ║
║  kubernetes.io/tls                 TLS certificate + private key     ║
║                                    Used by Ingress for HTTPS         ║
║                                                                      ║
║  kubernetes.io/dockerconfigjson    Docker registry credentials       ║
║                                    Used to pull private images       ║
║                                                                      ║
║  kubernetes.io/service-account-token  SA token (auto-created)       ║
║                                                                      ║
║  kubernetes.io/ssh-auth            SSH private key                  ║
║                                                                      ║
║  kubernetes.io/basic-auth          Username + password               ║
║                                                                      ║
║  bootstrap.kubernetes.io/token     Bootstrap tokens (kubeadm)       ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 5.2.3 Creating Secrets

```bash
# METHOD 1: From literal values (base64 encoded automatically)
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=supersecret \
  --from-literal=API_KEY=sk-abc123xyz

# METHOD 2: From a file (e.g., SSH key, TLS cert)
kubectl create secret generic ssh-key-secret \
  --from-file=ssh-privatekey=~/.ssh/id_rsa

# METHOD 3: TLS secret from cert files
kubectl create secret tls myapp-tls \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key

# METHOD 4: Docker registry credentials
kubectl create secret docker-registry regcred \
  --docker-server=registry.company.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=me@company.com

# Verify secret (values are base64 encoded)
kubectl get secret app-secret -o yaml
# data:
#   DB_PASSWORD: c3VwZXJzZWNyZXQ=    ← base64 of "supersecret"
#   API_KEY: c2stYWJjMTIzeHl6

# Decode a secret value
kubectl get secret app-secret \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# Output: supersecret
```

### 5.2.4 Secret YAML — Declarative Definition

```yaml
# secret-opaque.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: production
  labels:
    app: myapp
type: Opaque
data:
  # Values MUST be base64 encoded
  # echo -n "supersecret" | base64  → c3VwZXJzZWNyZXQ=
  DB_PASSWORD: c3VwZXJzZWNyZXQ=
  API_KEY: c2stYWJjMTIzeHl6
  JWT_SECRET: bXlzdXBlcnNlY3JldGp3dGtleQ==

# ALTERNATIVE: Use stringData (plain text — Kubernetes encodes it)
# Easier to write but shows plaintext in the YAML file
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret-plain
  namespace: production
type: Opaque
stringData:                    # Plain text — auto-encoded to base64
  DB_PASSWORD: "supersecret"   # DO NOT commit this to Git!
  API_KEY: "sk-abc123xyz"

---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>

---
# Docker Registry Secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config-json>
```

### 5.2.5 Injecting Secrets — All Methods

```yaml
# secret-injection-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo-pod
spec:
  # Use private registry credentials for image pull
  imagePullSecrets:
  - name: registry-credentials

  containers:
  - name: app
    image: registry.company.com/my-app:v1

    # ══════════════════════════════════════════════════════════════
    # METHOD 1: Single env var from specific Secret key
    # ══════════════════════════════════════════════════════════════
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_PASSWORD
          optional: false
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: API_KEY

    # ══════════════════════════════════════════════════════════════
    # METHOD 2: ALL keys from Secret as env vars
    # ══════════════════════════════════════════════════════════════
    envFrom:
    - secretRef:
        name: app-secret          # All keys become env vars

    # ══════════════════════════════════════════════════════════════
    # METHOD 3: Secret as files in a volume (most secure)
    # ══════════════════════════════════════════════════════════════
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true              # Always read-only for secrets!
    # Result: /etc/secrets/DB_PASSWORD, /etc/secrets/API_KEY as files
    # Files stored in tmpfs (RAM) — never written to disk

  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
      defaultMode: 0400           # Read-only by owner (most restrictive)
      items:                      # Optional: mount specific keys only
      - key: DB_PASSWORD
        path: db-password         # /etc/secrets/db-password
      - key: API_KEY
        path: api-key             # /etc/secrets/api-key
```

```
╔══════════════════════════════════════════════════════════════════════╗
║           SECRET INJECTION — SECURITY COMPARISON                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  METHOD          SECURITY    NOTES                                   ║
║  ─────────────   ─────────   ─────────────────────────────────────  ║
║  env var         MEDIUM      Visible in /proc/PID/environ            ║
║                              Leaked in crash dumps, logs             ║
║                              Simple and widely used                  ║
║                                                                      ║
║  envFrom         MEDIUM      Same risks as env var                   ║
║                              Easiest way to inject many secrets      ║
║                                                                      ║
║  Volume file     BEST        Stored in tmpfs (RAM, not disk)         ║
║                              Not visible in env output               ║
║                              App reads file explicitly               ║
║                              Auto-rotated when Secret is updated     ║
║                              Preferred for production secrets        ║
║                                                                      ║
║  PRODUCTION RECOMMENDATION:                                          ║
║  Use an external secrets manager:                                    ║
║    • HashiCorp Vault + Vault Agent Injector                          ║
║    • AWS Secrets Manager + External Secrets Operator                ║
║    • GCP Secret Manager + External Secrets Operator                 ║
║    • Azure Key Vault + Secrets Store CSI Driver                     ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 5.3 Environment Variables — Direct Injection

### 5.3.1 Types of Environment Variable Sources

```yaml
# all-env-sources.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo-pod
spec:
  containers:
  - name: app
    image: my-app:v1

    env:
    # ── TYPE 1: Hard-coded value (avoid for anything environment-specific)
    - name: APP_VERSION
      value: "1.2.3"

    # ── TYPE 2: From ConfigMap key
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL

    # ── TYPE 3: From Secret key
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_PASSWORD

    # ── TYPE 4: Downward API — Pod's own metadata as env var
    - name: MY_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: MY_POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: MY_NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName

    # ── TYPE 5: Resource info as env var
    - name: MY_CPU_REQUEST
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: requests.cpu
    - name: MY_MEM_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
```

### 5.3.2 Downward API — Pod Self-Awareness

The Downward API lets a pod access information about itself without calling the
Kubernetes API. This is useful for logging, tracing, and self-registration:

```yaml
# downward-api-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-pod
  labels:
    app: myapp
    version: v1
  annotations:
    build-number: "1234"
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /etc/podinfo/labels && sleep 3600"]
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"              # /etc/podinfo/labels
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"         # /etc/podinfo/annotations
        fieldRef:
          fieldPath: metadata.annotations
      - path: "pod-name"
        fieldRef:
          fieldPath: metadata.name
      - path: "pod-namespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "cpu-limit"
        resourceFieldRef:
          containerName: app
          resource: limits.cpu
          divisor: 1m              # Express in millicores
```

---

## 5.4 Projected Volumes — Combining Sources

A **Projected Volume** combines multiple volume sources into a single mount point:

```yaml
# projected-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-vol-pod
spec:
  containers:
  - name: app
    image: my-app:v1
    volumeMounts:
    - name: all-config
      mountPath: /etc/app-config
  volumes:
  - name: all-config
    projected:
      sources:
      - configMap:
          name: app-config
      - secret:
          name: app-secret
          items:
          - key: DB_PASSWORD
            path: secrets/db-password
            mode: 0400
      - downwardAPI:
          items:
          - path: pod-info/name
            fieldRef:
              fieldPath: metadata.name
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: my-app
  # All sources merged into /etc/app-config/ directory
```

---

## 5.5 Production Configuration Strategy

### 5.5.1 The Configuration Hierarchy

```
╔══════════════════════════════════════════════════════════════════════╗
║              PRODUCTION CONFIGURATION DECISION TREE                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Is the data sensitive? (password, token, key, cert)                 ║
║       │                                                              ║
║     YES → Secret                                                     ║
║       │   → For rotation:  External Secrets Operator + Vault/AWS SM ║
║       │   → For TLS certs: cert-manager auto-renews Let's Encrypt   ║
║       │                                                              ║
║     NO  → Is it environment-specific? (dev/staging/prod differ)     ║
║               │                                                      ║
║             YES → ConfigMap (per namespace/environment)              ║
║               │   → Use Kustomize or Helm values.yaml per env       ║
║               │                                                      ║
║             NO  → Is it the same across ALL environments?            ║
║                       │                                              ║
║                     YES → Hard-code in Deployment YAML              ║
║                           (e.g., APP_NAME, SERVER_PORT)              ║
║                     NO  → ConfigMap                                  ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 5.5.2 Full Production Configuration Example

```yaml
# production-app-config.yaml
# Complete configuration stack for a production microservice

---
# Non-sensitive runtime configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-service-config
  namespace: production
data:
  LOG_LEVEL: "INFO"
  SERVER_PORT: "8080"
  METRICS_PORT: "9090"
  DB_HOST: "postgres-svc.production.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "payments"
  REDIS_HOST: "redis-svc.production.svc.cluster.local"
  CACHE_TTL_SECONDS: "300"
  MAX_RETRY_ATTEMPTS: "3"
  PAYMENT_TIMEOUT_MS: "5000"
  application.yml: |
    server:
      port: ${SERVER_PORT:8080}
    spring:
      datasource:
        url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
        hikari:
          maximum-pool-size: 20
      redis:
        host: ${REDIS_HOST}
        ttl: ${CACHE_TTL_SECONDS}
    logging:
      level:
        root: ${LOG_LEVEL}
    management:
      server:
        port: ${METRICS_PORT}

---
# Sensitive credentials
apiVersion: v1
kind: Secret
metadata:
  name: payment-service-secret
  namespace: production
type: Opaque
stringData:
  DB_PASSWORD: "REPLACE_WITH_REAL_PASSWORD"    # Use external secrets in prod!
  STRIPE_API_KEY: "REPLACE_WITH_REAL_KEY"
  JWT_SECRET: "REPLACE_WITH_REAL_JWT_SECRET"

---
# Deployment using both ConfigMap and Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: myregistry/payment-service:v2.1.0
        ports:
        - containerPort: 8080
        - containerPort: 9090
        envFrom:
        - configMapRef:
            name: payment-service-config   # ALL config keys as env vars
        - secretRef:
            name: payment-service-secret   # ALL secret keys as env vars
        env:
        # Override specific values
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        # Mount full application.yml as a file
        - name: app-config-volume
          mountPath: /app/config
          readOnly: true
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 20
      volumes:
      - name: app-config-volume
        configMap:
          name: payment-service-config
          items:
          - key: application.yml
            path: application.yml
```

---

## 5.6 ConfigMap and Secret Commands

```bash
# ── CONFIGMAPS ────────────────────────────────────────────────────────
kubectl get configmaps                     # List ConfigMaps
kubectl get cm                             # Shorthand
kubectl describe cm app-config             # Full details with values
kubectl get cm app-config -o yaml          # YAML output
kubectl edit cm app-config                 # Edit in-place (triggers volume auto-update)
kubectl delete cm app-config

# Create from various sources
kubectl create cm app-config \
  --from-literal=KEY=VALUE \
  --from-file=nginx.conf \
  --from-env-file=app.env

# ── SECRETS ───────────────────────────────────────────────────────────
kubectl get secrets                        # List Secrets (values hidden)
kubectl describe secret app-secret         # Details (values still hidden)
kubectl get secret app-secret -o yaml      # Shows base64-encoded values

# Decode a secret value (two methods)
kubectl get secret app-secret \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

kubectl get secret app-secret \
  -o go-template='{{.data.DB_PASSWORD | base64decode}}'

# Create secrets
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=mypass
kubectl create secret tls tls-secret \
  --cert=cert.crt --key=cert.key
kubectl create secret docker-registry regcred \
  --docker-server=registry.io \
  --docker-username=user \
  --docker-password=pass

# Update a ConfigMap or Secret live
kubectl create cm app-config \
  --from-literal=LOG_LEVEL=DEBUG \
  --dry-run=client -o yaml | kubectl apply -f -

# Quick base64 encoding/decoding
echo -n "mysecret" | base64          # Encode
echo "bXlzZWNyZXQ=" | base64 -d     # Decode
```

---

## Chapter 5: Hands-On Labs

### Lab 5.1 — ConfigMap as File Volume with Live Reload Simulation

```bash
# Step 1: Create a ConfigMap with an nginx config
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-cm
data:
  default.conf: |
    server {
        listen 80;
        location / {
            return 200 'Version 1 - Hello from ConfigMap!\n';
            add_header Content-Type text/plain;
        }
    }
EOF

# Step 2: Deploy nginx using this ConfigMap as a volume
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-configmap-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cm
  template:
    metadata:
      labels:
        app: nginx-cm
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-cm
EOF

kubectl wait --for=condition=ready pod -l app=nginx-cm --timeout=60s

# Step 3: Test it
kubectl port-forward deploy/nginx-configmap-demo 8080:80 &
curl http://localhost:8080     # Should return "Version 1"

# Step 4: Update ConfigMap (triggers file update in ~60s)
kubectl patch configmap nginx-cm --type merge -p '
{"data":{"default.conf":"server {\n    listen 80;\n    location / {\n        return 200 '"'"'Version 2 - Updated from ConfigMap!\\n'"'"';\n        add_header Content-Type text/plain;\n    }\n}"}}'

# Wait ~60 seconds for kubelet sync
sleep 65
# Reload nginx config (app must support live reload)
kubectl exec deploy/nginx-configmap-demo -- nginx -s reload
curl http://localhost:8080     # Should now return "Version 2"
kill %1                        # Stop port forward
```

### Lab 5.2 — Secret Injection and Decoding

```bash
# Step 1: Create a secret with multiple values
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=S3cur3P@ssw0rd \
  --from-literal=host=postgres-svc \
  --from-literal=port=5432

# Step 2: Verify secret is base64 encoded
kubectl get secret db-credentials -o yaml
# data:
#   host: cG9zdGdyZXMtc3Zj
#   password: UzNjdXIzUEBzc3cwcmQ=
#   port: NTQzMg==
#   username: YWRtaW4=

# Step 3: Decode secrets
for key in username password host port; do
  echo -n "$key: "
  kubectl get secret db-credentials \
    -o jsonpath="{.data.$key}" | base64 -d
  echo
done

# Step 4: Inject into a pod and verify
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c"]
    args:
    - |
      echo "=== ENV VAR INJECTION ==="
      echo "DB_HOST: $DB_HOST"
      echo "DB_USER: $DB_USER"
      echo "DB_PASS: $DB_PASS"
      echo ""
      echo "=== VOLUME FILE INJECTION ==="
      ls /etc/db-secrets/
      echo "Password file content:"
      cat /etc/db-secrets/password
      echo ""
      sleep 3600
    env:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: host
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    volumeMounts:
    - name: db-secret-vol
      mountPath: /etc/db-secrets
      readOnly: true
  volumes:
  - name: db-secret-vol
    secret:
      secretName: db-credentials
      defaultMode: 0400
EOF

kubectl logs secret-test-pod

# Step 5: Verify tmpfs mount (in-memory, not on disk)
kubectl exec secret-test-pod -- mount | grep secrets
# Output: tmpfs on /etc/db-secrets type tmpfs
# Confirms it is stored in RAM, not on the node disk
```

### Lab 5.3 — Downward API and Environment Metadata

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-demo
  labels:
    app: my-app
    version: v1.2.3
  annotations:
    build-timestamp: "2024-01-15T10:30:00Z"
    git-commit: "abc1234"
spec:
  containers:
  - name: app
    image: busybox
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 500m
        memory: 256Mi
    command: ["sh", "-c"]
    args:
    - |
      echo "=== Downward API Env Vars ==="
      echo "Pod Name:      $POD_NAME"
      echo "Namespace:     $POD_NAMESPACE"
      echo "Pod IP:        $POD_IP"
      echo "Node Name:     $NODE_NAME"
      echo "CPU Request:   $CPU_REQUEST"
      echo "Mem Limit:     $MEM_LIMIT"
      echo ""
      echo "=== Downward API Volume Files ==="
      cat /etc/podinfo/labels
      echo ""
      cat /etc/podinfo/annotations
      sleep 3600
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: CPU_REQUEST
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: requests.cpu
    - name: MEM_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
EOF

kubectl logs downward-api-demo
```

---

## Chapter 5: Troubleshooting Guide

### Issue 1: Pod fails to start — missing ConfigMap or Secret

```bash
kubectl describe pod my-pod
# Events: Error: configmap "app-config" not found
# Events: Error: secret "app-secret" not found

# Fix: Create the missing resource BEFORE the pod
kubectl get cm app-config -n <namespace>    # Does it exist?
kubectl get secret app-secret -n <namespace>

# Check namespace match — pod and ConfigMap must be in SAME namespace
kubectl get cm -n production
kubectl get pod my-pod -o yaml | grep namespace

# Use optional: true to allow pod to start even if CM/Secret missing
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: LOG_LEVEL
      optional: true              # Pod starts even if CM missing
```

### Issue 2: Wrong value after ConfigMap update

```bash
# Env vars from ConfigMap are NEVER auto-updated after pod start
# You must restart the pod to pick up new env var values

kubectl rollout restart deployment/my-app

# For volume-mounted ConfigMaps, check the sync has happened
kubectl exec my-pod -- cat /etc/config/LOG_LEVEL
# If still old value → wait up to 60s or check kubelet sync period

# Check when file was last modified
kubectl exec my-pod -- ls -la /etc/config/
```

### Issue 3: base64 encoding error in Secret YAML

```bash
# WRONG — value is not base64 encoded
data:
  password: mysecretpassword    # Error: illegal base64 data

# Fix: Encode the value
echo -n "mysecretpassword" | base64
# Output: bXlzZWNyZXRwYXNzd29yZA==

# In the YAML:
data:
  password: bXlzZWNyZXRwYXNzd29yZA==

# OR use stringData (auto-encodes):
stringData:
  password: "mysecretpassword"
```

### Issue 4: Secret value has unexpected newline

```bash
# COMMON MISTAKE: echo without -n adds a newline
echo "mypassword" | base64        # Wrong! Includes \n
echo -n "mypassword" | base64     # Correct! -n = no newline

# To verify your secret has no trailing newline:
kubectl get secret my-secret \
  -o jsonpath='{.data.password}' | base64 -d | xxd | tail -1
# Last byte should NOT be 0a (0a = newline)
```

### Issue 5: ConfigMap or Secret changes not reflected in app

```bash
# For env vars: restart is required
kubectl rollout restart deployment/my-app

# For volume mounts: files update automatically within ~60s
# But the application must actively re-read the file
# Check if your app supports hot-reload (Spring Boot Actuator, etc.)

# Force immediate sync: delete and recreate pod
kubectl delete pod my-pod-xxx    # Deployment auto-recreates it
```

---

## Chapter 5: Interview Questions

**Q1: What is the difference between a ConfigMap and a Secret?**

> ConfigMaps store non-sensitive configuration data as plain text key-value pairs — URLs, ports, feature flags, config files. Secrets store sensitive data — passwords, tokens, certificates — as base64-encoded values. Secrets have additional protections: they are mounted as tmpfs (RAM), restricted by RBAC, and can be encrypted at rest in etcd. Crucially, base64 is NOT encryption — it is only encoding. Both are delivered separately from the container image so the same image can run in any environment.

**Q2: What are the three ways to inject a ConfigMap into a Pod?**

> (1) **env with configMapKeyRef** — inject a single ConfigMap key as a named environment variable. (2) **envFrom with configMapRef** — inject all keys from a ConfigMap as environment variables simultaneously. (3) **Volume mount** — mount the ConfigMap as files in a directory, where each key becomes a filename and the value is the file content. Volume mounts are the only method that auto-updates within ~60 seconds when the ConfigMap changes; env vars require a pod restart.

**Q3: Can you update a ConfigMap and have the pod pick it up without restarting?**

> Only when the ConfigMap is mounted as a volume. Kubernetes (via kubelet) syncs mounted ConfigMap files within approximately 60 seconds using an atomic symlink swap. Env vars injected via `env` or `envFrom` are captured at pod startup and are never updated; those require a pod restart (`kubectl rollout restart`). The application must also support config file watching — nginx, for example, requires `nginx -s reload` even after the files change.

**Q4: Is a Kubernetes Secret actually secure?**

> By default, Secrets are only base64-encoded in etcd — not encrypted. Anyone with etcd access or `kubectl get secret` RBAC permission can read them. To make Secrets truly secure you need: (1) Encryption at Rest for etcd (KMS provider); (2) RBAC policies restricting who can get/list Secrets; (3) Audit logging for Secret access; (4) Preferably, an external secrets manager like HashiCorp Vault, AWS Secrets Manager, or GCP Secret Manager with the External Secrets Operator, which stores actual secrets outside the cluster.

**Q5: What is the Downward API?**

> The Downward API is a mechanism that allows pods to access information about themselves without calling the Kubernetes API server. It can expose pod metadata (name, namespace, labels, annotations), pod status (IP, host IP), and resource limits/requests as environment variables or files in a volume. Common uses: injecting the pod name into logs for tracing, passing namespace to apps that self-register, exposing CPU/memory limits so the app can configure its thread pools accordingly.

**Q6: What is the difference between `data` and `stringData` in a Secret?**

> `data` expects values to be base64-encoded — you must encode manually. `stringData` accepts plain text — Kubernetes automatically encodes it to base64 before storing. When you `kubectl get secret -o yaml`, all values are shown under `data` as base64 regardless of how they were submitted. `stringData` makes writing Secret YAML easier but should never be committed to version control as-is, since the plain text password is visible in the YAML file.

**Q7: How do you rotate a Secret in a running application without downtime?**

> (1) Update the Secret: `kubectl apply -f new-secret.yaml` or patch with new values. (2) If using volume mounts, the files auto-update within 60s; the application must detect the change and reload credentials. (3) If using env vars, perform a rolling restart: `kubectl rollout restart deployment/my-app` — Kubernetes will bring up new pods with the updated Secret before terminating old ones. For zero-downtime rotation on databases, use a dual-credential approach during transition.

**Q8: What is `imagePullSecrets` and when do you need it?**

> `imagePullSecrets` is a list of Secrets referenced in the pod spec that provide credentials for pulling container images from private registries. Without it, Kubernetes can only pull from public registries like Docker Hub. You create a `kubernetes.io/dockerconfigjson` type Secret with registry credentials, then reference it. In production, the best practice is to attach the Secret to the namespace's default ServiceAccount so all pods in that namespace can pull private images automatically.

---

## CKA Exam Notes — Chapter 5

```
╔════════════════════════════════════════════════════════════════════╗
║                  CKA EXAM FOCUS — CHAPTER 5                       ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: Configuration is tested across many tasks.          ║
║  Secrets and ConfigMaps appear in RBAC, pods, deployments.        ║
║                                                                    ║
║  FASTEST COMMANDS IN EXAM:                                         ║
║  k create cm mymap --from-literal=key=val $do > cm.yaml           ║
║  k create secret generic mysec --from-literal=pw=abc $do > s.yaml ║
║                                                                    ║
║  DECODE A SECRET VALUE (one-liner):                                ║
║  k get secret mysec -o jsonpath='{.data.pw}' | base64 -d          ║
║                                                                    ║
║  MUST-KNOW YAML PATTERNS:                                          ║
║  envFrom:                                                          ║
║  - configMapRef:                                                   ║
║      name: my-cm                                                   ║
║  - secretRef:                                                      ║
║      name: my-secret                                               ║
║                                                                    ║
║  volumes:                                                          ║
║  - name: config-vol                                                ║
║    configMap:                                                      ║
║      name: my-cm                                                   ║
║                                                                    ║
║  EXAM TRAPS:                                                       ║
║  base64 encoding: always use echo -n (no newline!)                ║
║  Secret data: values MUST be base64 in data field                 ║
║  ConfigMap + Secret both in same namespace as pod                  ║
║  Volume mount auto-updates; env vars do NOT                        ║
║  optional: true prevents pod crash if CM/Secret missing            ║
║                                                                    ║
║  QUICK ENCODE/DECODE:                                              ║
║  echo -n "value" | base64        ← encode (use in Secret YAML)    ║
║  echo "dmFsdWU=" | base64 -d     ← decode                         ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Common Mistakes — Chapter 5

| Mistake | Why It Happens | How to Avoid |
|---|---|---|
| Storing secrets in Git | Easy to forget stringData is plaintext | Use sealed-secrets, SOPS, or external secrets manager |
| base64 encoding with `echo` (no -n) | Adds newline to secret value | Always `echo -n "value" | base64` |
| Expecting env vars to auto-update | Sees volume files update, assumes same | Only volume mounts auto-update; env vars need pod restart |
| ConfigMap in different namespace | Namespace is overlooked | CM and Pod must be in the same namespace |
| Using Opaque secret for TLS | Works but cert-manager won't manage it | Use `kubernetes.io/tls` type for TLS certificates |
| Hardcoding `stringData` in Git | "I'll remove it before committing" | Use external secrets from day one — not after a breach |

---

## Chapter 5 Summary

1. **ConfigMaps** — store non-sensitive config as key-value pairs; separate config from image
2. **ConfigMap injection** — three methods: single env var, all env vars (envFrom), volume file
3. **Volume auto-update** — mounted ConfigMap files update within ~60s; env vars do not
4. **Secrets** — store sensitive data, base64-encoded; NOT encrypted by default
5. **Secret types** — Opaque, TLS, docker-registry, service-account-token
6. **Secret injection** — same three methods as ConfigMap; volume is most secure (tmpfs)
7. **Downward API** — pods access own metadata (name, namespace, labels, IP) as env/files
8. **Projected volumes** — combine ConfigMap + Secret + DownwardAPI into one mount
9. **Production strategy** — use external secrets managers; never commit secrets to Git
10. **base64 rule** — always `echo -n` to avoid newline; base64 is encoding, not encryption

---

*Next: Chapter 6 — Workload Management: DaemonSets, StatefulSets, Jobs & CronJobs*
