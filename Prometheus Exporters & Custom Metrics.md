# 📊 Metrics & Exporters Hands-On Roadmap

**Goal 2 — Master Prometheus Exporters & Custom Metrics**

---

## 🔹 Phase 1 — Linux System Monitoring (Node Exporter) ✅

**Goal:** Get visibility into EC2 system metrics.

### What you did:
- ✅ Installed Node Exporter
- ✅ Configured Prometheus to scrape it
- ✅ Visualized CPU, memory, and disk usage in Grafana
- ✅ Added alerts for high CPU

**Status:** ✅ You've already completed this phase!

---

## 🔹 Phase 2 — Container Metrics (cAdvisor)

**Goal:** Monitor container-level CPU, memory, and network metrics.

### Steps:

**1. Install and run cAdvisor:**
```bash
docker run -d \
  --name=cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  gcr.io/cadvisor/cadvisor:latest
```

**2. Add a Prometheus job:**
```yaml
- job_name: 'cadvisor'
  static_configs:
    - targets: ['localhost:8080']
```

**3. Check in Prometheus → Targets**

**4. Visualize container metrics in Grafana (Docker Host Overview dashboard)**

### Expected learning:
- Difference between host and container metrics
- Resource isolation per container

---

## 🔹 Phase 3 — Endpoint Monitoring (Blackbox Exporter)

**Goal:** Monitor uptime and latency of APIs or websites.

### Steps:

**1. Run Blackbox Exporter:**
```bash
docker run -d --name=blackbox \
  -p 9115:9115 prom/blackbox-exporter
```

**2. Add to prometheus.yml:**
```yaml
- job_name: 'blackbox'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://prometheus.io
      - https://grafana.com
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: localhost:9115
```

**3. Check `probe_success` metric for status.**

**4. Create Grafana dashboard + alerts for "endpoint down".**

### Expected learning:
- External uptime monitoring
- Alerting on HTTP failures

---

## 🔹 Phase 4 — Database and Middleware Exporters

**Goal:** Monitor Redis, MySQL, and Kafka performance.

### Steps:

**1. Run Redis and its exporter:**
```bash
docker run -d --name redis -p 6379:6379 redis
docker run -d --name redis-exporter -p 9121:9121 oliver006/redis_exporter
```

**2. Add to prometheus.yml:**
```yaml
- job_name: 'redis'
  static_configs:
    - targets: ['localhost:9121']
```

**3. Repeat similar steps for MySQL (`mysqld_exporter`) and Kafka (`kafka_exporter`).**

### Expected learning:
- DB/Cache health metrics
- Latency, connections, memory usage, replication lag

---

## 🔹 Phase 5 — Custom Application Metrics

**Goal:** Expose and monitor metrics from your own app.

### Steps:

**1. Write a small Python web app with Prometheus client:**
```bash
pip install prometheus_client flask
```

```python
from flask import Flask
from prometheus_client import Counter, generate_latest

app = Flask(__name__)
REQUEST_COUNT = Counter('app_requests_total', 'Total requests count')

@app.route('/')
def hello():
    REQUEST_COUNT.inc()
    return "Hello, Metrics!"

@app.route('/metrics')
def metrics():
    return generate_latest()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**2. Run it and add to Prometheus:**
```yaml
- job_name: 'custom_app'
  static_configs:
    - targets: ['localhost:5000']
```

**3. View your own metric `app_requests_total`.**

### Expected learning:
- How apps expose `/metrics` endpoint
- Custom metrics instrumentation

---

## 🔹 Phase 6 — Metric Optimization & Label Cardinality

**Goal:** Prevent Prometheus performance issues.

### Learn and practice:

**1. Avoid high-cardinality labels:**
- ❌ Bad: `user_id`, `session_id`
- ✅ Good: `region`, `instance`, `status`

**2. Use recording rules for frequent queries:**
```yaml
recording_rules.yml:
  groups:
    - name: cpu_rules
      rules:
        - record: instance:cpu_usage:avg
          expr: avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)
```

### Expected learning:
- Query optimization
- Efficient metric design for large-scale systems

---

## 🔹 Phase 7 — Alerting Simulation

**Goal:** Practice alert rules for all metrics types.

### Examples:

| Alert Type | Source | Trigger |
|------------|--------|---------|
| High CPU | Node Exporter | CPU > 80% |
| Container OOM | cAdvisor | Memory limit reached |
| API down | Blackbox Exporter | `probe_success == 0` |
| Redis connection failures | Redis Exporter | Connection errors |
| Custom metric threshold | Your app | Custom business logic |

**Simulate and validate alerts in Alertmanager + Slack (or console).**

---

## 🔹 Phase 8 — Dashboard Organization

**Goal:** Build service-specific Grafana dashboards.

### Dashboards to create:

| Service Type | Key Metrics |
|--------------|-------------|
| API (Custom App) | Request count, latency, errors |
| Database (MySQL/Redis) | Connections, memory usage, ops/sec |
| Cache | Hit ratio, latency |
| Node (Linux) | CPU, memory, disk, network |
| Container (cAdvisor) | Per-container resource metrics |
| External (Blackbox) | Availability, response time |

---

## ✅ Summary of What You'll Learn After Goal 2

| Layer | Tool | Learning Outcome |
|-------|------|------------------|
| OS | Node Exporter | Host metrics |
| Containers | cAdvisor | Container visibility |
| Network/HTTP | Blackbox | Uptime and latency monitoring |
| DB/Cache | Redis/MySQL Exporters | Middleware health |
| App | Custom metrics | Business logic observability |
| Optimization | Recording rules + label control | Efficient scaling |
| Visualization | Grafana | End-to-end dashboards |
| Alerts | Alertmanager | Proactive incident detection |

---

## 🎯 Final Outcome

By completing this roadmap, you will have:
- ✅ Mastered multiple Prometheus exporters
- ✅ Built custom application metrics
- ✅ Optimized metrics for production scale
- ✅ Created comprehensive monitoring dashboards
- ✅ Implemented multi-layer alerting strategies