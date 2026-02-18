---
type: always_apply
---

# Development Principles

- **YAGNI**: Don't implement features not currently needed
- **KISS**: Prefer simple solutions over complex ones
- **DRY**: Eliminate code duplication

## File Standards

- Use kebab-case for file names — descriptive and self-documenting so LLMs understand purpose from filename alone
- Keep files under 200 lines — split into modules when larger
- Check for existing modules before creating new ones
- Analyze logical separation boundaries before modularizing

## Code Quality

- No syntax errors; code must compile/run before marking complete
- Use try-catch for error handling; cover security standards
- Follow security standards: validate inputs, sanitize outputs
- Write self-documenting code; add comments for complex logic only
- Prioritize functionality and readability over strict style enforcement
- Do NOT create new "enhanced" copies of files — update existing files directly
- Always implement real code, not mocks or simulations

## Pre-Commit Rules

- Run linting before commit; run tests before push
- Never commit secrets, `.env` files, API keys, or credentials to git
- Use conventional commit format: `feat:`, `fix:`, `docs:`, `refactor:`, etc.
- Keep commits focused on actual code changes (no AI references in messages)

## Implementation Rules

- Always read `./README.md` before planning or implementing anything
- Follow codebase structure and standards in `./docs` directory
- Run compile/build check after every significant code change
- After every implementation, search for downstream callers that need updating
- Sacrifice grammar for concision in reports; list unresolved questions at the end

