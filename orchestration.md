# Orchestration Protocol

Multi-agent coordination guide for Augment CLI, OpenCode, and Claude Code.

---

## Agent Roles

| Role | Responsibility |
|---|---|
| Orchestrator | Breaks work into phases, delegates to specialists, integrates results |
| Planner | Creates implementation plan with TODO tasks in `./plans/` |
| Researcher | Investigates unknowns, verifies library docs, reports findings |
| Implementer | Writes code following the plan exactly |
| Tester | Runs tests, validates functionality, reports pass/fail |
| Reviewer | Code quality, security, DRY violations, maintainability |
| Debugger | Root cause analysis, call stack tracing, fix verification |
| Docs Manager | Updates `./docs/` when architecture or APIs change |

---

## Coordination Patterns

### Sequential (dependent tasks)

Use when each phase depends on the output of the previous:

```
Plan → Research → Implement → Test → Review → Finalize
```

Each agent completes fully before the next begins. Pass outputs forward as context.

### Parallel (independent tasks)

Spawn agents simultaneously when there are no file conflicts:

- Code + Tests + Docs (separate non-overlapping files)
- Multiple isolated features on different modules
- Platform-specific implementations (e.g., iOS / Android)

**Rule:** Define integration points and file ownership before starting parallel work.

---

## Context Handoff

Always include in each agent prompt:

- **Work context** — git root of the files being worked on
- **Plans path** — `./plans/` for implementation plans and TODO lists
- **Reports path** — `./plans/reports/` for agent output summaries
- **Relevant files** — list specific files the agent needs to read or modify

**Example handoff:**
```
Task: Implement auth middleware
Work context: ./
Plans: ./plans/
Reports: ./plans/reports/
Files: src/middleware/auth.ts, src/types/user.ts
Follow: ./orchestration.md
```

---

## Development Phases

| # | Phase | Key actions |
|---|---|---|
| 1 | **Plan** | Read `./README.md` + `./docs/`; create plan in `./plans/` |
| 2 | **Implement** | Follow plan exactly; compile-check after each significant change |
| 3 | **Test** | Write and run tests; fix all failures before proceeding |
| 4 | **Review** | Security, DRY, no hardcoded credentials, no unused imports |
| 5 | **Document** | Update `./docs/` if architecture or APIs changed |
| 6 | **Debug** | Trace symptom → root cause; re-run tests after fix |

---

## Skills

52 skills — auto-selected by Augment via `.augment/rules/skill-*.md` description matching.

Key skills for orchestration: `sequential-thinking`, `planning`, `debug`, `code-review`, `research`

---

## Hard Rules

- Never commit secrets, `.env` files, API keys, or credentials
- Update existing files — never create "enhanced" copies alongside originals
- Fix root causes — never mask symptoms with workarounds
- All tests must pass before declaring work complete
- Read `./README.md` before planning or implementing anything

