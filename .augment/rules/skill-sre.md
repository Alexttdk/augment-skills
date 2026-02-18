---
type: agent_requested
description: Apply SRE practices: define SLIs/SLOs, manage error budgets, run incident response, write blameless postmortems, reduce toil, and plan capacity for production systems.
---

# SRE Engineer

## When to Use
- Defining or reviewing SLIs, SLOs, and error budgets
- Setting up reliability monitoring and burn-rate alerting
- Running incident response or writing postmortems
- Identifying and automating operational toil
- Capacity planning and scaling decisions
- Establishing on-call practices and runbooks

---

## SLI / SLO / SLA Definitions

| Term | Definition | Example |
|------|-----------|---------|
| **SLI** (Indicator) | Quantitative metric measuring behavior | 99.7% of requests returned 2xx in the last 30d |
| **SLO** (Objective) | Target threshold for an SLI | ≥ 99.9% availability over 30d rolling window |
| **SLA** (Agreement) | Legal/contractual commitment with consequences | "We guarantee 99.5%; credits if breached" |
| **Error Budget** | `1 − SLO target` — allowable unreliability | 0.1% = 43.2 min/month for 99.9% SLO |

**Picking SLIs**: focus on user-facing impact — availability (% successful requests), latency (% under threshold), throughput, correctness. 4xx errors are client errors; don't count against availability SLI.

**Common SLO tiers:**
```yaml
tier_1_critical:  availability: 99.99%  # 4m 23s/month downtime
tier_2_important: availability: 99.9%   # 43m 28s/month
tier_3_standard:  availability: 99.5%   # 3h 37m/month
```

---

## Error Budget Calculation & Policy

```
Error budget = 1 − SLO target
  → 99.9% SLO = 0.1% budget = 43.2 min/month

Remaining budget = (budget_total − budget_consumed) / budget_total
Burn rate = current_error_rate / total_error_budget
  → burn rate > 1.0 means exhausting budget faster than sustainable
```

**Multi-window burn rate alerts** (Google SRE Workbook):
| Window | Burn Rate Threshold | Budget Consumed | Action |
|--------|--------------------|-----------------|----|
| 1h     | 14.4×              | 2%              | Page now |
| 6h     | 6.0×               | 5%              | Page/warn |
| 3d     | 1.0×               | 10%             | Ticket |

**Error budget policy:**
| Budget Remaining | State | Actions |
|-----------------|-------|---------|
| > 75% | Normal | Full feature velocity, standard deploys |
| 25–75% | Careful | Enhanced code review, senior approval for risky deploys |
| < 25% | Restricted | Halt non-critical features, focus on reliability work |
| 0% | Feature Freeze | Emergency fixes only, exec review, mandatory postmortems |

Exceptions (security patches, critical business needs) require explicit VP + product lead approval.

---

## Incident Response Runbook Template

### Severity Levels
| SEV | Impact | Response |
|-----|--------|----------|
| SEV1 | Complete outage, major customer impact | All-hands war room, page VP, 15-min updates |
| SEV2 | Partial outage, significant impact | Team response, 30-min updates |
| SEV3 | Degraded performance, some users affected | On-call engineer, 1-hr updates |
| SEV4 | Minor issue, minimal impact | Ticket, next business day |

### Response Phases
```
1. DETECT    → Acknowledge alert, assess severity, join #incident channel
2. TRIAGE    → Identify blast radius, assign Incident Commander (IC) and Comms Lead
3. MITIGATE  → Apply fastest fix (rollback, feature flag, reroute traffic)
4. RESOLVE   → Verify metrics normalized, monitor 30 min, close incident
5. FOLLOW-UP → Schedule postmortem within 48h, create action items
```

### Roles
- **Incident Commander**: coordinates, delegates, makes decisions
- **Comms Lead**: status page updates, stakeholder notifications every 15-30 min
- **On-call Engineer**: investigation, fix implementation, verification

---

## Blameless Postmortem Template

```markdown
# Postmortem: [Incident Title]
Date: | Authors: | Severity: | Status: Draft/Complete

## Summary
One paragraph: what happened, impact, how resolved.

## Impact
- Duration: X minutes (HH:MM – HH:MM UTC)
- Users affected: ~N% of [feature/traffic]
- SLO impact: consumed X% of monthly error budget

## Timeline (UTC)
| Time  | Event |
| 14:30 | Deployment completed |
| 14:35 | Alert: HighErrorRate fires |
| 14:40 | Root cause identified |
| 14:50 | Rollback initiated |
| 15:00 | Metrics normalized |

## Root Cause
Technical description of why this happened.

## Detection
What triggered discovery? Time from start to detection?

## Resolution
What fixed it? Why did it work?

## What Went Well / What Didn't / Where We Got Lucky

## Action Items
| Action | Owner | Priority | Due Date |
|--------|-------|----------|----------|
| Add connection pool monitoring | @alice | P0 | YYYY-MM-DD |
```

**Postmortem principles**: no blame, focus on systems not people, share broadly, verify action items get completed.

---

## Toil Identification & Reduction Checklist

**Toil is work that is**: manual, repetitive, automatable, tactical (no enduring value), scales linearly with traffic.

Target: keep toil < 50% of any engineer's time.

```
Identify toil:
☐ What manual tasks does on-call run more than once a week?
☐ What runbook steps could be a script?
☐ What alerts require the same action every time?
☐ What deployments require human coordination vs. can be automated?
☐ What reports are generated manually?

Prioritize reduction by: frequency × time per occurrence × risk of human error

Reduction patterns:
☐ Auto-remediation scripts for common alert responses
☐ Self-service tooling so devs don't need SRE for routine ops
☐ Idempotent runbooks that can run as automation
☐ Eliminate alerts with <5% action rate (noise → fix or remove)
```

---

## Capacity Planning Framework

```
1. MEASURE baseline:
   - Current peak RPS, P99 latency, CPU/memory utilization
   - Resource headroom (% utilized vs provisioned)

2. FORECAST growth:
   - Historical trend (week-over-week, month-over-month)
   - Known upcoming events (product launches, campaigns)
   - Target: provision for 2× current peak (N+1 for node failures)

3. MODEL limits:
   - Load test to find saturation point per instance type
   - Identify first bottleneck (CPU? DB connections? memory?)
   - Calculate max RPS per node × planned node count

4. ACT:
   - Scale before hitting 60% sustained utilization
   - Use auto-scaling with conservative scale-in, aggressive scale-out
   - Review capacity quarterly or after major traffic events

5. ALERT on leading indicators:
   - CPU sustained > 70% for 15 min
   - DB connection pool > 80%
   - Disk projected full within 7 days
```

---

## Golden Signals (Monitor These on Every Service)

| Signal | What to measure | Alert threshold |
|--------|----------------|-----------------|
| **Latency** | p50, p95, p99 request duration | p99 > SLO threshold for 5 min |
| **Traffic** | Requests/sec (also messages, writes) | Sudden drop > 20% (could indicate issue) |
| **Errors** | % of requests returning 5xx | Error rate burn > 6× for 6h |
| **Saturation** | CPU, memory, connection pool utilization | > 80% sustained for 15 min |

---

## Must Do / Must Not Do

✅ Define SLOs with explicit user-impact justification  
✅ Write blameless postmortems for all SEV1/SEV2 incidents  
✅ Measure toil quarterly; have automation plan if > 50%  
✅ Balance reliability with feature velocity via error budgets  
✅ Test failure scenarios before they happen in production  
❌ Set SLOs without understanding what user impact actually is  
❌ Alert on causes (CPU spike) instead of symptoms (high error rate)  
❌ Assign blame in postmortems  
❌ Tolerate recurring toil without an automation plan  
❌ Deploy at capacity limits without headroom for failure  

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/sre-engineer

