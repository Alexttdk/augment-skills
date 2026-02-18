---
type: agent_requested
description: Apply when building CLI tools — command architecture, argument/flag conventions, config file handling, interactive prompts, output formatting, progress indicators, and error messages.
---

# CLI Development

## When to Use
- Building command-line tools or terminal applications
- Designing command hierarchy and subcommand structure
- Implementing argument parsing, flags, environment variables
- Adding interactive prompts, progress bars, colored output
- Writing actionable error messages and exit codes
- Supporting CI/CD non-interactive execution

## Command Architecture

```
mycli                            ← root command
├── init [options]               ← simple command
├── config
│   ├── get <key>               ← nested subcommand
│   ├── set <key> <value>
│   └── list
├── deploy [environment]         ← positional arg
│   ├── --dry-run               ← boolean flag
│   ├── --force
│   └── --config <file>         ← option with value
└── plugins
    ├── install <name>
    └── remove <name>
```

**Principles:**
- Commands are nouns (subjects), subcommands are verbs: `mycli deploy` not `mycli run-deploy`
- Keep hierarchy shallow (max 3 levels)
- Every command must have `--help` output and obey `--version` on root

## Argument Conventions

```bash
# Positional args: required, named clearly
mycli deploy production

# Boolean flags: presence = true
mycli deploy --force --dry-run

# Short + long forms
mycli -v --verbose
mycli -c config.yml --config config.yml

# Variadic args
mycli install pkg1 pkg2 pkg3

# Negation
mycli --no-color
```

**Flag naming rules:**
- Use `--kebab-case` for multi-word flags: `--dry-run`, `--output-format`
- Reserve `-v` for verbose, `-h` for help, `-V` for version
- `--verbose` prints human steps, `--debug` prints internals

## Configuration Layers

Priority order (highest wins):

```
1. CLI flags          --config ./custom.yml          explicit intent
2. Environment vars   MYCLI_ENV=production           runtime context
3. Project config     .myclirc, mycli.config.js      project scope
4. User config        ~/.config/mycli/config.yml     user preference
5. System config      /etc/mycli/config.yml          system defaults
6. Hard-coded defaults                               fallback
```

```javascript
// Config resolution pattern
const config = {
  ...systemDefaults,
  ...loadSystemConfig('/etc/mycli'),
  ...loadUserConfig('~/.config/mycli'),
  ...loadProjectConfig('.myclirc'),
  ...loadEnvVars('MYCLI_'),
  ...parseCliFlags(argv),
};
```

Sensitive data (tokens, passwords) → `~/.mycli/credentials.json` with `chmod 600`.

## Interactive Prompts

**Rule: detect CI/non-TTY and skip prompts:**
```javascript
const isInteractive = process.stdout.isTTY && !process.env.CI;

if (isInteractive) {
  environment = await select({ message: 'Target environment?',
    choices: ['development', 'staging', 'production'] });
} else {
  if (!options.environment)
    throw new Error('--environment required in non-interactive mode');
}
```

**Prompt types:**
```
Text:     Project name: my-app
Select:   ❯ development  staging  production
Multi:    ◉ TypeScript  ◯ ESLint  ◉ Prettier
Confirm:  Deploy to production? (y/N)   ← default No (safer)
Password: ********
```

**Show keyboard hints** ("Use arrows", "Space to select"). Allow Ctrl+C. Validate immediately.

## Output Formatting

### Colors — semantic use only
```
Red     → errors, failures, destructive actions
Yellow  → warnings, deprecations
Green   → success, completion
Blue    → information, hints
Cyan    → commands, code snippets
Gray    → metadata, timestamps
```

**Disable when piped:** `if (!process.stdout.isTTY || process.env.NO_COLOR) disableColors()` — respect the [NO_COLOR](https://no-color.org/) standard.

Always pair color with symbol — don't rely on color alone:
```
✓ Build successful   (green + symbol)
✗ Build failed       (red + symbol)
⚠ Deprecated flag    (yellow + symbol)
```

### Progress Indicators
```
Determinate:   [████████████░░░░] 60% · 120/200 MB · 2.4 MB/s · ETA 33s
Indeterminate: ⠋ Connecting to API...
Multi-step:    ✓ Dependencies installed (2.3s)
               ✓ Build complete (8.1s)
               ⠋ Running tests...
               ⏳ Deploy (pending)
```

### Tables and Structured Output
```
Fancy (TTY):    ┌─────────────┬──────────┐
                │ Environment │ Status   │
                ├─────────────┼──────────┤
                │ production  │ ✓ Active │

Plain (piped):  Environment  Status
                production   Active

JSON flag:      mycli list --json  → machine-readable output
```

## Error Messages

**Pattern: Context → Problem → Solution**

```
✗ Error: Config file not found

Searched locations:
  • ./mycli.config.yml
  • ~/.config/mycli/config.yml

Run 'mycli init' to create a config, or use --config to specify a path.
```

**Rules:**
- Be specific: "Port 3000 already in use" not "Connection error"
- Show searched paths, expected values, suggestions
- Use plain language — not `ENOENT`, not stack traces (save for `--debug`)
- Suggest the exact command to fix the problem

## Exit Codes

```javascript
const EXIT = {
  SUCCESS:          0,   // All good
  GENERAL_ERROR:    1,   // Runtime error
  MISUSE:           2,   // Invalid arguments / wrong usage
  PERMISSION:      77,   // Permission denied
  NOT_FOUND:      127,   // Command/resource not found
  SIGINT:         130,   // User pressed Ctrl+C
};
```

Always exit with non-zero on error — piped scripts depend on it.

## Performance

- Startup time target: **< 50ms** — lazy-load heavy deps
- Lazy loading: `if (cmd === 'deploy') { const d = require('./deploy'); ... }`
- Cache API responses: `~/.mycli/cache/` with TTL
- Check for updates non-blocking: `checkUpdates().catch(() => {})` — never block user

## Constraints

**MUST DO:** `--help` + `--version` on root · consistent `--kebab-case` flags · graceful `SIGINT` (Ctrl+C) · validate input early · support piped/non-interactive mode · cross-platform (Windows, macOS, Linux) · exit codes

**MUST NOT:** Print to stdout when output will be piped (use stderr for status) · use colors when not a TTY · break existing command signatures · require interactive input in CI · hardcode paths or platform-specific separators

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/cli-developer

