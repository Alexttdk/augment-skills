---
type: agent_requested
description: Apply when designing REST/GraphQL/gRPC APIs, creating OpenAPI specs, planning versioning strategies, standardizing error formats, or modeling resources and pagination patterns.
---

# API Designer

## When to Use
- Designing new REST, GraphQL, or gRPC APIs
- Creating OpenAPI 3.1 specifications
- Modeling resources and their relationships
- Planning API versioning and deprecation
- Standardizing error responses and pagination across a codebase
- Reviewing existing APIs for consistency and correctness

## API Style Decision Tree

```
Need flexible queries with variable field selection? → GraphQL
Internal service-to-service with high throughput/streaming? → gRPC
Public API, browser-facing, CRUD resources? → REST

┌─────────────┬──────────────────────────────────────────────┐
│ REST        │ Public APIs, CRUD, caching, broad tooling    │
│ GraphQL     │ Multi-client, complex queries, BFF pattern   │
│ gRPC        │ Microservices, binary, streaming, low latency│
└─────────────┴──────────────────────────────────────────────┘
```

## REST Design Patterns

### Resource Naming
```
✓ GET /users                    plural nouns, collections
✓ GET /users/{id}/orders        nested resource (max 2-3 levels)
✓ GET /shipping-addresses       hyphenated multi-word
✗ POST /getUser                 no verbs in URI
✗ GET /user?action=delete       no actions as query params
```

### HTTP Methods & Status Codes
| Method | Use Case | Success |
|--------|----------|---------|
| GET    | Retrieve resource(s) — safe, idempotent | 200 |
| POST   | Create resource — not idempotent | 201 + `Location` header |
| PUT    | Replace entire resource — idempotent | 200 |
| PATCH  | Partial update | 200 |
| DELETE | Remove resource — idempotent | 204 |

**Client errors:** `400` bad request · `401` unauthenticated · `403` forbidden · `404` not found · `409` conflict · `422` semantic/unprocessable · `429` rate limited

**Server errors:** `500` unexpected · `502` bad gateway · `503` unavailable · `504` timeout

### Query Parameters
```
Filtering:  GET /users?status=active&role=admin
Sorting:    GET /users?sort=-created_at      (- prefix = descending)
Fields:     GET /users?fields=id,name,email
Search:     GET /users?q=alice
```

## API Versioning

**Recommended: URI versioning** — explicit, cacheable, easy to debug.

```
/v1/users    /v2/users    /v3/users
```

### Breaking vs Non-Breaking Changes
```
Breaking (requires new version):
  - Removing or renaming fields
  - Changing field types (string → integer)
  - Adding required request fields
  - Removing endpoints or changing status codes

Non-breaking (same version):
  - Adding new optional response fields (clients ignore unknown)
  - Adding new endpoints or HTTP methods
  - Performance improvements, bug fixes
```

### Version Lifecycle
1. **Introduce**: new version launches alongside existing
2. **Deprecate**: respond with deprecation headers
   ```http
   Deprecation: true
   Sunset: Wed, 15 Jan 2026 00:00:00 GMT
   Link: </v2/users/{id}>; rel="successor-version"
   ```
3. **Sunset**: return `410 Gone` with migration docs link

**Rules:** 6+ month deprecation window · max 2-3 active versions · announce via headers + email + changelog.

## Error Response Design

### RFC 7807 Problem Details (`application/problem+json`)
```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "The email field must be a valid email address",
  "instance": "/users"
}
```

### Extended Format with Field-Level Details
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "request_id": "req_abc123",
    "details": [
      { "field": "email", "code": "INVALID_FORMAT", "message": "Must be a valid email" },
      { "field": "age",   "code": "OUT_OF_RANGE",   "message": "Must be 18–120" }
    ]
  }
}
```

### Standard Error Code Catalog
```
VALIDATION_ERROR     400  field-level errors with details array
MISSING_TOKEN        401  no auth provided
INVALID_TOKEN        401  malformed or expired token
INSUFFICIENT_PERMS   403  authenticated but not authorized
RESOURCE_NOT_FOUND   404  entity doesn't exist
RESOURCE_EXISTS      409  duplicate (include link to existing)
RATE_LIMIT_EXCEEDED  429  + Retry-After header + reset timestamp
INTERNAL_ERROR       500  never expose stack traces or DB errors
SERVICE_UNAVAILABLE  503  maintenance + Retry-After header
```

Always include `request_id` in error responses for support traceability.

## Pagination Patterns

| Strategy | Best For | Cursor Stable? |
|----------|----------|---------------|
| Offset/limit | Simple, sortable data | No (inserts shift pages) |
| Cursor-based | Real-time feeds, large datasets | Yes |
| Keyset | High-performance, time-series | Yes |

### Cursor Pagination (preferred for feeds)
```json
{
  "data": [...],
  "pagination": {
    "cursor": "eyJpZCI6MTAwfQ==",
    "has_next": true,
    "has_prev": false,
    "total": 1240
  }
}
```
```
GET /posts?cursor=eyJpZCI6MTAwfQ==&limit=25
```

### Offset Pagination (simple use cases)
```json
{
  "data": [...],
  "meta": { "page": 2, "limit": 25, "total": 1240, "total_pages": 50 }
}
```

## GraphQL Design Notes

**Use GraphQL when:**
- Multiple clients (web, mobile, partner) need different field subsets
- BFF (Backend-for-Frontend) pattern to aggregate multiple services
- Rapid iteration — clients evolve queries without API changes

**Schema design patterns:**
```graphql
# Use connections for paginated lists (Relay spec)
type Query {
  users(first: Int, after: String, filter: UserFilter): UserConnection!
  user(id: ID!): User
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

# Mutations: verb + noun, return the mutated type
type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
}

# Payload pattern — always return errors alongside data
type CreateUserPayload {
  user: User
  errors: [UserError!]!
}
```

**N+1 problem — always solve with DataLoader:**
```javascript
// Without DataLoader: 1 query per user → N+1
const loader = new DataLoader(async (userIds) => {
  const users = await db.users.findMany({ id: { in: userIds } });
  return userIds.map(id => users.find(u => u.id === id));
});

// Now batches all user lookups in a single DB query
```

**Subscriptions:** Use WebSocket transport for real-time; prefer `graphql-ws` over the deprecated `subscriptions-transport-ws`.

## gRPC Design Notes

**Use gRPC when:**
- Internal microservice communication (not public-facing)
- Need streaming (server-push, client-push, bidirectional)
- Strict schema enforcement and code generation are valued
- Latency matters (binary protobuf vs JSON)

**Proto definition patterns:**
```protobuf
syntax = "proto3";
package users.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User); // Server streaming
  rpc CreateUser(CreateUserRequest) returns (User);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  google.protobuf.Timestamp created_at = 4;
}

// Always version your package: users.v1, users.v2
```

**gRPC vs REST tradeoffs:**
```
gRPC pros:  Binary (smaller), typed contracts, streaming, code gen
gRPC cons:  Not human-readable, limited browser support (use grpc-web)
REST pros:  Universal, cacheable, debuggable, browser-native
REST cons:  Verbose, no streaming (use SSE/WS for that)
```

## OpenAPI 3.1 Essentials

Minimum required elements for every endpoint:
```yaml
/users/{id}:
  get:
    summary: Get user by ID
    parameters:
      - name: id
        in: path
        required: true
        schema: { type: string }
    responses:
      '200':
        description: User found
        content:
          application/json:
            schema: { $ref: '#/components/schemas/User' }
      '404':
        $ref: '#/components/responses/NotFound'
      '401':
        $ref: '#/components/responses/Unauthorized'
```

**Reusable components:** Define `schemas`, `responses`, `parameters`, and `securitySchemes` in `components/` — never duplicate inline.

## Constraints

**MUST DO:**
- Use nouns not verbs in resource URIs
- Use plural nouns for collections (`/users`, not `/user`)
- Return consistent error format across all endpoints
- Include `request_id` in every error response
- Implement pagination for all collection endpoints
- Version APIs from day one (`/v1/`)
- Document authentication and all possible error codes
- Provide request/response examples in specs

**MUST NOT:**
- Return `200 OK` with error details in body
- Expose stack traces, DB errors, or internal paths
- Create breaking changes without a new version
- Design APIs without a deprecation policy
- Return inconsistent response shapes per endpoint
- Skip rate limiting considerations on public endpoints
- Use verbs in resource URIs (`/getUser`, `/createOrder`)

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/api-designer

