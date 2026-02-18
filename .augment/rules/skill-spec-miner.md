---
type: agent_requested
description: Reverse-engineer specifications from existing codebases — map architecture, trace data flows, extract EARS-format requirements from code, and document undocumented systems.
---

# Spec Miner

## When to Use

Invoke when: understanding legacy or undocumented systems, creating documentation for existing code, onboarding to a new codebase, planning enhancements, extracting requirements from implementation.

Triggers: *"reverse engineer"*, *"legacy code"*, *"undocumented"*, *"understand this codebase"*, *"extract specs from code"*, *"what does this system do"*, *"code archaeology"*, *"create documentation for"*

---

## Role

You are a **senior software archaeologist** operating with two perspectives:
- **Arch Hat**: System architecture, data flows, module structure, integration points
- **QA Hat**: Observable behaviors, edge cases, error handling, boundary conditions

Ground ALL observations in actual code evidence. Distinguish facts from inferences. Flag uncertainties explicitly.

---

## Core Workflow

### Step 1: Scope
Define analysis boundaries — full system or specific feature? Ask if unclear.

### Step 2: Explore (Arch Hat)
Map structure using file system patterns:

```bash
# Entry points
Glob: **/main.{ts,js,py,go}, **/app.{ts,js,py}, **/index.{ts,js}

# Routes / controllers
Glob: **/routes/**/*.{ts,js}, **/controllers/**/*.{ts,js}
Grep: @Controller|@Get|@Post|router\.|app\.get

# Data models / schemas
Glob: **/models/**/*.{ts,js,py}, **/schema*.{ts,js,py,sql}, **/migrations/**/*
Grep: @Entity|class.*Model|schema\s*=

# Services / business logic
Glob: **/services/**/*.{ts,js}
Grep: async.*function|export.*class

# Auth / security
Glob: **/auth/**/*, **/guards/**/*
Grep: @Guard|middleware|passport|jwt

# External integrations
Grep: fetch\(|axios\.|HttpService|request\(
Glob: **/integrations/**/*, **/clients/**/*

# Config / env
Glob: **/*.config.{ts,js}, **/.env*, **/config/**/*
```

### Step 3: Trace
Follow data flows end-to-end:
```
Request → Middleware → Guard → Controller → Service → Repository → Database
                                                    ↓
                                              External API
```

### Step 4: Document (QA Hat)
Write observed requirements in EARS format (see below).

### Step 5: Flag
Mark areas needing clarification in an `Uncertainties` section.

---

## EARS Format — Complete Syntax Reference

| Type | Pattern | When to Use |
|------|---------|-------------|
| **Ubiquitous** | The system shall [action]. | Always true, no trigger |
| **Event-driven** | When [trigger], the system shall [action]. | On a specific event |
| **State-driven** | While [state], the system shall [action]. | Continuous condition |
| **Conditional** | While [state], when [trigger], the system shall [action]. | State + trigger combined |
| **Optional** | Where [feature enabled], the system shall [action]. | Feature flag controlled |

### Observed Requirements — Authentication

```
OBS-AUTH-001: While credentials are valid, when POST /auth/login is called,
              the system shall return JWT access token (15min) and refresh token (7d).

OBS-AUTH-002: While refresh token is valid, when POST /auth/refresh is called,
              the system shall issue a new access token.

OBS-AUTH-003: When an expired or invalid token is provided,
              the system shall return 401 Unauthorized.

OBS-AUTH-004: While failed login count exceeds 5, when login is attempted,
              the system shall reject the attempt without revealing the lockout reason.
```

### Observed Requirements — User Management

```
OBS-USER-001: While email is unique, when POST /users is called with valid data,
              the system shall create user with bcrypt-hashed password (rounds=12).

OBS-USER-002: When email format is invalid,
              the system shall return 400 with error "Invalid email format".

OBS-USER-003: When DELETE /users/:id is called by an admin,
              the system shall set deleted_at timestamp instead of removing the record.
```

### Observed Requirements — Error Handling

```
OBS-ERR-001: When required fields are missing from any request,
             the system shall return 400 with field-specific error messages.

OBS-ERR-002: When a requested resource does not exist,
             the system shall return 404 with message "Resource not found".
```




---

## Analysis Checklist

Run after Step 2 (Explore) to confirm coverage before writing requirements.

| Phase | What to Find | Done? |
|-------|-------------|-------|
| Entry points | `main.*`, `app.*`, `index.*`, server bootstrap | [ ] |
| Routes | All HTTP verbs, path params, query params, middleware chain | [ ] |
| Models | Entity fields, types, constraints, indexes, relations | [ ] |
| Migrations | Schema history, breaking changes, data transformations | [ ] |
| Auth | Authentication mechanism, authorization rules, role checks | [ ] |
| Validation | Input sanitization location, error message format, rules | [ ] |
| Business logic | Core algorithms, state transitions, computation rules | [ ] |
| State transitions | Enum values, valid transitions, guards | [ ] |
| External calls | APIs, queues, caches, third-party SDKs | [ ] |
| Background jobs | Schedulers, workers, retry logic, idempotency | [ ] |
| Tests | Existing test names reveal intended behavior | [ ] |
| Error handling | Exception types, HTTP codes, logging strategy | [ ] |

**Verification checklist before writing spec:**
- [ ] All EARS requirements grounded in specific file/line evidence
- [ ] Each `While` clause confirmed as actual runtime state
- [ ] All `When` triggers confirmed as actual system inputs
- [ ] Uncertainties section populated for anything inferred

---

## Specification Output Template

Save completed spec to: `specs/{project_name}_reverse_spec.md`

```markdown
# Reverse Specification: {Project Name}
**Date:** {date} | **Analyst:** {name} | **Codebase:** {repo/path}

## Overview
[2-3 sentence description of what the system does and its primary purpose]

## Architecture Summary

### Tech Stack
| Layer | Technology | Version |
|-------|-----------|---------|
| Runtime | [e.g. Node.js 20] | [x.y.z] |
| Framework | [e.g. NestJS] | [x.y.z] |
| Database | [e.g. PostgreSQL 15] | [x.y.z] |
| Auth | [e.g. JWT + Passport] | [x.y.z] |

### Module Structure
[Top-level modules and their responsibilities]

### Data Flow
[Request → Middleware → Controller → Service → Repository → DB]

## Observed Functional Requirements

[Write all requirements in EARS format with evidence citations]

OBS-XXX-001: [EARS statement]
  Evidence: [file path:line range]

## Observed Non-Functional Requirements

| Category | Observation | Evidence |
|----------|------------|---------|
| Security | [e.g. bcrypt rounds=12] | [file:line] |
| Performance | [e.g. DB query uses index on user_id] | [file:line] |
| Error Handling | [e.g. 4xx always returns {error, message, statusCode}] | [file:line] |

## Inferred Acceptance Criteria

[For key behaviors, translate EARS requirements to Given/When/Then]

**Feature: [Name]**
- Given [context], When [action], Then [outcome]

## Uncertainties and Questions

| ID | Question | Why It Matters |
|----|---------|---------------|
| Q1 | [What is unclear] | [Impact on spec accuracy] |

## Recommendations

[Gaps found, missing tests, undocumented behaviors, suggested improvements]
```

---

## References

Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/spec-miner

