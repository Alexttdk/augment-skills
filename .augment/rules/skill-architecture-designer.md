---
type: agent_requested
description: Design system architecture, document decisions as ADRs, select the right pattern (monolith vs microservices vs serverless), apply C4 diagrams, and evaluate trade-offs against NFRs.
---

# Architecture Designer

## When to Use

Invoke when: designing new system architecture, choosing between architectural patterns, reviewing existing architecture, creating Architecture Decision Records (ADRs), planning for scalability, evaluating technology choices.

Triggers: *"architecture"*, *"system design"*, *"design pattern"*, *"microservices"*, *"scalability"*, *"ADR"*, *"technical design"*, *"monolith vs microservices"*, *"infrastructure"*

---

## Core Workflow

1. **Understand requirements** — Gather functional + non-functional requirements (use NFR checklist)
2. **Identify patterns** — Match requirements to architectural patterns (see table below)
3. **Design** — Create architecture with trade-offs explicitly documented
4. **Document** — Write an ADR for every significant decision
5. **Review** — Validate with stakeholders before finalizing

**Rules**: Never over-engineer for hypothetical scale. Document ALL significant decisions as ADRs. Always evaluate trade-offs, not just benefits. Plan for failure modes. Consider operational complexity.

---

## ADR Template

```markdown
# ADR-{number}: {Title}

## Status
[Proposed | Accepted | Deprecated | Superseded by ADR-XXX]

## Context
[Situation, forces at play, constraints, and what we're trying to achieve.]

## Decision
[What we are going to do — stated clearly and unambiguously.]

## Consequences

### Positive
- [Benefit 1]

### Negative
- [Drawback 1]

### Neutral
- [Side effect — neither good nor bad]

## Alternatives Considered
| Alternative | Why Rejected |
|------------|-------------|
| [Option A] | [Reason] |

## References
- [RFC, discussion, or documentation links]
```

### ADR Example

```markdown
# ADR-001: Use PostgreSQL for primary database

## Status
Accepted

## Context
E-commerce platform needs relational DB with strong consistency for financial transactions,
JSON support for flexible product attributes, and Python/Node.js compatibility.
Team has existing PostgreSQL experience.

## Decision
Use PostgreSQL as primary database, hosted on AWS RDS.

## Consequences
### Positive
- ACID compliance for financial transactions
- Rich features: JSON, full-text search, CTEs, window functions
- Strong community and tooling ecosystem

### Negative
- Vertical scaling limits (mitigated by read replicas)
- Requires DBA expertise for query optimization at scale

## Alternatives Considered
| Alternative | Why Rejected |
|------------|-------------|
| MySQL | Less feature-rich JSON operations |
| MongoDB | Relational data model required for orders/inventory |
| CockroachDB | Higher cost, team unfamiliar, adds operational overhead |
```

Store ADRs at: `docs/adr/NNNN-title-in-kebab-case.md`

---

## Architecture Patterns Quick-Reference

| Pattern | Best For | Team Size | Key Trade-off |
|---------|----------|-----------|--------------|
| **Monolith** | Simple domain, new projects | 1–10 | Simple deploy vs. hard to scale parts independently |
| **Modular Monolith** | Growing complexity | 5–20 | Module boundaries vs. still a single deployment unit |
| **Microservices** | Complex domain, large org | 20+ | Independent scaling/teams vs. distributed system complexity |
| **Serverless** | Variable or event-driven load | Any | Auto-scale vs. cold starts + vendor lock-in |
| **Event-Driven** | Async processing, audit trails | 10+ | Loose coupling vs. debugging + eventual consistency |
| **CQRS** | Read-heavy, event sourcing | 10+ | Read/write optimization vs. consistency complexity |

**Decision guide**:
- Starting new project → Monolith
- Growing startup → Modular Monolith
- 20+ devs, clear bounded contexts → Microservices
- Variable/spiky load, no persistent connections needed → Serverless
- Async workflows, pub/sub, audit → Event-Driven
- Read:write ratio > 10:1, or complex queries → CQRS

---

## C4 Model Quick-Reference

| Level | Name | Shows | Audience |
|-------|------|-------|----------|
| L1 | **Context** | System in environment; external actors and external systems | Non-technical stakeholders |
| L2 | **Container** | Major deployable units: apps, DBs, APIs, queues, functions | Architects, tech leads |
| L3 | **Component** | Internal structure of one container | Developers |
| L4 | **Code** | Classes, interfaces, functions | Rarely needed in practice |

Start at L1 and zoom in only when more detail is needed. For most design discussions, L1 + L2 is sufficient.

---

## NFR Checklist

Before finalizing any architecture, confirm each dimension is addressed:

| Dimension | Key Questions |
|-----------|--------------|
| **Performance** | Latency targets (p50/p99), throughput, caching strategy |
| **Scalability** | Expected load growth, horizontal vs. vertical, bottlenecks |
| **Availability** | SLA target (99.9% = 8.7h/yr downtime), failover, multi-AZ |
| **Security** | AuthN/AuthZ, encryption at rest + transit, OWASP Top 10 |
| **Observability** | Logging, metrics, distributed tracing, alerting thresholds |
| **Data** | Retention policy, backup schedule, GDPR/compliance |
| **Operations** | Deployment strategy, rollback plan, on-call complexity |

---

## Output Template

```markdown
# System Architecture: [System Name]

## Requirements Summary
- Functional: [key capabilities]
- Non-functional: [from NFR checklist]
- Constraints: [team size, timeline, budget, existing systems]

## Chosen Pattern
[Pattern name] — [one-sentence rationale]

## High-Level Diagram (C4 L1/L2)
[Mermaid or text box diagram]

## Key Decisions (ADR Index)
- ADR-001: [title] — [one-line summary]
- ADR-002: [title] — [one-line summary]

## Technology Recommendations
| Component | Choice | Rationale |
|-----------|--------|-----------|
| Database | | |
| API layer | | |
| Cache | | |
| Message bus | | |

## Risks and Mitigations
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| | H/M/L | H/M/L | |
```

---

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/architecture-designer

