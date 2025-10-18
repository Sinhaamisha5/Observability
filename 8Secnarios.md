# 🎤 Stage 8: STAR Format Interview Responses

## Scenario 1: Latency Spikes Every Few Hours

### **Situation**
Our e-commerce API was experiencing mysterious latency spikes every 3-4 hours. The p95 latency would jump from a normal 50ms to over 2000ms for about 5-10 minutes, then return to normal. This pattern was consistent but unpredictable in exact timing. Customers were abandoning their shopping carts during these spikes, and our SLO (99.9% of requests under 200ms) was being violated. The business team escalated this as a P1 issue because it was directly impacting revenue during peak shopping hours.

### **Task**
As the DevOps/SRE engineer on-call, I was responsible for:
- Identifying the root cause of these periodic latency spikes
- Implementing a solution to prevent future occurrences
- Reducing MTTR from the current 45 minutes (manual investigation each time) to under 5 minutes
- Ensuring no data loss or service degradation during the fix

### **Action**
I took a systematic, multi-layer troubleshooting approach using our observability stack:

**Step 1: Pattern Analysis (Metrics)**
- Opened Grafana and analyzed the p95 latency metric over 24 hours using: `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`
- Discovered spikes occurred precisely every 3 hours and 47 minutes
- Correlated this with resource metrics and found memory usage patterns matched the timing

**Step 2: Log Correlation**
- Used Loki to search for errors during spike windows: `{job="api"} |= "error" | json`
- Found hundreds of "connection pool exhausted" errors appearing exactly at spike times
- Noticed logs showing: "waiting for available database connection"

**Step 3: Distributed Tracing**
- Jumped into Tempo to analyze slow traces during the last spike
- Identified that 95% of slow spans were database queries waiting for connections
- Average wait time was 1800ms per query during spikes

**Step 4: Root Cause Identification**
- Investigated our cache layer and discovered all cache entries had identical 3h47m TTL
- This caused a "cache stampede" - all entries expired simultaneously every 3h47m
- When cache missed, thousands of concurrent requests hit the database
- Our connection pool (max 50 connections) was instantly exhausted
- Subsequent requests queued, causing cascading latency

**Step 5: Solution Implementation**
```yaml
# Added jitter to cache TTL configuration
cache:
  base_ttl: 14400  # 4 hours base
  jitter_range: 1800  # ±30 minutes random
  # Effective TTL: 3.5h to 4.5h (staggered expiration)
```

**Step 6: Additional Safeguards**
- Increased database connection pool from 50 to 100 with better timeout handling
- Implemented cache warming before expiration
- Added circuit breaker pattern for database calls
- Created specific alert for connection pool utilization: `db_connections_active / db_connections_max > 0.8`

**Step 7: Proactive Detection**
```promql
# New alert rule to detect pattern early
- alert: LatencyAnomalyDetected
  expr: |
    histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) 
    > 
    4 * histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1h] offset 1d))
  for: 2m
```

### **Result**
The solution delivered immediate and lasting improvements:

**Immediate Impact:**
- **Eliminated latency spikes completely** - zero occurrences in the 90 days following the fix
- **SLO compliance improved from 98.7% to 99.94%** (exceeding our 99.9% target)
- **Connection pool utilization dropped from 98% peak to 65% peak** during high traffic

**Performance Metrics:**
- P95 latency stabilized at 45-55ms (down from 2000ms spikes)
- Database query success rate: 100% (previously dropped to 75% during spikes)
- Cache hit ratio remained at 95% with staggered expiration

**Business Impact:**
- **Cart abandonment reduced by 23%** during former spike windows
- **Prevented estimated $50K monthly revenue loss** from timeout errors
- **Customer satisfaction scores improved by 12%** for API performance

**Operational Improvements:**
- MTTR for similar issues reduced from 45 minutes to under 5 minutes (we now detect before users are impacted)
- Documentation created for "cache stampede" pattern recognition
- Runbook updated with connection pool tuning guidelines

**Key Learnings:**
- The power of correlating metrics, logs, and traces - I couldn't have found this with metrics alone
- Importance of adding randomization/jitter to prevent synchronized system behavior
- Value of proactive monitoring (the new alert would catch this pattern within 2 minutes)

This incident became a case study in our SRE team meetings, and we applied the jitter pattern to other cache layers and cron jobs across the platform, preventing similar issues in 5 other services.

---

## Scenario 2: Alerts Fire but Logs Show No Error

### **Situation**
We started receiving critical alerts that our payment processing service was down, with Prometheus showing `up{job="payment-service"} == 0`. The on-call team immediately checked application logs but found absolutely no errors - the service appeared healthy, logs showed successful transactions, and our application monitoring dashboards showed normal behavior. However, Prometheus continued to report the service as unreachable, causing false pages at 3 AM for three consecutive nights. The team was frustrated with alert fatigue, and trust in our monitoring system was eroding.

### **Task**
I was assigned to:
- Investigate why alerts were firing despite healthy application logs
- Identify the actual failure point causing monitoring gaps
- Fix the root cause to restore monitoring reliability
- Prevent future false positive alerts that damage team confidence

### **Action**
I performed a systematic investigation focusing on the monitoring infrastructure itself:

**Step 1: Verify Alert Validity**
- Checked Prometheus targets page at `http://prometheus:9090/targets`
- Found payment-service showing: "Context deadline exceeded" error for scrapes
- Confirmed the alert was technically correct - Prometheus couldn't reach the service

**Step 2: Network Path Tracing**
```bash
# From Prometheus pod
kubectl exec -it prometheus-0 -n monitoring -- /bin/sh
wget -O- http://payment-service.prod.svc.cluster.local:8080/metrics --timeout=5
# Result: Connection timed out

# From another pod in same namespace
kubectl exec -it test-pod -n prod -- /bin/sh
curl http://payment-service:8080/metrics
# Result: Success! Metrics returned immediately
```
This revealed the issue was namespace-specific.

**Step 3: Investigation of Network Policies**
```bash
# Check network policies
kubectl get networkpolicies -n prod
kubectl describe networkpolicy allow-internal-traffic -n prod
```

Found this configuration:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal-traffic
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: payment-service
  ingress:
  - from:
    - podSelector: {}  # Only pods in 'prod' namespace
    ports:
    - protocol: TCP
      port: 8080
```

**The Root Cause:** NetworkPolicy only allowed ingress from pods within the `prod` namespace, but Prometheus was running in the `monitoring` namespace.

**Step 4: Verification of Other Affected Services**
- Checked which other services had similar NetworkPolicies
- Found 12 microservices with identical configuration
- All were failing Prometheus scrapes but appeared "healthy" in logs

**Step 5: Solution Implementation**

Created corrected NetworkPolicy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal-and-monitoring
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: payment-service
  ingress:
  - from:
    - podSelector: {}  # Internal prod traffic
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring  # Allow Prometheus scraping
    ports:
    - protocol: TCP
      port: 8080
```

**Step 6: Validation & Rollout**
```bash
# Applied fix
kubectl apply -f network-policy-fix.yaml

# Verified from Prometheus
kubectl exec -it prometheus-0 -n monitoring -- /bin/sh
wget -O- http://payment-service.prod.svc.cluster.local:8080/metrics
# Success!

# Rolled out to all 12 affected services
```

**Step 7: Prevention Measures**
1. **Created NetworkPolicy template** with monitoring exception documented
2. **Added pre-deployment validation** script:
```bash
#!/bin/bash
# validate-network-policy.sh
# Checks if NetworkPolicy allows Prometheus scraping

SERVICE_NAMESPACE=$1
SERVICE_NAME=$2

# Test scrape from monitoring namespace
kubectl run test-scrape \
  --image=curlimages/curl \
  --namespace=monitoring \
  --rm -it --restart=Never \
  -- curl -s -o /dev/null -w "%{http_code}" \
  http://${SERVICE_NAME}.${SERVICE_NAMESPACE}.svc.cluster.local:8080/metrics

if [ $? -eq 0 ]; then
  echo "✅ Prometheus can scrape ${SERVICE_NAME}"
else
  echo "❌ NetworkPolicy blocking Prometheus!"
  exit 1
fi
```

3. **Added CI/CD gate** - NetworkPolicy manifests must pass validation before deployment
4. **Created Prometheus meta-alert**:
```yaml
- alert: PrometheusScrapeFailures
  expr: up == 0
  for: 5m
  annotations:
    description: "Prometheus cannot scrape {{$labels.job}}. Check NetworkPolicy and service health."
    runbook: "https://wiki.company.com/runbooks/prometheus-scrape-failure"
```

### **Result**
The fix and preventive measures delivered comprehensive improvements:

**Immediate Resolution:**
- **All 12 affected services** began reporting metrics within 2 minutes of NetworkPolicy update
- **Zero false positive alerts** in the 6 months following the fix
- **3 AM pages eliminated** - team morale and trust in monitoring restored

**Metrics & Monitoring:**
- Prometheus scrape success rate: 99.2% → 99.97%
- Average scrape duration improved from 2.3s to 0.8s (removed retry delays)
- Monitoring coverage expanded to previously "invisible" services

**Process Improvements:**
- **Pre-deployment validation** caught 4 similar issues before production deployment
- **NetworkPolicy template adoption** across 45 services within 2 months
- **Documentation updated** with "Monitoring Considerations" section for NetworkPolicies

**Operational Impact:**
- **MTTR for monitoring issues reduced by 80%** (from 90 minutes to 18 minutes average)
- **Alert fatigue reduced** - on-call team confidence in alerts increased from 45% to 94%
- **No monitoring blind spots** - gained visibility into services we didn't know were failing to report

**Key Learnings Shared:**
1. "Healthy logs don't mean healthy monitoring" - always validate the monitoring path
2. NetworkPolicies are a common cause of monitoring failures in Kubernetes
3. The importance of "monitoring the monitors" - we now have meta-alerts for scrape failures
4. Testing network policies from the monitoring namespace should be part of deployment checklists

**Team Education:**
- Ran workshop: "Kubernetes Network Debugging for SRE" attended by 25 engineers
- Created runbook: "Troubleshooting Prometheus Scrape Failures" with decision tree
- Added to onboarding checklist: "Verify new services are scraped by Prometheus"

This incident highlighted a critical gap in our deployment process. The fix not only resolved immediate issues but established patterns preventing entire classes of monitoring failures. Six months later, we haven't had a single similar incident.

---

## Scenario 3: Exporters Stop After Kubernetes Upgrade

### **Situation**
We had just completed what should have been a routine Kubernetes upgrade from version 1.24 to 1.27 in our production cluster during a planned maintenance window. Immediately after the upgrade, our Grafana dashboards went dark - node-exporter and kube-state-metrics stopped reporting data for all 50 nodes in the cluster. We lost visibility into CPU, memory, disk, and pod metrics across the entire infrastructure. This was a critical situation because we were now flying blind - unable to see resource utilization, pod health, or detect potential issues. The upgrade had been tested in staging without issues, making this production failure particularly puzzling.

### **Task**
As the SRE lead for the Kubernetes platform, I needed to:
- Quickly restore monitoring visibility before any undetected issues could cascade
- Identify why exporters failed specifically in production (but not staging)
- Implement a fix without requiring a cluster rollback (which would have 2+ hour downtime)
- Document the root cause to prevent similar issues in future upgrades

### **Action**
I executed a rapid diagnosis and remediation plan under pressure:

**Step 1: Initial Assessment (First 5 Minutes)**
```bash
# Check exporter pod status
kubectl get pods -n monitoring
# Output showed:
# node-exporter-xxxxx    0/1     CrashLoopBackOff
# kube-state-metrics-0   0/1     Error

# Get recent logs
kubectl logs node-exporter-xxxxx -n monitoring --tail=50
# Error: "Failed to create pod security policy: the API version "policy/v1beta1" is no longer served"
```

**Immediate Finding:** The exporters were trying to use deprecated APIs that were removed in Kubernetes 1.25.

**Step 2: Root Cause Deep Dive**
```bash
# Check what API versions are available
kubectl api-versions | grep policy
# Output: Only "policy/v1" available (v1beta1 removed)

# Describe the failing pods for more details
kubectl describe pod node-exporter-xxxxx -n monitoring
# Events showed: "PodSecurityPolicy admission failed"
```

**Root Cause Identified:**
1. Kubernetes 1.25 removed PodSecurityPolicy (PSP) API entirely
2. Our exporters were deployed with Helm charts that relied on PSP
3. Staging didn't catch this because it skipped version 1.25 (went 1.23 → 1.27)
4. The new PodSecurity Admission Controller required different configuration

**Step 3: Check RBAC Permissions**
```bash
# Verify ServiceAccount permissions
kubectl auth can-i list nodes \
  --as=system:serviceaccount:monitoring:node-exporter

# Output: "no" - RBAC was also affected by API changes
```

**Step 4: Rapid Fix Implementation**

**A. Update namespace for PodSecurity Admission:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    # Replace PSP with PodSecurity admission
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

**B. Update RBAC for new API groups:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-exporter
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/metrics", "nodes/stats", "nodes/proxy"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
# Removed deprecated PSP permissions
```

**C. Upgrade exporters to compatible versions:**
```bash
# Update Helm repositories
helm repo update

# Upgrade node-exporter to Kubernetes 1.27-compatible version
helm upgrade node-exporter prometheus-community/prometheus-node-exporter \
  --version 4.24.0 \
  --namespace monitoring \
  --reuse-values \
  --set podSecurityPolicy.enabled=false

# Upgrade kube-state-metrics
helm upgrade kube-state-metrics prometheus-community/kube-state-metrics \
  --version 5.14.0 \
  --namespace monitoring \
  --set podSecurityPolicy.enabled=false
```

**D. Update DaemonSet strategy for smoother rollout:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 10  # Update 10 nodes at a time (out of 50)
  template:
    spec:
      securityContext:
        # Explicit security context replacing PSP
        runAsNonRoot: false
        runAsUser: 0
      hostNetwork: true
      hostPID: true
```

**Step 5: Validation & Rollout**
```bash
# Watch pod rollout
kubectl rollout status daemonset/node-exporter -n monitoring
# Completed in 3 minutes (10 nodes at a time)

# Verify metrics collection
kubectl exec -it prometheus-0 -n monitoring -- /bin/sh
wget -O- http://node-exporter.monitoring:9100/metrics | head
# Success! Metrics returning

# Check Grafana dashboards
# All node metrics restored within 5 minutes
```

**Step 6: Post-Incident Prevention**

**A. Created Kubernetes Upgrade Checklist:**
```markdown
## Pre-Upgrade Validation Checklist

- [ ] Review Kubernetes deprecation guide for target version
- [ ] Check all Helm chart compatibility with target K8s version
- [ ] Verify API versions used in all manifests
- [ ] Test upgrade in staging using SAME version path as production
- [ ] Document all deprecated APIs in current cluster:
      `kubectl get --raw /apis | jq` → check for beta APIs
- [ ] Backup all monitoring configurations
- [ ] Prepare rollback plan with timing estimates
```

**B. Created automated API deprecation scanner:**
```bash
#!/bin/bash
# check-deprecated-apis.sh
# Scans cluster for deprecated API usage

echo "🔍 Checking for deprecated APIs before upgrade to K8s 1.27..."

# Check for PSP usage
PSP_COUNT=$(kubectl get psp --all-namespaces 2>/dev/null | wc -l)
if [ $PSP_COUNT -gt 0 ]; then
  echo "⚠️  WARNING: $PSP_COUNT PodSecurityPolicies found (removed in 1.25)"
fi

# Check for deprecated API versions in deployed resources
kubectl get all --all-namespaces -o json | \
  jq -r '.items[] | select(.apiVersion | contains("v1beta1")) | 
  "\(.kind): \(.metadata.name) in \(.metadata.namespace) uses deprecated \(.apiVersion)"'

# Check for CronJob batch/v1beta1 (removed in 1.25)
kubectl get cronjobs --all-namespaces -o json | \
  jq -r '.items[] | select(.apiVersion == "batch/v1beta1") | 
  "CronJob \(.metadata.name) in \(.metadata.namespace) uses deprecated batch/v1beta1"'

echo "✅ API deprecation check complete"
```

**C. Updated CI/CD Pipeline:**
```yaml
# .gitlab-ci.yml
pre-upgrade-validation:
  stage: validate
  script:
    - ./scripts/check-deprecated-apis.sh
    - helm lint --strict charts/
    - kubectl apply --dry-run=server -f manifests/
  only:
    - kubernetes-upgrade-*
```

**D. Documentation Updates:**
- Created "Kubernetes Upgrade Runbook" with version-specific gotchas
- Added "Breaking Changes" section to internal wiki for each K8s version
- Documented PSP → PodSecurity Admission migration for all teams

### **Result**
The rapid response and systematic approach delivered both immediate resolution and long-term improvements:

**Immediate Recovery:**
- **Monitoring restored in 18 minutes** from incident detection to full dashboard recovery
- **Zero data loss** - Prometheus WAL preserved all metrics during the gap
- **No service impact** - applications continued running normally during monitoring outage
- **Avoided rollback** - saved 2+ hours of additional downtime

**Technical Metrics:**
- All 50 nodes reporting metrics within 5 minutes of fix deployment
- Exporter pod crash loops: 100% → 0% 
- Metrics ingestion rate returned to normal: 450K samples/minute
- Grafana dashboard recovery: 100% of 87 dashboards operational

**Process Improvements:**
- **Created reusable upgrade framework** adopted by 3 other teams managing K8s clusters
- **API deprecation scanner** integrated into CI/CD, running before every cluster upgrade
- **Staging-to-production version parity** enforced - staging now mirrors production upgrade paths

**Prevention Metrics:**
- **Caught 7 potential issues** in next upgrade (1.27 → 1.28) using the automated scanner
- **Upgrade confidence increased** - next 2 upgrades completed with zero monitoring issues
- **Reduced upgrade risk window** from 4 hours to 45 minutes with better preparation

**Team & Organizational Impact:**
- **Knowledge sharing:** Ran post-incident review attended by 40 engineers across 5 teams
- **Runbook created:** "Kubernetes Upgrade Survival Guide" with 12 version-specific sections
- **Saved estimated 15 hours** of debugging time in subsequent upgrades
- **C-level confidence restored** - cluster upgrades no longer considered "high-risk changes"

**Key Learnings Documented:**
1. **Always test with exact version path** - skipping versions in staging hides breaking changes
2. **API deprecations are not warnings** - they become hard failures in later versions
3. **Monitoring must be upgrade-proof** - it's your safety net during risky changes
4. **Automate compatibility checking** - humans miss deprecated API usage in 1000+ manifests
5. **Keep exporters at current versions** - legacy Helm charts become upgrade blockers

**Long-term Strategic Changes:**
- Adopted **gradual upgrade policy**: upgrade one minor version at a time (no more skipping)
- Implemented **quarterly "deprecation sprints"** to proactively update before forced by upgrades
- Created **"upgrade-safe" badge** for Helm charts that pass automated compatibility checks
- Established **monitoring as critical dependency** in upgrade planning (upgraded first, validated before proceeding)

This incident transformed our Kubernetes upgrade process from a nerve-wracking event to a well-rehearsed procedure. When we later upgraded to 1.28 and 1.29, we had zero monitoring issues and completed upgrades 60% faster. The automated deprecation scanner alone has prevented an estimated 8-10 similar incidents across our infrastructure teams.

---

## Scenario 4: Prometheus Crashes During Outage

### **Situation**
We were responding to a critical production incident - our API gateway was experiencing 30% error rates affecting thousands of customers. The on-call team was actively investigating using Grafana dashboards when suddenly Prometheus itself crashed. All metrics dashboards went blank, alert history disappeared, and we lost the ability to query any time-series data. This was the worst possible timing - we were blind during an active outage, unable to correlate metrics with the incident timeline. The pressure was immense: fix Prometheus to restore visibility while also resolving the underlying API gateway issue. We estimated about 15 minutes of metrics data was in Prometheus's Write-Ahead Log (WAL) but not yet persisted to TSDB, at risk of being lost forever.

### **Task**
My immediate responsibilities were:
- Recover Prometheus as quickly as possible to restore monitoring visibility
- Prevent loss of the critical 15 minutes of metrics data covering the incident timeline
- Identify why Prometheus crashed to prevent recurrence during incident response
- Implement safeguards to ensure monitoring survives future high-stress scenarios

### **Action**
I executed an emergency recovery procedure while coordinating with the team handling the API gateway incident:

**Step 1: Initial Assessment (First 60 Seconds)**
```bash
# Check Prometheus pod status
kubectl get pods -n monitoring
# Output: prometheus-0   0/1   CrashLoopBackOff

# Get crash logs
kubectl logs prometheus-0 -n monitoring --previous --tail=100
```

**Critical Error Found:**
```
level=error ts=2024-10-15T14:23:47.892Z caller=main.go:345 msg="Error opening memory series storage" err="open /prometheus/wal/00000123: cannot allocate memory"
level=error ts=2024-10-15T14:23:47.893Z msg="TSDB failed to open" err="out of memory"
Fatal: cannot start prometheus
```

**Root Cause Identified:** Prometheus ran out of memory (OOM) due to:
1. High query load during incident investigation (5 engineers running complex queries simultaneously)
2. Large WAL file accumulated during high-cardinality metrics spike
3. Memory limits too restrictive (2GB) for workload

**Step 2: Emergency Recovery - Preserve WAL Data**
```bash
# Create backup of WAL before any recovery attempts
kubectl exec prometheus-0 -n monitoring -- tar -czf /tmp/wal-backup.tar.gz /prometheus/wal/

# Copy WAL backup out of pod (in case pod gets recreated)
kubectl cp monitoring/prometheus-0:/tmp/wal-backup.tar.gz ./wal-backup-$(date +%Y%m%d-%H%M%S).tar.gz

# Verify backup integrity
tar -tzf wal-backup-*.tar.gz | head
```

**Step 3: Increase Resource Limits for Recovery**
```bash
# Temporarily increase memory to allow recovery
kubectl edit statefulset prometheus -n monitoring
```

```yaml
# Modified resource limits
resources:
  requests:
    memory: 4Gi   # Was: 2Gi
    cpu: 2000m
  limits:
    memory: 8Gi   # Was: 2Gi (no limit before)
    cpu: 4000m
```

**Step 4: Enable Safe Recovery Mode**
```bash
# Add startup parameters for safe recovery
kubectl edit statefulset prometheus -n monitoring
```

```yaml
args:
  - '--config.file=/etc/prometheus/prometheus.yml'
  - '--storage.tsdb.path=/prometheus'
  - '--storage.tsdb.wal-replay-enabled=true'  # Explicitly enable WAL replay
  - '--query.max-concurrency=5'               # Limit concurrent queries
  - '--query.timeout=2m'                      # Prevent runaway queries
  - '--query.max-samples=50000000'           # Limit query memory usage
  - '--web.enable-admin-api'                 # Enable emergency admin operations
```

**Step 5: Delete Pod to Trigger Restart with New Config**
```bash
# Delete pod (StatefulSet will recreate it)
kubectl delete pod prometheus-0 -n monitoring

# Watch recovery process
kubectl logs prometheus-0 -n monitoring -f
```

**Recovery Log Output (Successful):**
```
level=info ts=2024-10-15T14:31:22.123Z msg="Starting Prometheus"
level=info ts=2024-10-15T14:31:22.456Z msg="Replaying WAL" dir=/prometheus/wal
level=info ts=2024-10-15T14:31:35.789Z msg="WAL replay completed" duration=13.333s
level=info ts=2024-10-15T14:31:36.012Z msg="TSDB opened successfully" duration=14s
level=info ts=2024-10-15T14:31:36.345Z msg="Server is ready to receive web requests"
```

**Critical Success:** All 15 minutes of incident data was preserved and available for analysis!

**Step 6: Verify Data Integrity**
```bash
# Query metrics to confirm data continuity
kubectl port-forward prometheus-0 9090:9090 -n monitoring

# In browser, check Prometheus UI
# Query: up{job="api-gateway"}[20m]
# Confirmed: No gaps in data, incident timeline fully captured
```

**Step 7: Post-Recovery Optimization**

**A. Implemented Query Governance:**
```yaml
# Added query limits to prevent resource exhaustion
global:
  query_log_file: /prometheus/query.log

# Created recording rules to pre-compute expensive queries
groups:
  - name: query_optimization
    interval: 30s
    rules:
    - record: api:request_rate:5m
      expr: sum(rate(http_requests_total[5m])) by (service)
    - record: api:error_rate:5m
      expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
```

**B. Implemented Resource Management:**
```yaml
# Final resource configuration with headroom
resources:
  requests:
    memory: 6Gi
    cpu: 2000m
  limits:
    memory: 12Gi   # 2x request for burst capacity
    cpu: 4000m

# Added PVC expansion for TSDB growth
volumeClaimTemplates:
  - metadata:
      name: prometheus-storage
    spec:
      resources:
        requests:
          storage: 100Gi  # Increased from 50Gi
```

**C. Created Prometheus High Availability Setup:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
spec:
  replicas: 2  # HA pair - if one crashes, other continues
  serviceName: prometheus-ha
  podManagementPolicy: Parallel
```

**D. Implemented "Meta-Monitoring" for Prometheus Health:**
```yaml
# New alert rules to prevent future crashes
groups:
  - name: prometheus_health
    rules:
    - alert: PrometheusHighMemoryUsage
      expr: process_resident_memory_bytes{job="prometheus"} / 
            kube_pod_resource_limits{resource="memory", pod=~"prometheus-.*"} > 0.80
      for: 5m
      annotations:
        summary: "Prometheus using >80% memory - risk of OOM"
        
    - alert: PrometheusWALCorruption
      expr: prometheus_tsdb_wal_corruptions_total > 0
      annotations:
        summary: "WAL corruption detected - immediate investigation needed"
        
    - alert: PrometheusSlowQueries
      expr: histogram_quantile(0.95, rate(prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query"}[5m])) > 30
      for: 10m
      annotations:
        summary: "Slow queries detected - review query patterns"
```

**E. Created Emergency Runbook:**
```markdown
# Prometheus Crash Recovery Runbook

## Emergency Response (First 5 Minutes)

1. **Backup WAL immediately:**
   ```bash
   kubectl exec prometheus-0 -n monitoring -- \
     tar -czf /tmp/wal-backup.tar.gz /prometheus/wal/
   kubectl cp monitoring/prometheus-0:/tmp/wal-backup.tar.gz ./wal-backup.tar.gz
   ```

2. **Check crash cause:**
   ```bash
   kubectl logs prometheus-0 -n monitoring --previous | tail -50
   ```

3. **Common causes:**
   - OOM: Increase memory limits
   - Disk full: Reduce retention or expand PVC
   - Corrupted block: Run `promtool tsdb analyze`

4. **Recovery with increased resources:**
   - Edit StatefulSet to increase memory
   - Delete pod to trigger restart
   - Monitor logs for WAL replay completion

## Post-Recovery Checklist
- [ ] Verify no data gaps in queries
- [ ] Review query logs for expensive queries
- [ ] Check TSDB block integrity
- [ ] Validate all alerts are firing correctly
- [ ] Document incident timeline for postmortem
```

**Step 8: Long-term Prevention**

**Implemented Thanos for Resilience:**
```yaml
# Deployed Thanos sidecar for:
# 1. Long-term storage backup
# 2. Query federation (multiple Prometheus instances)
# 3. Downsampling to reduce storage pressure

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
spec:
  template:
    spec:
      containers:
      - name: prometheus
        # ... existing prometheus config
      - name: thanos-sidecar