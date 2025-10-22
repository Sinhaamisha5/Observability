# Service Reliability Engineering (SRE) Fundamentals
## A Complete Guide to SLIs, SLOs, Error Budgets, and Monitoring Dashboards

---

## Table of Contents
1. [Service Level Indicators (SLIs)](#slis)
2. [Service Level Objectives (SLOs)](#slos)
3. [Error Budgets](#error-budgets)
4. [SRE Dashboards](#dashboards)
5. [Interview Preparation](#interview-prep)
6. [Implementation Guide](#implementation)

---

## Service Level Indicators (SLIs) {#slis}

### What Are SLIs?

Think of **Service Level Indicators (SLIs)** as your system's vital signs — just like a heartbeat or blood pressure for humans. They measure how well your system is performing for users.

**Definition:** An SLI is a quantitative measurement of some aspect of reliability — like availability, latency, errors, throughput, or saturation.

### Monitoring vs Observability

Understanding the difference between these two concepts is crucial:

| Concept | Analogy | Meaning |
|---------|---------|---------|
| **Monitoring** | Security cameras and alarms | Tells you when something happens — predefined alerts and metrics. |
| **Observability** | Forensic toolkit (footprints, fingerprints) | Lets you understand why it happened — by exploring logs, traces, and metrics. |

Together, they help you detect and diagnose reliability issues.

### The 5 Core Types of SLIs

| Type | Measures | Example | Why It Matters |
|------|----------|---------|----------------|
| **Availability** | % of successful requests | `200 OK` responses ÷ total requests | Tells if your service is up and reachable. |
| **Latency** | Speed of responses | % of requests < 200 ms | Reflects user experience — slow = bad UX. |
| **Errors** | Failure rate | `5xx` or timeouts ÷ total | Reveals when backend or logic fails. |
| **Throughput** | Work done per time | Requests/sec, data processed | Shows how much load your system handles. |
| **Saturation** | Resource pressure | CPU, memory, queue usage | Early warning before the system crashes. |

### How SLIs Are Measured

Here are Prometheus-style queries for common SLIs:

| SLI | Example Formula |
|-----|-----------------|
| **Availability** | `(successful_requests / total_requests) * 100` |
| **Latency** | `histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m]))` |
| **Error Rate** | `(error_responses / total_requests) * 100` |
| **Throughput** | `rate(http_requests_total[5m])` |
| **Saturation** | `CPU_usage / CPU_limit` or `queue_length / queue_capacity` |

**Critical Point:** Always ensure the query matches your definition — otherwise, you get misleading data.

### White Box Monitoring

"White box" monitoring means you're looking inside the system, not just from the outside.

You get data by:
- Instrumenting code (adding metrics in your application)
- Collecting via Prometheus exporters
- Visualizing with Grafana

**Example:** A Go app might expose metrics like:

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{status="200"} 5342
```

Prometheus scrapes this endpoint and stores time-series data for analysis.

### Practical Example: Online Record Store API

Consider an API service for an online vinyl record store where users browse, search, and order records.

| User Journey | SLI | What It Measures | Why It Matters |
|--------------|-----|------------------|----------------|
| Search records | Availability | % of requests returning 200 OK | Users can access catalog |
| Search records | Latency | % of searches under 300ms | Ensures fast browsing |
| Create order | Availability | % of successful POST /orders | Ensures customers can buy |
| Process order | Success rate | % of orders fully processed | Ensures backend is reliable |
| System health | Saturation | CPU/memory usage vs limit | Prevents overload |

### How to Collect SLI Data

Four main methods for collecting SLI data:

**1. Application Instrumentation**
- Add code that exposes metrics
- Tools: Prometheus client libraries
- Provides granular insights into application behavior

**2. Load Balancer/Proxy Metrics**
- Gather from infrastructure (no code changes required)
- Tools: NGINX, Envoy, AWS ALB
- Captures traffic patterns at the edge

**3. Client-Side Instrumentation**
- Measure from user's browser or device
- Tools: Real User Monitoring (RUM) solutions
- Shows actual user experience

**4. Synthetic Testing**
- Simulated requests to check uptime
- Tools: Blackbox Exporter, curl loops
- Tests reliability when no real users are online

**Best Practice:** Combine multiple data sources for accuracy.

### Synthetic Monitoring Example

Here's a simple bash check that monitors availability:

```bash
while true; do
  time curl -s -o /dev/null -w "%{http_code}\n" https://myapp.com/health
  sleep 60
done
```

This keeps checking if your API is up and how fast it responds — even when no real users are online.

---

## Service Level Objectives (SLOs) {#slos}

### What Are SLOs?

An **SLO** (Service Level Objective) is a target for how reliable or fast a service should be — based on what your users actually care about.

It's not about being perfect (100% uptime), but about finding the sweet spot between:
- User happiness
- Business goals
- Engineering feasibility

### Why 100% Uptime Is Unrealistic

Complex systems will always have some downtime or errors. Even big companies like Google or Amazon don't target 100% uptime. It's more effective to aim for something like 99.9% availability — high enough for users, but achievable for engineers.

### What SLOs Really Do

SLOs aren't just numbers — they give teams a shared language to talk about reliability:

| SLOs Help You... | Example |
|------------------|---------|
| Quantify acceptable risk | "We can tolerate 0.1% request failures per week." |
| Align engineering with business | "This service can be down for 10 minutes per month and still meet the SLA." |
| Prioritize real user-impacting issues | "Slow checkout times affect revenue more than slow search." |
| Guide investment | "Is it worth doubling cost just to gain another 'nine'?" |

### SLO = Built on SLI

The relationship between SLI, SLO, and SLA is fundamental:

| Concept | Meaning | Example |
|---------|---------|---------|
| **SLI** (Indicator) | What you measure | % of requests < 300 ms |
| **SLO** (Objective) | What you target | 99% of requests < 300 ms |
| **SLA** (Agreement) | What you promise to customers | If uptime < 99.9%, we refund users |

**Key Point:** SLIs feed into SLOs, and SLOs define SLAs.

### How to Build a Good SLO (Step-by-Step)

#### Step 1: Start with the User Journey

Ask: what are the critical steps users take in your product?

**Example (Online Record Store):**
- Searching for records
- Adding to cart
- Checking out and processing orders

#### Step 2: Map Each Step to What Users Care About

| User Journey | What Users Expect | SLI Type | Example SLO |
|--------------|-------------------|----------|-------------|
| Search | Fast, reliable results | Availability + Latency | 99.9% success; 99% under 300 ms |
| Order Processing | Quick checkout | Success Rate + Latency | 99.9% success; 95% under 3 s |

#### Step 3: Balance Between User Happiness and Cost

Each additional "9" (like going from 99.9% → 99.99%) costs exponentially more. The cost goes up fast while revenue or satisfaction improves slowly. The right SLO is the point where cost equals business value gained.

**Examples by Use Case:**
- A photo app might be fine with 99.9% uptime
- A payment service may require 99.999% (five nines)
- A social media platform might target 99.95%

### Setting and Reviewing SLOs

**When you first set them:**
- Use benchmarks (industry averages)
- Use historical system data
- Document why you chose each target

**Over time:**
- Review performance (did we meet targets?)
- Gather user feedback
- Recalculate based on cost, performance, and satisfaction

**Important:** SLOs are living metrics — not "set and forget."

### Example SLO Conversation

"Our search API has 99.9% availability and 99% of queries under 300 ms. We chose those numbers because users notice slowdowns after 300 ms, and that level fits our infrastructure's capacity."

That's exactly how an SRE or DevOps engineer might explain it in an interview.

---

## Error Budgets {#error-budgets}

### What Is an Error Budget?

An **error budget** represents how much unreliability you can afford before breaking your SLO (Service Level Objective).

**Simple Definition:** It's the difference between 100% reliability and your SLO target.

**Example:** If your SLO = 99.9% uptime, then your error budget = 0.1% downtime allowed.

That means your service can be down for 0.1% of the total time in a given period (e.g., a month) without violating the SLO.

### How to Calculate Error Budget

Let's take an example of **99.95% availability SLO**:

- Total time in 30 days = 43,200 minutes
- Error budget = 0.05% downtime = 0.0005 × 43,200 = **21.6 minutes of downtime allowed per month**

So your system can be unavailable for up to 21.6 minutes/month and still be within the target.

### Common Error Budget Examples

| SLO Target | Error Budget (per month) | Error Budget (per year) |
|------------|--------------------------|------------------------|
| 99.0% | 7.2 hours | 87.6 hours |
| 99.5% | 3.6 hours | 43.8 hours |
| 99.9% | 43.2 minutes | 8.76 hours |
| 99.95% | 21.6 minutes | 4.38 hours |
| 99.99% | 4.32 minutes | 52.6 minutes |
| 99.999% | 26 seconds | 5.26 minutes |

### Purpose: Balancing Reliability & Innovation

Error budgets help balance reliability and innovation. If you focus too much on reliability, you slow innovation (no changes, no new features). If you release too fast, you risk instability and downtime.

Error budgets give teams a measured, data-driven way to decide when to:
- Push new features
- Freeze deployments
- Focus on improving reliability

### How Teams Use Error Budgets

Teams set thresholds for budget consumption:

| Budget Used | Action |
|-------------|--------|
| **50%** | Review incidents, investigate performance |
| **75%** | Slow releases, increase testing |
| **100%** | Freeze new features, focus on reliability |

This helps automate decision-making — no emotional debates; just clear rules.

### Implementing Error Budgets (Step-by-Step)

1. **Define clear SLOs** for key services (e.g., availability, latency, success rate)
2. **Document the calculation** of how the error budget is computed
3. **Build monitoring dashboards** to track real-time consumption
4. **Set thresholds** (e.g., 50%, 75%, 100%) and define what actions to take
5. **Integrate with CI/CD** so deployment freezes or alerts happen automatically
6. **Communicate to everyone** — Dev, QA, Product — how it works
7. **Review & adjust regularly** based on incidents and metrics

### Example Interview Answer: Error Budgets

**Question:** "What is an error budget, and how would you implement it?"

**Answer:**
"An error budget is the amount of unreliability a service can tolerate while still meeting its SLOs. For example, if we have a 99.9% uptime SLO, the 0.1% downtime allowed per month is our error budget.

In practice, I'd implement it by tracking SLI metrics like availability in Prometheus, calculating budget burn in Grafana, and setting thresholds that trigger alerts or slow down deployments. This ensures a balance between reliability and innovation — if the error budget is mostly used up, we focus on fixing issues rather than releasing new features."

---

## SRE Dashboards {#dashboards}

### Goal of Dashboards in SRE

Once you've defined SLIs, SLOs, and Error Budgets, the next challenge is: "How do we make this data visible and actionable for the team?"

Dashboards help teams:
- See service reliability in real time
- Understand trends (Are we improving or getting worse?)
- Take action when reliability starts to drop
- Communicate system health clearly — not just to engineers, but also product managers, leadership, and business teams

### What Makes a Good SLO Dashboard

#### 1. User-Centric Focus

Show metrics that actually impact the user. Examples include:
- Availability (% of successful requests)
- Latency (response times)
- Error rate
- User journey success rate (checkout, login, upload, etc.)

**Key Point:** Make sure user-facing SLIs are prominent — right at the top of the dashboard.

#### 2. Visual Hierarchy

Design the dashboard like a story:
- **Top:** Overall service health
- **Middle:** Key metrics (availability, latency, error budget burn)
- **Bottom:** Supporting data (component or dependency health, historical trends)

This lets anyone quickly tell:
- "Is the service healthy?"
- "If not, what's causing the issue?"

#### 3. Color Coding

Use color to make the dashboard intuitive:
- 🟢 **Green:** SLOs are met comfortably
- 🟡 **Yellow:** Approaching SLO limits
- 🔴 **Red:** SLO violated, needs immediate action

Even non-technical viewers can instantly see the situation.

#### 4. Error Budget Visualization

Always include:
- Total error budget (how much unreliability is allowed)
- Burn rate (how fast you're consuming the budget)
- Projected exhaustion date (when you'll run out of error budget if trends continue)

This helps teams forecast problems before they cause SLO violations.

#### 5. Context and Clarity

Add context directly in the dashboard:
- Show SLO targets (e.g., 99.9% uptime over 30 days)
- Show time window (daily, weekly, monthly)
- Add links to incident playbooks or service dependencies

That way, anyone looking at the dashboard can understand it instantly.

### Recommended Dashboard Layout

Here's what a strong SLO dashboard might look like (using Prometheus + Grafana):

| Section | Example Panels |
|---------|-----------------|
| **Top (Summary)** | Overall Service Health (e.g., 99.95% uptime) |
| **Middle (Metrics)** | Availability %, Latency (p95), Error Budget Burn Rate |
| **Trend Graphs** | 7-day trend for availability and latency |
| **Bottom (Components)** | Subsystem health (e.g., API, DB, cache) + alerts panel |

### Practical Example Scenario

Let's say your SLO is: "95% of orders complete within 3 seconds."

You'd include:
- A latency panel showing the % of orders under 3s
- A trend graph over time
- A burn-down meter showing how much of the latency error budget has been used

Now, if latency spikes, everyone can immediately see that reliability is at risk and take preventive action.

---

## Interview Preparation {#interview-prep}

### Key Interview Questions and Answers

#### Question 1: "What is an SLI and how is it different from an SLO?"

**Answer:**
"An SLI (Service Level Indicator) is a quantitative measurement of system reliability — what we actually measure. Examples include availability, latency, error rate, throughput, and saturation.

An SLO (Service Level Objective) is a target goal based on SLIs — what we commit to. For example, if our SLI shows that 99.5% of requests succeed, our SLO might be 'we target 99% success rate.'

The key difference: SLIs measure reality; SLOs define ambition."

#### Question 2: "How would you define SLOs for a payment processing service?"

**Answer:**
"For a payment service, I'd focus on the user journeys that matter most:

1. Payment submission — 99.99% availability, 99.9% under 2 seconds
2. Transaction confirmation — 99.99% success rate
3. Error handling — 99% of errors logged and alertable within 10 seconds

I'd set these high targets (many nines) because payment failures directly impact revenue and customer trust. I'd use historical data and industry benchmarks to validate the targets, then review quarterly based on actual performance and cost."

#### Question 3: "How do you monitor SLOs and error budgets?"

**Answer:**
"Once SLOs and error budgets are defined, I make them visible through dashboards — typically in Grafana, fed by Prometheus metrics. I design dashboards that show user-facing metrics like availability, latency, and error budget burn at the top, color-coded for quick understanding. I also include trends, context (like SLO targets and measurement windows), and links to incident runbooks.

This makes reliability data actionable — teams can immediately see when we're close to burning our budget and slow down risky deployments before we violate SLOs."

#### Question 4: "How would you respond if the error budget is 75% consumed with two weeks left in the month?"

**Answer:**
"At 75% consumption with time remaining, I'd trigger our predefined action plan:

1. Slow down feature deployments — shift from rolling releases to scheduled, tested batches
2. Increase testing rigor — ensure only critical bug fixes go out
3. Focus on stability — pause any risky infrastructure changes
4. Monitor burn rate closely — if it continues at the current pace, we'll exceed the budget

I'd communicate this to product and engineering teams clearly: we have limited budget left, so we're deprioritizing new features in favor of reliability. Once we get closer to the end of the month and see if we stay within budget, we can resume normal release cadence next month."

#### Question 5: "Walk me through your complete SLI-SLO-Error Budget-Dashboard implementation"

**Answer:** (See structured answer below)

### Complete Implementation Walkthrough

**Scenario:** Implementing SLI–SLO–Error Budget tracking dashboard in Grafana for a microservices platform.

**Your Response:**

"To implement the SLI–SLO–SLA and error budget tracking dashboard in Grafana, I started by defining the right Service Level Indicators (SLIs) — the key metrics that represent user experience, such as availability, latency, and error rate.

Then, based on those indicators, I defined Service Level Objectives (SLOs) — for example, 99.9% of requests should succeed within 200 ms over a 30-day window. From that, I calculated the error budget — in this case, 0.1% unreliability allowed, or about 43 minutes of downtime per month.

Once the targets were set, I used Prometheus to collect and store the metrics. For example:
- `rate(http_requests_total{status!~"5.."}[5m])` for availability
- `histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m]))` for latency
- `rate(http_requests_total{status=~"5.."}[5m])` for error rate

These metrics were then visualized in Grafana.

In Grafana, I built a dashboard with a clear visual hierarchy:
- At the top: overall service health — showing uptime percentage and SLA compliance
- Middle section: key SLO panels for availability, latency, and error budget burn rate
- Below that: trend graphs showing performance over the last 7 or 30 days
- Bottom panels: component health — like API, database, and cache metrics

I used color coding for clarity — green when we're comfortably within SLO, yellow when approaching the threshold, and red if the SLO is violated.

I also included a dedicated error budget panel that shows:
- Total budget for the period
- % of budget consumed
- Current burn rate (how quickly we're using the budget)
- Projected exhaustion date if the trend continues

Finally, I added context like the defined SLO targets, measurement windows (e.g., monthly), and links to incident runbooks.

This dashboard helped the team track reliability in real time, identify trends before incidents, and make data-driven release decisions — for example, pausing deployments if 75% of the monthly error budget was already consumed."

---

## Implementation Guide {#implementation}

### Step 1: Define Your SLIs

Start by identifying critical user journeys and the metrics that matter:

```
User Journey: Customer checks out and pays
Critical SLIs:
- Checkout API availability: (successful requests / total requests) * 100
- Checkout latency: % of requests < 3 seconds
- Payment processing success rate: (completed payments / initiated payments) * 100
```

### Step 2: Set Your SLOs

Based on SLIs, define realistic targets:

```
SLO: Checkout API
- Availability: 99.95% over 30 days
- Latency: 99% of requests under 3 seconds
- Error budget: 0.05% × 43,200 minutes = 21.6 minutes/month
```

### Step 3: Instrument Your Application

Add Prometheus metrics to your code:

**Python Example:**
```python
from prometheus_client import Counter, Histogram

request_count = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

request_duration = Histogram(
    'request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

@app.route('/checkout', methods=['POST'])
def checkout():
    with request_duration.labels('POST', '/checkout').time():
        try:
            # Process checkout
            request_count.labels('POST', '/checkout', '200').inc()
        except Exception as e:
            request_count.labels('POST', '/checkout', '500').inc()
```

**Go Example:**
```go
import "github.com/prometheus/client_golang/prometheus"

var (
    httpRequests = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    httpDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "request_duration_seconds",
            Help: "HTTP request duration",
        },
        []string{"method", "endpoint"},
    )
)
```

### Step 4: Configure Prometheus

Create a `prometheus.yml` configuration:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'checkout-api'
    static_configs:
      - targets: ['localhost:8000']
```

### Step 5: Build Grafana Dashboard

Create dashboard JSON for SLO tracking:

```json
{
  "dashboard": {
    "title": "Checkout Service SLO Dashboard",
    "panels": [
      {
        "title": "Availability %",
        "targets": [
          {
            "expr": "rate(http_requests_total{status!~\"5..\"}[5m]) * 100"
          }
        ]
      },
      {
        "title": "P95 Latency",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Error Budget Burn Rate",
        "targets": [
          {
            "expr": "(1 - (rate(http_requests_total{status!~\"5..\"}[5m]))) / 0.0005"
          }
        ]
      }
    ]
  }
}
```

### Step 6: Set Up Alerting

Define alerting rules in Prometheus:

```yaml
groups:
  - name: slo_alerts
    rules:
      - alert: AvailabilityBelowSLO
        expr: rate(http_requests_total{status!~"5.."}[5m]) < 0.9995
        for: 5m
        annotations:
          summary: "Availability dropped below 99.95% SLO"
      
      - alert: ErrorBudgetExhausted
        expr: rate(http_requests_total{status=~"5.."}[5m]) / 0.0005 > 0.75
        for: 1m
        annotations:
          summary: "Error budget 75% consumed - slow deployments"
```

### Step 7: Integrate with CI/CD

Automate deployment decisions based on error budget:

```bash
#!/bin/bash
# Check error budget before deployment

ERROR_BUDGET_CONSUMED=$(curl http://prometheus:9090/api/v1/query \
  --data-urlencode 'query=error_budget_consumed' | jq '.data.result[0].value[1]')

if (( $(echo "$ERROR_BUDGET_CONSUMED > 75" | bc -l) )); then
  echo "Error budget > 75% consumed - BLOCKING deployment"
  exit 1
else
  echo "Error budget OK - proceeding with deployment"
  exit 0
fi
```

### Step 8: Review and Adjust

Schedule monthly reviews:
- Did we meet our SLOs?
- Did the SLO targets make sense?
- What incidents consumed the error budget?
- Should we adjust targets for next month?

---

## Key Takeaways

1. **SLIs are measurements** of what users actually experience (availability, latency, errors)
2. **SLOs are targets** based on SLIs that balance user happiness, business goals, and engineering feasibility
3. **Error budgets** quantify how much unreliability you can afford, enabling data-driven decisions about deployments and feature velocity
4. **Dashboards make SLOs actionable** by visualizing key metrics, trends, and burn rates in real time
5. **Together, they create a culture** where reliability is measurable, understandable, and prioritized alongside feature development

---

## Quick Reference: Common SLO Values by Industry

| Service Type | Typical Availability SLO | Typical Latency SLO | Rationale |
|--------------|--------------------------|---------------------|-----------|
| **Payment Processing** | 99.99% | 99% < 2s | Critical for revenue; failures cost money |
| **E-commerce Checkout** | 99.95% | 95% < 3s | Revenue-impacting but slightly more forgiving |
| **Search/Discovery** | 99.9% | 90% < 500ms | User can retry; not revenue-critical |
| **Background Processing** | 99% | 99% < 5m | User not waiting; some delay acceptable |
| **Internal Tools** | 95% | 90% < 2s | Lower priority; some downtime acceptable |

---

## Glossary

- **Availability:** % of time the service is reachable and returning successful responses
- **Burn Rate:** Speed at which error budget is being consumed
- **Error Budget:** Maximum allowable unreliability before SLO is violated
- **Latency:** Time taken for requests to complete (measured as percentiles like p50, p95, p99)
- **Observability:** Ability to understand internal system behavior through logs, traces, and metrics
- **SLI:** Service Level Indicator — what you measure
- **SLO:** Service Level Objective — what you target
- **SLA:** Service Level Agreement — what you promise to customers
- **Saturation:** Level of resource utilization (CPU, memory, disk, etc.)
- **Throughput:** Amount of work completed per unit time

---

## References & Further Reading

- Google's SRE Book: Site Reliability Engineering
- Google's SLO Documentation
- Prometheus Documentation: https://prometheus.io/docs/
- Grafana Dashboard Documentation: https://grafana.com/docs/grafana/latest/dashboards/
- Observability Best Practices by O'Reilly

---

**Last Updated:** October 22, 2025
**For Questions or Feedback:** Refer to your SRE team or documentation repository