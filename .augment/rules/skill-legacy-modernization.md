---
type: agent_requested
description: Activate when modernizing legacy codebases, implementing strangler fig migrations, reducing technical debt incrementally, or upgrading frameworks/databases without business disruption.
---

# Legacy Modernizer

## When to Use
- Modernizing aging systems or outdated technology stacks
- Implementing strangler fig or branch-by-abstraction
- Migrating monoliths to microservices or modular architectures incrementally
- Upgrading databases, frameworks, or languages safely
- Reducing technical debt while preserving business continuity
- Planning migration roadmaps with rollback strategies

**Core philosophy:** Never do a big-bang rewrite. Always migrate incrementally with rollback capability at every step.

---

## Phase 1: Assessment Framework

### Codebase Health Signals

| Metric | Green | Yellow | Red |
|--------|-------|--------|-----|
| Test coverage | >70% | 30–70% | <30% |
| Avg function length | <30 lines | 30–80 | >80 lines |
| Circular dependencies | 0 | 1–5 | >5 |
| Raw SQL in app code | None | Isolated | Widespread |
| CVE-affected deps | 0 | 1–3 | >3 |
| Last touched (days) | <90 | 90–365 | >365 |

### Dependency Mapping
1. Build a dependency graph (use `pydeps`, `madge`, or `depcruise`)
2. Identify leaf modules (no outbound deps) — extract these first
3. Find circular deps — must break before extraction
4. Map external integrations — third-party APIs, queues, cron jobs

### Technical Debt Estimation (SQALE method)

```
Effort days = Σ (issue_count × severity_multiplier)
  critical: 1.0 day/issue
  major:    0.5 day/issue
  minor:    0.25 day/issue

Cost = effort_days × daily_rate
```

Typical issue categories: long functions, circular deps, missing tests, security vulns, deprecated deps, code duplication.

### Risk Assessment Matrix

| Area | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Database migration | Critical | Medium | Dual-write + lazy migration |
| Auth system change | Critical | Low | Shadow-test in prod + feature flags |
| UI framework swap | Medium | Medium | Incremental component replacement |
| API deprecation | High | High | 12-month sunset + versioning |

---

## Phase 2: Safety Net First

**Build tests BEFORE touching any code.**

### Characterisation Tests (Golden Master)
```python
# Capture current behaviour — don't assert what it "should" do, 
# assert what it actually does today.
def test_golden_master_order_total():
    result = legacy_order_service.calculate_total(ORDER_FIXTURE)
    assert result == snapshot("order_total_golden")  # serialized snapshot
```

Target: **80%+ coverage on all migration-critical paths** before first change.

### Monitoring Baseline
Record current production metrics before any changes:
- Error rate (per endpoint), latency p50/p95/p99, throughput, memory/CPU

---

## Phase 3: Strangler Fig Pattern

**Concept:** Build new functionality alongside the old system. Route traffic incrementally. Strangle old code when traffic = 0%.

### Step-by-Step Implementation

**Step 1 — Introduce a Facade/Router**
```python
MIGRATION_CONFIG = {
    "users.create":  {"new_pct": 100},
    "users.update":  {"new_pct": 50},
    "orders.create": {"new_pct": 0},   # not started yet
}

async def route(feature: str, request):
    pct = MIGRATION_CONFIG.get(feature, {}).get("new_pct", 0)
    if should_use_new(request, pct):
        return await new_service.handle(request)
    return await legacy_service.handle(request)

def should_use_new(request, pct: int) -> bool:
    if pct == 0: return False
    if pct == 100: return True
    return hash(request.user_id) % 100 < pct  # sticky routing per user
```

**Step 2 — Adapter Pattern (wrap legacy behind interface)**
```python
class OrderServiceInterface(ABC):
    @abstractmethod
    async def create_order(self, user_id, items) -> dict: ...

class LegacyOrderAdapter(OrderServiceInterface):
    async def create_order(self, user_id, items):
        return await asyncio.to_thread(self._legacy.create_order, user_id, items)

class ModernOrderService(OrderServiceInterface):
    async def create_order(self, user_id, items):
        async with self.db.begin():
            order = Order(user_id=user_id, items=items)
            self.db.add(order)
            await self.db.flush()
            await self.events.publish("order.created", {"id": order.id})
            return order.to_dict()
```

**Step 3 — Traffic Migration Phases**

| Phase | Traffic % | Validation Gate |
|-------|-----------|-----------------|
| Setup | 0% | Smoke tests pass |
| Canary | 10% | Error rate < 1%; latency within 20% of baseline |
| Ramp | 50% | Error rate < 0.5%; p95 within baseline |
| Full | 100% | All metrics green for 48h |
| Cleanup | — | Legacy code unused for 30 days → delete |

Auto-rollback: if error rate > threshold, restore previous traffic split immediately.

---

## Phase 4: Database Migration

### Expand-Contract Pattern (zero-downtime schema change)
```
Step 1 EXPAND:  Add new column with default/nullable
  ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;

Step 2 WRITE BOTH: Application writes to old + new column simultaneously
  self.is_confirmed = verified
  self.email_verified = verified  # new field

Step 3 BACKFILL: Migrate existing data in batches
  UPDATE users SET email_verified = is_confirmed WHERE email_verified IS NULL;

Step 4 READ NEW: Switch reads to new column

Step 5 CONTRACT: Drop old column after all deployments complete
```

### Dual-Write Repository
```python
class DualWriteRepo:
    async def create(self, data):
        # Modern DB is source of truth
        async with self.modern.begin():
            record = Model(**data)
            self.modern.add(record)
            await self.modern.flush()
        # Best-effort sync to legacy
        try:
            asyncio.create_task(self._sync_to_legacy(record))
        except Exception as e:
            logger.error("Legacy sync failed", extra={"id": record.id, "err": str(e)})
        return record

    async def get(self, id):
        record = await self.modern.get(Model, id)
        if record: return record
        # Lazy migration: read from legacy, migrate, return
        legacy = await self._read_legacy(id)
        return await self._migrate(legacy) if legacy else None
```

---

## Phase 5: API Versioning During Migration

```python
# Support both versions simultaneously during transition
@app.get("/api/users/{id}")          # v1 — keep working
@app.get("/api/v2/users/{id}")       # v2 — new structure

# Deprecation signals
response.headers["Deprecation"] = "true"
response.headers["Sunset"] = "2025-12-31"
response.headers["Link"] = '</api/v2/users>; rel="successor-version"'
```

Migration timeline: announce → dual support → sunset warning → deprecate.

---

## Feature Flag Strategy

```python
class FeatureFlags:
    def is_enabled(self, flag: str, user_id: str = None) -> bool:
        config = self.store.get(flag)
        if not config: return False
        if config["enabled_for_all"]: return True
        if user_id and user_id in config.get("allowlist", []): return True
        # Percentage rollout via consistent hash
        if user_id:
            return hash(f"{flag}:{user_id}") % 100 < config.get("percentage", 0)
        return False
```

One flag per migration surface. Flags are the rollback lever — always test the "off" path.

---

## Modernization Roadmap Template

| Phase | Duration | Goal | Exit Criteria |
|-------|----------|------|---------------|
| 0. Assess | 2w | Map system, identify risks | Roadmap approved |
| 1. Safety Net | 4w | 80%+ test coverage on critical paths | Tests green in CI |
| 2. DB Setup | 3w | Dual-write working | Data consistency 99.9% |
| 3. Extract Leaf Services | 4–6w per service | Strangler fig at 100% traffic | Metrics match baseline |
| 4. Extract Core | 6–12w | Core business services live | Legacy idle 30 days |
| 5. Cleanup | 2w | Delete legacy code | No references remain |

---

## Hard Rules

**MUST DO**
- [ ] Characterisation tests before first code change
- [ ] Feature flags for every rollout (rollback = flip flag)
- [ ] Monitor error rates and latency before/during/after each phase
- [ ] Dual-write with modern DB as source of truth
- [ ] Expand-contract for every schema change

**MUST NOT**
- [ ] Big-bang rewrites — always incremental
- [ ] Delete legacy code until new is proven stable (30+ days)
- [ ] Skip test coverage — it's the safety net
- [ ] Deploy without a rollback procedure documented
- [ ] Accumulate new tech debt in new code during migration

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/legacy-modernizer
- Strangler Fig Pattern: https://martinfowler.com/bliki/StranglerFigApplication.html
- Branch by Abstraction: https://martinfowler.com/bliki/BranchByAbstraction.html
- Expand-Contract: https://www.martinfowler.com/bliki/ParallelChange.html

