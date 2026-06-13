# KUBERNETES MASTERY HANDBOOK
# Part 9: Monitoring and Logging
# Chapter 9: Metrics Server, Prometheus, Grafana & Logging Architecture

---

> **"You cannot fix what you cannot see.
>  In Kubernetes, observability is not optional — it is how you sleep at night."**

---

## Chapter Introduction

Your cluster is running. Your apps are deployed. Security is in place.
But how do you know if everything is actually *working well*?

How do you know when a node is about to run out of memory?
When a pod is quietly crashing every 6 hours?
When your API response time doubled at 2 AM?
When the disk on a worker node is 95% full?

**Observability** answers these questions. It has three pillars:

```
╔══════════════════════════════════════════════════════════════════════╗
║              THE THREE PILLARS OF OBSERVABILITY                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  METRICS — What is the system doing RIGHT NOW?                       ║
║  ─────────────────────────────────────────────                       ║
║  Numerical measurements over time                                   ║
║  CPU usage, memory usage, request rate, error rate, latency         ║
║  Tools: Metrics Server, Prometheus, Grafana                         ║
║  Retention: Days to months (aggregated)                             ║
║                                                                      ║
║  LOGS — What happened and WHEN?                                      ║
║  ──────────────────────────────                                      ║
║  Text records of discrete events                                    ║
║  "User alice logged in", "Database query failed", "Pod restarted"   ║
║  Tools: kubectl logs, Fluentd, Fluent Bit, EFK/PLG Stack            ║
║  Retention: Days to weeks (expensive to store)                      ║
║                                                                      ║
║  TRACES — How did a request flow through ALL services?               ║
║  ─────────────────────────────────────────────────────              ║
║  Distributed request tracing across microservices                   ║
║  "Request ID xyz: 5ms in frontend → 120ms in auth → 800ms in DB"   ║
║  Tools: Jaeger, Zipkin, OpenTelemetry                               ║
║  (Covered in Platform Engineering — outside CKA scope)              ║
║                                                                      ║
║  KUBERNETES MONITORING STACK:                                        ║
║  ┌────────────────────────────────────────────────────────────────┐ ║
║  │  kubectl top     — Quick CLI metrics (via Metrics Server)     │ ║
║  │  Metrics Server  — Short-term in-memory metrics (HPA source)  │ ║
║  │  Prometheus      — Long-term metrics storage + alerting       │ ║
║  │  Grafana         — Dashboards and visualisation               │ ║
║  │  AlertManager    — Alert routing and notifications            │ ║
║  │  Fluent Bit      — Lightweight log collector (per node)       │ ║
║  │  Elasticsearch   — Log storage and search (EFK stack)         │ ║
║  │  Kibana          — Log visualisation (EFK stack)              │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 9.1 Metrics Server — The Foundation

### 9.1.1 What Is Metrics Server?

#### In Plain English

Metrics Server is a **lightweight speedometer** for your cluster. It collects
CPU and memory usage from every node and pod, stores it briefly in memory,
and makes it available via the Kubernetes API. It is what powers `kubectl top`
and what the **Horizontal Pod Autoscaler (HPA)** uses to decide when to scale.

#### In Technical Language

**Metrics Server** is a scalable, efficient source of container resource
metrics. It collects metrics from the Summary API exposed by kubelets on each
node and exposes them through the Kubernetes Metrics API
(`metrics.k8s.io/v1beta1`). It stores metrics in memory only — no historical
data, no persistence, no long-term storage.

```
╔══════════════════════════════════════════════════════════════════════╗
║                  METRICS SERVER ARCHITECTURE                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  EVERY NODE:                                                         ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  kubelet                                                    │    ║
║  │    │                                                        │    ║
║  │    └── cAdvisor (container metrics collector)               │    ║
║  │          │ CPU usage, memory usage, filesystem, network     │    ║
║  │          └── Exposed at: /stats/summary                     │    ║
║  └──────────────────────────────┬──────────────────────────────┘    ║
║                                 │ scrapes every 60s                  ║
║  ┌──────────────────────────────▼──────────────────────────────┐    ║
║  │  METRICS SERVER POD (in kube-system)                        │    ║
║  │  Collects → Aggregates → Stores in MEMORY (no disk)         │    ║
║  │  Exposes: /apis/metrics.k8s.io/v1beta1                      │    ║
║  └──────────────────────────────┬──────────────────────────────┘    ║
║                                 │                                    ║
║              ┌──────────────────┼──────────────────┐                ║
║              ▼                  ▼                  ▼                ║
║      kubectl top nodes   kubectl top pods    HPA Controller         ║
║      (human readable)    (human readable)   (auto-scaling)          ║
║                                                                      ║
║  WHAT METRICS SERVER PROVIDES:                                       ║
║  • Current CPU usage per pod/node (in millicores)                   ║
║  • Current memory usage per pod/node (in bytes)                     ║
║  WHAT IT DOES NOT PROVIDE:                                           ║
║  • Historical data (no time series)                                 ║
║  • Custom metrics (only CPU + memory)                               ║
║  • Long-term storage (in-memory only)                               ║
║  • Alerting                                                          ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 9.1.2 Installing Metrics Server

```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For local clusters (minikube, kind) — needs insecure TLS flag
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for local clusters (minikube/kind — self-signed certs)
kubectl patch deployment metrics-server \
  -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-",
       "value":"--kubelet-insecure-tls"}]'

# Enable in minikube directly
minikube addons enable metrics-server

# Verify installation
kubectl get pods -n kube-system | grep metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io
# Should show: AVAILABLE = True

# Wait for metrics to be available (60-90 seconds after install)
watch kubectl top nodes
```

### 9.1.3 kubectl top — Using Metrics

```bash
# ── NODE METRICS ──────────────────────────────────────────────────────
kubectl top nodes
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# worker-1   250m         12%    1024Mi          52%
# worker-2   180m         9%     896Mi           45%
# worker-3   420m         21%    1536Mi          78%

kubectl top nodes --sort-by=cpu        # Sort by CPU usage
kubectl top nodes --sort-by=memory     # Sort by memory usage

# ── POD METRICS ───────────────────────────────────────────────────────
kubectl top pods                        # Current namespace
kubectl top pods -n production          # Specific namespace
kubectl top pods -A                     # All namespaces
kubectl top pods -A --sort-by=cpu       # Highest CPU first
kubectl top pods -A --sort-by=memory    # Highest memory first

# Per-container breakdown
kubectl top pods --containers
# POD                  NAME          CPU(cores)  MEMORY(bytes)
# my-app-xyz           app           45m         128Mi
# my-app-xyz           sidecar       5m          32Mi

# Find resource hogs
kubectl top pods -A --sort-by=memory | head -10
kubectl top pods -A --sort-by=cpu | head -10

# ── RESOURCE UTILISATION CHECK ────────────────────────────────────────
# Compare requests vs actual usage
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.requests) |
  [.metadata.namespace, .metadata.name,
   .spec.containers[0].resources.requests.cpu // "none",
   .spec.containers[0].resources.requests.memory // "none"]
  | @tsv' | head -20
```

---

## 9.2 Horizontal Pod Autoscaler (HPA)

HPA is the primary consumer of Metrics Server data. It automatically scales
pod replicas based on CPU, memory, or custom metrics.

### 9.2.1 HPA Architecture

```
╔══════════════════════════════════════════════════════════════════════╗
║                   HPA — HOW IT WORKS                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  DESIRED REPLICAS = ceil(currentReplicas ×                          ║
║                          (currentMetric / targetMetric))            ║
║                                                                      ║
║  EXAMPLE:                                                            ║
║  target CPU utilisation = 50%                                        ║
║  current replicas = 3                                                ║
║  current average CPU = 90%                                           ║
║                                                                      ║
║  desired = ceil(3 × (90% / 50%)) = ceil(5.4) = 6 replicas           ║
║                                                                      ║
║  HPA CONTROL LOOP (every 15 seconds by default):                    ║
║                                                                      ║
║  Metrics Server → HPA Controller → Current avg CPU = 90%           ║
║                        │                                             ║
║                        │ desired replicas = 6                        ║
║                        ▼                                             ║
║               Deployment scaled: 3 → 6                              ║
║                        │                                             ║
║               New pods start, CPU pressure drops                    ║
║                        │                                             ║
║               Next cycle: avg CPU = 30% → scale down to 3          ║
║               (scale-down has 5-minute cool-down by default)        ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 9.2.2 HPA YAML

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app              # Target deployment to scale

  minReplicas: 2               # Never scale below 2
  maxReplicas: 20              # Never scale above 20

  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization      # Utilization (%) or AverageValue (cores)
        averageUtilization: 70 # Scale when avg CPU > 70% of request

  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi    # Scale when avg memory > 512Mi

  # Custom metric (requires Prometheus Adapter)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"    # Scale when avg RPS > 100 per pod

  behavior:                    # Scaling speed control
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
      policies:
      - type: Percent
        value: 10              # Scale down max 10% per minute
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0   # Scale up immediately
      policies:
      - type: Percent
        value: 100             # Double replicas per 30 seconds if needed
        periodSeconds: 30
      - type: Pods
        value: 4               # Or add max 4 pods per 30 seconds
        periodSeconds: 30
      selectPolicy: Max        # Use whichever policy adds more pods
```

```bash
# Create HPA imperatively
kubectl autoscale deployment web-app \
  --min=2 --max=20 --cpu-percent=70

# Check HPA status
kubectl get hpa
# NAME         REFERENCE         TARGETS     MINPODS  MAXPODS  REPLICAS
# web-app-hpa  Deployment/web-app 45%/70%    2        20       3

kubectl describe hpa web-app-hpa   # Detailed status and events

# Watch HPA in real-time
kubectl get hpa -w

# Simulate load to trigger scale-up (in another terminal)
kubectl run load-gen --image=busybox --rm -it --restart=Never -- \
  sh -c "while true; do wget -q -O- http://web-app-svc; done"
```

---

## 9.3 Prometheus — Production Metrics Platform

### 9.3.1 What Is Prometheus?

#### In Plain English

Prometheus is like a **very diligent data scientist** who visits every service
in your cluster every 15 seconds, asks "How are you doing?", records the
answer with a timestamp, and stores it all in a time-series database. Later,
you can ask questions like "What was the CPU usage of the payments service
between 2 AM and 3 AM last Tuesday?" — and get an exact answer.

#### In Technical Language

**Prometheus** is an open-source monitoring system and time-series database.
It uses a **pull model** — Prometheus scrapes HTTP endpoints (`/metrics`) from
targets at configurable intervals. It stores data in a time-series database
and provides PromQL (Prometheus Query Language) for querying.

```
╔══════════════════════════════════════════════════════════════════════╗
║              PROMETHEUS ARCHITECTURE IN KUBERNETES                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  TARGETS (expose /metrics endpoint):                                 ║
║  ┌────────────┐  ┌────────────┐  ┌───────────────┐                  ║
║  │  App Pods  │  │  kubelet   │  │  Node Exporter│                  ║
║  │ /metrics   │  │ /metrics   │  │  /metrics     │                  ║
║  │ port:9090  │  │ port:10250 │  │  port:9100    │                  ║
║  └─────┬──────┘  └─────┬──────┘  └───────┬───────┘                  ║
║        │               │                 │                           ║
║        └───────────────┼─────────────────┘                           ║
║                        │  SCRAPE (pull every 15s)                   ║
║                        ▼                                             ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  PROMETHEUS SERVER                                           │   ║
║  │  ┌──────────────┐  ┌────────────────┐  ┌───────────────┐    │   ║
║  │  │  Retrieval   │  │  TSDB Storage  │  │  HTTP Server  │    │   ║
║  │  │  (Scraper)   │  │  (Time Series  │  │  (PromQL API) │    │   ║
║  │  │              │→ │  Database)     │→ │               │    │   ║
║  │  └──────────────┘  └────────────────┘  └───────┬───────┘    │   ║
║  │  ┌──────────────────────────────┐              │             │   ║
║  │  │  Alerting Rules              │──────────────┘             │   ║
║  │  │  (PrometheusRule objects)    │  trigger alerts            │   ║
║  │  └──────────────────────────────┘                            │   ║
║  └──────────────────────────────┬───────────────────────────────┘   ║
║                                 │                                    ║
║              ┌──────────────────┼──────────────────┐                ║
║              ▼                  ▼                  ▼                ║
║         AlertManager         Grafana           Other tools          ║
║         (route alerts)       (dashboards)      (Thanos, etc.)       ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 9.3.2 Installing Prometheus — kube-prometheus-stack

The recommended way to install Prometheus in Kubernetes is via the
**kube-prometheus-stack** Helm chart, which includes Prometheus,
Grafana, AlertManager, and all required exporters in one package.

```bash
# Add Helm repo
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=MySecurePassword \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=gp3-ssd \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi

# Verify installation
kubectl get pods -n monitoring
# NAME                                                  READY
# alertmanager-monitoring-kube-prometheus-alertmanager-0 2/2
# monitoring-grafana-xxx                                3/3
# monitoring-kube-prometheus-operator-xxx               1/1
# monitoring-kube-state-metrics-xxx                     1/1
# monitoring-prometheus-node-exporter-xxx               1/1  (DaemonSet - one per node)
# prometheus-monitoring-kube-prometheus-prometheus-0    2/2

# Access Prometheus UI
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090 &
# Open http://localhost:9090

# Access Grafana UI
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80 &
# Open http://localhost:3000
# Default login: admin / MySecurePassword
```

### 9.3.3 How Apps Expose Metrics

```yaml
# app-with-metrics.yaml
# Application exposing Prometheus metrics
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        # For legacy Prometheus scraping (without Operator):
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: my-app:v1
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090   # Prometheus metrics on separate port
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

### 9.3.4 ServiceMonitor — Prometheus Operator Pattern

The **Prometheus Operator** uses custom resources to configure scraping
declaratively:

```yaml
# servicemonitor.yaml
# Tells Prometheus Operator which services to scrape
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: production
  labels:
    release: monitoring          # Must match Prometheus's serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: my-app                # Selects Services with this label
  endpoints:
  - port: metrics                # Port name from Service spec
    path: /metrics               # Metrics path
    interval: 15s                # Scrape every 15 seconds
    scrapeTimeout: 10s
  namespaceSelector:
    matchNames:
    - production                 # Only services in these namespaces

---
# The Service that ServiceMonitor targets
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  namespace: production
  labels:
    app: my-app                  # Must match ServiceMonitor selector
spec:
  selector:
    app: my-app
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: metrics                # ServiceMonitor references this port name
    port: 9090
    targetPort: 9090
```

### 9.3.5 PrometheusRule — Alerting Rules

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: production
  labels:
    release: monitoring          # Must match Prometheus's ruleSelector
spec:
  groups:
  - name: app.rules
    interval: 30s                # Evaluation interval
    rules:

    # ALERT: Pod crash loop
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m                    # Must be true for 5 minutes before firing
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.pod }} in {{ $labels.namespace }}
          has restarted {{ $value | humanize }} times in 15 minutes."
        runbook: "https://wiki.company.com/runbooks/pod-crash-loop"

    # ALERT: High CPU usage
    - alert: HighCPUUsage
      expr: |
        (sum(rate(container_cpu_usage_seconds_total{
          container!="", image!=""}[5m])) by (pod, namespace)
        /
        sum(kube_pod_container_resource_requests{
          resource="cpu", container!=""}[5m]) by (pod, namespace))
        > 0.9
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High CPU: {{ $labels.pod }}"
        description: "Pod {{ $labels.pod }} CPU is at {{ $value | humanizePercentage }}"

    # ALERT: Node disk pressure
    - alert: NodeDiskPressure
      expr: |
        (node_filesystem_avail_bytes{mountpoint="/"}
        / node_filesystem_size_bytes{mountpoint="/"}) < 0.10
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.instance }} disk below 10%"
        description: "{{ $labels.instance }} has only
          {{ $value | humanizePercentage }} disk available"

    # RECORDING RULE: pre-compute expensive query
    - record: job:http_requests_total:rate5m
      expr: |
        sum(rate(http_requests_total[5m])) by (job, namespace)
```

### 9.3.6 AlertManager — Routing Alerts

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-monitoring-kube-prometheus-alertmanager
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s           # Wait 30s to group alerts before sending
      group_interval: 5m        # Send grouped updates every 5 minutes
      repeat_interval: 12h      # Re-notify if still firing after 12h
      receiver: 'default-slack'

      routes:
      # Critical alerts → PagerDuty (immediate)
      - matchers:
        - severity = critical
        receiver: pagerduty
        repeat_interval: 1h

      # Warning alerts → Slack #platform-alerts
      - matchers:
        - severity = warning
        receiver: slack-warnings

    receivers:
    - name: 'default-slack'
      slack_configs:
      - channel: '#platform-alerts'
        send_resolved: true
        title: '{{ .Status | toUpper }}: {{ .CommonLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

    - name: 'pagerduty'
      pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_KEY'
        description: '{{ .CommonAnnotations.summary }}'

    - name: 'slack-warnings'
      slack_configs:
      - channel: '#platform-warnings'
        send_resolved: true
```

### 9.3.7 Essential PromQL Queries

```promql
# ── POD METRICS ──────────────────────────────────────────────────────

# CPU usage by pod (as % of request)
sum(rate(container_cpu_usage_seconds_total{namespace="production",container!=""}[5m]))
by (pod) /
sum(kube_pod_container_resource_requests{namespace="production",resource="cpu"})
by (pod) * 100

# Memory usage by pod (bytes)
sum(container_memory_working_set_bytes{namespace="production",container!=""})
by (pod)

# Pod restart rate (last 15 minutes)
rate(kube_pod_container_status_restarts_total{namespace="production"}[15m])

# Pods not in Running state
kube_pod_status_phase{phase!="Running", namespace="production"}

# ── NODE METRICS ──────────────────────────────────────────────────────

# Node CPU utilisation
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)

# Node memory available
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# Node disk usage
1 - (node_filesystem_avail_bytes{mountpoint="/"} /
     node_filesystem_size_bytes{mountpoint="/"})

# ── APPLICATION METRICS ───────────────────────────────────────────────

# HTTP request rate (if app exposes this)
sum(rate(http_requests_total{namespace="production"}[5m])) by (service)

# HTTP error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100

# P99 latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m]))
  by (le, service))
```

---

## 9.4 Grafana — Dashboards and Visualisation

### 9.4.1 What Is Grafana?

Grafana is the **display screen** of your monitoring setup. Prometheus
collects and stores metrics; Grafana queries Prometheus and renders
beautiful dashboards showing the state of your cluster.

```
╔══════════════════════════════════════════════════════════════════════╗
║                GRAFANA — KEY CONCEPTS                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  DATA SOURCE                                                         ║
║  Connected backend where data comes from.                           ║
║  In Kubernetes: Prometheus, Loki, Jaeger, Elasticsearch             ║
║                                                                      ║
║  DASHBOARD                                                           ║
║  A collection of panels organised on a canvas.                     ║
║  Can be imported (JSON), exported, and version-controlled.          ║
║  Example: Kubernetes Cluster Overview dashboard.                    ║
║                                                                      ║
║  PANEL                                                               ║
║  A single visualisation on a dashboard.                             ║
║  Types: Graph, Stat, Table, Gauge, Heatmap, Bar chart              ║
║                                                                      ║
║  ALERT                                                               ║
║  Grafana can also alert based on dashboard panel thresholds.        ║
║  Simpler than PrometheusRule but less powerful.                     ║
║                                                                      ║
║  POPULAR KUBERNETES DASHBOARDS (import by ID):                      ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  3119  — Kubernetes Cluster Overview (most popular)          │   ║
║  │  6417  — Kubernetes Pods Overview                            │   ║
║  │  1860  — Node Exporter Full (node metrics)                   │   ║
║  │  7249  — Kubernetes Cluster Monitoring                       │   ║
║  │  13770 — Kubernetes All-in-One Monitoring                    │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 9.4.2 Grafana Dashboard as ConfigMap (GitOps)

```yaml
# grafana-dashboard-configmap.yaml
# Provision dashboards automatically via ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"       # Label used by Grafana dashboard sidecar
data:
  my-app-dashboard.json: |
    {
      "title": "My App Dashboard",
      "uid": "my-app-001",
      "panels": [
        {
          "title": "Request Rate",
          "type": "graph",
          "datasource": "Prometheus",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{namespace=\"production\"}[5m])) by (service)",
              "legendFormat": "{{service}}"
            }
          ]
        },
        {
          "title": "Error Rate %",
          "type": "stat",
          "datasource": "Prometheus",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100"
            }
          ]
        }
      ],
      "refresh": "30s",
      "time": {"from": "now-1h", "to": "now"}
    }
```

---

## 9.5 Logging Architecture in Kubernetes

### 9.5.1 The Three Levels of Logging

```
╔══════════════════════════════════════════════════════════════════════╗
║              THREE LOGGING LEVELS IN KUBERNETES                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  LEVEL 1: APPLICATION-LEVEL LOGGING                                  ║
║  ──────────────────────────────────                                  ║
║  Application writes to stdout/stderr.                               ║
║  Container runtime captures it.                                     ║
║  Accessible via: kubectl logs <pod>                                 ║
║  Stored at: /var/log/containers/<pod>_<ns>_<container>-<id>.log     ║
║  on the node.                                                        ║
║  PROBLEM: Lost when pod is deleted or node fails.                   ║
║                                                                      ║
║  LEVEL 2: NODE-LEVEL LOGGING                                         ║
║  ───────────────────────────                                         ║
║  Container runtime rotates logs on node disk.                       ║
║  kubelet manages log rotation.                                       ║
║  Default retention: until disk pressure or rotation limit.          ║
║  Still lost when node fails.                                        ║
║                                                                      ║
║  LEVEL 3: CLUSTER-LEVEL LOGGING (Production Requirement)            ║
║  ──────────────────────────────────────────────────────             ║
║  A log agent (Fluentd/Fluent Bit) runs on every node as DaemonSet. ║
║  It ships logs to an external log backend.                          ║
║  Logs survive pod deletion, node failure, and cluster recreation.   ║
║  Queryable across all pods and namespaces from one place.           ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  CLUSTER-LEVEL LOGGING FLOW:                                 │   ║
║  │                                                              │   ║
║  │  Pod → stdout → /var/log/containers/*.log (node disk)        │   ║
║  │                         │                                    │   ║
║  │                 Fluent Bit DaemonSet reads                   │   ║
║  │                         │                                    │   ║
║  │                  Parses + enriches with                      │   ║
║  │                  pod name, namespace, labels                 │   ║
║  │                         │                                    │   ║
║  │                  Ships to backend:                           │   ║
║  │                  Elasticsearch / Loki / Splunk / CloudWatch  │   ║
║  │                         │                                    │   ║
║  │                  Query via:                                  │   ║
║  │                  Kibana / Grafana (Loki) / Splunk UI          │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 9.5.2 kubectl logs — Built-in Log Access

```bash
# ── BASIC LOG COMMANDS ────────────────────────────────────────────────
kubectl logs <pod-name>                    # Current container logs
kubectl logs <pod-name> -f                 # Follow (stream) logs
kubectl logs <pod-name> --previous         # Logs from PREVIOUS container instance
kubectl logs <pod-name> --tail=100         # Last 100 lines
kubectl logs <pod-name> --since=1h         # Last 1 hour
kubectl logs <pod-name> --since-time="2024-01-15T10:00:00Z"  # From specific time

# Multi-container pod
kubectl logs <pod-name> -c <container>     # Specific container
kubectl logs <pod-name> --all-containers   # All containers

# Deployment / ReplicaSet (picks a random pod)
kubectl logs deployment/web-app            # Logs from one pod
kubectl logs deployment/web-app --all-pods # Logs from ALL pods (noisy)

# Label selector
kubectl logs -l app=web --all-containers   # All pods matching label

# ── ADVANCED LOG PATTERNS ─────────────────────────────────────────────
# Save logs before deleting a pod
kubectl logs my-pod --previous > /tmp/crash-logs.txt

# Watch logs from all pods in a deployment simultaneously (use stern)
# stern web-app --namespace production --since 1h

# Get logs from all pods in a namespace that match a label
for pod in $(kubectl get pods -l app=web -o name); do
  echo "=== $pod ===" && kubectl logs $pod --tail=20
done

# Filter logs for errors
kubectl logs my-pod | grep -i "error\|warn\|fatal"
```

### 9.5.3 Fluent Bit — Lightweight Log Shipper

Fluent Bit is the preferred log collector for Kubernetes — it is lighter
than Fluentd and designed specifically for containerised environments.

```yaml
# fluent-bit-daemonset.yaml
# Deploy Fluent Bit to collect and ship all pod logs

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   Off
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On           # Parse JSON logs and merge
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off
        Labels              On           # Include pod labels
        Annotations         Off          # Exclude annotations (verbose)

    [FILTER]
        Name    record_modifier
        Match   *
        Record  cluster my-production-cluster
        Record  environment production

    [OUTPUT]
        Name            es
        Match           *
        Host            elasticsearch-svc.logging.svc.cluster.local
        Port            9200
        Index           kubernetes-logs
        Type            _doc
        Retry_Limit     5
        tls             Off

  parsers.conf: |
    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit-sa
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: config
          mountPath: /fluent-bit/etc/
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: config
        configMap:
          name: fluent-bit-config

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit-sa
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-role
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces", "nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit-binding
roleRef:
  kind: ClusterRole
  name: fluent-bit-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluent-bit-sa
  namespace: logging
```

### 9.5.4 Loki — Prometheus for Logs

**Loki** is a log aggregation system from Grafana Labs. Unlike Elasticsearch,
it indexes only metadata (labels) not the full log content — making it
much cheaper to run and tightly integrated with Grafana.

```
╔══════════════════════════════════════════════════════════════════════╗
║           LOKI vs ELASTICSEARCH — COMPARISON                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║                  LOKI                  ELASTICSEARCH                 ║
║  Cost:           Low (index only       High (full-text               ║
║                  labels/metadata)      indexing)                     ║
║  Storage:        S3/GCS compatible     Block storage needed          ║
║  Query language: LogQL                 Lucene / KQL                  ║
║  UI:             Grafana (built-in)    Kibana                        ║
║  Alerting:       Grafana Alerts        ElastAlert / Kibana           ║
║  Kubernetes fit: Excellent (label-     Good (more complex           ║
║                  based, like Prom)     to operate)                   ║
║  Best for:       Cloud-native K8s,    Full-text search,             ║
║                  cost-conscious teams  compliance, complex search    ║
║                                                                      ║
║  PLG STACK: Promtail + Loki + Grafana (all from Grafana Labs)        ║
║  EFK STACK: Elasticsearch + Fluentd/Fluent Bit + Kibana              ║
╚══════════════════════════════════════════════════════════════════════╝
```

```bash
# Install Loki + Promtail via Helm (PLG stack)
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi

# Loki is now a data source in Grafana
# Query logs with LogQL:
# {namespace="production", app="web-app"} |= "error"
# {container="nginx"} | json | status >= 500
```

### 9.5.5 LogQL — Querying Logs in Loki

```logql
# ── BASIC QUERIES ─────────────────────────────────────────────────────

# All logs from production namespace
{namespace="production"}

# Logs from specific app
{namespace="production", app="payment-service"}

# Filter for error lines
{namespace="production"} |= "ERROR"

# Filter excluding debug
{namespace="production"} != "DEBUG"

# Regex filter
{namespace="production"} |~ "error|warn|fatal"

# ── LOG PARSING ───────────────────────────────────────────────────────

# Parse JSON logs
{app="my-app"} | json | level="error"

# Parse logfmt
{app="my-app"} | logfmt | duration > 1s

# ── METRICS FROM LOGS ─────────────────────────────────────────────────

# Count error rate per minute
sum(rate({namespace="production"} |= "ERROR" [1m])) by (app)

# P99 latency from log fields (if app logs duration)
quantile_over_time(0.99,
  {app="api"} | json | unwrap duration_ms [5m])
```

---

## 9.6 The Full Observability Stack Deployment

```yaml
# observability-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
apiVersion: v1
kind: Namespace
metadata:
  name: logging
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

```bash
# COMPLETE OBSERVABILITY STACK SETUP

# 1. Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 2. Install kube-prometheus-stack (Prometheus + Grafana + AlertManager)
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f prometheus-values.yaml    # Your custom values

# 3. Install Loki + Promtail (logging)
helm install loki grafana/loki-stack \
  -n logging --create-namespace \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=20Gi

# 4. Add Loki as Grafana data source
kubectl apply -f loki-datasource-configmap.yaml

# Verify everything is running
kubectl get pods -n monitoring
kubectl get pods -n logging
kubectl top nodes
kubectl top pods -A
```

---

## Chapter 9: Hands-On Labs

### Lab 9.1 — Metrics Server and kubectl top

```bash
# Install metrics server
minikube addons enable metrics-server

# Wait for it to be ready
kubectl rollout status deployment/metrics-server -n kube-system
sleep 60   # Allow initial metrics collection

# Generate some load
kubectl create deployment load-test --image=nginx --replicas=5
for i in $(seq 1 5); do
  kubectl run loader-$i --image=busybox --restart=Never \
    -- sh -c "while true; do wget -q -O/dev/null http://load-test-svc 2>/dev/null; done" &
done

# Monitor resource usage
watch kubectl top pods -A --sort-by=cpu

# Create HPA
kubectl expose deployment load-test --port=80 --name=load-test-svc
kubectl autoscale deployment load-test --min=2 --max=10 --cpu-percent=50

# Watch HPA in action
kubectl get hpa -w

# Cleanup
kubectl delete deployment load-test
kubectl delete hpa load-test
kubectl delete svc load-test-svc
kubectl delete pods -l run=loader --force
```

### Lab 9.2 — kubectl logs Deep Dive

```bash
# Deploy a chatty application
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: log-demo
  template:
    metadata:
      labels:
        app: log-demo
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sh", "-c"]
        args:
        - |
          i=0
          while true; do
            case $((i % 4)) in
              0) echo "{\"level\":\"INFO\",\"msg\":\"Request processed\",\"id\":$i}";;
              1) echo "{\"level\":\"WARN\",\"msg\":\"Slow query detected\",\"duration\":\"800ms\"}";;
              2) echo "{\"level\":\"ERROR\",\"msg\":\"Database connection failed\",\"retry\":$i}";;
              3) echo "{\"level\":\"DEBUG\",\"msg\":\"Cache hit\",\"key\":\"user-$i\"}";;
            esac
            i=$((i+1))
            sleep 2
          done
EOF

# Basic log access
kubectl get pods -l app=log-demo
kubectl logs deployment/log-demo
kubectl logs deployment/log-demo -f --tail=20

# Access specific pod
POD=$(kubectl get pods -l app=log-demo -o name | head -1)
kubectl logs $POD --since=1m
kubectl logs $POD | grep ERROR
kubectl logs $POD | grep -E "ERROR|WARN"

# All pods simultaneously
kubectl logs -l app=log-demo --prefix=true --tail=5

# Count errors across all pods
kubectl logs -l app=log-demo 2>/dev/null | grep -c ERROR

# Save logs before deleting
kubectl logs $POD > /tmp/pod-logs-backup.txt
cat /tmp/pod-logs-backup.txt | wc -l
```

### Lab 9.3 — Deploy Prometheus and Grafana (Quick)

```bash
# Install kube-prometheus-stack
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false

# Wait for all pods
kubectl wait --for=condition=ready pod \
  -l "release=monitoring" -n monitoring --timeout=300s

# Port-forward Prometheus
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-prometheus 9090:9090 &

# Port-forward Grafana
kubectl port-forward -n monitoring \
  svc/monitoring-grafana 3000:80 &

echo "Prometheus: http://localhost:9090"
echo "Grafana: http://localhost:3000 (admin/admin123)"

# Create a test ServiceMonitor
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: app
        image: prom/prometheus:latest
        ports:
        - name: metrics
          containerPort: 9090
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app-svc
  namespace: default
  labels:
    app: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - name: metrics
    port: 9090
    targetPort: 9090
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-app-monitor
  namespace: default
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: sample-app
  endpoints:
  - port: metrics
    interval: 15s
EOF

# In Prometheus UI: Status → Targets → verify sample-app appears
# In Grafana: Import dashboard ID 3119 for K8s overview
```

### Lab 9.4 — Alerting Rule

```bash
# Create an alert rule that fires when any pod restarts
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: lab-alerts
  namespace: default
  labels:
    release: monitoring
spec:
  groups:
  - name: lab.rules
    rules:
    - alert: PodRestartingTooMuch
      expr: increase(kube_pod_container_status_restarts_total[5m]) > 0
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} restarted"
        description: "Pod {{ $labels.pod }} restarted in namespace {{ $labels.namespace }}"
EOF

# Trigger a restart to test the alert
kubectl run crasher --image=busybox \
  --restart=Always \
  -- sh -c "exit 1"

# Watch restarts
kubectl get pod crasher -w

# In Prometheus UI: Alerts → see PodRestartingTooMuch firing
kubectl delete pod crasher
```

---

## Chapter 9: Troubleshooting Guide

### Issue 1: `kubectl top` returns "error: metrics not available"

```bash
kubectl top nodes
# Error from server (ServiceUnavailable):
# the server is currently unable to handle the request (get nodes.metrics.k8s.io)

# Check Metrics Server is running
kubectl get pods -n kube-system | grep metrics-server
kubectl describe pod -n kube-system -l k8s-app=metrics-server

# Check Metrics Server logs
kubectl logs -n kube-system -l k8s-app=metrics-server

# Common fix: TLS issue on local cluster
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-",
       "value":"--kubelet-insecure-tls"}]'

# Check API service
kubectl get apiservice v1beta1.metrics.k8s.io
# STATUS must be True — if False, metrics-server is not healthy
```

### Issue 2: HPA stuck at minimum replicas (not scaling up)

```bash
kubectl describe hpa my-hpa
# Conditions:
# AbleToScale: True
# ScalingActive: False  ← Problem is here
# Message: "the HPA was unable to compute the replica count"

# CAUSE: Metrics Server not running or pods have no resource requests
# Check pod resource requests
kubectl get pod my-pod -o yaml | grep -A5 resources

# Without CPU requests, HPA cannot calculate utilisation percentage
# Fix: Add resource requests to the deployment

# CAUSE: Metrics not yet available
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods | jq .

# Check HPA events
kubectl get events --field-selector reason=FailedGetScale
```

### Issue 3: Prometheus not scraping a target

```bash
# In Prometheus UI: Status → Targets → find your service

# State: DOWN
# Error: "connection refused"
# Fix: Check the pod is running and metrics port is correct
kubectl get svc my-app-svc -o yaml | grep ports
kubectl exec my-pod -- wget -qO- http://localhost:9090/metrics

# State: DOWN
# Error: "Get http://...: context deadline exceeded"
# Fix: Network Policy may be blocking Prometheus
# Allow monitoring namespace to scrape production
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - port: 9090
EOF

# ServiceMonitor not being picked up
# Check labels match Prometheus's serviceMonitorSelector
kubectl get prometheus -n monitoring -o yaml | grep serviceMonitorSelector
# ServiceMonitor must have matching labels!
kubectl label servicemonitor my-monitor release=monitoring
```

### Issue 4: Logs missing from Fluent Bit / pods not appearing

```bash
# Check Fluent Bit DaemonSet
kubectl get ds fluent-bit -n logging
kubectl get pods -n logging -l app=fluent-bit

# Check Fluent Bit logs (meta!)
kubectl logs -n logging -l app=fluent-bit --tail=50

# Common issues:
# 1. RBAC: Fluent Bit SA can't read pod info
kubectl auth can-i get pods \
  --as system:serviceaccount:logging:fluent-bit-sa

# 2. hostPath /var/log not accessible
kubectl describe pod -n logging -l app=fluent-bit | grep -A5 Events

# 3. Parser mismatch — logs not in expected format
# Check if containerd uses cri format vs docker format
kubectl exec -n logging -it $(kubectl get pod -n logging \
  -l app=fluent-bit -o name | head -1) -- \
  tail /var/log/containers/kube-apiserver*.log | head -3
```

### Issue 5: Grafana showing "No data" for Prometheus queries

```bash
# Step 1: Verify Prometheus data source is connected
# Grafana → Configuration → Data Sources → Prometheus → Test

# Step 2: Try the PromQL query directly in Prometheus UI
# If it works there → Grafana datasource URL is wrong

# Step 3: Check time range
# Grafana default is "Last 6 hours" — make sure data exists
# in that range

# Step 4: Check for recording rule vs raw metric name mismatch
kubectl get prometheusrule -A
# Verify rule names match what Grafana dashboard expects

# Step 5: Reload Prometheus configuration
kubectl exec -n monitoring \
  prometheus-monitoring-kube-prometheus-prometheus-0 \
  -- kill -HUP 1
```

---

## Chapter 9: Interview Questions

**Q1: What is the difference between Metrics Server and Prometheus?**

> Metrics Server is a lightweight, in-memory metrics source that collects only current CPU and memory usage from kubelets. It has no persistence, no historical data, and only exposes two metric types. It powers `kubectl top` and HPA. Prometheus is a full time-series monitoring system that scrapes any number of custom metrics endpoints, stores data with configurable retention (days to months), supports rich querying via PromQL, and drives alerting via AlertManager. Use Metrics Server for autoscaling; use Prometheus for observability, alerting, and historical analysis.

**Q2: How does Prometheus discover targets in Kubernetes?**

> Prometheus uses Kubernetes service discovery (`kubernetes_sd_configs`) to automatically find scrape targets. With the Prometheus Operator, it uses `ServiceMonitor` and `PodMonitor` custom resources — you declare which services/pods to scrape using label selectors, and the Operator configures Prometheus accordingly. Without the Operator, Prometheus can be configured to discover all pods or services with specific annotations (`prometheus.io/scrape: "true"`). The pull model means Prometheus initiates connections to targets' `/metrics` endpoints at configured intervals.

**Q3: What are the three logging levels in Kubernetes and which is production-grade?**

> Level 1 (Application) — app writes to stdout/stderr, accessible via `kubectl logs`, but lost when pod is deleted. Level 2 (Node) — container runtime writes logs to node disk at `/var/log/containers/`, lost if node fails. Level 3 (Cluster-level) — a log agent DaemonSet (Fluent Bit, Fluentd) runs on every node, collects logs from the node filesystem, enriches them with Kubernetes metadata (pod name, namespace, labels), and ships them to a persistent backend (Elasticsearch, Loki). Only Level 3 is suitable for production — it survives pod deletion, node failure, and cluster maintenance.

**Q4: What is a ServiceMonitor and why is it used?**

> A ServiceMonitor is a custom resource defined by the Prometheus Operator. It declaratively specifies which Kubernetes Services Prometheus should scrape, the port name and path to scrape, and the interval. The Prometheus Operator watches ServiceMonitor objects and automatically generates the corresponding Prometheus scrape configuration. This is the GitOps-friendly approach — teams define their own ServiceMonitors alongside their application deployments, rather than requiring a central Prometheus config update.

**Q5: What is HPA and what does it require to work?**

> Horizontal Pod Autoscaler automatically scales the number of pod replicas in a Deployment, StatefulSet, or ReplicaSet based on observed metrics. Requirements: (1) Metrics Server must be installed and running; (2) The target pods must have CPU/memory resource requests set — without requests, HPA cannot calculate utilisation percentage; (3) The HPA object itself with `scaleTargetRef`, `minReplicas`, `maxReplicas`, and metric targets. HPA checks metrics every 15 seconds and has a built-in scale-down stabilisation window (default 5 minutes) to prevent thrashing.

**Q6: What is the PLG stack vs the EFK stack?**

> EFK stack: Elasticsearch (log storage and search) + Fluentd or Fluent Bit (log collection) + Kibana (log UI). Powerful full-text search, mature, but resource-intensive and complex to operate. PLG stack: Promtail (log collector) + Loki (log aggregation) + Grafana (UI). Loki only indexes labels/metadata, not full log content — much cheaper to store and operate. Tightly integrated with Grafana (same UI for metrics and logs). Loki uses LogQL (similar to PromQL). PLG is recommended for most Kubernetes-native teams; EFK for compliance-heavy environments needing full-text search.

**Q7: What is a PrometheusRule?**

> A PrometheusRule is a custom resource (from the Prometheus Operator) that defines alerting and recording rules. Alerting rules evaluate PromQL expressions and fire alerts to AlertManager when conditions are met (e.g., pod crash-looping, high CPU). Recording rules pre-compute expensive queries into new metric series for performance. The Operator watches PrometheusRule objects and automatically loads them into Prometheus without restart. Rules are grouped with evaluation intervals and support labels and annotations for alert routing and context.

**Q8: How do you debug a pod that is crashing and you cannot get its current logs?**

> Use `kubectl logs <pod-name> --previous` to get the logs from the previous (crashed) container instance. The `--previous` flag accesses the terminated container's logs before it was restarted. If the pod has already been deleted, check if your cluster-level logging (Fluent Bit + Elasticsearch/Loki) has captured the logs — query by pod name in Kibana or Grafana Explore. Also `kubectl describe pod <pod-name>` shows the last exit code and reason in the `Last State` section, which may give enough context (e.g., OOMKilled = exit code 137 = need to increase memory limit).

---

## CKA Exam Notes — Chapter 9

```
╔════════════════════════════════════════════════════════════════════╗
║                  CKA EXAM FOCUS — CHAPTER 9                       ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  EXAM WEIGHT: ~5% directly, but monitoring commands appear        ║
║  throughout troubleshooting tasks (30% of exam).                  ║
║                                                                    ║
║  MUST-KNOW COMMANDS:                                               ║
║  k top nodes                          ← node resource usage       ║
║  k top pods -A --sort-by=cpu          ← find CPU hogs             ║
║  k top pods -A --sort-by=memory       ← find memory hogs          ║
║  k logs <pod> --previous              ← crashed container logs    ║
║  k logs <pod> -f --tail=100           ← follow live logs          ║
║  k logs -l app=web --all-containers   ← all pods by label         ║
║  k describe pod <pod> | grep -A5 Events  ← always check events   ║
║                                                                    ║
║  HPA EXAM PATTERN:                                                 ║
║  k autoscale deployment <n>                                       ║
║    --min=2 --max=10 --cpu-percent=70                               ║
║  k get hpa                                                         ║
║  k describe hpa <name>    ← check conditions and events           ║
║                                                                    ║
║  METRICS SERVER EXAM:                                              ║
║  If top doesn't work → check metrics-server in kube-system        ║
║  k get pods -n kube-system | grep metrics                         ║
║  k get apiservice v1beta1.metrics.k8s.io                          ║
║                                                                    ║
║  EXAM TRAPS:                                                       ║
║  HPA requires resource REQUESTS set on pods to work               ║
║  kubectl top requires metrics-server to be healthy                ║
║  --previous flag for crashed container logs is often the answer   ║
║  k logs deployment/<n> shows ONE pod, not all pods               ║
║  Metrics Server stores data in MEMORY only — no history           ║
║                                                                    ║
║  TROUBLESHOOTING SEQUENCE (in exam context):                      ║
║  1. k get pods -A                    ← find problematic pods      ║
║  2. k describe pod <pod>             ← read Events section        ║
║  3. k logs <pod> --previous          ← crash logs                ║
║  4. k top nodes                      ← resource pressure?         ║
║  5. k top pods -A --sort-by=memory   ← memory hog?               ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Common Mistakes — Chapter 9

| Mistake | Why It Happens | How to Avoid |
|---|---|---|
| HPA not scaling: no resource requests | Forgot to set CPU requests | Always set `resources.requests.cpu` when using CPU-based HPA |
| `kubectl top` fails after fresh install | Metrics Server needs 60-90s to collect | Wait ~90s after install before running top |
| Missing `--previous` for crash logs | Using `kubectl logs` on current container | Always try `--previous` when pod is in CrashLoopBackOff |
| Prometheus not discovering ServiceMonitor | Label mismatch with Prometheus selector | Check `kubectl get prometheus -o yaml` for `serviceMonitorSelector` |
| Fluent Bit missing RBAC | SA has no permission to read pod metadata | Create ClusterRole with get/list/watch on pods + ClusterRoleBinding |
| Network Policy blocking Prometheus scrape | Default-deny without prometheus allow rule | Add ingress rule from monitoring namespace to port 9090 |
| Loki not receiving logs from Promtail | Wrong Loki URL in Promtail config | Use internal DNS: `loki.logging.svc.cluster.local:3100` |
| AlertManager sending duplicate alerts | Missing `group_by` configuration | Set `group_by: [alertname, namespace]` in AlertManager route |

---

## Chapter 9 Summary

1. **Three observability pillars** — Metrics (what's happening now), Logs (what happened), Traces (how a request flowed)
2. **Metrics Server** — lightweight, in-memory CPU/memory metrics; powers `kubectl top` and HPA; no history
3. **kubectl top** — instant resource snapshot; `--sort-by=cpu/memory` for finding hogs
4. **HPA** — auto-scales replicas based on CPU/memory/custom metrics; requires resource requests on pods
5. **Prometheus** — pull-based time-series monitoring; scrapes `/metrics` endpoints; PromQL for querying
6. **Prometheus Operator** — ServiceMonitor, PodMonitor, PrometheusRule custom resources for declarative config
7. **AlertManager** — routes Prometheus alerts to Slack, PagerDuty, email with grouping and deduplication
8. **Grafana** — dashboards connecting to Prometheus (metrics) and Loki (logs); import by dashboard ID
9. **kubectl logs** — built-in log access; `--previous` for crashed containers; `-f` to stream
10. **Three logging levels** — Application → Node → Cluster-level (only cluster-level survives failures)
11. **Fluent Bit** — lightweight DaemonSet log shipper; enriches with K8s metadata; ships to Elasticsearch/Loki
12. **EFK vs PLG** — EFK: full-text search, complex, expensive; PLG: label-based, simple, cheap, cloud-native

---

*Next: Chapter 10 — Cluster Administration: kubeadm, Cluster Upgrade, etcd Backup & Restore*

*"You can observe your cluster perfectly. Now let's learn to OPERATE it —
 setting it up from scratch, upgrading it safely, and restoring it from disaster."*
