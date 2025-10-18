# 🎯 Goal 3: Prometheus Advanced Concepts

**Objective:** Master Prometheus internals, scalability, and performance tuning with hands-on implementation.

---

## 📑 Table of Contents
1. [Concepts Overview](#concepts-overview)
2. [Project Structure](#project-structure)
3. [Phase 1: Base Stack Setup](#phase-1-base-stack-setup)
4. [Phase 2: Federation Setup](#phase-2-federation-setup)
5. [Phase 3: Thanos Integration](#phase-3-thanos-integration)
6. [Phase 4: Tuning & Optimization](#phase-4-tuning--optimization)
7. [Key Learnings Summary](#key-learnings-summary)

---

## 🧱 Concepts Overview

### 1️⃣ Prometheus Data Storage & Retention

**🧠 Concept:**
Prometheus stores all metric data locally on disk (usually in `/var/lib/prometheus`). This includes time series data, which grows quickly as metrics accumulate over time.

**🪣 Why It Matters:**
If not managed, Prometheus can run out of disk space or slow down queries due to too much data.

**⚙️ How to Control It:**
You can limit how much data Prometheus keeps:
- `--storage.tsdb.retention.time` → how long to retain data (e.g. 15 days)
- `--storage.tsdb.retention.size` → how much disk space to use (e.g. 5GB)
- `scrape_interval` → how often to collect data (e.g. every 30 seconds)

**📊 Example:**
```yaml
global:
  scrape_interval: 30s
  evaluation_interval: 30s
```

```bash
--storage.tsdb.retention.time=15d
--storage.tsdb.retention.size=5GB
```

---

### 2️⃣ Prometheus Federation

**🧠 Concept:**
Federation allows one Prometheus server to collect data from another Prometheus. Think of it like a "Prometheus of Prometheuses."

**🏗️ Why It's Used:**
- To combine multiple environments (like dev, test, prod) into a single dashboard
- To reduce load on local Prometheus servers by only aggregating needed metrics

**⚙️ How It Works:**
1. Each environment runs its own Prometheus
2. A top-level Prometheus scrapes data from them using `/federate` endpoint

**📊 Example Config:**
```yaml
scrape_configs:
  - job_name: 'federate'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]': ['{job=~".*"}']
    static_configs:
      - targets: ['prometheus1:9090', 'prometheus2:9090']
```

**✅ Result:** You get a single Prometheus dashboard with combined metrics from multiple servers.

---

### 3️⃣ Thanos Integration (Long-Term Storage + Global View)

**🧠 Concept:**
Thanos is an extension of Prometheus that provides:
- Long-term metric storage (S3, GCS, Azure)
- Global query view across clusters
- Deduplication and downsampling

**🏗️ Why It's Important:**
Prometheus alone can't store months/years of data or query across regions easily. Thanos solves that by acting as a distributed layer on top of Prometheus.

**⚙️ How It Works:**
1. A Thanos Sidecar runs next to each Prometheus
2. Sidecar uploads old data to object storage
3. Thanos Query component aggregates all Prometheus + sidecars for global queries

**🪣 Example Bucket Config:**
```yaml
type: FILESYSTEM
config:
  directory: /data/thanos
```

**✅ Result:** You can query historical data from all clusters using Thanos Query UI.

---

### 4️⃣ Recording Rules

**🧠 Concept:**
Recording rules pre-calculate frequently used queries and store the result as a new metric. This saves time and reduces query load on Grafana dashboards.

**📊 Example:**
```yaml
groups:
- name: node.rules
  rules:
  - record: instance:cpu:avg5m
    expr: avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)
```

**✅ Result:** When you open Grafana, it reads the precomputed metric `instance:cpu:avg5m` — making dashboards super fast.

---

### 5️⃣ Cardinality Explosion

**🧠 Concept:**
Each unique combination of metric name + label values is a separate time series. If you use labels like `user_id` or `session_id`, you'll create millions of time series — causing OOM (out-of-memory) issues.

**⚠️ Example of Bad Metrics:**
```
http_requests_total{user_id="12345", endpoint="/login"}
```

**🩹 Prevention Tips:**
- Avoid high-cardinality labels (uuid, ip, timestamp)
- Use aggregation: `sum by (endpoint)` instead of using all labels
- Check cardinality:
```promql
topk(10, count by (__name__)({__name__!=""}))
```

**✅ Result:** Lower memory footprint, faster queries, and stable Prometheus.

---

### 6️⃣ Meta-Monitoring Prometheus Itself

**🧠 Concept:**
Prometheus can scrape its own metrics (yes, it monitors itself).

**⚙️ Example:**
```yaml
scrape_configs:
  - job_name: 'self'
    static_configs:
      - targets: ['localhost:9090']
```

**📊 Important Metrics:**
- `prometheus_tsdb_head_series` → how many active time series
- `prometheus_engine_query_duration_seconds` → how long queries take

**✅ Result:** You can detect if Prometheus is becoming overloaded or slow.

---

### 7️⃣ Query Optimization & Profiling

**🧠 Concept:**
Prometheus includes a `/debug/pprof` endpoint to profile memory and CPU usage. You can use it to analyze performance bottlenecks.

**⚙️ Commands:**
```bash
curl http://localhost:9090/debug/pprof/heap > heap.out
go tool pprof heap.out
```

**✅ Result:** You can visualize what consumes memory and optimize slow queries or scrape configs.

---

## 📂 Project Structure

Create this folder structure in your GitHub repo:

```
prometheus-advanced/
│
├── phase1-base-stack/
│   ├── docker-compose.yml
│   └── prometheus.yml
│
├── phase2-federation/
│   ├── prometheus-main.yml
│   ├── prometheus-secondary.yml
│   └── docker-compose.yml
│
├── phase3-thanos/
│   ├── docker-compose.yml
│   ├── bucket_config.yaml
│   └── sidecar-config/
│
└── phase4-tuning/
    ├── recording_rules.yml
    ├── meta-monitoring.yml
    └── notes.md
```

---

## 🔹 Phase 1: Base Stack Setup

### docker-compose.yml
```yaml
version: "3"
services:
  prometheus:
    image: prom/prometheus:v2.52.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--storage.tsdb.retention.time=15d'
      - '--storage.tsdb.retention.size=5GB'
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: always

  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: always

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    restart: always

volumes:
  prometheus_data:
```

### prometheus.yml
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

### 🚀 Run the Stack
```bash
docker-compose up -d
```

### 🌐 Access Points
- Prometheus → `http://<EC2-IP>:9090`
- Grafana → `http://<EC2-IP>:3000` (admin/admin)

**✅ Outcome:** Optimized disk usage and better performance with retention policies.

---

## 🔹 Phase 2: Federation Setup

Create two Prometheus instances:
- `prometheus-main` (top-level)
- `prometheus-secondary` (child)

### docker-compose.yml
```yaml
version: "3"
services:
  prometheus-main:
    image: prom/prometheus
    container_name: prometheus-main
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus-main.yml:/etc/prometheus/prometheus.yml

  prometheus-secondary:
    image: prom/prometheus
    container_name: prometheus-secondary
    ports:
      - "9091:9090"
    volumes:
      - ./prometheus-secondary.yml:/etc/prometheus/prometheus.yml
```

### prometheus-secondary.yml
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

### prometheus-main.yml
```yaml
scrape_configs:
  - job_name: 'federate'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]': ['{job=~".*"}']
    static_configs:
      - targets: ['prometheus-secondary:9090']
```

### 🚀 Run and Verify
```bash
docker-compose up -d
```

Check `http://localhost:9090` → you'll see metrics from both servers.

**✅ Outcome:** Centralized monitoring across multiple environments.

---

## 🔹 Phase 3: Thanos Integration

### bucket_config.yaml
```yaml
type: FILESYSTEM
config:
  directory: /data/thanos
```

### docker-compose.yml
```yaml
version: "3"
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./data:/prometheus
    ports:
      - "9090:9090"

  thanos-sidecar:
    image: quay.io/thanos/thanos:v0.35.0
    container_name: thanos-sidecar
    command:
      - sidecar
      - --http-address=0.0.0.0:10902
      - --grpc-address=0.0.0.0:10901
      - --tsdb.path=/prometheus
      - --objstore.config-file=/etc/thanos/bucket_config.yaml
      - --prometheus.url=http://prometheus:9090
    ports:
      - "10901:10901"
      - "10902:10902"
    volumes:
      - ./data:/prometheus
      - ./bucket_config.yaml:/etc/thanos/bucket_config.yaml
```

### 🚀 Run Thanos
```bash
docker-compose up -d
```

**✅ Outcome:** Scalable, distributed monitoring with long-term storage in object storage.

---

## 🔹 Phase 4: Tuning & Optimization

### recording_rules.yml
```yaml
groups:
- name: system.rules
  rules:
  - record: instance:cpu:avg5m
    expr: avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)
```

Add it in `prometheus.yml`:
```yaml
rule_files:
  - "recording_rules.yml"
```

**✅ Benefits:**
- Speeds up Grafana dashboards
- Reduces query load

---

### meta-monitoring.yml
```yaml
scrape_configs:
  - job_name: 'self'
    static_configs:
      - targets: ['localhost:9090']
```

**✅ You'll now get metrics like:**
- `prometheus_tsdb_head_series`
- `prometheus_engine_query_duration_seconds`

Use Grafana to create dashboards monitoring Prometheus itself.

---

### Profiling Prometheus

Check memory and CPU profiling:
```bash
curl http://localhost:9090/debug/pprof/heap > heap.out
go tool pprof heap.out
```

**✅ Use to detect performance bottlenecks or query slowness.**

---

## 📊 Summary Table

| Concept | Purpose | Benefit |
|---------|---------|---------|
| Retention tuning | Manage disk usage | Better performance |
| Federation | Combine multiple Prometheus | Global visibility |
| Thanos | Long-term storage | Historical insights |
| Recording rules | Precompute queries | Faster dashboards |
| Cardinality control | Optimize memory | Stable Prometheus |
| Meta-monitoring | Self-observation | Early detection of issues |
| Profiling | Performance tuning | Efficient queries |

---

## 🧠 Key Learnings Summary

By completing this guide, you will have:

✅ **Deep understanding of Prometheus internals**
- Data storage mechanisms
- TSDB architecture
- Retention policies

✅ **Hands-on scalability setup**
- Multi-instance federation
- Distributed monitoring
- Cross-region aggregation

✅ **Long-term storage implementation**
- Thanos sidecar configuration
- Object storage integration
- Historical data querying

✅ **Performance optimization**
- Recording rules for efficiency
- Cardinality management
- Query profiling and debugging

✅ **Production-ready monitoring**
- Self-monitoring capabilities
- Alerting on Prometheus health
- Scalable architecture patterns

---

## 🚀 Quick Start Guide

### Step 1: Clone and Setup
```bash
git clone <your-repo>
cd prometheus-advanced/phase1-base-stack
```

### Step 2: Start Services
```bash
docker-compose up -d
```

### Step 3: Access Interfaces
- **Prometheus UI:** `http://localhost:9090`
- **Grafana:** `http://localhost:3000` (admin/admin)
- **Node Exporter:** `http://localhost:9100/metrics`

### Step 4: Progress Through Phases
Complete each phase sequentially to build comprehensive knowledge.

---

## 📝 Interview-Ready Talking Points

**"I built an advanced Prometheus observability platform featuring:**
- **Multi-tier federation** for aggregating metrics across dev, test, and production environments
- **Thanos integration** for long-term storage with 90+ days retention in S3
- **Recording rules** that reduced dashboard query time by 60%
- **Cardinality optimization** preventing memory issues across 200+ microservices
- **Meta-monitoring** with self-observability to detect Prometheus performance degradation
- **Query profiling** to identify and resolve bottlenecks in metric collection

**This architecture handled 10M+ time series with sub-second query performance."**

---

## # 🎯 Prometheus Advanced Observability Platform - STAR Case Study

## Interview Question Variations This Answers:
- "Tell me about a time you improved system observability"
- "Describe a complex monitoring solution you implemented"
- "Have you worked with Prometheus at scale?"
- "Tell me about a time you optimized infrastructure performance"

---

## 📋 Situation

**Context:**
Our organization was running **200+ microservices** across multiple AWS environments (dev, staging, production) with a fragmented monitoring approach. Each team was running their own Prometheus instances independently, leading to several critical issues:

**Problems:**
- **No centralized visibility** - Teams couldn't correlate issues across services
- **Data retention limited to 15 days** - Unable to analyze long-term trends or perform year-over-year comparisons
- **Performance degradation** - Prometheus instances were experiencing OOM crashes due to cardinality explosion (15M+ time series)
- **Slow dashboard loading** - Grafana dashboards took 30+ seconds to load due to complex queries
- **Legacy Nagios alerts** - Still dependent on aging infrastructure that wasn't integrated with modern stack

**Business Impact:**
- Mean Time to Detection (MTTD) was 20+ minutes
- Root cause analysis was difficult without historical data
- Teams spent 40% of on-call time just correlating metrics across systems

---

## 🎯 Task

**My Responsibility:**
As the **DevOps/SRE engineer**, I was tasked with:

1. **Design and implement** a unified, scalable observability platform
2. **Reduce monitoring costs** by optimizing resource usage
3. **Enable long-term metric retention** (90+ days) for compliance and trend analysis
4. **Improve query performance** to achieve sub-second dashboard loading
5. **Integrate legacy monitoring** (Nagios) into the modern stack
6. **Ensure high availability** with no single point of failure

**Success Criteria:**
- Achieve centralized visibility across all environments
- Dashboard load time under 3 seconds
- Support 90-day retention with queryable historical data
- Reduce Prometheus memory usage by 50%
- Zero data loss during migration

---

## ⚡ Action

I implemented a **four-phase approach** to build an enterprise-grade Prometheus observability platform:

### **Phase 1: Foundation & Optimization (Week 1-2)**

**What I Did:**
- **Deployed base monitoring stack** using Docker Compose on AWS EC2
  - Prometheus with optimized retention policies (`--storage.tsdb.retention.time=15d`, `--storage.tsdb.retention.size=5GB`)
  - Node Exporter for system metrics
  - Grafana for visualization
  - Alertmanager for alert routing

- **Implemented cardinality controls** to prevent memory issues:
  - Audited all metrics using `topk(10, count by (__name__)({__name__!=""}))`
  - Identified problematic labels (user_id, session_id causing 12M unnecessary time series)
  - Removed high-cardinality labels and replaced with aggregations
  - **Result:** Reduced active time series from 15M to 3M (80% reduction)

**Technical Details:**
```yaml
# Optimized scrape configuration
global:
  scrape_interval: 30s  # Reduced from 15s
  evaluation_interval: 30s
```

---

### **Phase 2: Federation Architecture (Week 3)**

**What I Did:**
- **Designed hierarchical federation** to aggregate metrics across environments
  - Each environment (dev/staging/prod) retained its local Prometheus
  - Deployed central "federation" Prometheus to aggregate critical metrics
  - Used selective metric federation (only business-critical metrics)

- **Configured federation endpoints:**
```yaml
scrape_configs:
  - job_name: 'federate'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]': 
        - '{job="api"}' # Only federate API metrics
        - '{job="database"}'
    static_configs:
      - targets: 
          - 'prometheus-prod:9090'
          - 'prometheus-staging:9090'
```

**Benefits:**
- Single pane of glass for leadership dashboards
- Reduced load on individual Prometheus instances
- Maintained environment isolation for troubleshooting

---

### **Phase 3: Long-Term Storage with Thanos (Week 4-5)**

**What I Did:**
- **Integrated Thanos** for unlimited retention and global querying
  - Deployed Thanos Sidecar alongside each Prometheus instance
  - Configured S3 bucket for object storage (cost: $0.023/GB vs EC2 disk)
  - Set up Thanos Query for unified query interface
  - Implemented downsampling (5m resolution after 7 days, 1h after 30 days)

- **Architecture components:**
  - **Thanos Sidecar:** Uploaded blocks to S3 every 2 hours
  - **Thanos Store Gateway:** Queried historical data from S3
  - **Thanos Query:** Federated real-time + historical queries
  - **Thanos Compactor:** Optimized storage with deduplication

**Configuration:**
```yaml
# bucket_config.yaml
type: S3
config:
  bucket: "company-metrics-prod"
  endpoint: "s3.us-east-1.amazonaws.com"
  region: "us-east-1"
```

**Technical Challenges Solved:**
- Handled clock skew between Prometheus instances using `--min-time` flags
- Implemented retention policies in S3 (90 days standard, 2 years glacier)
- Set up VPC endpoints to avoid S3 data transfer costs

---

### **Phase 4: Performance Optimization (Week 6)**

**What I Did:**

**A. Recording Rules for Query Performance**
- Created **25 recording rules** for frequently accessed metrics
- Pre-computed expensive aggregations (95th percentile latency, error rates)

```yaml
groups:
- name: api.rules
  interval: 30s
  rules:
  - record: api:request_duration:p95
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
  
  - record: instance:cpu:avg5m
    expr: avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)
```

**Impact:** Dashboard load time reduced from 30s to 2s (93% improvement)

**B. Meta-Monitoring**
- Configured Prometheus to monitor itself
- Set up alerts for Prometheus health issues:
  - High memory usage (>80%)
  - Query latency (>5s)
  - Scrape failures (>5% error rate)

```yaml
- alert: PrometheusHighMemory
  expr: process_resident_memory_bytes / node_memory_MemTotal_bytes > 0.8
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Prometheus memory usage high"
```

**C. Query Profiling**
- Used `/debug/pprof` endpoints to identify memory leaks
- Optimized slow queries causing timeouts
- Implemented query result caching in Grafana (5-minute TTL)

---

### **Phase 5: Integration & Security (Week 7)**

**What I Did:**

**Legacy Nagios Integration:**
- Deployed `nagios_exporter` to bridge legacy alerting
- Mapped Nagios checks to Prometheus metrics
- Gradually migrated critical alerts to Prometheus rules

**Security Hardening:**
- Implemented NGINX reverse proxy with TLS termination
- Configured OAuth2 proxy for SSO integration
- Applied network policies (only allowed access from corporate VPN)
- Enabled basic auth for Prometheus admin endpoints

**GitOps Implementation:**
- Stored all configurations in Git (prometheus rules, dashboards, alerts)
- Set up CI/CD pipeline for automated deployment
- Implemented automated dashboard provisioning in Grafana

```yaml
# Grafana provisioning
apiVersion: 1
providers:
  - name: 'default'
    folder: 'Microservices'
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

---

## 📊 Result

### **Quantifiable Outcomes:**

**Performance Improvements:**
- ✅ **Dashboard load time:** 30s → **2s** (93% faster)
- ✅ **Prometheus memory usage:** 32GB → **12GB** (62% reduction)
- ✅ **Active time series:** 15M → **3M** (80% reduction)
- ✅ **Query success rate:** 92% → **99.8%**

**Operational Impact:**
- ✅ **MTTD (Mean Time to Detect):** 20 minutes → **2 minutes** (90% faster)
- ✅ **MTTR (Mean Time to Resolve):** Reduced by 40% due to better correlation
- ✅ **Data retention:** 15 days → **90 days** (with 2-year historical archive)
- ✅ **Cost savings:** $4,800/month in EC2 costs (moved to S3 storage at $200/month)

**Business Value:**
- ✅ **Prevented 15+ production incidents** through proactive alerting in first quarter
- ✅ **Enabled compliance** with SOC2 requirements (90-day metric retention)
- ✅ **Improved on-call experience** - Engineers spent 60% less time investigating issues
- ✅ **Facilitated capacity planning** - Leadership could analyze year-over-year trends

**Team & Process Improvements:**
- ✅ Unified monitoring reduced tool sprawl from 5 systems to 1
- ✅ Standardized SLO/SLI definitions across all services
- ✅ Created self-service dashboards - Teams could create custom views without SRE intervention
- ✅ Established monitoring-as-code practices (100% of configs in Git)

---

## 🎤 Key Talking Points for Interview

### **Technical Depth:**
*"I implemented a hierarchical Prometheus federation with Thanos for long-term storage, reducing our monitoring infrastructure costs by 85% while extending retention from 15 days to 90 days with 2-year archives in S3. The key challenge was handling cardinality explosion—I reduced active time series from 15M to 3M through label optimization and recording rules."*

### **Problem-Solving:**
*"When dashboards were taking 30+ seconds to load, I used Prometheus profiling tools to identify inefficient queries. By implementing 25 recording rules for pre-computed aggregations, I reduced load time by 93%. This was critical for our on-call engineers who needed real-time visibility during incidents."*

### **Business Impact:**
*"The platform improvement directly impacted incident response—our MTTD dropped from 20 minutes to 2 minutes, and we prevented 15+ production incidents in the first quarter through proactive alerting. The CFO was particularly pleased with the $55K annual cost savings from moving to object storage."*

### **Leadership & Collaboration:**
*"I worked with 8 development teams to standardize metric labeling conventions and created documentation and training materials. I also implemented GitOps workflows so teams could self-service their dashboards and alerts through pull requests, reducing SRE bottlenecks."*

---

## 💡 Lessons Learned

**What Worked Well:**
- Phased approach prevented big-bang migration risks
- Recording rules dramatically improved user experience
- Thanos solved both retention and cost challenges elegantly

**Challenges Overcome:**
- Initial resistance from teams comfortable with existing tools - solved through training and proving value with pilot metrics
- Clock skew between instances causing Thanos issues - resolved with NTP synchronization and time flags
- Query timeouts during migration - implemented gradual rollout and query result caching

**What I'd Do Differently:**
- Start with cardinality controls earlier - would have saved 2 weeks of optimization work
- Implement automated capacity planning from day one
- Create runbooks for common failure scenarios before going to production

---

## 🔧 Technical Skills Demonstrated

✅ **Monitoring & Observability:** Prometheus, Grafana, Alertmanager, Thanos, PromQL  
✅ **Cloud Infrastructure:** AWS (EC2, S3, VPC), Docker, Docker Compose  
✅ **Performance Optimization:** Query profiling, cardinality management, recording rules  
✅ **Architecture Design:** Federation patterns, distributed systems, high availability  
✅ **Security:** TLS/SSL, OAuth2, network policies, RBAC  
✅ **DevOps Practices:** GitOps, CI/CD, Infrastructure-as-Code, monitoring-as-code  
✅ **Data Engineering:** Time-series databases, retention policies, downsampling  
✅ **Cost Optimization:** Resource rightsizing, cloud cost management

---

## 📝 Follow-Up Questions You Might Get

**Q: "How did you handle the migration without downtime?"**
*A: "I implemented a blue-green approach where both old and new systems ran in parallel for 2 weeks. We gradually shifted teams over and maintained rollback capability. Critical alerts were duplicated in both systems until validation was complete."*

**Q: "What was your biggest technical challenge?"**
*A: "Cardinality explosion causing OOM crashes. The root cause was developers adding user_id and session_id as labels. I solved it by educating teams on label best practices and implementing automated validation in our CI pipeline that rejected metrics with high-cardinality labels."*

**Q: "How do you ensure this scales as the company grows?"**
*A: "The Thanos architecture is horizontally scalable—we can add more Query and Store Gateway instances. I also implemented automated capacity alerts that trigger when we hit 70% of resource limits, giving us time to scale proactively."*

---

**Duration:** 7 weeks from design to production  
**Team Size:** Solo implementation, collaborated with 8 dev teams  
**Technologies:** Prometheus, Thanos, Grafana, Alertmanager, Docker, AWS (EC2, S3), NGINX, Git