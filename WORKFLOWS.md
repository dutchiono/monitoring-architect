# Monitoring Architect - Operational Workflows

## Workflow Overview

This document outlines operational workflows for managing, maintaining, and responding to monitoring systems. Each workflow includes step-by-step procedures, escalation paths, and best practices.

## Daily Operations Workflows

### 1. Morning Health Check Workflow

**Schedule**: Every day at 9:00 AM

**Workflow Steps**:
```
1. Check Monitoring System Health
   ├─> Verify Prometheus scrape targets (all UP)
   ├─> Check Loki ingestion rate (no drops)
   ├─> Verify Grafana dashboard load times (< 2s)
   └─> Check Alertmanager status (no silenced alerts)

2. Review Active Alerts
   ├─> List all firing alerts by severity
   ├─> Verify alert ownership and assignments
   ├─> Check for alert fatigue (repeated alerts)
   └─> Update incident tracking

3. Check SLO Compliance
   ├─> Review error budget burn rate
   ├─> Check availability SLO (target: 99.9%)
   ├─> Verify latency SLO (p99 < 500ms)
   └─> Document any SLO violations

4. System Capacity Review
   ├─> Prometheus storage usage (target: < 80%)
   ├─> Loki storage usage (target: < 80%)
   ├─> CPU/Memory usage of monitoring stack
   └─> Forecast capacity needs (next 30 days)

5. Generate Daily Report
   └─> Email summary to team
```

**Dashboard**: `Monitoring Health Overview`

**Expected Duration**: 15-20 minutes

---

### 2. Alert Triage Workflow

**Trigger**: New alert received

**Workflow Steps**:
```
1. Alert Reception (< 1 minute)
   ├─> Acknowledge alert in PagerDuty/Slack
   ├─> Check if alert is duplicate or known issue
   └─> Assign severity level

2. Initial Assessment (< 5 minutes)
   ├─> Review alert description and labels
   ├─> Check related dashboards
   ├─> Query logs for error context
   ├─> Check recent deployments
   └─> Determine if customer-impacting

3. Investigation (< 15 minutes)
   ├─> Gather metrics, logs, traces
   ├─> Identify affected services/components
   ├─> Check dependency status
   ├─> Review historical patterns
   └─> Form initial hypothesis

4. Decision Point
   ├─> Option A: Immediate action required → Incident Response
   ├─> Option B: Known issue → Apply fix
   ├─> Option C: False positive → Tune alert
   └─> Option D: Requires further investigation → Create ticket

5. Resolution
   ├─> Document findings
   ├─> Update runbook if needed
   └─> Close or escalate alert
```

**Alert Severity Matrix**:
- **Critical**: Customer-impacting, immediate response required
- **High**: Potential customer impact, respond within 30 min
- **Medium**: Internal impact, respond within 2 hours
- **Low**: Informational, respond within 1 day

---

## Incident Response Workflows

### 3. Incident Response Workflow

**Trigger**: Critical or high-severity alert

**Workflow Steps**:
```
1. Incident Declaration (< 2 minutes)
   ├─> Create incident channel (#incident-YYYY-MM-DD-XXX)
   ├─> Page on-call engineer
   ├─> Update status page (if customer-impacting)
   └─> Start incident timeline

2. Assemble Response Team (< 5 minutes)
   ├─> Incident Commander (coordinates response)
   ├─> Technical Lead (drives resolution)
   ├─> Communications Lead (customer updates)
   └─> Subject Matter Experts (as needed)

3. Assessment & Diagnosis (< 15 minutes)
   ├─> Review relevant dashboards
   ├─> Query logs for error patterns
   ├─> Trace requests to identify bottleneck
   ├─> Check infrastructure status
   ├─> Review recent changes (deployments, config)
   └─> Form hypothesis

4. Mitigation (< 30 minutes)
   ├─> Option A: Rollback recent deployment
   ├─> Option B: Scale up resources
   ├─> Option C: Restart affected services
   ├─> Option D: Fail over to backup region
   ├─> Option E: Enable maintenance mode
   └─> Verify mitigation effectiveness

5. Monitoring & Validation (< 10 minutes)
   ├─> Verify error rate normalized
   ├─> Check latency returned to baseline
   ├─> Test critical user flows
   ├─> Monitor for recurring issues
   └─> Confirm customer impact resolved

6. Communication
   ├─> Update status page with resolution
   ├─> Post internal update
   ├─> Notify customer support team
   └─> Schedule postmortem

7. Post-Incident (within 48 hours)
   ├─> Write incident report
   ├─> Conduct blameless postmortem
   ├─> Identify action items
   ├─> Update runbooks and alerts
   └─> Track remediation work
```

**SLA Targets**:
- Detection: < 1 minute
- Acknowledgment: < 2 minutes
- Mitigation: < 30 minutes
- Full Resolution: < 2 hours

---

### 4. Alert Rule Creation Workflow

**Trigger**: Need for new monitoring coverage

**Workflow Steps**:
```
1. Define Alert Requirements
   ├─> What condition triggers the alert?
   ├─> What is the business impact?
   ├─> What is the appropriate severity?
   ├─> Who should be notified?
   └─> What action should responders take?

2. Write Alert Rule
   PromQL Example:
   ```yaml
   - alert: HighErrorRate
     expr: |
       sum(rate(http_requests_total{status=~"5.."}[5m])) 
       / sum(rate(http_requests_total[5m])) > 0.05
     for: 5m
     labels:
       severity: critical
       team: backend
       runbook: https://wiki.example.com/runbooks/high-error-rate
     annotations:
       summary: "High error rate detected"
       description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
   ```

3. Test Alert Rule
   ├─> Deploy to staging environment
   ├─> Trigger alert condition (chaos testing)
   ├─> Verify alert fires correctly
   ├─> Check notification delivery
   └─> Validate runbook instructions

4. Review & Approve
   ├─> Peer review alert logic
   ├─> Check for alert overlap/duplication
   ├─> Verify severity and routing
   └─> Approve for production

5. Deploy to Production
   ├─> Merge alert rule to main branch
   ├─> Apply via GitOps (Flux/ArgoCD)
   ├─> Verify alert appears in Prometheus
   └─> Document in alert catalog

6. Monitor Alert Performance
   ├─> Track alert frequency (first 7 days)
   ├─> Measure time-to-detect
   ├─> Gather feedback from responders
   └─> Tune thresholds if needed
```

---

### 5. Dashboard Creation Workflow

**Trigger**: Need for new observability view

**Workflow Steps**:
```
1. Define Dashboard Purpose
   ├─> Who is the audience? (SRE, Dev, Business)
   ├─> What questions does it answer?
   ├─> What time range is relevant?
   └─> What actions will users take?

2. Identify Key Metrics
   ├─> Primary metrics (red/yellow/green status)
   ├─> Supporting metrics (context and drill-down)
   ├─> Correlation metrics (related systems)
   └─> Historical trends

3. Build Dashboard
   ├─> Create dashboard in Grafana
   ├─> Add panels with appropriate visualizations
   ├─> Configure variables for filtering
   ├─> Set appropriate time ranges
   ├─> Add annotations for deployments/incidents
   └─> Write panel descriptions

4. Review & Refine
   ├─> Share draft with stakeholders
   ├─> Gather feedback on usefulness
   ├─> Optimize query performance
   ├─> Ensure mobile responsiveness
   └─> Add links to related dashboards/runbooks

5. Deploy & Document
   ├─> Export dashboard JSON
   ├─> Store in Git repository
   ├─> Deploy via provisioning or API
   ├─> Add to dashboard catalog
   └─> Train team on usage

6. Maintain Dashboard
   ├─> Review usage monthly (view count)
   ├─> Update based on feedback
   ├─> Archive unused dashboards
   └─> Keep queries optimized
```

**Dashboard Best Practices**:
- Use templating variables for flexibility
- Show current value and trend
- Include drill-down links
- Add contextual help text
- Optimize query performance (< 2s load time)

---

## Maintenance Workflows

### 6. Prometheus Storage Maintenance

**Schedule**: Weekly

**Workflow Steps**:
```
1. Check Storage Usage
   ├─> Current usage: prometheus_tsdb_storage_size_bytes
   ├─> Growth rate: delta over last 7 days
   └─> Projected full date

2. Clean Up Old Data (if needed)
   ├─> Verify retention policy (default: 15 days)
   ├─> Manually delete old blocks (if urgent)
   └─> Restart Prometheus to reclaim space

3. Optimize Storage
   ├─> Run compaction manually if needed
   ├─> Check for high-cardinality metrics
   └─> Remove unused metrics

4. Verify Thanos Backup
   ├─> Check Thanos sidecar upload status
   ├─> Verify blocks in object storage
   └─> Test restore procedure (monthly)
```

**Prometheus Retention Configuration**:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
spec:
  retention: 15d
  retentionSize: 50GB
```

---

### 7. Log Retention & Cleanup

**Schedule**: Daily

**Workflow Steps**:
```
1. Check Loki Storage
   ├─> Storage usage by tenant
   ├─> Ingestion rate trends
   └─> Query performance

2. Apply Retention Policies
   ├─> Delete logs older than retention period
   ├─> Compress old log chunks
   └─> Archive critical logs to long-term storage

3. Optimize Queries
   ├─> Identify slow queries
   ├─> Add indexes for common filters
   └─> Update LogQL best practices docs
```

**Loki Retention Configuration**:
```yaml
limits_config:
  retention_period: 30d
  
table_manager:
  retention_deletes_enabled: true
  retention_period: 30d
```

---

### 8. Alert Tuning Workflow

**Trigger**: Alert fatigue or false positives

**Workflow Steps**:
```
1. Identify Problematic Alerts
   ├─> Alerts firing too frequently (> 5/day)
   ├─> Alerts with high false positive rate (> 30%)
   ├─> Alerts not actionable
   └─> Duplicate or overlapping alerts

2. Analyze Alert History
   ├─> Review last 30 days of alert data
   ├─> Calculate signal-to-noise ratio
   ├─> Interview responders for feedback
   └─> Check if alert led to action

3. Tune Alert Parameters
   ├─> Adjust threshold values
   ├─> Increase 'for' duration (reduce flapping)
   ├─> Add additional conditions
   ├─> Change alert severity
   └─> Update notification routing

4. Test Tuned Alert
   ├─> Deploy to staging
   ├─> Monitor for 1 week
   ├─> Verify reduced noise
   └─> Ensure real issues still detected

5. Document Changes
   ├─> Update alert documentation
   ├─> Explain tuning rationale
   └─> Track improvement metrics
```

**Alert Tuning Example**:
```yaml
# Before: Alert fires every 30s when CPU spikes briefly
- alert: HighCPU
  expr: node_cpu_usage > 80
  for: 1m

# After: Alert only fires for sustained high CPU
- alert: HighCPU
  expr: avg_over_time(node_cpu_usage[5m]) > 85
  for: 10m
```

---

## SLO Management Workflows

### 9. SLO Definition Workflow

**Workflow Steps**:
```
1. Define SLO
   ├─> Select SLI (availability, latency, error rate)
   ├─> Set target (e.g., 99.9% availability)
   ├─> Define measurement window (30 days)
   └─> Calculate error budget

2. Implement SLI Measurement
   ├─> Write PromQL query for SLI
   ├─> Create recording rules
   ├─> Set up alerting on burn rate
   └─> Build SLO dashboard

3. Monitor & Report
   ├─> Track SLO compliance daily
   ├─> Calculate remaining error budget
   ├─> Alert when burn rate too high
   └─> Generate monthly SLO reports

4. Review & Adjust
   ├─> Quarterly SLO review meeting
   ├─> Adjust targets based on business needs
   ├─> Update error budget policies
   └─> Document changes
```

**SLO Example**:
```yaml
# Availability SLO: 99.9% (30-day window)
- record: service:availability:30d
  expr: |
    avg_over_time(up[30d]) * 100

# Error Budget: 0.1% (43 minutes of downtime)
- alert: HighErrorBudgetBurn
  expr: |
    (1 - avg_over_time(up[1h])) > (0.001 * 14.4)  # 14.4x burn rate
  annotations:
    description: "Error budget burning 14.4x faster than sustainable rate"
```

---

## Automation Workflows

### 10. Automated Remediation Workflow

**Trigger**: Specific alert patterns

**Workflow Steps**:
```
1. Alert Detection
   ├─> Alertmanager receives alert
   ├─> Webhook sent to automation service
   └─> Alert evaluated for auto-remediation

2. Safety Checks
   ├─> Verify alert is known and safe to auto-fix
   ├─> Check if remediation already in progress
   ├─> Validate current system state
   └─> Confirm no manual intervention active

3. Execute Remediation
   ├─> Option A: Restart service
   ├─> Option B: Scale up resources
   ├─> Option C: Clear cache
   ├─> Option D: Run cleanup script
   └─> Option E: Fail over to standby

4. Validation
   ├─> Wait for service recovery
   ├─> Verify metrics returned to normal
   ├─> Check for side effects
   └─> Confirm alert resolved

5. Notification
   ├─> Post to incident channel
   ├─> Document automated action
   └─> Alert team if remediation failed
```

**Automation Example (Kubernetes CronJob)**:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: auto-restart-on-oom
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: auto-restart
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              # Get pods with OOMKilled status
              kubectl get pods --all-namespaces \
                -o json | jq -r '.items[] | 
                select(.status.containerStatuses[]?.lastState.terminated.reason=="OOMKilled") |
                "\(.metadata.namespace) \(.metadata.name)"' |
              while read namespace pod; do
                echo "Restarting OOMKilled pod: $namespace/$pod"
                kubectl delete pod -n $namespace $pod
              done
```

---

## Reporting Workflows

### 11. Weekly Monitoring Report

**Schedule**: Every Monday at 10:00 AM

**Report Sections**:
```
1. Executive Summary
   ├─> Overall system health (Green/Yellow/Red)
   ├─> Key metrics vs. last week
   ├─> Major incidents (if any)
   └─> Action items

2. Availability & Performance
   ├─> Uptime percentage
   ├─> P50/P95/P99 latencies
   ├─> Error rates by service
   └─> SLO compliance status

3. Alert Analysis
   ├─> Total alerts fired
   ├─> Alerts by severity
   ├─> Mean time to acknowledge (MTTA)
   ├─> Mean time to resolve (MTTR)
   └─> Alert trends

4. Capacity Planning
   ├─> Resource utilization trends
   ├─> Growth projections
   ├─> Scaling recommendations
   └─> Cost optimization opportunities

5. Action Items
   ├─> Alert rules to tune
   ├─> Dashboards to create
   ├─> Monitoring gaps to fill
   └─> Training needs
```

**Automation**:
```python
#!/usr/bin/env python3
# generate_weekly_report.py

import requests
from datetime import datetime, timedelta

PROMETHEUS_URL = "http://prometheus:9090"

def query_prometheus(query, start, end):
    response = requests.get(
        f"{PROMETHEUS_URL}/api/v1/query_range",
        params={"query": query, "start": start, "end": end, "step": "1h"}
    )
    return response.json()

def generate_report():
    end = datetime.now()
    start = end - timedelta(days=7)
    
    # Query metrics
    uptime = query_prometheus("avg_over_time(up[7d]) * 100", start, end)
    latency_p99 = query_prometheus("histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[7d]))", start, end)
    error_rate = query_prometheus("rate(http_requests_total{status=~'5..'}[7d])", start, end)
    
    # Generate report
    report = f"""
    Weekly Monitoring Report
    Week of {start.strftime('%Y-%m-%d')}
    
    Availability: {uptime:.2f}%
    P99 Latency: {latency_p99:.0f}ms
    Error Rate: {error_rate:.2f}%
    """
    
    return report

if __name__ == "__main__":
    print(generate_report())
```

---

## Best Practices

### Monitoring Workflow Best Practices

1. **Automate Repetitive Tasks**: Use scripts and tools for routine checks
2. **Document Everything**: Maintain runbooks for all alert types
3. **Practice Incident Response**: Regular fire drills and game days
4. **Review and Improve**: Continuous improvement of alerts and dashboards
5. **Monitor the Monitors**: Meta-monitoring of the monitoring stack
6. **Keep It Simple**: Avoid over-complication in alerts and dashboards
7. **Train the Team**: Regular training on monitoring tools and procedures

### Alert Workflow Checklist

- [ ] Alert has clear description
- [ ] Alert has appropriate severity
- [ ] Alert has runbook link
- [ ] Alert has been tested
- [ ] Alert routing configured
- [ ] Alert has owner/team
- [ ] Alert threshold tuned
- [ ] Alert false positive rate < 20%

### Dashboard Workflow Checklist

- [ ] Dashboard has clear purpose
- [ ] Dashboard audience identified
- [ ] Key metrics defined
- [ ] Visualizations appropriate
- [ ] Queries optimized
- [ ] Variables used for flexibility
- [ ] Documentation added
- [ ] Team trained on usage

---

## Emergency Procedures

### Critical System Failure

**If Prometheus is down**:
1. Check alternative monitoring (CloudWatch, Datadog)
2. Restart Prometheus pods
3. Verify storage is available
4. Check for OOM or crash loops
5. Restore from backup if needed

**If Grafana is down**:
1. Use Prometheus UI directly
2. Query via API or CLI
3. Restart Grafana
4. Check database connectivity
5. Verify configuration

**If Loki is down**:
1. Use kubectl logs directly
2. Restart Loki components
3. Check object storage connectivity
4. Verify ingester memory
5. Restore from backup

---

## Workflow Metrics

Track these metrics for workflow effectiveness:

- **MTTA** (Mean Time to Acknowledge): < 5 minutes
- **MTTR** (Mean Time to Resolve): < 30 minutes
- **Alert Accuracy**: > 80% actionable
- **Dashboard Usage**: Views per week
- **Incident Postmortem Completion**: Within 48 hours
- **Runbook Coverage**: > 90% of alerts
