---
type: agent_requested
description: ClaudeKit development workflow phases for implementing features, planning tasks, debugging, and complex multi-step work
---

# Workflow Orchestration

## Development Phases

Execute these phases sequentially for feature work:

### 1. Research & Planning
- Read `./README.md` and relevant docs in `./docs` for project context
- Use sequential-thinking for complex multi-step problems
- Look up current library docs before implementing
- Create a plan with TODO tasks in `./plans/` directory
- Identify which skills are needed before starting implementation

### 2. Implementation
- Follow the plan from the Planning phase exactly
- Write clean, readable, maintainable code
- Handle edge cases and error scenarios
- Run compile/build check after each significant change
- Update existing files — do NOT create new "enhanced" copies

### 3. Testing
- Write unit tests for all new functions
- Fix all failing tests before proceeding — never skip or mock to bypass
- Validate actual functionality, not simulated behavior
- Achieve meaningful test coverage on new code

### 4. Code Review
- Self-review for security vulnerabilities, DRY violations, and maintainability issues
- Verify no hardcoded credentials, no syntax errors, no unused imports

### 5. Documentation
- Update `./docs/codebase-summary.md` if architecture changed
- Update relevant docs in `./docs/` if APIs or behavior changed
- Keep docs concise and accurate

### 6. Debugging (when issues arise)
- Trace call stack backwards from symptom to root cause
- Never mask symptoms — fix the actual underlying issue
- Re-run tests after fix; repeat from Step 3 if tests fail

## Orchestration Patterns

**Sequential** (for dependent tasks): Research → Plan → Implement → Test → Review → Finalize

**Parallel reasoning** (for independent sub-problems): Analyze multiple concerns simultaneously before integrating

