# 🚀 Goal 6: Scaling & Reliability of Monitoring Stack

**Objective:** Build a production-grade, highly available, and infinitely scalable observability platform capable of monitoring thousands of microservices across multiple regions.

---

## 📑 Table of Contents

1. [Understanding the Challenge](#understanding-the-challenge)
2. [Core Concepts](#core-concepts)
3. [High Availability (HA)](#high-availability-ha)
4. [Federation Architecture](#federation-architecture)
5. [Sharding Strategy](#sharding-strategy)
6. [Thanos Integration](#thanos-integration)
7. [HA Alertmanager](#ha-alertmanager)
8. [Exporters at Scale](#exporters-at-scale)
9. [Resource Optimization](#resource-optimization)
10. [Hands-On Implementation](#hands-on-implementation)
11. [Load Testing & Benchmarking](#load-testing--benchmarking)
12. [STAR Case Study](#star-case-study)

---

## 🎯 Understanding the Challenge

### The Scaling Problem

**Scenario: Your Company is Growing**

```
Year 1: 50 microservices
├── 1 Prometheus server
├── Basic monitoring
└── Works fine ✅

Year 2: 500 microservices
├── 1 Prometheus server (struggling)
├── Slow queries (10s+)
├── Occasional OOM crashes
└── Missing data ⚠️

Year 3: 5,000 microservices
├── 1 Prometheus server (failing)
├── Cannot scrape all targets
├── Constant crashes
├── 50% data loss
└── Production incidents ❌
```

### Why Single Prometheus Fails at Scale

**Architectural Limitations:**

1. **Single-node architecture** - No built-in clustering
2. **Memory constraints** - Time series stored in RAM
3. **Scrape capacity limits** - ~10K targets maximum
4. **Query performance** - Slows with data volume
5. **No replication** - Single point of failure
6. **Limited retention** - Disk space constraints

### What "Production-Grade" Means

**Requirements:**
- ✅ **99.9% availability** - No monitoring downtime
- ✅ **Horizontal scalability** - Add capacity without limits
- ✅ **Data redundancy** - No single point of failure
- ✅ **Long-term storage** - Years of historical data
- ✅ **Fast queries** - Sub-second response time
- ✅ **Global visibility** - Cross-region aggregation
- ✅ **Cost efficiency** - Optimized resource usage

### The Solution Stack

```
┌─────────────────────────────────────────────────┐
│        Production Observability Stack           │
├─────────────────────────────────────────────────┤
│  High Availability (HA)                         │
│  ├── Multiple Prometheus replicas               │
│  ├── HA Alertmanager cluster                    │
│  └── Load-balanced Grafana                      │
├─────────────────────────────────────────────────┤
│  Horizontal Scaling                             │
│  ├── Sharding (distributed scraping)            │
│  ├── Federation (hierarchical aggregation)      │
│  └── Auto-scaling (Kubernetes HPA)              │
├─────────────────────────────────────────────────┤
│  Long-Term Storage                              │
│  ├── Thanos (object storage backend)            │
│  ├── Unlimited retention                        │
│  └── Downsampling for efficiency                │
├─────────────────────────────────────────────────┤
│  Performance Optimization                       │
│  ├── Recording rules                            │
│  ├── Cardinality management                     │
│  └── Resource tuning                            │
└─────────────────────────────────────────────────┘
```

---

## 🧠 Core Concepts

### Concept Summary Table

| Concept | Purpose | Benefit | Use Case |
|---------|---------|---------|----------|
| **High Availability** | Eliminate single points of failure | 99.9% uptime | Production systems |
| **Federation** | Hierarchical metric aggregation | Global visibility | Multi-region deployments |
| **Sharding** | Distribute scrape load | Horizontal scaling | 10K+ targets |
| **Thanos** | Long-term storage + HA queries | Unlimited retention | Compliance, analytics |
| **HA Alertmanager** | Deduplicated alerting | Reliable notifications | Critical alerts |
| **Auto-scaling** | Dynamic resource allocation | Cost optimization | Variable load |

---

## 🏗️ High Availability (HA)

### What is High Availability?

**Definition:**  
Running multiple instances of the same service to ensure continuous operation even when individual components fail.

**The Problem Without HA:**

```
Single Prometheus Server
        │
        ▼
    [Server Crashes]
        │
        ▼
    ❌ No metrics
    ❌ No alerts
    ❌ Blind to production
    ❌ Incidents undetected
```

**The Solution With HA:**

```
┌─────────────┐  ┌─────────────┐
│ Prometheus  │  │ Prometheus  │
│   Node 1    │  │   Node 2    │
└──────┬──────┘  └──────┬──────┘
       │                │
       ├────────────────┤
       │   Both scrape  │
       │   same targets │
       └────────┬───────┘
                ▼
         [If Node 1 fails,
          Node 2 continues]
              ✅
```

### HA Architecture Components

#### 1. Prometheus HA Setup

**Configuration:**

```
Deployment:
├── prometheus-0 (scraping all targets)
├── prometheus-1 (scraping same targets)
└── Both send to Alertmanager
```

**Key Points:**
- Both instances scrape **identical targets**
- Both instances have **same configuration**
- Both instances fire alerts
- Alertmanager **deduplicates** alerts
- Query either instance (or both via load balancer)

**Benefits:**
- ✅ Zero data loss if one node fails
- ✅ Continuous alerting
- ✅ Seamless failover
- ✅ No SPOF (Single Point of Failure)

**Trade-offs:**
- ⚠️ 2x resource usage (acceptable cost for reliability)
- ⚠️ Slight alert delay (deduplication window)

#### 2. Grafana HA Setup

**Configuration:**

```yaml
grafana:
  replicas: 3
  persistence:
    enabled: true
    storageClass: ssd
  database:
    type: postgres  # Shared external DB
    host: postgres.service
```

**Architecture:**

```
┌─────────────────────────────────────┐
│      Load Balancer (NGINX)          │
└────┬────────┬────────┬──────────────┘
     │        │        │
┌────▼────┐ ┌─▼──────┐ ┌▼─────────┐
│Grafana 1│ │Grafana 2│ │Grafana 3 │
└────┬────┘ └─┬──────┘ └┬─────────┘
     │        │        │
     └────────┴────────┴──────────┐
                                  │
                         ┌────────▼────────┐
                         │ Shared Database │
                         │   (PostgreSQL)  │
                         └─────────────────┘
```

**Key Features:**
- Session persistence (shared DB)
- Dashboard synchronization
- User authentication shared
- Load distribution

#### 3. Storage Considerations

**Local vs Shared Storage:**

| Storage Type | Use Case | Pros | Cons |
|--------------|----------|------|------|
| **Local Disk** | Prometheus data | Fast, simple | Lost if node fails |
| **Network Storage** | Grafana configs | Shared across replicas | Slower, complexity |
| **Object Storage** | Thanos long-term | Unlimited, cheap | Query latency |

### HA Deployment Example

#### Docker Compose Setup

**docker-compose-ha.yml:**

```yaml
version: "3.8"

services:
  # Prometheus Node 1
  prometheus-1:
    image: prom/prometheus:v2.52.0
    container_name: prometheus-1
    hostname: prometheus-1
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
      - '--web.external-url=http://prometheus-1:9090'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts.yml:/etc/prometheus/alerts.yml
      - prometheus-1-data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped
    networks:
      - monitoring

  # Prometheus Node 2 (HA replica)
  prometheus-2:
    image: prom/prometheus:v2.52.0
    container_name: prometheus-2
    hostname: prometheus-2
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
      - '--web.external-url=http://prometheus-2:9091'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts.yml:/etc/prometheus/alerts.yml
      - prometheus-2-data:/prometheus
    ports:
      - "9091:9090"
    restart: unless-stopped
    networks:
      - monitoring

  # Alertmanager Node 1
  alertmanager-1:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager-1
    hostname: alertmanager-1
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--cluster.listen-address=0.0.0.0:9094'
      - '--cluster.peer=alertmanager-2:9094'
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-1-data:/alertmanager
    ports:
      - "9093:9093"
    restart: unless-stopped
    networks:
      - monitoring

  # Alertmanager Node 2 (HA replica)
  alertmanager-2:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager-2
    hostname: alertmanager-2
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--cluster.listen-address=0.0.0.0:9094'
      - '--cluster.peer=alertmanager-1:9094'
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-2-data:/alertmanager
    ports:
      - "9094:9093"
    restart: unless-stopped
    networks:
      - monitoring

  # Grafana with HA support
  grafana:
    image: grafana/grafana:10.0.0
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_DATABASE_TYPE=postgres
      - GF_DATABASE_HOST=postgres:5432
      - GF_DATABASE_NAME=grafana
      - GF_DATABASE_USER=grafana
      - GF_DATABASE_PASSWORD=grafana
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - postgres
    restart: unless-stopped
    networks:
      - monitoring

  # PostgreSQL for Grafana shared state
  postgres:
    image: postgres:15
    container_name: postgres
    environment:
      - POSTGRES_DB=grafana
      - POSTGRES_USER=grafana
      - POSTGRES_PASSWORD=grafana
    volumes:
      - postgres-data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - monitoring

  # Node Exporter (example target)
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  prometheus-1-data:
  prometheus-2-data:
  alertmanager-1-data:
  alertmanager-2-data:
  grafana-data:
  postgres-data:

networks:
  monitoring:
    driver: bridge
```

**prometheus.yml (for both nodes):**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    replica: '0'  # Change to '1' for prometheus-2

# Alert to both Alertmanagers
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager-1:9093'
            - 'alertmanager-2:9093'

rule_files:
  - 'alerts.yml'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # Scrape other Prometheus instance for federation
  - job_name: 'prometheus-ha'
    static_configs:
      - targets: ['prometheus-1:9090', 'prometheus-2:9090']
```

**alertmanager.yml:**

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        api_url: 'YOUR_SLACK_WEBHOOK_URL'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

# Inhibition to prevent alert storms
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

### Testing HA Setup

**1. Verify both Prometheus instances are running:**
```bash
# Check targets on both instances
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].health'
curl http://localhost:9091/api/v1/targets | jq '.data.activeTargets[].health'
```

**2. Test failover:**
```bash
# Stop Prometheus node 1
docker stop prometheus-1

# Verify metrics still available from node 2
curl http://localhost:9091/api/v1/query?query=up

# Restart node 1
docker start prometheus-1
```

**3. Verify Alertmanager clustering:**
```bash
# Check cluster status on both nodes
curl http://localhost:9093/api/v1/status
curl http://localhost:9094/api/v1/status
```

---

## 🌐 Federation Architecture

### What is Federation?

**Definition:**  
A hierarchical Prometheus setup where a top-level (global) Prometheus scrapes aggregated metrics from multiple lower-level (regional) Prometheus instances.

### Why Federation?

**The Problem:**

```
Single Global Prometheus
├── Scrapes 10,000 targets directly
├── High CPU/Memory usage
├── Slow queries
├── Cannot scale further
└── Regional latency issues
```

**The Solution:**

```
        Global Prometheus
        (100 targets)
              │
      ┌───────┴───────┐
      │               │
US-East Prometheus  EU-West Prometheus
(5,000 targets)     (5,000 targets)
      │                     │
  [Local Services]    [Local Services]
```

### Federation Patterns

#### Pattern 1: Geographic Federation

**Use Case:** Multi-region deployments

```
                 ┌─────────────────┐
                 │ Global/HQ       │
                 │ Prometheus      │
                 │ (Aggregated)    │
                 └────────┬────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
│ US-East      │  │ US-West     │  │ EU-Central  │
│ Prometheus   │  │ Prometheus  │  │ Prometheus  │
│ (Regional)   │  │ (Regional)  │  │ (Regional)  │
└──────────────┘  └─────────────┘  └─────────────┘
```

**Benefits:**
- ✅ Reduced latency (scrape locally)
- ✅ Regional isolation
- ✅ Global visibility
- ✅ Disaster recovery per region

#### Pattern 2: Environment Federation

**Use Case:** Dev/Staging/Production separation

```
            ┌──────────────────┐
            │ Production       │
            │ Federation       │
            │ Prometheus       │
            └────────┬─────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌────▼────┐    ┌────▼────┐
│ Prod    │    │ Staging │    │ Dev     │
│ Cluster │    │ Cluster │    │ Cluster │
└─────────┘    └─────────┘    └─────────┘
```

#### Pattern 3: Team/Domain Federation

**Use Case:** Large organizations with team ownership

```
         ┌───────────────────┐
         │ Platform Team     │
         │ Global Prometheus │
         └─────────┬─────────┘
                   │
      ┌────────────┼────────────┐
      │            │            │
┌─────▼──────┐ ┌──▼──────┐ ┌──▼──────┐
│ API Team   │ │ Data    │ │ Infra   │
│ Prometheus │ │ Team    │ │ Team    │
└────────────┘ └─────────┘ └─────────┘
```

### Implementing Federation

#### Regional Prometheus Configuration

**us-east-prometheus.yml:**

```yaml
global:
  scrape_interval: 15s
  external_labels:
    region: 'us-east'
    cluster: 'production'

scrape_configs:
  - job_name: 'api-services'
    static_configs:
      - targets:
          - 'api-1:8080'
          - 'api-2:8080'
          # ... 5000 more targets

  - job_name: 'databases'
    static_configs:
      - targets:
          - 'mysql-1:9104'
          - 'postgres-1:9187'

  - job_name: 'node'
    static_configs:
      - targets:
          - 'node-1:9100'
          - 'node-2:9100'
          # ... 500 nodes
```

**eu-west-prometheus.yml:**

```yaml
global:
  scrape_interval: 15s
  external_labels:
    region: 'eu-west'
    cluster: 'production'

scrape_configs:
  # Similar configuration for EU region
  - job_name: 'api-services'
    static_configs:
      - targets:
          - 'api-eu-1:8080'
          - 'api-eu-2:8080'
```

#### Global Federation Prometheus

**global-prometheus.yml:**

```yaml
global:
  scrape_interval: 30s  # Slower scrape for federation
  external_labels:
    datacenter: 'global'

scrape_configs:
  # Federate from US-East
  - job_name: 'federate-us-east'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        # Only fetch aggregated metrics
        - '{job="api-services"}'
        - '{job="databases"}'
        - '{__name__=~".*:.*"}'  # Recording rules only
    static_configs:
      - targets:
          - 'prometheus-us-east:9090'
        labels:
          region: 'us-east'

  # Federate from EU-West
  - job_name: 'federate-eu-west'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="api-services"}'
        - '{job="databases"}'
        - '{__name__=~".*:.*"}'
    static_configs:
      - targets:
          - 'prometheus-eu-west:9090'
        labels:
          region: 'eu-west'

  # Federate from US-West
  - job_name: 'federate-us-west'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="api-services"}'
        - '{job="databases"}'
        - '{__name__=~".*:.*"}'
    static_configs:
      - targets:
          - 'prometheus-us-west:9090'
        labels:
          region: 'us-west'
```

### Federation Best Practices

#### 1. Use Recording Rules for Aggregation

**Regional Prometheus (recording_rules.yml):**

```yaml
groups:
  - name: federation_rules
    interval: 30s
    rules:
      # Pre-aggregate for federation
      - record: region:api_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (region, service)

      - record: region:api_errors:rate5m
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (region, service)

      - record: region:api_latency:p95
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, region, service))
```

**Benefits:**
- ✅ Reduced data transfer
- ✅ Faster federation queries
- ✅ Lower global Prometheus load

#### 2. Selective Metric Federation

**Don't federate everything:**

```yaml
# ❌ BAD: Federate all metrics
params:
  'match[]':
    - '{__name__!=""}'  # Everything

# ✅ GOOD: Federate only necessary metrics
params:
  'match[]':
    - '{job="api"}'                    # Specific jobs
    - '{__name__=~".*:.*"}'           # Recording rules
    - '{__name__=~"up|node_.*"}'      # Essential metrics
```

#### 3. Use `honor_labels: true`

```yaml
scrape_configs:
  - job_name: 'federate'
    honor_labels: true  # ← Keep original labels from source
    metrics_path: '/federate'
```

**Why:** Preserves original labels from regional Prometheus, avoiding conflicts.

### Docker Compose Federation Example

**docker-compose-federation.yml:**

```yaml
version: "3.8"

services:
  # Regional Prometheus - US East
  prometheus-us-east:
    image: prom/prometheus:v2.52.0
    container_name: prometheus-us-east
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    volumes:
      - ./configs/us-east-prometheus.yml:/etc/prometheus/prometheus.yml
      - ./configs/recording-rules.yml:/etc/prometheus/recording-rules.yml
      - prom-us-east-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - monitoring

  # Regional Prometheus - EU West
  prometheus-eu-west:
    image: prom/prometheus:v2.52.0
    container_name: prometheus-eu-west
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    volumes:
      - ./configs/eu-west-prometheus.yml:/etc/prometheus/prometheus.yml
      - ./configs/recording-rules.yml:/etc/prometheus/recording-rules.yml
      - prom-eu-west-data:/prometheus
    ports:
      - "9091:9090"
    networks:
      - monitoring

  # Global Federation Prometheus
  prometheus-global:
    image: prom/prometheus:v2.52.0
    container_name: prometheus-global
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    volumes:
      - ./configs/global-prometheus.yml:/etc/prometheus/prometheus.yml
      - prom-global-data:/prometheus
    ports:
      - "9092:9090"
    depends_on:
      - prometheus-us-east
      - prometheus-eu-west
    networks:
      - monitoring

  # Example targets (simulate services)
  node-exporter-us:
    image: prom/node-exporter:latest
    container_name: node-us
    networks:
      - monitoring

  node-exporter-eu:
    image: prom/node-exporter:latest
    container_name: node-eu
    networks:
      - monitoring

  # Grafana to visualize global view
  grafana:
    image: grafana/grafana:10.0.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - monitoring

volumes:
  prom-us-east-data:
  prom-eu-west-data:
  prom-global-data:

networks:
  monitoring:
    driver: bridge
```

### Testing Federation

**1. Verify regional Prometheus instances:**
```bash
# US-East
curl http://localhost:9090/api/v1/label/__name__/values | jq '.data[]' | wc -l

# EU-West
curl http://localhost:9091/api/v1/label/__name__/values | jq '.data[]' | wc -l
```

**2. Test federation endpoint:**
```bash
# Check what metrics are exposed for federation
curl 'http://localhost:9090/federate?match[]={job="node"}' | head -20
```

**3. Verify global Prometheus scraping:**
```bash
# Check federated metrics in global Prometheus
curl 'http://localhost:9092/api/v1/query?query=up{job="federate-us-east"}'
```

**4. Query across regions:**
```promql
# Total requests across all regions
sum(region:api_requests:rate5m)

# Compare regions
sum(region:api_requests:rate5m) by (region)

# Find highest error rate region
topk(1, sum(region:api_errors:rate5m) by (region))
```

---

## ⚡ Sharding Strategy

### What is Sharding?

**Definition:**  
Splitting Prometheus into multiple smaller servers, each responsible for scraping a **subset** of targets.

**Why Needed:**
- Prometheus is single-node (no clustering)
- Limited by CPU/memory of single machine
- Sharding enables horizontal scaling

### Sharding vs Federation

| Aspect | Sharding | Federation |
|--------|----------|------------|
| **Purpose** | Distribute scrape load | Aggregate metrics |
| **Targets** | Different targets per shard | Same targets, different regions |
| **Query** | Query specific shard or use global query layer | Query global Prometheus |
| **Use Case** | 10K+ targets in one location | Multi-region deployments |

### Sharding Strategies

#### Strategy 1: Hash-based Sharding

**Distribute targets by hash of target address:**

```yaml
# Shard 0 - Handles targets where hash(address) % 4 == 0
- job_name: 'node'
  static_configs:
    - targets:
        - 'node-1:9100'
        - 'node-2:9100'
        # ... all nodes
  relabel_configs:
    - source_labels: [__address__]
      modulus: 4
      target_label: __tmp_hash
      action: hashmod
    - source_labels: [__tmp_hash]
      regex: '0'
      action: keep

# Shard 1 - hash % 4 == 1
# Shard 2 - hash % 4 == 2
# Shard 3 - hash % 4 == 3
```

#### Strategy 2: Domain-based Sharding

**Separate by service type:**

```
Shard 1: API services (microservices)
Shard 2: Databases (MySQL, PostgreSQL, Redis)
Shard 3: Infrastructure (nodes, networks)
Shard 4: Applications (custom exporters)
```

**Configuration:**

```yaml
# shard-1-prometheus.yml (API Services)
scrape_configs:
  - job_name: 'api-services'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['api-*']

# shard-2-prometheus.yml (Databases)
scrape_configs:
  - job_name: 'databases'
    static_configs:
      - targets:
          - 'mysql-exporter:9104'
          - 'postgres-exporter:9187'
          - 'redis-exporter:9121'
```

#### Strategy 3: Namespace-based Sharding (Kubernetes)

**One shard per Kubernetes namespace:**

```yaml
# shard-team-a.yml
scrape_configs:
  - job_name: 'team-a-services'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['team-a']

# shard-team-b.yml
scrape_configs:
  - job_name: 'team-b-services'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['team-b']
```

### Implementing Sharding

#### Docker Compose Sharding Example

**docker-compose-sharding.yml:**

```yaml
version: "3.8"

services:
  # Shard 0 - API Services
  prometheus-shard-0:
    image: prom/prometheus:v2.52.0
    container_name: prom-shard-0
    command:
      - '--config.file=/etc/prometheus