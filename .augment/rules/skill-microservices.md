---
type: agent_requested
description: Activate when designing distributed systems, decomposing monoliths, defining service boundaries, choosing inter-service communication, or implementing saga/CQRS/event-sourcing patterns.
---

# Microservices Architect

## When to Use
- Decomposing a monolith into services (or deciding whether to)
- Defining bounded contexts and service ownership
- Choosing between REST, gRPC, messaging, or event-driven communication
- Designing distributed transactions (saga, outbox)
- Setting up API gateway, service mesh, or observability
- Evaluating CQRS, event sourcing, or database-per-service patterns

---

## Decision Gate: Do You Need Microservices?

Answer "yes" to **3 or more** before splitting:
- [ ] Independent scalability required for different components
- [ ] Different teams need autonomous deployment cycles
- [ ] Technology diversity is genuinely necessary
- [ ] Isolated failure is acceptable/desirable
- [ ] CI/CD pipelines and observability already in place
- [ ] Team has microservices operational experience

**If fewer than 3 "yes"** → start with a **modular monolith**. Extract services later.

---

## Service Decomposition

### Bounded Context Identification (DDD)
1. **Event Storming** — map domain events, commands, aggregates on a timeline
2. **Ubiquitous Language** — each context has its own vocabulary (same word = same meaning inside, different outside)
3. **Subdomain Classification** — Core (competitive advantage), Supporting, Generic (buy/outsource)

### Right-Sizing a Service

| Signal | Too Small | Just Right | Too Large |
|--------|-----------|------------|-----------|
| Team | Can't fill a 2-person team | 2–9 people (2-pizza) | Multiple teams conflict |
| Endpoints | 1–2 | 5–15 | 50+ |
| Business scope | Single CRUD entity | Single business capability | Multiple capabilities |
| Deploy coupling | Must deploy with others | Independent pipeline | Deploys together |

### Extraction Order (Strangler Fig)
1. **Leaf services** first — no downstream calls (notifications, file storage)
2. **Supporting services** — auth, user management
3. **Core business** — orders, payments, inventory (last, highest risk)

### Anti-Patterns
- **Distributed Monolith** — services that must deploy together or share a DB
- **Entity Services** — `UserService` that's just CRUD with no domain logic; prefer capability-based names
- **Shared business logic libraries** — creates hidden coupling; share only technical infrastructure

---

## Communication Patterns Matrix

| Pattern | Protocol | When to Use | Avoid When |
|---------|----------|-------------|------------|
| Sync REST | HTTP/JSON | Public APIs, simple request/response, browser clients | Long-running ops (>5s), fan-out |
| Sync gRPC | HTTP/2 + Protobuf | Internal low-latency, streaming, polyglot | Public-facing or browser clients |
| Async Queue | AMQP/SQS | Single consumer, task distribution, guaranteed delivery | Multiple consumers need same message |
| Event Stream | Kafka/Kinesis | Multiple consumers, event replay, CQRS projections, high throughput | Simple task queues |
| GraphQL Federation | HTTP/JSON | BFF pattern, frontend-driven flexible queries | Internal service-to-service calls |

### Sync vs. Async Decision
```
Use SYNCHRONOUS when:
  - User waiting for immediate result
  - ≤ 2 service hops
  - Strong consistency required
  - Latency budget < 200ms

Use ASYNCHRONOUS when:
  - Long-running operation (>5s)
  - Multiple downstream consumers
  - Eventual consistency acceptable
  - Need to decouple producers from consumers
```

### Event Schema Standard
```json
{
  "eventId": "uuid",
  "eventType": "order.placed",
  "eventVersion": "1.0",
  "timestamp": "ISO-8601",
  "aggregateId": "order-123",
  "correlationId": "request-uuid",
  "payload": { "...domain fields..." }
}
```
Name events in **past tense** (`order.placed`, `payment.failed`). Version schemas; use upcasting for backward compatibility.

---

## Data Consistency Strategies

### Database per Service — Non-Negotiable Rules
- No service reads another service's DB directly
- Cross-service data access → API calls or subscribed events
- Use **denormalisation** (copy relevant fields) or **CQRS read models** for complex queries

### Saga Pattern — Distributed Transactions

**Choreography (event-driven, no central coordinator):**
```
OrderService ──order.created──► PaymentService
                                    │ payment.completed
                                    ▼
                             InventoryService
                                    │ inventory.reserved
                                    ▼
                             ShippingService
```
✅ Loose coupling  ❌ Hard to trace full workflow

**Orchestration (central state machine):**
```
OrderSaga orchestrates:
  1. command → PaymentService → payment.completed
  2. command → InventoryService → inventory.reserved
  3. command → ShippingService → shipment.created

On failure at step N: issue compensating commands for steps 1..N-1
```
✅ Visible workflow  ❌ Orchestrator can become bottleneck

**Always make saga steps idempotent** — check `saga_id` before processing to handle retries safely.

### Outbox Pattern (Reliable Event Publishing)
```
Within single DB transaction:
  1. Write business record
  2. Write event to outbox table (same transaction)
Separate publisher process:
  3. Poll outbox → publish to message broker → mark sent
```
Prevents the "write to DB but fail to publish event" split-brain problem.

### CQRS
- **Command side** (write model): enforces invariants, emits events
- **Query side** (read model): materialized views rebuilt from events, optimised for specific queries
- Rebuild read model by replaying event stream — enables schema evolution

---

## Resilience Patterns

| Pattern | Purpose | Config guidance |
|---------|---------|-----------------|
| Circuit Breaker | Stop cascading failures | Open after 5 failures in 10s; half-open retry after 30s |
| Retry with backoff | Transient faults | Max 3 retries; exponential backoff + jitter |
| Timeout | Prevent indefinite blocking | Set per-call; fail fast over hanging |
| Bulkhead | Isolate failure domains | Separate thread pools per downstream |
| Health checks | Readiness & liveness | `/health/ready` and `/health/live` on every service |

---

## API Gateway & Service Mesh

**API Gateway** — edge layer (external clients):
- Authentication, rate limiting, routing, SSL termination, versioning

**Service Mesh (Istio/Linkerd)** — internal (service-to-service):
- mTLS, circuit breaking, distributed tracing, traffic shaping
- Use when you have 10+ services; overhead not worth it below that

---

## Observability Must-Haves

1. **Structured logs** — JSON with `correlationId`, `service`, `level`, `timestamp`
2. **Distributed tracing** — inject `X-Correlation-ID` / trace context on all requests
3. **Metrics** — expose Prometheus-format `/metrics`; track error rate, latency (p50/p95/p99), saturation
4. **Health endpoints** — `/health/live` (process alive) and `/health/ready` (dependencies reachable)

---

## Implementation Checklist

**Service Independence**
- [ ] Deploys independently via its own CI/CD pipeline
- [ ] Owns its database (no shared tables)
- [ ] Can degrade gracefully when dependencies are unavailable
- [ ] Exposes health check + readiness probe

**Communication**
- [ ] Correlation IDs propagated on all requests and events
- [ ] Circuit breakers on all synchronous external calls
- [ ] Idempotency keys on all async message handlers

**Data**
- [ ] Cross-aggregate operations use async events, not synchronous calls
- [ ] Saga steps are idempotent (checked via saga_id)
- [ ] Outbox pattern used where at-least-once delivery is critical

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/microservices-architect
- DDD: Evans, *Domain-Driven Design* (2003)
- Saga Pattern: https://microservices.io/patterns/data/saga.html
- Outbox Pattern: https://microservices.io/patterns/data/transactional-outbox.html

