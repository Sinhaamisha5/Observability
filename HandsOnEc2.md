# 🧭 Monitoring & Alerting Architecture for 200+ Microservices

This project provides a complete **Monitoring & Alerting architecture** built with:

- **Prometheus** → Metrics collection  
- **Alertmanager** → Alert routing  
- **Grafana** → Visualization  
- **Node Exporter / App Exporter** → Host & app metrics  
- **Nagios Integration** → Legacy monitoring  
- **Service Discovery** → EC2 or Kubernetes  
- **Security Hardening** → HTTPS + Auth  
- **GitOps Automation** → Dashboards & config sync  

---

## 🧩 Phase 1 — Base Setup: EC2 + Prometheus + Node Exporter

### 🎯 Objective
Install **Prometheus** and **Node Exporter** on EC2 to collect host-level metrics.

### 🔧 Steps

**1. Create folder structure**
```bash
mkdir -p ~/monitoring-stack/{prometheus,grafana,alertmanager,exporters}
cd ~/monitoring-stack
```

**2. Install Node Exporter**
```bash
cd exporters
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvfz node_exporter*.tar.gz
cd node_exporter-1.7.0.linux-amd64
./node_exporter &
```

**3. Install Prometheus**
```bash
cd ~/monitoring-stack/prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
tar xvf prometheus*.tar.gz
cd prometheus-2.53.0.linux-amd64
```

**4. Configure prometheus.yml**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

**5. Start Prometheus**
```bash
./prometheus --config.file=prometheus.yml &
```

**6. Access**
- Prometheus UI → `http://<EC2-IP>:9090`
- Node Exporter metrics → `http://<EC2-IP>:9100/metrics`

✅ **Prometheus is now collecting EC2 metrics.**

---

## 📊 Phase 2 — Grafana Visualization

### 🎯 Objective
Install Grafana and connect it to Prometheus.

### 🔧 Steps
```bash
cd ~/monitoring-stack/grafana
docker run -d -p 3000:3000 --name=grafana grafana/grafana
```

Then open Grafana in your browser:
- `http://<EC2-IP>:3000`
- Login → `admin` / `admin`
- Add Data Source → Prometheus → `http://<EC2-IP>:9090`
- Import Dashboard → ID: **1860** (Node Exporter Full)

✅ **Grafana dashboards now visualize EC2 metrics.**

---

## 🚨 Phase 3 — Alerting Setup (Prometheus + Alertmanager)

### 🎯 Objective
Create alert rules (e.g., CPU > 80%) and route via Alertmanager.

### 🔧 Steps

**1. Create alert rules (`prometheus/alerts.yml`):**
```yaml
groups:
  - name: system_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage > 80% for 2m"
```

**2. Configure Alertmanager:**
```yaml
route:
  receiver: 'email-alert'

receivers:
  - name: 'email-alert'
    email_configs:
      - to: 'your@email.com'
        from: 'alert@example.com'
```

**3. Link Alertmanager to Prometheus:**
```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'
rule_files:
  - "alerts.yml"
```

✅ **Alerts will now appear in Prometheus and route through Alertmanager.**

---

## 🧩 Phase 4 — Integrate Nagios Alerts into Prometheus

### 🎯 Objective
Expose Nagios alerts to Prometheus using an exporter or webhook.

### 🔧 Option 1: Use nagios_exporter
```bash
docker run -d -p 9115:9115 --name=nagios-exporter \
  -e NAGIOS_URL=http://nagios-server/nagiosxi \
  -e NAGIOS_USER=admin \
  -e NAGIOS_PASS=password \
  nagios_exporter
```

Then add a job to Prometheus:
```yaml
- job_name: 'nagios'
  static_configs:
    - targets: ['localhost:9115']
```

✅ **Prometheus now scrapes Nagios alerts.**

---

## 🔄 Phase 5 — Dynamic Service Discovery

### 🎯 Objective
Enable Prometheus to auto-discover new EC2 or Kubernetes services.

### 🏗️ Example: EC2 Service Discovery
```yaml
- job_name: 'ec2_instances'
  ec2_sd_configs:
    - region: us-east-1
      access_key: <ACCESS_KEY>
      secret_key: <SECRET_KEY>
  relabel_configs:
    - source_labels: [__meta_ec2_private_ip]
      target_label: instance
```

✅ **New EC2 instances are automatically added to monitoring.**

---

## 🔐 Phase 6 — Secure Observability Endpoints

### 🎯 Objective
Enable HTTPS, Basic Auth, and/or OAuth2 proxy for secure access.

### Example: NGINX Reverse Proxy
```nginx
server {
    listen 443 ssl;
    server_name monitoring.example.com;

    ssl_certificate /etc/ssl/certs/ssl-cert.pem;
    ssl_certificate_key /etc/ssl/private/ssl-key.pem;

    location / {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://localhost:9090;
    }
}
```

✅ **Prometheus and Grafana are now accessible securely via HTTPS.**

---

## ⚙️ Phase 7 — GitOps for Dashboard & Config Management

### 🎯 Objective
Automate Grafana & Prometheus configurations via GitHub and CI/CD.

**1. Store all YAMLs & JSON dashboards in Git.**

**2. Grafana provisioning (`provisioning/dashboards.yml`):**
```yaml
apiVersion: 1
providers:
  - name: 'Dashboards'
    options:
      path: /var/lib/grafana/dashboards
```

**3. Auto-deploy dashboards with GitHub Actions:**
```yaml
- name: Deploy Grafana Dashboards
  run: |
    scp dashboards/* grafana@<EC2-IP>:/var/lib/grafana/dashboards/
```

✅ **Dashboards and configs are deployed automatically (GitOps).**

---

## 🧠 Phase 8 — Interview Story (Example Narrative)

> "I designed and implemented a full observability stack on AWS EC2. Prometheus collected metrics from 200+ microservices using EC2 service discovery. Grafana was used for visualization, with dashboards provisioned via GitOps. I integrated legacy Nagios alerts through exporters, secured all endpoints using NGINX TLS + Basic Auth, and automated alert routing through Alertmanager."

---

## 🧱 Folder Structure

```
monitoring-stack/
├── prometheus/
│   ├── prometheus.yml
│   ├── alerts.yml
├── grafana/
│   ├── dashboards/
│   ├── provisioning/
├── alertmanager/
│   ├── alertmanager.yml
├── exporters/
│   ├── node_exporter/
│   ├── nagios_exporter/
```

---

## ✅ Final Outcome

A production-grade Monitoring & Alerting Stack with:

- ✅ Prometheus metrics from 200+ microservices
- ✅ Grafana dashboards (auto-provisioned via GitOps)
- ✅ Alertmanager notifications (email/webhook)
- ✅ Nagios integration
- ✅ EC2/Kubernetes service discovery
- ✅ Secured observability endpoints