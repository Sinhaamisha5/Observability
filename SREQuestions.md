# Incident Management & Operations Questions - Interview Answers

## 1. Walk me through your incident response process from detection to resolution

**ANSWER (60 seconds):**

*"Detection first - we have multi-window burn rate alerts that page on-call within a minute. I acknowledge immediately, join the incident bridge, and do quick triage - is it affecting users, what's the scope, what changed recently.*

*I follow a systematic troubleshooting process: check monitoring dashboards, review recent deployments, examine logs, check dependencies. The goal is stabilization first, root cause later. If I can't resolve in 15 minutes, I escalate and pull in experts.*

*Communication is critical - I update stakeholders every 15 minutes even if there's no new information. Post status updates to our incident channel and external status page if customer-facing.*

*Once resolved, I track recovery metrics and ensure we're fully restored. Within 48 hours, I write a blameless postmortem focusing on what happened, why detection/response took as long as it did, and action items to prevent recurrence. We review postmortems as a team and assign owners to follow-up items.*

*Key metrics I track: MTTD under 1 minute, MTTR under 5 minutes for SEV-1s, and we close the loop on 100% of action items within 30 days."*

**INCIDENT RESPONSE PROCESS:**

```
1. DETECTION (Goal: <1 minute)
   ├─ Alert fires
   ├─ On-call acknowledges
   └─ Initial triage

2. RESPONSE (Goal: <5 minutes to stabilize)
   ├─ Join incident bridge
   ├─ Quick assessment (scope, impact, urgency)
   ├─ Check recent changes
   └─ Begin troubleshooting

3. COMMUNICATION (Every 15 minutes)
   ├─ Update incident channel
   ├─ Status page if customer-facing
   └─ Escalate if needed

4. RESOLUTION
   ├─ Stabilize first (restore service)
   ├─ Root cause investigation later
   └─ Verify full recovery

5. POST-INCIDENT (Within 48 hours)
   ├─ Blameless postmortem
   ├─ Action items with owners
   └─ Track completion
```

**KEY METRICS:**
- **MTTD** (Mean Time to Detect): <1 minute
- **MTTR** (Mean Time to Resolve): <5 minutes for SEV-1
- **Action Item Completion**: 100% within 30 days

---

## 2. What's your strategy for post-incident reviews / blameless RCAs?

**ANSWER (60 seconds):**

*"I run blameless postmortems within 48 hours while details are fresh. The format: timeline of events, impact metrics, root cause analysis using the 5-whys technique, what went well, what went poorly, and action items.*

*Critical rule: no blame, no punishment. We focus on process and system improvements, not individual mistakes. If someone made an error, we ask why the system allowed that error - was documentation unclear, was the tool confusing, did we lack proper testing?*

*I involve everyone who participated - we do a collaborative session, not a blame game. The goal is learning, not finger-pointing. Action items are specific, assigned to owners, with deadlines. We track completion in our next team meetings.*

*We share postmortems across teams and maintain a searchable repository. Before major changes, we review past incidents to avoid repeating mistakes. Some of our best reliability improvements came from postmortem action items - like implementing canary deployments after a bad rollout, or adding better monitoring after an undetected degradation."*

**POSTMORTEM TEMPLATE:**

```markdown
# Incident Postmortem: [Brief Title]

**Date**: 2024-01-15
**Duration**: 2 hours 15 minutes
**Severity**: SEV-1
**Impact**: 30% of users unable to checkout

## Timeline
- 14:00 - Deployment started
- 14:15 - Error rate spiked to 35%
- 14:17 - Alert fired, on-call paged
- 14:20 - Incident declared
- 14:45 - Root cause identified (bad config)
- 15:30 - Rollback completed
- 16:15 - Full recovery confirmed

## Impact
- Users affected: ~50,000
- Revenue impact: ~$120K
- Customer complaints: 47

## Root Cause
Database connection pool size set to 10 (should be 100) in new config

## 5-Whys Analysis
1. Why did checkout fail? → Database connection timeout
2. Why timeout? → Not enough connections available
3. Why not enough? → Pool size was 10 instead of 100
4. Why was it 10? → Config template had wrong default
5. Why wrong default? → Template wasn't reviewed in code review

## What Went Well
✅ Quick detection (2 minutes)
✅ Clear communication
✅ Rollback procedure worked perfectly

## What Went Poorly
❌ Config change not caught in review
❌ Staging didn't have production load to catch this
❌ No automated config validation

## Action Items
1. [Owner: Alice] Add config validation to CI pipeline - Due: Jan 22
2. [Owner: Bob] Update code review checklist - Due: Jan 19
3. [Owner: Charlie] Implement load testing in staging - Due: Feb 5
4. [Owner: Diana] Document config best practices - Due: Jan 26
```

**BLAMELESS PRINCIPLES:**
- Focus on **systems**, not individuals
- Ask "why did the system allow this?" not "who did this?"
- Assume good intentions
- Learn and improve, don't punish
- Share knowledge across teams

---

## 3. What metrics and dashboards do you consider essential for observability?

**ANSWER (60 seconds):**

*"I organize dashboards into three layers. First, the RED metrics for services: Rate of requests, Errors, and Duration. These tell me if users are experiencing issues. Second, USE metrics for resources: Utilization, Saturation, and Errors - CPU, memory, disk, network. Third, business metrics like checkout completion rate or API success rate.*

*For SRE-specific dashboards: SLO compliance showing error budget burn rate, incident response metrics with MTTD and MTTR, capacity utilization trending toward 80% thresholds, and deployment frequency with success rates.*

*I believe in alert-on-symptoms, not-causes. Alert when users are impacted - high error rate, slow response times - not when a single server's CPU is high. And every alert must be actionable with a clear runbook.*

*For distributed systems, I add distributed tracing to track requests across services and understand where latency accumulates. Logs, metrics, and traces together give complete observability. I also implement anomaly detection - alerting when patterns are unusual even if absolute thresholds aren't breached."*

**ESSENTIAL DASHBOARDS:**

**1. Service Health Dashboard (RED Metrics)**
```
Rate:     Request volume over time
Errors:   Error rate (%) and count
Duration: Latency (P50, P95, P99)
```

**2. Resource Dashboard (USE Metrics)**
```
Utilization: % of resource in use (CPU, memory, disk, network)
Saturation:  Queue depth, wait times
Errors:      Failed operations, packet loss
```

**3. SLO Dashboard**
```
- SLI current value
- Error budget remaining
- Burn rate (how fast consuming budget)
- 30-day trend
```

**4. Incident Response Dashboard**
```
- MTTD (Mean Time to Detect)
- MTTR (Mean Time to Resolve)
- Incident frequency
- On-call load
```

**5. Capacity Dashboard**
```
- Current utilization (compute, storage, network)
- Growth trends
- Forecast (when will we hit 80%?)
- Procurement pipeline
```

**THE THREE PILLARS:**
- **Metrics**: Numbers over time (Prometheus)
- **Logs**: Event details (ELK, Splunk)
- **Traces**: Request flow across services (Jaeger, Zipkin)

**ALERT PRINCIPLES:**
- Alert on **symptoms** (user impact), not causes
- Every alert must be **actionable**
- Include **runbook** link in alert
- Avoid alert fatigue (quality over quantity)

---

## 4. How do you differentiate between symptom and root cause in alerts?

**ANSWER (60 seconds):**

*"Symptoms are what users experience - high latency, errors, service unavailable. Root causes are why it's happening - database overload, memory leak, network partition.*

*I always alert on symptoms because that's what matters to users. If checkout is failing, I don't care initially whether it's database, network, or application - I just need to know users can't check out.*

*During investigation, I work backwards from symptom to root cause. High API latency is a symptom. I dig into metrics - is it slow database queries? That's closer to root cause. Why are queries slow? Table lacks an index. That's the root cause.*

*The 5-whys technique helps. Users can't log in. Why? Authentication service timing out. Why? Database queries taking 30 seconds. Why? Lock contention on users table. Why? Batch job running without proper transaction handling. Why? No code review caught the missing transaction boundaries. That's your root cause.*

*Key rule: fix symptoms first to restore service, then investigate root cause to prevent recurrence. In the postmortem, identify both - the symptom we alerted on and the root cause we fixed."*

**SYMPTOMS VS ROOT CAUSES:**

| Symptom (Alert On This) | Possible Root Causes |
|-------------------------|---------------------|
| High error rate | Database down, network partition, code bug, overload |
| High latency | Slow queries, memory pressure, network congestion |
| Service unavailable | Host failure, OOM kill, dependency failure |
| Failed checkouts | Payment API down, database timeout, cache failure |

**5-WHYS EXAMPLE:**

```
Symptom: Users can't log in

Why #1: Authentication service timing out
  → Why #2: Database queries taking 30 seconds
    → Why #3: Lock contention on users table
      → Why #4: Batch job running without proper transactions
        → Why #5: Code review didn't catch transaction issue
          ROOT CAUSE: Missing code review checklist item
```

**INVESTIGATION FLOW:**
```
1. Detect SYMPTOM (alert fires)
   ↓
2. Stabilize (restore service)
   ↓
3. Investigate ROOT CAUSE (5-whys)
   ↓
4. Fix ROOT CAUSE (prevent recurrence)
   ↓
5. Document in postmortem
```

**KEY PRINCIPLE:**
> "Alert on what users see (symptoms), fix what causes it (root cause)"

---

## 5. Describe a time you automated an operational task that reduced toil

**ANSWER (60 seconds):**

*"We had a manual process for certificate renewals - tracking 500+ certificates across AWS, GCP, and OpenStack, checking expiration dates monthly, manually requesting renewals, deploying them. Took 20 hours monthly and we had three outages from missed expirations.*

*I built a Python tool called cert-guardian that scans all platforms daily, identifies certificates expiring within 30 days, automatically renews them using ACM, Certificate Manager, or Let's Encrypt depending on platform, deploys to load balancers, and sends Slack notifications.*

*Implementation took 3 weeks. Now runs as a Kubernetes CronJob daily. Impact: zero certificate outages in 18 months since deployment, reduced manual work from 20 hours to 2 hours monthly - 90% reduction in toil, improved security compliance score by 35%, and manages 750 certificates automatically.*

*The key was making it production-ready - comprehensive error handling, detailed logging, dry-run mode for testing, monitoring with Prometheus metrics, and a web dashboard for visibility. This freed up time for actual engineering work instead of repetitive operational tasks."*

**TOIL AUTOMATION EXAMPLE:**

**Before Automation:**
```
Manual Process:
- Track 500+ certificates in spreadsheet
- Check expiration dates monthly
- Request renewals manually (different process per platform)
- Deploy to load balancers manually
- Hope nothing was missed

Time: 20 hours/month
Outages: 3 in 12 months
Error-prone: Yes
Scalability: No
```

**After Automation:**
```
Automated Process:
- cert-guardian scans daily
- Identifies certs expiring < 30 days
- Auto-renews via ACM/GCP/Let's Encrypt
- Auto-deploys to load balancers
- Slack notifications

Time: 2 hours/month (monitoring only)
Outages: 0 in 18 months
Error-prone: No
Scalability: Yes (750+ certs now)
```

**IMPACT METRICS:**
- ✅ 90% reduction in manual toil
- ✅ Zero outages (vs 3 previously)
- ✅ 35% better security compliance
- ✅ Scales to 750+ certificates
- ✅ Team time freed for engineering work

**WHAT MAKES AUTOMATION PRODUCTION-READY:**
1. Comprehensive error handling
2. Detailed logging
3. Dry-run mode for testing
4. Monitoring and alerting
5. Dashboard for visibility
6. Documentation and runbooks

---

## 6. What alerting anti-patterns have you encountered?

**ANSWER (75 seconds):**

*"The biggest is alert fatigue - too many alerts that aren't actionable. Early in my career, we'd get 50+ alerts daily. Most were noise. People started ignoring them, and we missed a critical database failure buried in the spam.*

*Specific anti-patterns I've seen: First, alerting on causes instead of symptoms - 'disk 80% full' isn't actionable unless it's actually affecting users. Alert when the service degrades. Second, duplicate alerts - five alerts for the same incident from different systems. Use alert aggregation.*

*Third, no clear ownership - alert goes to a general channel, no one responds thinking someone else will handle it. Every alert needs a clear owner. Fourth, vague descriptions - 'Service unhealthy' doesn't tell you what to do. Alerts should include context and link to runbooks.*

*Fifth, alert thresholds too sensitive - constant flapping. Use multi-window alerts and require the condition to persist. Sixth, no alert on recovery - you get paged at 3 AM but no notification when it auto-recovers, so you stay awake investigating.*

*My principle: every alert must wake someone up for a reason, be actionable, have a runbook, and clearly indicate what's broken and the impact. If an alert doesn't meet these criteria, it shouldn't page anyone - maybe just log it or show on a dashboard."*

