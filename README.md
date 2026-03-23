# ccstack

My Claude Code stack. 13 slash commands for the workflows I actually use, distilled from 34K+ prompts.

## Install

```bash
./setup
```

## Commands

**Daily drivers**

| Command | What it does |
|---------|-------------|
| `/rs-check` | Typecheck + lint + build in parallel. Auto-fixes formatting. |
| `/rs-review` | Routes changed files to the right specialist agent. |
| `/rs-test [fix\|add\|improve]` | Fix broken tests, add missing coverage, or both. |
| `/rs-debug <error>` | Root-cause investigation. No fixes without evidence. |
| `/rs-simplify` | Find and remove over-engineering. |

**Before shipping**

| Command | What it does |
|---------|-------------|
| `/rs-arch [module]` | Architecture review — layers, dependencies, god files. |
| `/rs-contract` | Validate Go structs match TS interfaces. Catches PII leaks. |
| `/rs-release` | Full pre-release checklist — gates, blockers, migrations, env vars. |
| `/rs-ship` | Close beads, commit, push, sync. Land the plane. |

**When you need context**

| Command | What it does |
|---------|-------------|
| `/rs-explore <question>` | CodeGraph-powered codebase search with call chain tracing. |
| `/rs-deps` | Stale packages, circular imports, layer violations. |
| `/rs-standup` | Generate standup from git log + beads. |
| `/rs-onboard [module]` | Explain a module: purpose, structure, patterns, pitfalls. |
