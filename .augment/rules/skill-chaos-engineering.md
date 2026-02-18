---
type: agent_requested
description: Design and run chaos experiments, inject faults with controlled blast radius, define steady state, and plan game days to improve production system resilience.
---

# Chaos Engineering

## When to Use
- Designing controlled failure injection experiments
- Planning and running game days
- Defining blast radius and safety controls before testing
- Building system resilience by proactively finding weaknesses
- Integrating chaos testing into CI/CD pipelines
- Validating incident response and monitoring effectiveness

---

## Experiment Design: Hypothesis → Injection → Observation → Analysis

### Step 1: Define Steady State
Before injecting anything, establish what "normal" looks like:
```yaml
steady_state:
  metrics:
    - name: Error Rate
      threshold: "< 0.1%"
      query: "rate(http_requests_total{status=~'5..'}[5m])"
    - name: Latency P99
      threshold: "< 500ms"
    - name: Active DB Connections
      threshold: "> 10"
```
**Rule**: abort the experiment if steady state is not met at start.

### Step 2: Write the Hypothesis
Format: *"Given [normal state], when [failure occurs], then [expected behavior], measured by [metrics]."*

Example:
```
Given: payment-api handles 500 RPS with <0.1% errors
When: 50% of API pods are killed
Then: error rate stays <5% and latency p99 < 1s (load balancer re-routes)
Measured by: Prometheus error_rate, latency_p99 dashboards
```

### Step 3: Control Blast Radius
```python
# Progressive rollout — never start in production
stages = [
    { "env": "dev",        "percentage": 100, "duration": "5 min"  },
    { "env": "staging",    "percentage": 100, "duration": "10 min" },
    { "env": "production", "percentage": 1,   "duration": "5 min"  },  # canary
    { "env": "production", "percentage": 10,  "duration": "10 min" },  # widen
]
```

**Hard limits for production**:
- `target_percentage > 10%` requires feature flag AND auto-rollback enabled
- `max_duration > 10 min` requires explicit VP approval
- Auto-rollback must trigger within 30 seconds

### Step 4: Execute & Monitor
- Monitor continuously during the experiment (never walk away)
- Abort immediately if error rate exceeds threshold or manual kill switch activated
- Log all observations with timestamps

### Step 5: Analyze & Improve
- Did the hypothesis hold? Document why or why not
- File action items for any gaps found
- Update runbooks and monitoring based on learnings

---

## Fault Types Catalog

| Category | Fault Type | Method | Tool |
|----------|-----------|--------|------|
| **Compute** | Pod/process kill | `kubectl delete pod` | Chaos Monkey, Litmus |
| **Compute** | CPU throttle | `stress-ng --cpu` | Gremlin |
| **Network** | Latency injection | `tc netem delay 200ms` | toxiproxy, Pumba |
| **Network** | Packet loss | `tc netem loss 30%` | toxiproxy |
| **Network** | Network partition | `iptables -A OUTPUT -d <svc> -j DROP` | Chaos Mesh |
| **Resource** | Memory pressure | `stress-ng --vm` | Gremlin |
| **Resource** | Connection pool exhaust | Open N connections, don't close | custom script |
| **Dependency** | Downstream timeout | Delay responses via proxy | toxiproxy |
| **Dependency** | Dependency unavailable | Block traffic to service | iptables |
| **Infrastructure** | AZ/region failure | Terminate all instances in AZ | AWS FIS |

---

## Blast Radius Control Checklist

```
Before running any experiment:
☐ Steady state defined and verified
☐ Hypothesis written and reviewed by team
☐ Start in dev/staging, not production
☐ Feature flag / traffic percentage set (≤ 1% for first prod run)
☐ Auto-rollback configured with ≤ 30s trigger time
☐ Kill switch / abort plan documented and tested
☐ Team notified (Slack, war room link ready)
☐ Customer support aware (for any prod experiments)
☐ Max experiment duration set (default ≤ 10 min)
☐ Monitoring dashboards open during experiment
```

**Abort immediately if**:
- Error rate exceeds `max_error_rate` threshold
- Latency p99 exceeds SLO × 2
- Any unintended service affected
- Manual kill switch activated

---

## Game Day Planning Template

```yaml
game_day:
  name: "Database Failover Drill"
  date: "YYYY-MM-DD"
  duration: "2 hours"
  environment: "staging"      # Always start in staging

  objectives:
    - Verify RDS failover completes in < 2 minutes
    - Validate application auto-reconnect logic
    - Test monitoring and alerting accuracy
    - Practice incident response communication

  participants:
    facilitator: "chaos-lead"
    observers: ["sre-team", "dev-team"]
    responders: ["on-call-engineer", "dba"]
    stakeholders: ["eng-manager"]

  scenarios:
    - name: "Primary DB failure"
      inject: "Force RDS reboot with failover"
      success_criteria: ["Failover < 2 min", "No data loss", "Alerts fired"]
    - name: "Network partition to DB"
      inject: "Block traffic to RDS security group"
      success_criteria: ["Read-only mode activated", "Clear user error messages"]
    - name: "Surprise scenario"      # Don't reveal in advance
      inject: "Facilitator decides day-of (cascading failure)"
      success_criteria: ["Root cause identified < 15 min"]

  safety:
    rollback_triggers: ["error_rate > 10%", "production traffic affected", "manual abort"]
    rollback_time_limit_seconds: 60
    abort_contacts: ["VP Engineering phone"]

  post_game:
    postmortem_scheduled: "next day, 14:00"
    report_includes: ["timeline", "metrics dashboard export", "action items"]
```

**Game Day Phases**:
| Phase | Duration | Activity |
|-------|----------|----------|
| Pre-game | 30 min | Verify environment, brief participants, test rollback |
| Execution | 90 min | Run scenarios, observe, document every action + timestamp |
| Debrief | 30 min | Immediate learnings, top 3 action items |
| Post-mortem | 1 week later | Detailed analysis, verify improvements |

**Debrief questions**: What went well? What surprised us? What would we do differently? What are our top 3 action items?

---

## Must Do / Must Not Do

✅ Define steady state and hypothesis BEFORE injecting anything  
✅ Start in dev/staging, expand to production gradually  
✅ Enable auto-rollback with ≤ 30s trigger time  
✅ Monitor continuously during all experiments  
✅ Document all learnings and create action items  
✅ Share findings broadly — the whole org benefits  
❌ Run experiments without a written hypothesis  
❌ Skip blast radius controls (cost: real customer impact)  
❌ Test multiple failure variables simultaneously (isolate to one)  
❌ Run in production without prior staging validation  
❌ Leave systems in degraded state after experiment  
❌ Skip team communication before/during experiments  

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/chaos-engineer

