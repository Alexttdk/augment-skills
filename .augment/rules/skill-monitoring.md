---
type: agent_requested
description: Set up monitoring, metrics, alerting, dashboards, or distributed tracing. Use for Prometheus, Grafana, OpenTelemetry, structured logging, and observability instrumentation.
---

# Monitoring & Observability

## When to Use
- Adding metrics, logs, or traces to a service
- Setting up Prometheus scraping, Grafana dashboards, or alerting rules
- Implementing distributed tracing with OpenTelemetry
- Designing alert thresholds to avoid alert fatigue
- Building RED/USE method dashboards
- Debugging production issues through observability

## The Three Pillars

| Pillar | What it covers | Tooling |
|--------|---------------|---------|
| **Metrics** | Aggregated numeric measurements over time (rates, counts, gauges, histograms) | Prometheus, DataDog, CloudWatch |
| **Logs** | Timestamped structured events per request/operation | Loki, ELK, structured JSON |
| **Traces** | End-to-end request flow across services with span timing | OpenTelemetry, Jaeger, Tempo |

## Metric Types (Prometheus)

| Type | Use Case | Example |
|------|----------|---------|
| **Counter** | Cumulative total, only increases | `http_requests_total` |
| **Gauge** | Point-in-time value, goes up/down | `active_connections` |
| **Histogram** | Distribution with configurable buckets | `http_request_duration_seconds` |
| **Summary** | Pre-calculated percentiles | `http_response_size_bytes` |

**Naming conventions**: use unit suffix (`_seconds`, `_bytes`, `_total`), base units only (seconds not ms), prefix with service name.

## Instrumentation Pattern

```typescript
// 1. Define metrics at module level
const httpRequests = new Counter({ name: 'http_requests_total', labelNames: ['method', 'status'] });
const httpDuration = new Histogram({ name: 'http_request_duration_seconds',
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5], labelNames: ['method'] });

// 2. Middleware: measure on every request
app.use((req, res, next) => {
  const end = httpDuration.startTimer({ method: req.method });
  res.on('finish', () => {
    httpRequests.inc({ method: req.method, status: res.statusCode });
    end();
  });
  next();
});

// 3. Expose /metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.send(await register.metrics());
});
```

Also collect business metrics (orders, revenue, active users) — not just technical ones.

## PromQL Quick Reference

```promql
# Request rate (5m window)
sum(rate(http_requests_total[5m]))

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# p95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# CPU utilization
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Top 10 endpoints by error rate
topk(10, sum by (path) (rate(http_requests_total{status=~"5.."}[5m]))
  / sum by (path) (rate(http_requests_total[5m])))
```

## Dashboard Layout (RED + USE)

```
┌──────────────────────────────────────────────────────┐
│  SERVICE OVERVIEW: Rate │ Error Rate │ p50 │ p99     │
├──────────────────────────────────────────────────────┤
│  REQUEST METRICS: Requests/s by endpoint, Error rate │
├──────────────────────────────────────────────────────┤
│  LATENCY: Heatmap distribution, p50/p95/p99 trends   │
├──────────────────────────────────────────────────────┤
│  INFRASTRUCTURE: CPU │ Memory │ Disk │ Network       │
└──────────────────────────────────────────────────────┘
```

**Panel choices**: Stat for KPIs, Time Series for trends, Heatmap for latency distribution, Table for top-N.

**RED** (services): Rate, Errors, Duration  
**USE** (resources): Utilization, Saturation, Errors

## OpenTelemetry Setup (Node.js)

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  resource: new Resource({ [SEMRESATTRS_SERVICE_NAME]: 'my-service' }),
  traceExporter: new OTLPTraceExporter({ url: 'http://jaeger:4318/v1/traces' }),
  instrumentations: [getNodeAutoInstrumentations()],  // auto-instruments HTTP, DB, etc.
});
sdk.start();
```

**Manual spans** for business operations:
```typescript
const tracer = trace.getTracer('my-service');
async function processOrder(orderId: string) {
  return tracer.startActiveSpan('processOrder', async (span) => {
    span.setAttribute('order.id', orderId);
    try {
      await doWork();
      span.setStatus({ code: SpanStatusCode.OK });
    } catch (e) {
      span.recordException(e); span.setStatus({ code: SpanStatusCode.ERROR });
      throw e;
    } finally { span.end(); }
  });
}
```

**Always propagate context** across service boundaries via `propagation.inject/extract` on HTTP headers.

## Alert Design Principles

1. **Alert on symptoms, not causes** — alert on high error rate, not "CPU spike"
2. **Actionable only** — every alert needs a runbook; if no action needed, it's not an alert
3. **SLO-based alerting** — use burn-rate alerts tied to error budget consumption, not static thresholds
4. **Multi-window burn rates** (from Google SRE Workbook):
   - Fast burn: rate × 14.4 on 1h window → page immediately (2% budget in 1h)
   - Medium burn: rate × 6.0 on 6h window → ticket/warn
   - Slow burn: rate × 1.0 on 3d window → review
5. **Avoid flapping** — require sustained violations (e.g., `for: 5m` in Prometheus rules)
6. **Severity tiers**: P1 (page now), P2 (page if persists), P3 (ticket)

## Must Do / Must Not Do

✅ Use structured JSON logging with request IDs for correlation  
✅ Monitor business metrics alongside technical metrics  
✅ Implement `/health` and `/metrics` endpoints on every service  
✅ Include correlation IDs across all distributed service calls  
❌ Log sensitive data (passwords, tokens, PII)  
❌ Alert on every individual error — leads to alert fatigue  
❌ Use string interpolation in log messages (use structured fields)  
❌ Skip correlation IDs in distributed systems  

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/monitoring-expert

