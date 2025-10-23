# Case Study: Preventing a Black Friday Disaster Through Proactive Capacity Planning

### For "Tell me about a time you prevented a major incident"
**Focus on**: The proactive planning aspects, historical analysis, and how you identified risks before they became problems.

**Key points to emphasize**:
- Previous year's Black Friday failure provided clear context
- Comprehensive monitoring revealed the 72% disk usage issue weeks in advance
- Load testing discovered the memory leak before production impact
- Result: Zero capacity-related outages during peak event

### For "Describe a situation where you had to balance competing priorities"
**Focus on**: Cost vs. reliability trade-offs and strategic decision-making.

**Key points to emphasize**:
- Could have over-provisioned all month (35% cost increase) but chose hybrid approach
- Balanced proactive pre-provisioning with reactive autoscaling
- Final result: Only 12% cost increase while achieving 100% reliability
- Demonstrated business acumen alongside technical expertise

### For "Give an example of data-driven decision making"
**Focus on**: Historical pattern analysis and metrics-based forecasting.

**Key points to emphasize**:
- Analyzed 12 months of data to identify seasonal pattern with linear baseline
- Used specific metrics (5% monthly growth, 15-20x traffic spikes) to drive decisions
- Validated assumptions through load testing
- Forecasting accuracy within 5% in subsequent quarters

### For "Tell me about a complex technical project you led"
**Focus on**: The complete 11-week timeline and cross-functional coordination.

**Key points to emphasize**:
- Structured approach: monitoring → analysis → planning → testing → execution
- Coordinated with multiple teams (engineering, business, vendors)
- Created reusable playbook adopted by other teams
- Demonstrated project management and technical leadership

### For "How do you handle pressure or unexpected challenges?"
**Focus on**: The load testing discovery and rapid problem-solving.

**Key points to emphasize**:
- Memory leak discovered during week 9 load testing (tight timeline)
- Quickly diagnosed root cause and implemented fix
- API rate limit issue required vendor negotiation under deadline
- Remained calm, systematic, and solution-oriented

### For "Describe your monitoring or observability philosophy"
**Focus on**: Comprehensive monitoring strategy and actionable thresholds.

**Key points to emphasize**:
- "Without visibility, there's no capacity to plan" - monitored all resource types
- Graduated thresholds (warning at 70%, critical at 85%) provided early signals
- Linked thresholds to automated actions (not just alerts)
- Created real-time dashboards for operational awareness

---

## Situation
I was working as an SRE on an e-commerce platform that had experienced a significant outage during the previous Black Friday due to insufficient capacity. The site went down for 3 hours during our highest traffic period, resulting in substantial revenue loss and customer trust issues. Leadership made it clear this couldn't happen again.

## Task
I was tasked with leading the capacity planning initiative to ensure our systems could handle the upcoming holiday season. My goals were to:
- Prevent any capacity-related outages during Black Friday and Cyber Monday
- Optimize costs by avoiding over-provisioning during normal periods
- Build a sustainable, repeatable process for future peak events

## Action

### 1. Established Comprehensive Monitoring (Weeks 1-2)
I started by implementing monitoring across all critical resource types:
- **Compute**: CPU utilization and queue lengths across our FastAPI services
- **Memory**: RAM usage patterns, especially during traffic spikes
- **Storage**: Database disk space (PostgreSQL) and log storage (Loki)
- **Network**: Bandwidth usage and connection counts
- **Database**: Query throughput and connection pool utilization
- **API Limits**: External payment gateway rate limits

I integrated everything into Prometheus and created Grafana dashboards for real-time visibility.

### 2. Analyzed Historical Patterns (Weeks 3-4)
I analyzed 12 months of historical data and identified our growth pattern as **seasonal with a linear baseline**:
- Steady 5-8% monthly growth during normal periods
- 15-20x traffic spikes during major sales events
- Weekend peaks every week (2-3x normal traffic)

Key finding: Our Postgres database was at 72% capacity with 5% monthly growth, meaning we'd hit critical levels before Black Friday without intervention.

### 3. Set Graduated Thresholds (Week 5)
I established warning and critical thresholds for each resource:

| Resource | Warning (70-80%) | Critical (85-90%) | Automated Action |
|----------|------------------|-------------------|------------------|
| CPU | 70% | 85% | Scale out pods (HPA) |
| Memory | 80% | 90% | Add memory capacity |
| Database Connections | 70% | 85% | Increase pool size |
| Disk Space | 70% | 85% | Provision more storage |

These thresholds triggered automated alerts and, where possible, automated scaling responses.

### 4. Implemented Hybrid Strategy (Weeks 6-8)
I designed a two-pronged approach:

**Proactive:**
- Pre-provisioned 3x baseline capacity for Black Friday weekend
- Scheduled database storage expansion 4 weeks before the event
- Increased connection pool limits ahead of time

**Reactive:**
- Configured Kubernetes Horizontal Pod Autoscaler (HPA) with:
  - Min replicas: 10 (vs. 3 normally)
  - Max replicas: 50 (vs. 10 normally)
  - CPU trigger: 70%
- Set up auto-scaling for our Celery task queue workers (3x-15x capacity)

### 5. Load Testing & Validation (Weeks 9-10)
I conducted three rounds of load testing:
- Simulated 20x normal traffic
- Tested autoscaling response times
- Validated database connection pool behavior under load
- Identified and fixed a memory leak that only appeared at 10x+ load

Discovered our API gateway would hit rate limits at 18x traffic, so I negotiated increased limits with the vendor.

### 6. Created Runbooks & Monitoring (Week 11)
I documented:
- Capacity escalation procedures
- Manual scaling commands if automation failed
- Rollback procedures
- Real-time monitoring dashboard for the on-call team

## Result

### Black Friday Performance:
- **Zero capacity-related outages** during the entire holiday weekend
- Peak traffic reached 22x baseline (higher than forecasted)
- Autoscaling responded within 90 seconds to traffic spikes
- All thresholds held below critical levels
- Average response time remained under 200ms (vs. 2+ seconds during previous year's failures)

### Business Impact:
- **$2.3M additional revenue** compared to previous Black Friday (no downtime)
- Customer satisfaction scores improved by 40%
- Cloud costs increased only 12% overall (vs. 35% if we'd over-provisioned for peak all month)

### Long-term Improvements:
- Established quarterly capacity review meetings
- Built forecasting models that predicted Q1 growth within 5% accuracy
- Created a capacity planning playbook adopted by other teams
- Reduced on-call incidents related to capacity by 75%

## Key Learnings

1. **Balance proactive and reactive**: Pre-provisioning for known events combined with autoscaling for unexpected spikes gave us the best coverage.

2. **Monitor everything**: The memory leak we discovered during load testing would have caused failures under real traffic. You can't plan for what you can't see.

3. **Graduated thresholds matter**: Having warning thresholds at 70% gave us time to investigate before hitting critical 85% levels, preventing several would-be incidents.

4. **Load testing is non-negotiable**: Our assumptions were wrong about several components. Testing revealed these issues when we still had time to fix them.

5. **Document and automate**: Under pressure during Black Friday, the team executed flawlessly because everything was documented and most responses were automated.

---

## Interview Adaptation Guide

### For "Tell me about a time you prevented a major incident"
**Focus on**: The proactive planning aspects, historical analysis, and how you identified risks before they became problems.

**Key points to emphasize**:
- Previous year's Black Friday failure provided clear context
- Comprehensive monitoring revealed the 72% disk usage issue weeks in advance
- Load testing discovered the memory leak before production impact
- Result: Zero capacity-related outages during peak event

### For "Describe a situation where you had to balance competing priorities"
**Focus on**: Cost vs. reliability trade-offs and strategic decision-making.

**Key points to emphasize**:
- Could have over-provisioned all month (35% cost increase) but chose hybrid approach
- Balanced proactive pre-provisioning with reactive autoscaling
- Final result: Only 12% cost increase while achieving 100% reliability
- Demonstrated business acumen alongside technical expertise

### For "Give an example of data-driven decision making"
**Focus on**: Historical pattern analysis and metrics-based forecasting.

**Key points to emphasize**:
- Analyzed 12 months of data to identify seasonal pattern with linear baseline
- Used specific metrics (5% monthly growth, 15-20x traffic spikes) to drive decisions
- Validated assumptions through load testing
- Forecasting accuracy within 5% in subsequent quarters

### For "Tell me about a complex technical project you led"
**Focus on**: The complete 11-week timeline and cross-functional coordination.

**Key points to emphasize**:
- Structured approach: monitoring → analysis → planning → testing → execution
- Coordinated with multiple teams (engineering, business, vendors)
- Created reusable playbook adopted by other teams
- Demonstrated project management and technical leadership

### For "How do you handle pressure or unexpected challenges?"
**Focus on**: The load testing discovery and rapid problem-solving.

**Key points to emphasize**:
- Memory leak discovered during week 9 load testing (tight timeline)
- Quickly diagnosed root cause and implemented fix
- API rate limit issue required vendor negotiation under deadline
- Remained calm, systematic, and solution-oriented

### For "Describe your monitoring or observability philosophy"
**Focus on**: Comprehensive monitoring strategy and actionable thresholds.

**Key points to emphasize**:
- "Without visibility, there's no capacity to plan" - monitored all resource types
- Graduated thresholds (warning at 70%, critical at 85%) provided early signals
- Linked thresholds to automated actions (not just alerts)
- Created real-time dashboards for operational awareness

---

## Delivery Tips

**Timing**:
- **2-minute version**: Situation + 2-3 key actions + results
- **5-minute version**: Full STAR with some technical details
- **10-minute version**: Deep dive with trade-offs and learnings

**Memorize These Numbers**:
- 72% disk usage, 5% monthly growth
- 15-20x traffic spikes during sales
- 22x actual peak traffic on Black Friday
- $2.3M additional revenue
- 75% reduction in capacity incidents
- 12% cost increase (vs 35% alternative)

**Be Ready for Follow-ups**:
- "What would you do differently?" → Wish we'd started earlier, would've automated more
- "What challenges did you face?" → Initial resistance to increased costs, vendor negotiations
- "How did you convince leadership?" → Showed revenue impact of previous failure vs. cost of prevention
- "What if autoscaling failed?" → Had manual runbooks and practiced procedures

**Adjust Technical Depth**:
- **For technical interviewers**: Include HPA configs, specific metrics, Prometheus/Grafana details
- **For non-technical interviewers**: Focus on business impact, decision-making process, leadership