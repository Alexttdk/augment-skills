---
type: agent_requested
description: Elicit and document feature requirements via structured interview, EARS-format functional specs, Given/When/Then acceptance criteria, and an implementation checklist.
---

# Feature Forge

## When to Use

Invoke when: defining a new feature, gathering requirements, writing functional specs, creating acceptance criteria, planning implementation TODO lists.

Triggers: *"requirements"*, *"spec"*, *"feature definition"*, *"user stories"*, *"EARS format"*, *"what should this feature do"*, *"gather requirements"*, *"define this feature"*

---

## Core Workflow

1. **Discover** — Ask about feature goal, target users, and user value
2. **Interview** — Systematic questioning from both hats:
   - **PM Hat**: user value, business goals, success metrics, MoSCoW priority
   - **Dev Hat**: feasibility, security, performance, edge cases, integrations
3. **Document** — Write requirements in EARS format (see below)
4. **Validate** — Review acceptance criteria; surface key trade-offs for stakeholder sign-off
5. **Plan** — Create implementation checklist

**Rules**: Conduct full interview before writing spec. Accept only specific, testable requirements. Always include non-functional requirements and error handling.

---

## EARS Syntax Reference

| Type | Pattern | Example |
|------|---------|---------|
| **Ubiquitous** | The system shall [action]. | The system shall encrypt all passwords using bcrypt. |
| **Event-driven** | When [trigger], the system shall [action]. | When the user clicks "Submit", the system shall validate the form. |
| **State-driven** | While [state], the system shall [action]. | While the user is logged in, the system shall display the dashboard. |
| **Conditional** | While [state], when [trigger], the system shall [action]. | While the cart has items, when checkout is clicked, the system shall navigate to payment. |
| **Optional** | Where [feature enabled], the system shall [action]. | Where 2FA is enabled, the system shall require a verification code. |

### EARS Examples

```
FR-AUTH-001: While credentials are valid, when POST /auth/login is called,
             the system shall return JWT (15min) and refresh token (7d).
FR-AUTH-002: When invalid credentials are provided,
             the system shall return 401 and increment failed login counter.
FR-AUTH-003: While failed login count exceeds 5, when login is attempted,
             the system shall reject the attempt and require password reset.

FR-CART-001: While user is logged in, when they click "Add to Cart",
             the system shall add the item and update the cart badge count.
FR-ORDER-001: While payment method is valid, when user confirms order,
              the system shall create order, charge payment, and send confirmation email.
```

---

## Acceptance Criteria Format

```markdown
### AC-001: [Scenario Name]
Given [context/precondition]
When [action taken]
Then [expected result]
And [additional assertion]
```

Include all scenario types:
- **Happy path**: valid state + valid action → success
- **Error cases**: invalid input/state → appropriate error message
- **Edge cases**: boundary conditions → graceful handling
- **Authorization**: role-based access → appropriate allow/deny

**INVEST checklist**: Independent, Negotiable, Valuable, Estimable, Small, Testable

---

## Interview Question Banks

**PM Hat** (ask first):
- What problem does this solve? Who has this problem today?
- What does success look like? What metric will you track?
- MoSCoW priority? (Must / Should / Could / Won't this release)
- What's explicitly out of scope?

**Dev Hat** (ask second):
- What are the edge cases? What happens when it fails?
- Performance expectations: load, latency, throughput targets?
- Security considerations: auth, data sensitivity, abuse vectors?
- What existing systems does this integrate with?
- What are the non-obvious constraints?

---

## Output Spec Template

```markdown
# Feature Specification: [Feature Name]

## Overview
As a [user type], I want [capability] so that [benefit].

## Functional Requirements (EARS)
FR-XXX-001: [EARS requirement]
FR-XXX-002: [EARS requirement]

## Non-Functional Requirements
- **Performance**: [latency, throughput, load targets]
- **Security**: [auth, data sensitivity, OWASP concerns]
- **Reliability**: [uptime, error rate, retry behavior]

## Acceptance Criteria
### AC-001: [Happy path name]
Given [precondition]
When [action]
Then [result]

### AC-002: [Error case name]
...

## Error Handling
| Error | Condition | Response |
|-------|-----------|----------|
| 400   | Validation failure | { error, details } |
| 401   | Auth failure | { error: "Unauthorized" } |
| 404   | Not found | { error: "Not found" } |

## Implementation TODO
- [ ] [Backend task]
- [ ] [Frontend task]
- [ ] [Tests]
- [ ] [Documentation]
```

Save as: `specs/{feature_name}.spec.md`

---

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/feature-forge

