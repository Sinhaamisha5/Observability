# 🎯 Goal 5: Incident Management & Troubleshooting

**Objective:** Master the art of detecting, diagnosing, and resolving production incidents using observability best practices.

---

## 📑 Table of Contents
1. [Understanding the Purpose](#understanding-the-purpose)
2. [Key Concepts](#key-concepts)
3. [Incident Management Process](#incident-management-process)
4. [Troubleshooting Framework](#troubleshooting-framework)
5. [Common Incident Scenarios](#common-incident-scenarios)
6. [Tools and Techniques](#tools-and-techniques)
7. [Post-Incident Analysis](#post-incident-analysis)
8. [STAR Case Study](#star-case-study)

---

## 🎯 Understanding the Purpose

### What Are We Doing Here?

When things break in production — **and they always do** — the SRE's job is to:

1. **Detect** the issue quickly (MTTD - Mean Time to Detect)
2. **Diagnose** the root cause accurately
3. **Restore** service fast (MTTR - Mean Time to Resolve)
4. **Prevent** it from happening again

### Why This Matters

**The Reality:**
- Production incidents cost businesses $300K/hour on average
- 70% of incidents are discovered by customers, not monitoring
- Average MTTR is 3-4 hours across the industry
- Engineers spend 40% of their time firefighting

**The Goal:**
Transform from reactive firefighting to proactive incident management through:
- **Observability** (not just monitoring)
- **Automation** (intelligent alerting)
- **Process** (structured incident response)
- **Learning** (blameless postmortems)

### Observability vs Monitoring

**Monitoring** tells you "something is wrong"
```
✅ Server CPU at 95%
✅ API latency increased
✅ Error rate spiked
```

**Observability** tells you "why it's wrong and where"
```
✅ Which service is causing high CPU
✅ Which endpoint has high latency
✅ Which dependency is failing
✅ Which code path is slow
```

### The Three Pillars of Observability

```
┌─────────────────────────────────────────┐
│         OBSERVABILITY                   │
├─────────────┬───────────────┬───────────┤
│   METRICS   │     LOGS      │  TRACES   │
├─────────────┼───────────────┼───────────┤
│ What        │ Why           │ Where     │
│ happened?   │ did it happen?│ did it    │
│             │               │ happen?   │
└─────────────┴───────────────┴───────────┘
```

**Metrics:** Numerical measurements over time (CPU, latency, errors)  
**Logs:** Detailed event records (stack traces, error messages)  
**Traces:** Request journey through microservices (distributed tracing)

---

## 🧠 Key Concepts

### 1️⃣ Incident Management Process

An **incident** is any unplanned interruption to a service.

#### Incident Lifecycle

| Stage | Description | Key Actions |
|-------|-------------|-------------|
| **Detection** | Alert triggers or user report | Monitor dashboards, check alerts |
| **Triage** | Assess severity and impact | Determine affected users/services |
| **Diagnosis** | Investigate root cause | Check metrics, logs, traces |
| **Resolution** | Mitigate or fix the issue | Restart, rollback, patch |
| **Recovery** | Verify service is restored | Monitor stability, check SLOs |
| **Postmortem** | Document learnings | Write RCA, action items |

#### Key Performance Metrics

**MTTD (Mean Time to Detect)**
```
How quickly did we detect the issue?

Industry Average: 10-20 minutes
Target: < 5 minutes
```

**MTTR (Mean Time to Resolve)**
```
How long did it take to fix?

Industry Average: 3-4 hours
Target: < 30 minutes
```

**MTBF (Mean Time Between Failures)**
```
How reliable is our service?

Target: Maximize uptime
```

#### Severity Classification

| Severity | Impact | Response Time | Example |
|----------|--------|---------------|---------|
| **P0 - Critical** | Complete service outage | Immediate | Payment system down |
| **P1 - High** | Major feature degraded | 15 minutes | 50% error rate |
| **P2 - Medium** | Minor feature affected | 1 hour | Dashboard loading slow |
| **P3 - Low** | Cosmetic/minor issue | 4 hours | UI alignment issue |

---

### 2️⃣ Metrics in Troubleshooting

#### What They Are
Numerical data collected over time:
- System metrics: CPU%, memory, disk I/O
- Application metrics: Request rate, latency, errors
- Business metrics: Orders/sec, revenue, conversions

#### Why They Matter
Metrics provide the **"what"** - they tell you something is wrong, but not necessarily **why**.

#### Common Metric Patterns During Incidents

| Symptom | Possible Cause | Investigation Path |
|---------|---------------|-------------------|
| CPU spike | Infinite loop, high load, CPU-intensive task | Check process CPU%, thread dumps |
| Memory growth | Memory leak, cache overflow | Monitor heap usage, GC patterns |
| Disk usage increase | Log bloat, time-series explosion | Check disk I/O, file sizes |
| Request rate drop | Network issue, app crash, traffic shift | Check upstream/downstream |
| Error rate increase | Dependency failure, bug deployment | Check error logs, recent changes |
| Latency increase | Slow query, network latency, resource contention | Check p95/p99 latency, traces |

#### Prometheus Troubleshooting Endpoints

**Check Target Health:**
```bash
# View scrape targets and their status
http://localhost:9090/targets
```

**Check Prometheus Status:**
```bash
# View TSDB stats, memory usage, series count
http://localhost:9090/status
```

**Check Internal Metrics:**
```bash
# Prometheus monitoring itself
http://localhost:9090/metrics
```

**Key Internal Metrics:**
```promql
# Number of active time series
prometheus_tsdb_head_series

# Scrape duration (detect slow exporters)
prometheus_target_scrape_duration_seconds

# Query duration (detect slow queries)
prometheus_engine_query_duration_seconds
```

---

### 3️⃣ Logs in Troubleshooting

#### What They Are
Detailed text records of events:
- Application logs (info, warning, error)
- System logs (syslog, journald)
- Access logs (nginx, apache)
- Error logs (stack traces, exceptions)

#### Why They Matter
**Metrics** tell you "something's wrong"  
**Logs** tell you "what exactly went wrong"

#### Log Levels and Their Use

| Level | Purpose | When to Use |
|-------|---------|-------------|
| **DEBUG** | Detailed diagnostic info | Development, deep troubleshooting |
| **INFO** | General informational messages | Normal operations, flow tracking |
| **WARN** | Potentially harmful situations | Degraded performance, retries |
| **ERROR** | Error events that might allow continuation | Handled exceptions, fallbacks |
| **FATAL** | Severe errors causing application abort | Unrecoverable failures |

#### Loki + Promtail Architecture

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Application │───▶│   Promtail   │───▶│     Loki     │
│     Logs     │    │   (Collect)  │    │   (Storage)  │
└──────────────┘    └──────────────┘    └──────────────┘
                                               │
                                               ▼
                                        ┌──────────────┐
                                        │   Grafana    │
                                        │    (Query)   │
                                        └──────────────┘
```

**Promtail:** Collects logs from files, containers, systemd  
**Loki:** Indexes logs by labels (job, instance, severity)  
**Grafana:** Queries logs alongside metrics for correlation

#### Example Use Case

**Scenario:** You see a latency spike on Grafana

**Step 1:** Identify the time window (e.g., 14:30-14:35)

**Step 2:** Switch to Loki panel in Grafana

**Step 3:** Query logs from that timeframe:
```logql
{job="payment-api"} |= "error" | json | latency > 1000
```

**Step 4:** Find the root cause:
```
Error: DBConnectionTimeout
Service: Payment API
Query: SELECT * FROM orders WHERE user_id = ?
Duration: 5000ms
```

**Result:** Immediately know the root cause (slow database query)

---

### 4️⃣ Traces for Root Cause Analysis

#### What They Are
Traces show a request's journey through multiple microservices, broken down into **spans**.

**Span:** A single operation in the trace (e.g., "call auth service")  
**Trace:** Complete journey from start to finish

#### Why They Matter
They pinpoint **where** in the service chain latency or failure occurred.

#### Distributed Tracing Example

```
User Request → Frontend → Auth → Orders → Payment → Database
   (100ms)      (10ms)    (5ms)   (15ms)   (200ms)   (150ms)
   
Total Latency: 480ms
Bottleneck: Payment service (200ms) → Database query (150ms)
```

#### Tempo Architecture

```
┌──────────────┐
│ Application  │
│ (OpenTelemetry)
└──────┬───────┘
       │ traces
       ▼
┌──────────────┐
│    Tempo     │
│  (Storage)   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Grafana    │
│ (Visualization)
└──────────────┘
```

#### Example Troubleshooting Flow

**Problem:** API endpoint is slow (2 seconds response time)

**Step 1:** Open Grafana trace view

**Step 2:** Find a slow trace (2000ms total)

**Step 3:** Analyze span breakdown:
```
frontend:      50ms  ✅
auth:          30ms  ✅
orders:        45ms  ✅
payment:       1800ms ⚠️  ← BOTTLENECK
  ├─ validate: 20ms
  ├─ process:  30ms
  └─ database: 1750ms ⚠️  ← ROOT CAUSE
```

**Step 4:** Check logs for that database query

**Step 5:** Find slow query, optimize index

**Result:** Latency reduced from 2000ms to 150ms

---

### 5️⃣ Correlation Across Metrics, Logs, and Traces

#### What It Is
The ability to jump between **metrics**, **logs**, and **traces** in one unified interface (Grafana).

#### Why It Matters
**Faster root cause detection** - no more jumping between tools or copy-pasting timestamps.

#### Correlation Flow Example

```
┌─────────────────────────────────────────────────────┐
│ 1. Metrics Dashboard (Grafana)                      │
│    • Latency spike at 14:30                         │
│    • Click on anomaly point                         │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│ 2. Exemplar Link (Trace ID)                         │
│    • Opens distributed trace                        │
│    • Shows payment service delay                    │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│ 3. Logs from Same Timeframe                         │
│    • Click "View Logs"                              │
│    • Shows: "DBConnectionTimeout"                   │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│ 4. Root Cause Identified                            │
│    • Database connection pool exhausted             │
│    • Action: Increase pool size from 10 to 50      │
└─────────────────────────────────────────────────────┘
```

#### Implementation in Grafana

**Step 1: Enable Exemplars in Prometheus**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  # Enable exemplars
  exemplar_storage:
    max_exemplars: 100000
```

**Step 2: Configure Data Source Links in Grafana**
```json
{
  "datasources": [
    {
      "name": "Prometheus",
      "type": "prometheus",
      "url": "http://prometheus:9090"
    },
    {
      "name": "Loki",
      "type": "loki",
      "url": "http://loki:3100",
      "derivedFields": [
        {
          "matcherRegex": "traceID=(\\w+)",
          "name": "TraceID",
          "url": "http://tempo:3200/trace/${__value.raw}"
        }
      ]
    },
    {
      "name": "Tempo",
      "type": "tempo",
      "url": "http://tempo:3200"
    }
  ]
}
```

**Step 3: Use in Dashboard**
- Metrics panel shows latency graph
- Click on data point → opens trace
- Trace shows slow span → click to view logs
- Logs show exact error message

---

### 6️⃣ Alerting and Deduplication

#### The Alert Fatigue Problem

**Scenario:** Database server goes down

**Without Deduplication:**
```
🔔 Alert 1: Database unreachable
🔔 Alert 2: API connection error
🔔 Alert 3: Payment service failing
🔔 Alert 4: Orders service failing
🔔 Alert 5: User service timeout
... (50 more alerts)
```

**Result:** 55 notifications for ONE root cause → **Alert fatigue** → Engineers ignore alerts

#### Solution: Alertmanager Grouping

**Group related alerts** into single notification:

```yaml
route:
  group_by: ['alertname', 'service', 'cluster']
  group_wait: 30s        # Wait 30s to collect similar alerts
  group_interval: 5m     # Send grouped summary every 5m
  repeat_interval: 3h    # Re-send if not resolved
```

**Result:**
```
🔔 Grouped Alert: "Database issues affecting 5 services"
   - DatabaseDown (critical)
   - APIConnectionError (warning) - 3 instances
   - ServiceTimeout (warning) - 2 instances
```

#### Solution: Alert Inhibition

**Suppress dependent alerts** when root cause is active:

```yaml
inhibit_rules:
  # If database is down, suppress connection errors
  - source_match:
      alertname: 'DatabaseDown'
      severity: 'critical'
    target_match:
      alertname: 'DatabaseConnectionError'
      severity: 'warning'
    equal: ['cluster']
```

**Logic:** "Don't alert about symptoms when root cause is known"

#### Intelligent Alert Routing

```yaml
route:
  receiver: 'default'
  routes:
    # Critical → PagerDuty + Slack
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    
    # Business hours only
    - match:
        severity: warning
      receiver: 'slack-warnings'
      active_time_intervals:
        - business-hours
    
    # Database team
    - match:
        team: database
      receiver: 'database-oncall'
```

#### Example Configuration

```yaml
# alertmanager.yml
route:
  receiver: 'slack'
  group_by: ['alertname', 'service']
  group_wait: 30s
  repeat_interval: 3h

receivers:
  - name: 'slack'
    slack_configs:
      - channel: '#sre-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

---

## 🔍 Troubleshooting Framework

### 7️⃣ Investigating Common Incident Scenarios

#### Scenario 1: Metrics Stopped Updating

**Symptoms:**
- Grafana shows "No data"
- Graphs are flat or missing

**Troubleshooting Path:**

**Step 1:** Check Prometheus targets
```bash
# Access Prometheus UI
http://localhost:9090/targets

# Look for DOWN targets
Target: node-exporter (DOWN)
Last Scrape: 5m ago
Error: connection refused
```

**Step 2:** Check if exporter is running
```bash
# SSH to target server
systemctl status node_exporter
# or
docker ps | grep node-exporter
```

**Step 3:** Check network connectivity
```bash
# From Prometheus server
curl http://target-server:9100/metrics
```

**Step 4:** Check exporter logs
```bash
journalctl -u node_exporter -f
# or
docker logs node-exporter
```

**Common Causes:**
- Exporter crashed
- Firewall blocking port
- DNS resolution failure
- Target misconfigured in prometheus.yml

---

#### Scenario 2: Prometheus Scrape Errors (HTTP 500)

**Symptoms:**
- Target shows UP but errors in scraping
- Partial data in Grafana

**Troubleshooting Path:**

**Step 1:** Check exporter health directly
```bash
curl http://exporter:9100/metrics
# If returns 500, exporter has issues
```

**Step 2:** Check exporter resource usage
```promql
# High CPU on exporter host
rate(process_cpu_seconds_total{job="exporter"}[5m])

# High memory usage
process_resident_memory_bytes{job="exporter"}
```

**Step 3:** Review exporter logs
```bash
# Look for errors, panics, crashes
docker logs exporter | grep -i error
```

**Common Causes:**
- Exporter bug/crash
- Too many metrics (cardinality)
- Resource exhaustion
- Configuration error

---

#### Scenario 3: Grafana Shows "No Data"

**Symptoms:**
- Dashboard panels empty
- "No data" message

**Troubleshooting Path:**

**Step 1:** Verify data in Prometheus
```bash
# Open Prometheus UI
http://localhost:9090

# Run the same query from Grafana
sum(rate(http_requests_total[5m]))
```

**Step 2:** Check time range
- Is selected time range in the past?
- Is retention period exceeded?

**Step 3:** Check query syntax
```promql
# Wrong (no data)
http_requests_total{service="$service"}

# Right (has data)
sum(rate(http_requests_total{service="$service"}[5m]))
```

**Step 4:** Verify data source connection
```
Grafana → Configuration → Data Sources → Test
```

**Common Causes:**
- Query syntax error
- Wrong time range
- Data source misconfigured
- Metric name typo
- No data for selected labels

---

#### Scenario 4: Disk Usage Spike

**Symptoms:**
- Prometheus disk filling up fast
- System alerts for low disk space

**Troubleshooting Path:**

**Step 1:** Check TSDB size
```bash
# Prometheus data directory
du -sh /prometheus/data
```

**Step 2:** Check cardinality
```promql
# Top metrics by series count
topk(10, count by (__name__)({__name__!=""}))
```

**Step 3:** Identify high-cardinality labels
```promql
# Check unique values per label
count by (user_id) ({__name__="http_requests_total"})
```

**Step 4:** Review recent changes
- New exporters added?
- New labels introduced?
- Scrape interval changed?

**Common Causes:**
- High-cardinality labels (user_id, session_id)
- Too many time series
- Short retention with high scrape frequency
- Memory leak in Prometheus

**Solutions:**
```yaml
# Reduce retention
--storage.tsdb.retention.time=7d

# Limit series
--storage.tsdb.retention.size=50GB

# Drop problematic metrics
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'expensive_metric_.*'
    action: drop
```

---

#### Scenario 5: Duplicate Alerts

**Symptoms:**
- Same alert fires multiple times
- Alert spam in Slack/email

**Troubleshooting Path:**

**Step 1:** Check Alertmanager config
```yaml
# Look for multiple matching routes
route:
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
    - match:  # ⚠️ Both rules match same alert
        alertname: 'HighCPU'
      receiver: 'slack'
```

**Step 2:** Review grouping configuration
```yaml
# Missing or incorrect group_by
route:
  group_by: ['alertname']  # Add more labels
  # Should be: ['alertname', 'instance', 'job']
```

**Step 3:** Check inhibition rules
```yaml
# Are inhibition rules working?
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['instance']
```

**Common Causes:**
- Multiple Alertmanager instances
- Overlapping routing rules
- Missing group_by configuration
- Inhibition rules not working

---

#### Scenario 6: Memory Leak Investigation

**Symptoms:**
- Memory usage grows continuously
- Eventually OOM (Out of Memory)

**Troubleshooting Path:**

**Step 1:** Identify the leaking process
```promql
# Memory usage over time
process_resident_memory_bytes{job="myapp"}
```

**Step 2:** Check heap profile
```bash
# Take heap snapshot
curl http://app:6060/debug/pprof/heap > heap.out

# Analyze with pprof
go tool pprof heap.out
```

**Step 3:** Check for common patterns
```promql
# Growing time series count
prometheus_tsdb_head_series

# Growing cache size
cache_size_bytes{job="myapp"}

# Unclosed connections
process_open_fds{job="myapp"}
```

**Step 4:** Review recent deployments
- Check deployment annotations in Grafana
- Correlate memory growth with release

**Common Causes:**
- Unbounded cache growth
- Connection leaks
- Goroutine leaks (Go apps)
- High cardinality metrics
- Missing GC configuration

---

## 📝 Post-Incident Analysis

### 8️⃣ Root Cause Analysis (Postmortem)

#### Why Postmortems Matter

**Benefits:**
- ✅ Learn from failures
- ✅ Prevent recurrence
- ✅ Improve systems and processes
- ✅ Build team knowledge
- ✅ Foster blameless culture

**Without Postmortems:**
- ❌ Same issues repeat
- ❌ Knowledge stays with individuals
- ❌ No systemic improvements
- ❌ Blame culture develops

#### Blameless Culture Principles

1. **Focus on systems, not people**
   - Wrong: "John deployed bad code"
   - Right: "Lack of automated testing allowed bug to reach production"

2. **Assume good intentions**
   - People made the best decisions with information available

3. **Create psychological safety**
   - Encourage honest discussion without fear

4. **Learn and improve**
   - Every incident is a learning opportunity

#### Postmortem Template

```markdown
# Incident Postmortem: [Title]

## Metadata
- **Date:** 2025-10-13
- **Duration:** 45 minutes (14:15 - 15:00 UTC)
- **Severity:** P1 - High
- **Detected By:** Automated alert (HighLatency)
- **Incident Commander:** Sarah Chen
- **Author:** DevOps Team

## Executive Summary
Brief description of what happened and impact.

Example:
"Payment API experienced 95% error rate for 45 minutes during peak hours, 
affecting approximately 5,000 customers. Root cause was Redis cache 
saturation due to missing eviction policy."

## Impact
- **Users Affected:** 5,000 customers
- **Revenue Impact:** $25,000 in lost transactions
- **Customer Support Tickets:** 127
- **SLO Status:** Consumed 15% of monthly error budget

## Timeline (All times UTC)

| Time | Event |
|------|-------|
| 14:10 | Redis memory reaches 95% capacity |
| 14:15 | HighLatency alert fires in Alertmanager |
| 14:17 | On-call engineer acknowledges alert |
| 14:20 | Investigation begins - checking metrics |
| 14:25 | Identified Redis as bottleneck via traces |
| 14:30 | Checked Redis logs - "OOM command not allowed" |
| 14:35 | Increased Redis memory limit temporarily |
| 14:40 | Configured maxmemory-policy=allkeys-lru |
| 14:45 | Service recovery begins |
| 15:00 | Full service restored, SLOs met |

## Root Cause
Redis cache hit 100% memory capacity without eviction policy configured.
When cache was full, all SET commands failed, causing application errors.

### Contributing Factors
1. Missing eviction policy in Redis configuration
2. No monitoring for Redis memory usage
3. No alerts for cache saturation
4. Insufficient load testing didn't reveal this issue

## Detection
- **How:** Automated Prometheus alert (HighLatency > 500ms)
- **MTTD:** 5 minutes (good)
- **Could be improved:** Add Redis-specific alerts

## Resolution
1. Temporarily increased Redis memory limit (maxmemory 4gb → 8gb)
2. Configured eviction policy: `maxmemory-policy allkeys-lru`
3. Restarted affected application pods
4. Monitored recovery until SLOs restored

## Action Items

| Action | Owner | Deadline | Status |
|--------|-------|----------|--------|
| Add Redis memory metrics to dashboard | DevOps | 2025-10-15 | ✅ Done |
| Create alert for Redis memory > 80% | SRE | 2025-10-15 | ✅ Done |
| Review all Redis instances for eviction policy | DevOps | 2025-10-20 | 🔄 In Progress |
| Add Redis load testing to staging | QA | 2025-10-25 | 📝 Planned |
| Document Redis configuration standards | SRE | 2025-10-30 | 📝 Planned |
| Conduct runbook drill for cache issues | Team | 2025-11-01 | 📝 Planned |

## Lessons Learned

### What Went Well
✅ Alert fired quickly (5 min MTTD)
✅ Team responded promptly
✅ Traces helped identify bottleneck fast
✅ Clear communication in incident channel

### What Didn't Go Well
❌ No Redis-specific monitoring
❌ Missing eviction policy (config gap)
❌ No load testing for cache scenarios
❌ Runbook didn't cover cache saturation

### Preventive Measures
1. **Monitoring:** Add comprehensive Redis metrics
2. **Alerting:** Create proactive cache alerts
3. **Testing:** Include cache stress tests
4. **Documentation:** Update runbooks
5. **Configuration:** Standardize Redis settings

## Appendix

### Relevant Dashboards
- [Payment API Dashboard](http://grafana/payment-api)
- [Redis Dashboard](http://grafana/redis)

### Related Alerts
- `HighLatency` (triggered)
- `HighErrorRate` (triggered)

### Chat Logs
[Link to Slack incident channel]

### Metrics Snapshots
- Error rate peak: 95% at 14:20
- Latency peak: 3500ms at 14:22
- Redis memory: 100% at 14:15
```

---

## 🔧 Tools Summary

| Tool | Purpose | Key Features | Example Usage |
|------|---------|--------------|---------------|
| **Prometheus** | Metrics collection & alerting | Time-series database, PromQL queries | `http://localhost:9090/targets` |
| **Grafana** | Visualization & correlation | Dashboards, multi-source queries | Link metrics → logs → traces |
| **Loki** | Log aggregation | Label-based indexing, LogQL | Query logs by service, severity |
| **Tempo** | Distributed tracing | Trace storage, span analysis | Identify slow microservices |
| **Alertmanager** | Alert routing & deduplication | Grouping, inhibition, routing | Reduce alert noise by 70% |
| **Promtail** | Log shipping | Collect logs from multiple sources | Forward logs to Loki |

### Prometheus Troubleshooting Commands

```bash
# Check target health
curl http://localhost:9090/api/v1/targets

# Check internal metrics
curl http://localhost:9090/metrics | grep prometheus_tsdb

# Reload configuration
curl -X POST http://localhost:9090/-/reload

# Check TSDB status
curl http://localhost:9090/api/v1/status/tsdb

# Query metrics
curl 'http://localhost:9090/api/v1/query?query=up'
```

### Loki Query Examples

```logql
# Errors in last hour
{job="api"} |= "error" | json

# Slow requests
{job="api"} | json | duration > 1000

# Specific time range
{job="api"}[5m] | json | severity="critical"

# Count by severity
sum(count_over_time({job="api"}[1h])) by (severity)
```

---

## 🧱 Real-World Example Summary (Interview STAR)

# Situation:
During peak traffic hours, multiple alerts triggered — “API latency high,” “Database slow,” and “Cache hit ratio low.” The Grafana dashboards were showing gaps and “No data” for several exporters.

# Task:
Diagnose the root cause and reduce Mean Time to Resolve (MTTR).

# Action:

Checked Prometheus /targets → found Redis exporter down.

Queried logs in Loki → connection timeout errors.

Used Tempo traces → confirmed high latency between API and Redis.

Restarted Redis exporter and tuned connection pool.

Added new alert rule for Redis exporter availability.

Conducted a post-incident review and added dashboard annotations.

# Result:

Restored service in 12 minutes (down from 1+ hour).

Reduced false alerts by 40%.

Improved observability correlation between metrics, logs, and traces.

# 📦 Tools Summary
Tool	Purpose	Example Command
Prometheus	Metrics & alert rules	http://localhost:9090/targets
Grafana	Visualization & correlation	Dashboards with logs + traces
Loki	Centralized log storage	http://localhost:3100
Tempo	Trace storage	View distributed traces
Alertmanager	Alert routing & deduplication	http://localhost:9093
Promtail	Log collector	Send logs from host to Loki