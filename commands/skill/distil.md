---
description: ⚡⚡⚡ Distil skills from a GitHub repo into augment-skills format
argument-hint: [source-github-url] [target-github-url]
---

Think harder.
Activate `planning`, `skill-creator`, `repomix`, and `research` skills.

## Your mission
Distil skills from a source GitHub repository and contribute them as augment-skills to a target repository.

## Variables
SOURCE_REPO: $1 (GitHub URL of source skills repo)
TARGET_REPO: $2 (defaults to `https://github.com/Alexttdk/augment-skills`)

## Pre-Creation Check (Active vs Suggested Plan)

Check the `## Plan Context` section in the injected context:
- If "Plan:" shows a path → Active plan exists. Ask user: "Continue with this? [Y/n]"
- If "Suggested:" shows a path → Branch-matched hint only. Ask if they want to activate or create new.
- If "Plan: none" → Create new plan using naming from `## Naming` section.

## Workflow

### Step 0 — Clarification
- If `SOURCE_REPO` is empty, use `AskUserQuestion` to ask for the source GitHub URL.
- If `TARGET_REPO` is empty, default to `https://github.com/Alexttdk/augment-skills`.

### Step 1 — Setup
- Clone `TARGET_REPO` to `/tmp/augment-skills` if not already present: `git clone $TARGET_REPO /tmp/augment-skills`
- Use `repomix` to pack `SOURCE_REPO` for analysis: `npx repomix --remote $SOURCE_REPO --output /tmp/source-repo-packed.xml`
- Create plan directory using `Plan dir:` from `## Naming` section, then run `node $HOME/.claude/scripts/set-active-plan.cjs {plan-dir}`

### Step 2 — Research (Parallel)
Spawn 2 `researcher` agents in parallel (max 5 tool calls each):

- **Researcher 1 — Source Catalog**: Analyze `/tmp/source-repo-packed.xml`. List all skill names, categories, brief description of each, key unique patterns and depth of each skill.
- **Researcher 2 — Gap Analysis**: Compare source skills against existing skills in `/tmp/augment-skills/.augment/rules/` and `$HOME/.claude/skills/`. Identify skills NOT yet present. For each gap skill, note uniqueness and recommended depth (fuller ~10-15KB vs lean ~3-7KB).

Save reports to `{plan-dir}/research/`.

### Step 3 — Plan
Pass both research reports to `planner` subagent:
- Group gap skills into 2-5 phases by category/theme (thinking/specs, testing/API, infra/ops, security/arch, AI/ML, etc.)
- Each phase: list skills, depth treatment, target filenames (`skill-{name}.md`)
- Depth heuristic: core/unique skills → fuller; utility/common skills → lean
- Output: `{plan-dir}/plan.md` + `{plan-dir}/phase-XX-*.md` per phase

### Step 4 — Validation (Optional)
After plan creation, ask: "Validate this plan with a brief interview? [Y/n]"
If yes, execute `/plan:validate {plan-dir}` SlashCommand.

### Step 5 — Implementation
Execute phases using `fullstack-developer` subagents:
- **Wave 1**: Launch phases 01 + 02 in parallel
- **Wave 2**: Launch remaining phases in parallel (after Wave 1 completes)
- Each agent writes output to `/tmp/augment-skills/.augment/rules/skill-{name}.md`
- Each skill file: YAML frontmatter `type: agent_requested` + concise self-contained markdown

### Step 6 — Contribute
After all waves complete:
```bash
cd /tmp/augment-skills
git add .augment/rules/skill-*.md
git commit -m "feat: distil skills from {SOURCE_REPO_NAME}"
git push origin main
```
Report: number of skills added, file sizes, commit hash.

## Output Requirements
- Every skill file must be self-contained (no external references required)
- No duplicate skills — skip any skill already present in target repo or `$HOME/.claude/skills/`
- Skill files follow augment format: `type: agent_requested` frontmatter + practical workflow instructions
- Token efficiency: fuller skills 10-15KB max, lean skills 3-7KB max

## Important Notes
**IMPORTANT:** Activate skills catalog at `$HOME/.claude/skills/*` intelligently throughout the process.
**IMPORTANT:** Sacrifice grammar for concision in reports.
**IMPORTANT:** In reports, list any unresolved questions at the end.
**IMPORTANT:** Ensure token efficiency while maintaining high quality.

