# Augment Skills

52 production-ready AI skills for **[Augment Code](https://augmentcode.com/)** — drop-in workspace rules that give Augment's AI expert-level knowledge across frontend, backend, DevOps, design, databases, testing, and more.

## What's Included

| Category | Skills |
|---|---|
| **Frontend** | React, Next.js, Vue, Svelte, Three.js, Remotion, Shader (GLSL) |
| **Backend** | Node.js, Python, Go, NestJS, FastAPI, Django |
| **Mobile** | React Native, Flutter, SwiftUI, Jetpack Compose |
| **Databases** | PostgreSQL, MongoDB, schema design, migrations |
| **DevOps** | Docker, Kubernetes, GCP, Cloudflare, CI/CD |
| **AI / MCP** | Google ADK, MCP builder, MCP management |
| **Design** | UI/UX Pro, shadcn/ui, frontend design |
| **Documents** | Word, PDF, PowerPoint, Excel |
| **Payments** | Stripe, Paddle, Polar, SePay |
| **Dev workflow** | Debug, Fix, Code review, Planning, Git, Scout |
| **Auth** | Better Auth (email, OAuth, 2FA, passkeys) |
| **Content** | Copywriting, Markdown viewer, Repomix |

## How It Works

Augment reads `.augment/rules/` automatically. Each skill file contains:

- **Frontmatter** — tells Augment *when* to activate (trigger description)
- **Body** — tells the AI *what to do* (expert instructions)

Skills with `type: agent_requested` are loaded on-demand when the task context matches. Skills with `type: always_apply` are always active.

## Quick Install

Copy `.augment/` and the root files into your project:

```bash
# Clone into a temp dir and copy rules into your project
git clone https://github.com/Alexttdk/augment-skills.git /tmp/augment-skills
cp -r /tmp/augment-skills/.augment /path/to/your-project/
cp /tmp/augment-skills/AGENTS.md /path/to/your-project/
cp /tmp/augment-skills/orchestration.md /path/to/your-project/
```

Or add as a git subtree:

```bash
git subtree add --prefix=.augment-skills \
  https://github.com/Alexttdk/augment-skills.git main --squash
```

## Project Structure

```
augment-skills/
├── .augment/
│   └── rules/                    ← Auto-loaded by Augment Code
│       ├── development-principles.md   (always_apply)
│       ├── privacy-security.md         (always_apply)
│       ├── workflow-orchestration.md   (agent_requested)
│       ├── agent-personas.md           (agent_requested)
│       └── skill-*.md × 52            (agent_requested)
├── AGENTS.md                     ← Augment entry point
├── orchestration.md              ← Multi-agent coordination
├── system-prompt.md              ← Task delegation patterns
└── README.md
```

## Core Files

| File | Purpose |
|---|---|
| `AGENTS.md` | Entry point — project overview, rules table, skill catalog |
| `orchestration.md` | How agents coordinate (sequential vs parallel, handoff template) |
| `system-prompt.md` | Who gets which task, when to delegate, what to pass forward |
| `agent-personas.md` | Thinking modes: Planner, Researcher, Tester, Reviewer, Debugger |
| `workflow-orchestration.md` | 6-phase dev workflow: Research → Implement → Test → Review → Docs → Debug |

## Development Principles

- **YAGNI** — don't implement features not currently needed
- **KISS** — prefer simple solutions over complex ones
- **DRY** — eliminate code duplication

## Contributing

1. Fork this repository
2. Add or improve a skill in `.augment/rules/skill-{name}.md`
3. Follow the frontmatter format: `type: agent_requested` + clear `description`
4. Submit a Pull Request

## License

MIT — see [LICENSE](LICENSE)