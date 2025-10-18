# 🔧 Stage 8: Real-World Troubleshooting & Optimization

## 🎯 Goal

Practice diagnosing and resolving complex incidents under real constraints. Move from theoretical knowledge to hands-on incident response, root cause analysis, and long-term system optimization.

---

## 🧠 Core Concepts

| Concept | Meaning | Example |
|---------|---------|---------|
| **MTTR** (Mean Time To Recovery) | Average time to restore service after an incident | 15 minutes from alert to resolution |
| **MTBF** (Mean Time Between Failures) | Average time between system failures | 720 hours (30 days) between outages |
| **RCA** (Root Cause Analysis) | Systematic investigation to find underlying problem | Network partition caused cascading failures |
| **Postmortem** | Blameless review document after incidents | "What happened, why, and how to prevent" |
| **WAL** (Write-Ahead Log) | Prometheus log file for crash recovery | `/data/wal/` directory contains uncommitted data |
| **Correlation** | Linking metrics, logs, and traces to find patterns | High latency + DB connection errors + trace span |
| **Self-Healing** | Automated remediation of known issues | Auto-restart pod when memory threshold exceeded |
| **Continuous Profiling** | Always-on performance monitoring | CPU flamegraphs, memory allocations over time |

---

## 🔍 Key Troubleshooting Scenarios

### 🔹 Scenario 1: Latency Spikes Every Few Hours

**Problem:**  
API latency jumps from 50ms to 2000ms every 3-4 hours, then recovers.

**Investigation Steps:**

1. **Check Metrics Pattern**
```promql
# Visualize latency over 24h
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Check if correlated with resource exhaustion
rate(container_memory_working_set_bytes[5m])
```

2. **Correlate with Logs**
```logql
# Look for errors during spike windows
{job="api"} |= "error" | logfmt | line_format "{{.timestamp}} {{.message}}"
```

3. **Analyze Traces**
- Open Grafana → Explore → Tempo
- Search traces during spike timeframe
- Identify slow spans (likely DB queries, external API calls)

4. **Common Root Causes**
- **Garbage collection pauses** → Check JVM/Go GC metrics
- **Database connection pool exhaustion** → Monitor `db_connections_active`
- **Periodic batch jobs** → Cross-reference with cron schedules
- **Memory leak** → Check memory growth pattern
- **Cache expiration** → All cache entries expire simultaneously

**Example Solution:**
```yaml
# If caused by cache stampede, implement staggered TTL
cache:
  ttl: 3600
  jitter: 300  # Random 0-300s added to TTL
```

**📊 Dashboard Query for Detection:**
```promql
# Alert when p95 latency exceeds baseline by 4x
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) 
  > 
4 * histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1h] offset 1d))
```

---

### 🔹 Scenario 2: Alerts Fire but Logs Show No Error

**Problem:**  
Prometheus alerts indicate service degradation, but application logs appear clean.

**Investigation Steps:**

1. **Verify Alert Accuracy**
```promql
# Check what triggered the alert
up{job="my-service"} == 0
```

2. **Check Network Dependencies**
```bash
# Test connectivity from monitoring stack
curl -v http://service:8080/health

# Check DNS resolution
nslookup service.namespace.svc.cluster.local

# Verify service mesh / proxy logs
kubectl logs -n istio-system istio-proxy
```

3. **Inspect Infrastructure Layer**
- Network policies blocking Prometheus scraper
- Load balancer health checks failing
- Service mesh sidecar issues
- TLS certificate expiration

4. **Check Prometheus Targets Page**
```
http://prometheus:9090/targets
```
Look for scrape errors: "context deadline exceeded", "connection refused"

**Common Root Causes:**
- **Firewall rules** blocking Prometheus scraper IP
- **Service mesh mTLS** misconfiguration
- **Kubernetes NetworkPolicy** blocking monitoring namespace
- **Exporter crash** (not application crash)
- **Port mismatch** in Service definition

**Example Fix:**
```yaml
# NetworkPolicy allowing Prometheus scraping
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scraping
spec:
  podSelector:
    matchLabels:
      app: my-service
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 8080
```

---

### 🔹 Scenario 3: Exporters Stop After Kubernetes Upgrade

**Problem:**  
After upgrading from K8s 1.25 → 1.27, node-exporter and kube-state-metrics stop reporting.

**Root Cause Analysis Steps:**

1. **Check Exporter Pod Status**
```bash
kubectl get pods -n monitoring
kubectl describe pod node-exporter-xxxxx -n monitoring
kubectl logs node-exporter-xxxxx -n monitoring
```

2. **Check for API Deprecations**
- K8s 1.25+ removed several beta APIs
- Check if exporters use deprecated APIs

3. **Verify RBAC Permissions**
```bash
# Check if ServiceAccount still has permissions
kubectl auth can-i list nodes --as=system:serviceaccount:monitoring:node-exporter
```

4. **Review Upgrade Notes**
```bash
# Check Kubernetes release notes
# Common breaking changes:
# - PodSecurityPolicy removed (replaced by PodSecurity admission)
# - DaemonSet rolling update changes
# - API version bumps
```

**Common Issues & Fixes:**

**Issue 1: PodSecurityPolicy Removed**
```yaml
# Replace with PodSecurity admission controller
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

**Issue 2: DaemonSet Update Strategy**
```yaml
# Update DaemonSet to use new rolling update
apiVersion: apps/v1
kind: DaemonSet
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
```

**Issue 3: Deprecated Metrics API**
```bash
# Upgrade exporter to compatible version
helm upgrade node-exporter prometheus-community/node-exporter \
  --version 4.24.0 \
  --namespace monitoring
```

---

### 🔹 Scenario 4: Prometheus Crashes During Outage

**Problem:**  
Prometheus crashes mid-incident, losing recent metrics data. Need to recover and investigate.

**Recovery Steps:**

1. **Check Crash Cause**
```bash
# Review Prometheus logs
kubectl logs prometheus-0 -n monitoring --previous

# Common causes:
# - OOM (Out of Memory)
# - Disk full
# - Corrupted TSDB block
# - Query timeout causing cascading failure
```

2. **Recover from WAL (Write-Ahead Log)**
```bash
# Prometheus automatically replays WAL on startup
# WAL files are in /prometheus/wal/

# If manual recovery needed:
docker run -v /prometheus:/prometheus \
  prom/prometheus:latest \
  --storage.tsdb.path=/prometheus \
  --storage.tsdb.wal-replay-enabled=true
```

3. **Check TSDB Health**
```bash
# Use promtool to verify TSDB integrity
promtool tsdb analyze /prometheus/data

# Check for corrupted blocks
promtool tsdb list /prometheus/data
```

4. **Prevent Future Crashes**

**If OOM:**
```yaml
# Increase memory limits
resources:
  limits:
    memory: 8Gi
  requests:
    memory: 4Gi

# Reduce retention
--storage.tsdb.retention.time=15d

# Limit query concurrency
--query.max-concurrency=10
```

**If Disk Full:**
```bash
# Enable retention by size
--storage.tsdb.retention.size=50GB

# Set up disk alerts
- alert: PrometheusDiskFull
  expr: (node_filesystem_avail_bytes{mountpoint="/prometheus"} / node_filesystem_size_bytes) < 0.1
  for: 5m
```

**If Query Overload:**
```yaml
# Set query timeout
--query.timeout=2m

# Limit query samples
--query.max-samples=50000000
```

5. **Implement High Availability**
```yaml
# Deploy Prometheus HA pair
apiVersion: v1
kind: Service
metadata:
  name: prometheus-ha
spec:
  selector:
    app: prometheus
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
spec:
  replicas: 2  # HA pair
  serviceName: prometheus-ha
```

---

### 🔹 Scenario 5: Reduce Monitoring Cost by 40%

**Problem:**  
Prometheus storage costs are $5000/month. Leadership wants 40% reduction without losing critical visibility.

**Cost Analysis Steps:**

1. **Identify Top Metric Consumers**
```bash
# Get metrics cardinality
curl http://localhost:9090/api/v1/status/tsdb | jq -r '
  .data.seriesCountByMetricName[] | 
  "\(.name): \(.value)"
' | sort -t: -k2 -nr | head -20
```

2. **Categorize Metrics**
```
Critical (keep):
- SLI metrics (latency, errors, availability)
- Resource metrics (CPU, memory, disk)
- Business metrics (transactions, revenue)

Nice-to-have (downsample):
- Detailed HTTP route metrics
- Per-pod network stats

Redundant (drop):
- Duplicate metrics from multiple exporters
- Unused dashboard metrics
- High-cardinality debug metrics
```

3. **Implement Cost Reduction Strategy**

**A. Drop Unused Metrics (20% savings)**
```yaml
# prometheus.yml
metric_relabel_configs:
  # Drop Go runtime metrics for non-critical services
  - source_labels: [__name__]
    regex: 'go_(gc|memstats|threads).*'
    action: drop
    
  # Drop verbose kube-state-metrics
  - source_labels: [__name__]
    regex: 'kube_pod_container_status_.*'
    action: drop
    
  # Drop high-cardinality HTTP metrics
  - source_labels: [__name__, path]
    regex: 'http_requests_total;/api/v1/users/.*'
    action: drop
```

**B. Reduce Scrape Frequency (10% savings)**
```yaml
# Less critical services: 60s → 5m
- job_name: 'batch-jobs'
  scrape_interval: 5m
  static_configs:
    - targets: ['batch-service:8080']
```

**C. Implement Retention Tiers (15% savings)**
```bash
# Short retention for high-res data
--storage.tsdb.retention.time=15d

# Deploy Thanos for long-term storage
# with aggressive downsampling
thanos:
  compact:
    retention:
      raw: 30d
      5m: 90d
      1h: 1y
```

**D. Use Federation for Multi-Cluster (10% savings)**
```yaml
# Central Prometheus federates from edge clusters
# Only critical metrics sent to central
- job_name: 'federate'
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
      - '{__name__=~"up|http_requests_total"}'
      - '{__name__=~"node_cpu_seconds_total"}'
  static_configs:
    - targets:
      - 'edge-prometheus-1:9090'
      - 'edge-prometheus-2:9090'
```

**4. Measure Results**

```promql
# Track metrics ingestion rate
rate(prometheus_tsdb_head_samples_appended_total[1h])

# Track storage usage
prometheus_tsdb_storage_blocks_bytes / 1024 / 1024 / 1024  # GB

# Calculate cost per metric
(monthly_cost / prometheus_tsdb_head_series)
```

**Expected Savings Breakdown:**
- Drop unused metrics: -$1000 (20%)
- Reduce scrape frequency: -$500 (10%)
- Retention optimization: -$750 (15%)
- Federation strategy: -$500 (10%)
- **Total: -$2750 (55% reduction)**

---

### 🔹 Scenario 6: Unified Observability Stack Migration

**Problem:**  
Migrate from disparate tools (Prometheus + ELK + Jaeger) to unified stack (Prometheus + Loki + Tempo) in Grafana.

**Migration Plan:**

**Phase 1: Assessment (Week 1)**
```bash
# Inventory current setup
- Prometheus: 500k active series
- Elasticsearch: 2TB logs, 500GB/day ingestion
- Jaeger: 100k spans/minute
- Grafana: 150 dashboards, 200 alerts

# Identify dependencies
- Which dashboards query multiple sources?
- Which alerts combine logs + metrics?
- Custom integrations to migrate
```

**Phase 2: Deploy New Stack (Week 2-3)**

```yaml
# Deploy Loki for logs
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
data:
  loki.yaml: |
    auth_enabled: false
    server:
      http_listen_port: 3100
    ingester:
      lifecycler:
        ring:
          kvstore:
            store: inmemory
          replication_factor: 1
    schema_config:
      configs:
      - from: 2024-01-01
        store: boltdb-shipper
        object_store: s3
        schema: v11
        index:
          prefix: loki_index_
          period: 24h
    storage_config:
      boltdb_shipper:
        active_index_directory: /loki/index
        cache_location: /loki/cache
      aws:
        s3: s3://my-bucket/loki
        region: us-east-1

---
# Deploy Promtail for log collection
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
spec:
  selector:
    matchLabels:
      app: promtail
  template:
    metadata:
      labels:
        app: promtail
    spec:
      serviceAccountName: promtail
      containers:
      - name: promtail
        image: grafana/promtail:latest
        args:
          - -config.file=/etc/promtail/promtail.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: promtail-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

```yaml
# Deploy Tempo for traces
apiVersion: v1
kind: ConfigMap
metadata:
  name: tempo-config
data:
  tempo.yaml: |
    server:
      http_listen_port: 3200
    distributor:
      receivers:
        jaeger:
          protocols:
            grpc:
              endpoint: 0.0.0.0:14250
            thrift_http:
              endpoint: 0.0.0.0:14268
        otlp:
          protocols:
            grpc:
              endpoint: 0.0.0.0:4317
            http:
              endpoint: 0.0.0.0:4318
    ingester:
      trace_idle_period: 10s
      max_block_bytes: 1_000_000
      max_block_duration: 5m
    compactor:
      compaction:
        block_retention: 168h  # 7 days
    storage:
      trace:
        backend: s3
        s3:
          bucket: my-tempo-bucket
          endpoint: s3.amazonaws.com
```

**Phase 3: Parallel Run (Week 4-6)**
```bash
# Run both systems simultaneously
# Compare data quality:
# - Log completeness (Loki vs ELK)
# - Trace accuracy (Tempo vs Jaeger)
# - Query performance
# - Storage costs

# Migrate dashboards incrementally
# - Update 10 dashboards/week
# - Keep old queries commented for rollback
```

**Phase 4: Cutover (Week 7)**
```yaml
# Update application instrumentation
# OLD: Send logs to Logstash
filebeat.output.logstash:
  hosts: ["logstash:5044"]

# NEW: Send logs to Loki via Promtail
# (Promtail auto-discovers pod logs)

# OLD: Send traces to Jaeger
jaeger:
  agent_host: jaeger-agent
  agent_port: 6831

# NEW: Send traces to Tempo
tempo:
  endpoint: tempo:4317
  protocol: grpc
```

**Phase 5: Decommission (Week 8)**
```bash
# Stop Jaeger collectors
kubectl scale deployment jaeger-collector --replicas=0

# Archive Elasticsearch data
# Keep 90 days for compliance
curl -X POST "elasticsearch:9200/_snapshot/my_backup/snapshot_final"

# Remove old exporters
helm uninstall elasticsearch-exporter
```

**Benefits Achieved:**
- **30% cost reduction** (Loki cheaper than Elasticsearch)
- **Single pane of glass** in Grafana
- **Faster queries** (10s → 2s for log searches)
- **Better correlation** (click from metric → log → trace)
- **Simplified maintenance** (one vendor, unified config)

---

### 🔹 Scenario 7: Build MTTR/MTBF Tracking Dashboard

**Problem:**  
Leadership wants visibility into reliability trends over time.

**Implementation:**

1. **Define Incident Tracking**
```yaml
# Create Prometheus recording rule
groups:
- name: incident_tracking
  interval: 1m
  rules:
  - record: incident:active
    expr: |
      count(ALERTS{alertstate="firing", severity=~"critical|warning"}) OR vector(0)
      
  - record: incident:duration_seconds
    expr: |
      time() - ALERTS_FOR_STATE{alertstate="firing"}
      
  - record: incident:resolved_total
    expr: |
      changes(ALERTS{alertstate="firing"}[24h]) / 2
```

2. **Calculate MTTR**
```promql
# Mean Time To Recovery (last 30 days)
avg_over_time(
  (
    timestamp(ALERTS{alertstate="resolved"}) 
    - 
    timestamp(ALERTS_FOR_STATE{alertstate="firing"})
  )[30d:]
)

# Or use incident tracking system API
# MTTR = sum(resolution_time) / count(incidents)
```

3. **Calculate MTBF**
```promql
# Mean Time Between Failures (last 30 days)
30 * 24 * 3600 / count(changes(ALERTS{alertstate="firing"}[30d]) / 2)

# More accurate with incident labels
(
  max(timestamp(incident_end)) - min(timestamp(incident_start))
) / count(incidents)
```

4. **Build Grafana Dashboard**

```json
{
  "dashboard": {
    "title": "Reliability Metrics",
    "panels": [
      {
        "title": "MTTR Trend (30d rolling)",
        "targets": [
          {
            "expr": "avg_over_time(incident:duration_seconds[30d]) / 60",
            "legendFormat": "MTTR (minutes)"
          }
        ],
        "thresholds": [
          {"value": 15, "color": "green"},
          {"value": 30, "color": "yellow"},
          {"value": 60, "color": "red"}
        ]
      },
      {
        "title": "MTBF Trend",
        "targets": [
          {
            "expr": "(30 * 24 * 3600) / (count(changes(ALERTS{alertstate=\"firing\"}[30d])) / 2)",
            "legendFormat": "MTBF (hours)"
          }
        ]
      },
      {
        "title": "Incident Frequency",
        "targets": [
          {
            "expr": "count(changes(ALERTS{alertstate=\"firing\"}[24h])) / 2",
            "legendFormat": "Incidents per day"
          }
        ]
      },
      {
        "title": "SLO Compliance",
        "targets": [
          {
            "expr": "(1 - (sum(rate(http_requests_total{status=~\"5..\"}[30d])) / sum(rate(http_requests_total[30d])))) * 100",
            "legendFormat": "Availability %"
          }
        ],
        "thresholds": [
          {"value": 99.9, "color": "green"},
          {"value": 99.5, "color": "yellow"},
          {"value": 99.0, "color": "red"}
        ]
      }
    ]
  }
}
```

5. **Add Executive Summary Variables**
```promql
# This month vs last month comparison
MTTR_current = avg_over_time(incident:duration_seconds[30d])
MTTR_previous = avg_over_time(incident:duration_seconds[30d] offset 30d)
MTTR_change_percent = ((MTTR_current - MTTR_previous) / MTTR_previous) * 100
```

---

### 🔹 Scenario 8: Automate Post-Incident Report Generation

**Problem:**  
Manual postmortem creation takes 2-3 hours per incident. Automate data collection from Grafana.

**Implementation:**

1. **Create Grafana Snapshot API Script**

```python
#!/usr/bin/env python3
import requests
import json
from datetime import datetime, timedelta

GRAFANA_URL = "http://grafana:3000"
GRAFANA_TOKEN = "your-api-token"

def generate_postmortem(incident_start, incident_end, alert_name):
    """
    Generate postmortem report with embedded Grafana data
    """
    
    # 1. Get relevant dashboard
    dashboard_uid = "incident-dashboard"
    
    # 2. Create snapshot for incident window
    snapshot_data = {
        "dashboard": get_dashboard(dashboard_uid),
        "expires": 86400,  # 24 hours
        "external": False,
        "time": {
            "from": incident_start,
            "to": incident_end
        }
    }
    
    snapshot = requests.post(
        f"{GRAFANA_URL}/api/snapshots",
        headers={"Authorization": f"Bearer {GRAFANA_TOKEN}"},
        json=snapshot_data
    ).json()
    
    # 3. Query metrics during incident
    metrics = {
        "error_rate": query_prometheus(
            f"rate(http_requests_total{{status=~'5..'}}[5m])",
            incident_start,
            incident_end
        ),
        "latency_p95": query_prometheus(
            f"histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            incident_start,
            incident_end
        ),
        "cpu_usage": query_prometheus(
            f"rate(container_cpu_usage_seconds_total[5m])",
            incident_start,
            incident_end
        )
    }
    
    # 4. Get logs during incident
    logs = query_loki(
        f'{{job="api"}} |= "error"',
        incident_start,
        incident_end,
        limit=100
    )
    
    # 5. Get traces
    traces = query_tempo(
        incident_start,
        incident_end,
        service="api-gateway"
    )
    
    # 6. Calculate impact metrics
    impact = calculate_impact(metrics, incident_start, incident_end)
    
    # 7. Generate report
    report = generate_report_markdown(
        alert_name=alert_name,
        incident_start=incident_start,
        incident_end=incident_end,
        snapshot_url=snapshot['url'],
        metrics=metrics,
        logs=logs[:20],  # Top 20 errors
        traces=traces[:10],  # Top 10 slow traces
        impact=impact
    )
    
    return report

def query_prometheus(query, start, end):
    """Query Prometheus range data"""
    response = requests.get(
        f"{GRAFANA_URL}/api/datasources/proxy/1/api/v1/query_range",
        headers={"Authorization": f"Bearer {GRAFANA_TOKEN}"},
        params={
            "query": query,
            "start": start,
            "end": end,
            "step": "60s"
        }
    )
    return response.json()

def query_loki(query, start, end, limit=100):
    """Query Loki logs"""
    response = requests.get(
        f"{GRAFANA_URL}/api/datasources/proxy/2/loki/api/v1/query_range",
        headers={"Authorization": f"Bearer {GRAFANA_TOKEN}"},
        params={
            "query": query,
            "start": start,
            "end": end,
            "limit": limit
        }
    )
    return response.json()

def calculate_impact(metrics, start, end):
    """Calculate business impact"""
    duration_minutes = (end - start) / 60
    
    # Calculate requests affected
    error_rate = metrics['error_rate']['data']['result'][0]['values']
    total_errors = sum([float(v[1]) for v in error_rate]) * 60  # per second * 60
    
    # Estimate revenue impact (example: $0.50 per transaction)
    revenue_impact = total_errors * 0.50
    
    return {
        "duration_minutes": duration_minutes,
        "total_errors": int(total_errors),
        "requests_affected": int(total_errors),
        "revenue_impact_usd": round(revenue_impact, 2),
        "users_impacted": int(total_errors * 0.3)  # Estimate
    }

def generate_report_markdown(alert_name, incident_start, incident_end, 
                            snapshot_url, metrics, logs, traces, impact):
    """Generate markdown postmortem report"""
    
    report = f"""
# Incident Postmortem: {alert_name}

**Date:** {datetime.fromtimestamp(incident_start).strftime('%Y-%m-%d %H:%M:%S')} - {datetime.fromtimestamp(incident_end).strftime('%Y-%m-%d %H:%M:%S')}  
**Duration:** {impact['duration_minutes']:.1f} minutes  
**Severity:** Critical  
**Status:** Resolved  

---

## Executive Summary

Service experienced elevated error rates impacting {impact['users_impacted']} users over {impact['duration_minutes']:.0f} minutes.

**Impact:**
- **Requests Affected:** {impact['requests_affected']:,}
- **Error Rate Peak:** {max([float(v[1]) for v in metrics['error_rate']['data']['result'][0]['values']]):.2%}
- **Latency Peak:** {max([float(v[1]) for v in metrics['latency_p95']['data']['result'][0]['values']]):.0f}ms
- **Estimated Revenue Impact:** ${impact['revenue_impact_usd']:,.2f}

---

## Timeline

**{datetime.fromtimestamp(incident_start).strftime('%H:%M:%S')}** - Alert fired: {alert_name}  
**{datetime.fromtimestamp(incident_start + 120).strftime('%H:%M:%S')}** - On-call engineer acknowledged  
**{datetime.fromtimestamp(incident_start + 300).strftime('%H:%M:%S')}** - Root cause identified  
**{datetime.fromtimestamp(incident_end - 60).strftime('%H:%M:%S')}** - Fix deployed  
**{datetime.fromtimestamp(incident_end).strftime('%H:%M:%S')}** - Service recovered  

---

## Root Cause

[TO BE FILLED BY ENGINEER]

---

## Detection

Alert `{alert_name}` fired at {datetime.fromtimestamp(incident_start).strftime('%H:%M:%S')}.

**Alert Query:**
```promql
{metrics['error_rate']['data']['result'][0]['metric']}
```

---

## Impact Metrics

### Error Rate
![Grafana Dashboard]({snapshot_url})

### Top Error Messages
```
{chr(10).join([f"- {log['line'][:100]}" for log in logs[:5]])}
```

### Slowest Traces
```
{chr(10).join([f"- {trace['traceID']}: {trace['duration']}ms" for trace in traces[:5]])}
```

---

## Resolution

[TO BE FILLED BY ENGINEER]

---

## Action Items

- [ ] **Prevent:** [Add preventive measures]
- [ ] **Detect:** [Improve detection]
- [ ] **Mitigate:** [Faster mitigation steps]
- [ ] **Document:** [Update runbooks]

---

## Lessons Learned

### What Went Well
- Alert fired within 2 minutes of issue starting
- On-call responded quickly
- Rollback executed successfully

### What Could Be Improved
- [TO BE FILLED BY TEAM]

---

**Full Dashboard Snapshot:** {snapshot_url}  
**Generated:** {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
"""
    
    return report

# Example usage
if __name__ == "__main__":
    incident_start = int((datetime.now() - timedelta(hours=2)).timestamp())
    incident_end = int((datetime.now() - timedelta(hours=1)).timestamp())
    
    report = generate_postmortem(
        incident_start=incident_start,
        incident_end=incident_end,
        alert_name="HighErrorRate"
    )
    
    # Save to file
    with open(f"postmortem_{datetime.now().strftime('%Y%m%d_%H%M%S')}.md", "w") as f:
        f.write(report)
    
    print("✅ Postmortem report generated!")
```

2. **Integrate with Alertmanager**

```yaml
# alertmanager.yml
receivers:
- name: 'postmortem-generator'
  webhook_configs:
  - url: 'http://postmortem-api:8080/generate'
    send_resolved: true

route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 24h
  receiver: 'postmortem-generator'