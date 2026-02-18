---
type: agent_requested
description: Activate when implementing features that span frontend and backend, defining API contracts before coding, or coordinating cross-stack work across models, endpoints, and UI components.
---

# Fullstack Guardian

## When to Use
- Implementing any feature that touches both backend API and frontend UI
- Defining API contracts before splitting work across stack layers
- Reviewing data flows from database through service layer to component
- Coordinating deployment of interdependent frontend + backend changes
- Deciding monolith vs microservice vs BFF for a new feature

---

## Cross-Stack Feature Workflow

Sequence always follows: **Design → Backend → API → Frontend → Tests → Deploy**

```
1. CONTRACT   Define API schema (request/response types + error codes)
2. BACKEND    Models → migrations → service logic → endpoint → unit tests
3. FRONTEND   API client types → data-fetching hook → component → UI tests  
4. INTEGRATE  Integration tests covering full request→response→render path
5. DEPLOY     Backend first, then frontend; validate in staging
```

Never build frontend against a guessed API shape — agree the contract first.

---

## Contract-First Development

### API Contract Template

```typescript
// Shared types package — define before implementation

// Request
interface CreateOrderRequest {
  customerId: string;
  items: Array<{ productId: string; quantity: number }>;
  shippingAddress: Address;
}

// Success response
interface CreateOrderResponse {
  data: {
    orderId: string;
    status: 'pending' | 'confirmed';
    total: number;
    estimatedDelivery: string; // ISO-8601
  };
}

// Error response — standard shape for all endpoints
interface ApiError {
  error: {
    code: string;       // machine-readable: "VALIDATION_ERROR"
    message: string;    // human-readable
    details?: Record<string, string[]>;  // field-level errors
    requestId: string;
  };
}
```

Write the OpenAPI/Swagger spec or shared TypeScript types **before writing handlers or components**. This enables parallel work and eliminates integration surprises.

---

## Backend Implementation Order

```
1. Database model + migration (define schema)
2. Validation schema (Zod/Pydantic) matching contract
3. Service/use-case layer (business logic, no HTTP)
4. HTTP handler/controller (thin: validate → delegate → respond)
5. Unit tests for service layer
6. Integration test: POST /endpoint → verify DB state
```

**Key invariants:**
- Input validated with schema on every endpoint (never trust req.body directly)
- Resource ownership checked server-side (never trust client-supplied userId/role)
- Errors mapped to standard `ApiError` shape with semantic HTTP status codes
- Sensitive fields (passwords, secrets) stripped from all responses

---

## Frontend Implementation Order

```
1. Add types from shared contract (or generate from OpenAPI)
2. API client function: fetch wrapper with typed request/response
3. Data-fetching hook: useQuery / useSuspenseQuery wrapping API call
4. Component: renders hook data, handles loading/error states
5. Component tests: mock API, assert render paths
```

**Key invariants:**
- Validate forms client-side for UX, but never rely on it for security
- All API errors surfaced to users with actionable messages
- Loading, error, and empty states handled in every data-fetching component
- No hardcoded API URLs — use environment config

---

## Frontend-Backend Sync Checklist

Run this checklist before considering a feature "done":

**API Contract**
- [ ] Request/response types match between server and client
- [ ] All error codes handled by frontend (400, 401, 403, 404, 422, 429, 500)
- [ ] Pagination / cursor implemented consistently
- [ ] API versioned if breaking changes introduced

**Data Flow**
- [ ] Required fields present in response (no undefined access in UI)
- [ ] Date/time fields in ISO-8601; client converts to local time
- [ ] File/image URLs are absolute or use a known CDN base
- [ ] Enum values shared (not duplicated as strings in FE and BE separately)

**Auth**
- [ ] Protected backend routes return 401 when unauthenticated
- [ ] Frontend redirects to login on 401 responses
- [ ] Role-restricted UI elements also restricted at API level
- [ ] CSRF token included on state-mutating requests (if using cookies)

**Validation**
- [ ] Form field constraints mirror backend schema limits (max length, regex)
- [ ] Backend 422 validation errors mapped to field-level UI messages

---

## Deployment Coordination

### Order of Deployment

```
Rule: Backend changes first, then frontend.

Pattern: Backward-compatible API update
  1. Deploy backend with new optional field / endpoint version
  2. Old frontend still works (old endpoint still served)
  3. Deploy frontend consuming new field/endpoint
  4. Deprecate old version; sunset after migration window

Forbidden: Deploy frontend requiring new field before backend serves it
```

### Environment Parity
- Staging must mirror production (same infrastructure, same data shape)
- Run E2E tests in staging before production promotion
- Blue-green or canary deploys for high-risk backend changes

### Rollback Plan (document before deploying)
```
If backend fails: revert migration if destructive; feature flag off
If frontend fails: CDN rollback to previous bundle; backend unaffected
If both fail: restore both in reverse deployment order
```

---

## Common Patterns Quick Reference

### CRUD Feature
```
Backend:  GET /resources, POST /resources, GET /resources/:id,
          PATCH /resources/:id, DELETE /resources/:id
Frontend: useResourceList(), useResource(id), useCreateResource(),
          useUpdateResource(), useDeleteResource()
Contract: ResourceSchema (Zod) shared or mirrored
```

### Real-Time Feature (WebSocket / SSE)
```
Backend:  HTTP endpoint bootstraps connection; push events on state change
Frontend: useWebSocket(url) hook manages connection + reconnect
Contract: Event types typed (e.g. OrderStatusEvent); versioned
Auth:     Ticket-based auth (exchange JWT for short-lived WS token)
```

### File Upload
```
Backend:  Validate MIME + size + magic bytes; store; return URL
Frontend: Upload to signed URL (S3/GCS) or multipart POST; poll status
Security: Server re-validates; scan for malware; strip EXIF if images
```

---

## Deliverables Before Calling a Feature Complete

- [ ] Backend: endpoint + service + migration + unit tests
- [ ] Frontend: component + hook + API client + component tests
- [ ] Integration test covering full request → response → render
- [ ] Error states tested (network failure, 4xx, 5xx)
- [ ] Deployed to staging; E2E tests pass
- [ ] Security checklist run (auth, authz, input validation, output encoding)
- [ ] Performance: API p95 < 200ms; TTI < 2s; bundle delta acceptable
- [ ] Accessibility: keyboard navigable; ARIA labels; colour contrast passes

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/fullstack-guardian
- REST API Standards: https://restfulapi.net/
- OpenAPI Specification: https://swagger.io/specification/

