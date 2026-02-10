# Kubernetes Observability Stack - Learning Guide

## Overview

This guide walks through deploying a complete **observability stack** for your K3s cluster. Observability means understanding what's happening inside your cluster through **metrics, logs, and traces** - the three pillars of observability.

### Why Observability Matters

**Without observability**:
- "Why is the pod crashing?" → No logs
- "Is the CPU usage high?" → No metrics
- "Why is the API slow?" → No traces
- Debugging = educated guessing

**With observability**:
- Detailed logs with search
- Real-time metrics and dashboards
- Distributed tracing
- Proactive alerts before users complain

### The Three Pillars

**1. Metrics** (Prometheus + Grafana)
- Numeric data over time (CPU, memory, request rate)
- Dashboards and graphs
- Alerting on thresholds

**2. Logs** (Loki + Grafana)
- Application output, errors, debug messages
- Centralized log aggregation
- Log searching and correlation

**3. Traces** (Tempo + Grafana - Optional)
- Request flow through microservices
- Latency analysis
- Performance bottlenecks

### What You'll Deploy

```
Observability Stack on K3s:

┌─────────────────────────────────────────┐
│         Grafana (Frontend)              │
│  - Dashboards for all 3 pillars         │
│  - Unified query interface              │
│  - Alerting                             │
└─────┬───────────┬───────────┬───────────┘
      │           │           │
      ▼           ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│Prometheus│ │  Loki   │ │ Tempo   │
│ (Metrics)│ │ (Logs)  │ │(Traces) │
└─────────┘ └─────────┘ └─────────┘
      ▲           ▲           ▲
      │           │           │
      └───────────┴───────────┘
      Scraped from K8s pods/nodes

Resources: ~6GB RAM total
```

---

## Phase 1: Metrics with Prometheus

### Goal
Deploy Prometheus to collect and store metrics from your cluster.

### Understanding Prometheus

**How it works**:
1. Pods expose `/metrics` endpoint (HTTP)
2. Prometheus scrapes these endpoints every 15s
3. Stores time-series data in local database
4. Provides PromQL query language
5. Grafana queries Prometheus for dashboards

**What it monitors**:
- Node metrics (CPU, RAM, disk, network)
- Pod metrics (container stats)
- K8s API metrics (# of pods, deployments, etc.)
- Application metrics (custom metrics you add)

### Step 1: Install kube-prometheus-stack

**Why this stack?**: Bundles Prometheus + Grafana + Alertmanager + exporters

```bash
# Add Helm repo (Helm = K8s package manager)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install with custom values
cat <<EOF > prometheus-values.yaml
# Prometheus configuration
prometheus:
  prometheusSpec:
    retention: 7d  # Keep metrics for 7 days
    resources:
      requests:
        memory: 2Gi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: 2
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi  # Adjust based on cluster size

# Grafana configuration
grafana:
  enabled: true
  adminPassword: "change-me-please"  # Set strong password!
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - grafana.local  # Change to your domain
  persistence:
    enabled: true
    size: 10Gi

# Node exporter (metrics from nodes)
nodeExporter:
  enabled: true

# kube-state-metrics (K8s object metrics)
kubeStateMetrics:
  enabled: true

# Alertmanager (alert routing)
alertmanager:
  enabled: true
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi
EOF

# Install
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f prometheus-values.yaml

# Watch installation
kubectl get pods -n monitoring -w
```

**Installation time**: 2-5 minutes

### Step 2: Access Grafana

```bash
# Port-forward (temporary access)
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Open browser to http://localhost:3000
# Login: admin / change-me-please

# Or get LoadBalancer IP (if ingress configured)
kubectl get ingress -n monitoring
```

**First login**:
1. Navigate to http://localhost:3000
2. Login with admin credentials
3. You'll see pre-built dashboards!

### Step 3: Explore Pre-Built Dashboards

**Navigate**: Dashboards → Browse

**Key dashboards**:
- **Kubernetes / Compute Resources / Cluster**: Overall cluster health
- **Kubernetes / Compute Resources / Namespace (Pods)**: Pod-level metrics
- **Node Exporter / Nodes**: Detailed node metrics
- **Kubernetes / API Server**: K8s API performance

**What to look for**:
- CPU/Memory usage trends
- Pod restart counts
- Network traffic
- Disk I/O

### Step 4: Understanding PromQL

**PromQL = Prometheus Query Language**

**Basic queries** (try in Grafana → Explore):

```promql
# Total CPU usage across cluster
sum(rate(container_cpu_usage_seconds_total[5m]))

# Memory usage per pod
sum by (pod) (container_memory_usage_bytes)

# Pods in CrashLoopBackOff
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}

# HTTP request rate (if app exposes metrics)
rate(http_requests_total[5m])
```

**Query breakdown**:
- `rate()`: Calculates per-second rate
- `[5m]`: Over 5 minute window
- `sum()`: Aggregate across labels
- `by (pod)`: Group by pod name

### Step 5: Create Custom Dashboard

**Example: Application metrics dashboard**

1. Go to Dashboards → New Dashboard
2. Add visualization → Prometheus data source
3. Query: `rate(http_requests_total{job="your-app"}[5m])`
4. Set title: "HTTP Request Rate"
5. Save dashboard

---

## Phase 2: Logs with Loki

### Goal
Deploy Loki to aggregate and query logs from all pods.

### Understanding Loki

**How it works**:
1. Promtail (agent) runs on every node
2. Tails logs from `/var/log/pods/*`
3. Sends to Loki server
4. Loki indexes labels (not full text - more efficient than ElasticSearch)
5. Grafana queries Loki with LogQL

**Why Loki > ElasticSearch?**:
- Lower resource usage (no full-text indexing)
- Better integration with Prometheus
- Same query language style (LogQL ~ PromQL)
- Purpose-built for K8s

### Step 1: Install Loki Stack

```bash
# Add Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create values file
cat <<EOF > loki-values.yaml
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1  # Single instance (HA needs 3)
  storage:
    type: filesystem  # Use PVC for production

# Promtail (log shipper)
promtail:
  enabled: true
  config:
    clients:
      - url: http://loki:3100/loki/api/v1/push

# Grafana datasource (auto-configure)
grafana:
  enabled: false  # Already have Grafana from Prometheus stack

# Storage
persistence:
  enabled: true
  size: 50Gi  # Adjust based on log volume

# Resource limits
resources:
  limits:
    cpu: 1
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 1Gi
EOF

# Install
helm install loki grafana/loki-stack \
  -n monitoring \
  -f loki-values.yaml

# Verify
kubectl get pods -n monitoring -l app=loki
kubectl get pods -n monitoring -l app.kubernetes.io/name=promtail
```

### Step 2: Configure Loki as Grafana Data Source

**In Grafana**:

1. Configuration (gear icon) → Data Sources → Add data source
2. Select "Loki"
3. URL: `http://loki:3100`
4. Click "Save & test"

**Or via kubectl**:

```bash
# Create datasource ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-loki
  namespace: monitoring
  labels:
    grafana_datasource: "1"
data:
  loki.yaml: |-
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki:3100
      isDefault: false
EOF

# Restart Grafana
kubectl rollout restart deployment -n monitoring prometheus-grafana
```

### Step 3: Query Logs with LogQL

**In Grafana → Explore → Select Loki datasource**

**Basic queries**:

```logql
# All logs from namespace
{namespace="default"}

# Logs from specific pod
{pod="nginx-xyz"}

# Logs containing "error" (case-insensitive)
{namespace="default"} |~ "(?i)error"

# Logs NOT containing "debug"
{namespace="default"} !~ "debug"

# Rate of errors per minute
rate({namespace="default"} |~ "error" [1m])

# Logs with JSON parsing
{namespace="default"} | json | level="error"
```

**LogQL operators**:
- `|~`: Regex match
- `!~`: Regex not match
- `|=`: Contains (faster than regex)
- `!=`: Not contains
- `| json`: Parse JSON logs

### Step 4: Create Log Dashboard

**Example: Application error tracking**

```logql
# Query for error rate
sum(rate({namespace="production"} |~ "(?i)error" [5m])) by (pod)

# Query for specific error messages
{namespace="production"} |~ "database connection failed"
```

**Dashboard panels**:
1. Error rate over time (graph)
2. Top 10 pods by error count (table)
3. Recent error logs (log panel)

---

## Phase 3: Distributed Tracing with Tempo

### Goal
Add distributed tracing to understand request flows.

### Understanding Distributed Tracing

**The problem**:
```
User reports: "API is slow"
Your microservices:
  API → Auth Service → User Service → Database
                    → Cache
```

**Question**: Which hop is slow?

**Tracing answer**:
```
Trace ID: abc123
├─ API Gateway: 245ms total
   ├─ Auth Service: 50ms
   │  └─ Database query: 45ms
   └─ User Service: 180ms  ← SLOW!
      ├─ Cache lookup: 2ms
      └─ Database query: 175ms  ← BOTTLENECK
```

### Step 1: Install Tempo

```bash
# Create values file
cat <<EOF > tempo-values.yaml
tempo:
  resources:
    limits:
      cpu: 500m
      memory: 1Gi
    requests:
      cpu: 250m
      memory: 512Mi

  storage:
    trace:
      backend: local  # Use S3 for production
      local:
        path: /var/tempo/traces

# Persistence
persistence:
  enabled: true
  size: 20Gi

# Configure for Grafana
tempoQuery:
  enabled: true
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
EOF

# Install
helm install tempo grafana/tempo \
  -n monitoring \
  -f tempo-values.yaml
```

### Step 2: Configure Tempo in Grafana

**Add data source**:

1. Configuration → Data Sources → Add → Tempo
2. URL: `http://tempo:3100`
3. Save & test

### Step 3: Instrument Applications

**Applications need to send traces to Tempo**

**Option 1: OpenTelemetry** (Recommended):

```python
# Python example with OpenTelemetry
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Configure tracer
trace.set_tracer_provider(TracerProvider())
otlp_exporter = OTLPSpanExporter(endpoint="tempo:4317", insecure=True)
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(otlp_exporter))

# Create span
tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("my-operation"):
    # Your code here
    do_work()
```

**Option 2: Jaeger client** (if already using Jaeger):

```python
# Jaeger is compatible with Tempo
import opentracing
from jaeger_client import Config

config = Config(
    config={
        'sampler': {'type': 'const', 'param': 1},
        'local_agent': {'reporting_host': 'tempo', 'reporting_port': 6831}
    },
    service_name='my-service'
)
tracer = config.initialize_tracer()
```

### Step 4: View Traces

**In Grafana → Explore → Tempo**:

1. Query by Trace ID (if you have one)
2. Or search traces by service name
3. Click trace to see flame graph
4. Analyze slow spans

---

## Phase 4: Unified Observability

### Goal
Correlate metrics, logs, and traces in Grafana.

### The Power of Correlation

**Scenario**: High error rate alert

**Traditional debugging**:
1. Check Prometheus: Error rate spike at 10:15
2. Check Loki: Find error logs manually
3. No connection to traces

**With correlation**:
1. Prometheus alert → click "View in Grafana"
2. Dashboard shows error rate spike
3. Click "View logs" → Loki shows errors from that time
4. Click "View trace" → Tempo shows slow requests
5. Root cause found in 2 minutes!

### Step 1: Enable Exemplars

**Exemplars** = links from metrics to traces

**Update Prometheus values**:

```yaml
# prometheus-values.yaml
prometheus:
  prometheusSpec:
    enableFeatures:
      - exemplar-storage
    exemplars:
      maxSize: 100000
```

**Upgrade Prometheus**:

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f prometheus-values.yaml
```

### Step 2: Configure Data Source Links

**In Grafana UI**:

1. Configuration → Data Sources → Prometheus
2. Scroll to "Exemplars" section
3. Internal link → Add
4. Data source: Tempo
5. URL label: traceID
6. Save

**Now**: Metrics can link to traces!

### Step 3: Create Unified Dashboard

**Example dashboard with all three**:

**Row 1: Metrics**
- Panel: Request rate (Prometheus)
- Query: `rate(http_requests_total[5m])`
- On click: Link to logs

**Row 2: Logs**
- Panel: Error logs (Loki)
- Query: `{job="my-app"} |~ "error"`
- On click: Link to trace

**Row 3: Traces**
- Panel: Trace timeline (Tempo)
- Filtered by time range

---

## Phase 5: Alerting

### Goal
Get notified before users complain.

### Understanding Alert Rules

**Alert anatomy**:
1. **Condition**: When to fire (e.g., CPU > 80%)
2. **Duration**: How long condition lasts (e.g., 5 minutes)
3. **Severity**: Critical, warning, info
4. **Route**: Where to send (email, Slack, PagerDuty)

### Step 1: Create Prometheus Alert Rules

```bash
# Create PrometheusRule custom resource
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: monitoring
spec:
  groups:
  - name: cluster-health
    interval: 30s
    rules:
    # Alert: Node down
    - alert: NodeDown
      expr: up{job="node-exporter"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ \$labels.instance }} is down"
        description: "Node has been down for more than 5 minutes"

    # Alert: High CPU
    - alert: HighCPUUsage
      expr: |
        100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High CPU on {{ \$labels.instance }}"
        description: "CPU usage is {{ \$value }}%"

    # Alert: Pod CrashLoop
    - alert: PodCrashLoop
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ \$labels.pod }} is crash looping"
        description: "Container {{ \$labels.container }} has restarted {{ \$value }} times"

    # Alert: High memory
    - alert: HighMemoryUsage
      expr: |
        (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High memory on {{ \$labels.instance }}"
        description: "Memory usage is {{ \$value }}%"
EOF

# Verify rules loaded
kubectl get prometheusrules -n monitoring
```

### Step 2: Configure Alertmanager

**Create Alertmanager config**:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-prometheus-kube-prometheus-alertmanager
  namespace: monitoring
type: Opaque
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m

    # Route alerts to receivers
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'default'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty'
      - match:
          severity: warning
        receiver: 'slack'

    # Receivers (notification endpoints)
    receivers:
    - name: 'default'
      email_configs:
      - to: 'alerts@example.com'
        from: 'prometheus@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'your-email@gmail.com'
        auth_password: 'your-app-password'

    - name: 'slack'
      slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: 'YOUR-PAGERDUTY-KEY'
EOF

# Restart Alertmanager
kubectl rollout restart statefulset -n monitoring alertmanager-prometheus-kube-prometheus-alertmanager
```

### Step 3: Test Alerts

**Trigger test alert**:

```bash
# Create high CPU pod
kubectl run cpu-hog --image=progrium/stress -- --cpu 4

# Wait 10-15 minutes for HighCPUUsage alert

# Check alerts in Grafana
# Alerting → Alert Rules

# Clean up
kubectl delete pod cpu-hog
```

---

## Monitoring Best Practices

### 1. Resource Limits

**Set limits on observability stack** to prevent it from consuming all resources:

```yaml
# In helm values
resources:
  limits:
    cpu: 2
    memory: 4Gi
  requests:
    cpu: 500m
    memory: 1Gi
```

### 2. Retention Policies

**Don't keep data forever**:

```yaml
# Prometheus
retention: 7d  # 7 days for metrics

# Loki
retention_period: 168h  # 7 days for logs

# Tempo
retention: 336h  # 14 days for traces
```

### 3. Alert Fatigue

**Avoid too many alerts**:
- Use `for: 10m` to avoid flapping
- Group related alerts
- Route low-priority to different channels
- Silence known issues during maintenance

### 4. Dashboard Organization

**Organize dashboards by**:
- **Infrastructure**: Nodes, pods, networking
- **Applications**: Per-service dashboards
- **Business**: User-facing metrics (sign-ups, revenue)

---

## Troubleshooting

### Common Issues

**Issue 1: Prometheus High Memory**

```bash
# Check Prometheus memory
kubectl top pod -n monitoring -l app.kubernetes.io/name=prometheus

# Reduce retention or add more RAM
# In prometheus-values.yaml:
retention: 3d  # Reduce from 7d
```

**Issue 2: No Logs in Loki**

```bash
# Check Promtail pods
kubectl get pods -n monitoring -l app.kubernetes.io/name=promtail

# Check Promtail logs
kubectl logs -n monitoring -l app.kubernetes.io/name=promtail

# Common cause: Promtail can't reach Loki
kubectl exec -n monitoring -it <promtail-pod> -- wget -O- http://loki:3100/ready
```

**Issue 3: Grafana Can't Reach Datasources**

```bash
# Check Grafana pod
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana

# Test connectivity from Grafana pod
kubectl exec -n monitoring -it <grafana-pod> -- curl http://prometheus-kube-prometheus-prometheus:9090/-/healthy
kubectl exec -n monitoring -it <grafana-pod> -- curl http://loki:3100/ready
```

---

## Next Steps

1. Deploy observability stack (Prometheus, Loki, Tempo)
2. Access Grafana and explore dashboards
3. Create custom dashboards for your apps
4. Configure alerting rules
5. Instrument applications with traces
6. Set up long-term storage (S3, etc.)

---

*This guide provides a complete observability solution for Kubernetes. With metrics, logs, and traces unified in Grafana, you have full visibility into your cluster.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial K8s observability stack guide
