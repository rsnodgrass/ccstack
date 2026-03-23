# rs-dev

Personal Claude Code skills distilled from 34K+ prompts across 577 projects.

## Install

```bash
cd ~/Code/rs-dev && ./setup
```

Symlinks all `rs-*.md` files into `~/.claude/commands/`. Idempotent — safe to re-run.

## Skills

| Skill | Purpose | Tier |
|-------|---------|------|
| `/rs-check` | Pre-merge quality gates (typecheck + lint + build in parallel) | 1 |
| `/rs-review` | Expert-routed code review (auto-dispatches specialist agents) | 1 |
| `/rs-test` | Test improvement — fix, add, or improve tests | 1 |
| `/rs-arch` | Architecture review against project conventions | 1 |
| `/rs-debug` | Structured root-cause debugging (no fixes without investigation) | 1 |
| `/rs-simplify` | Over-engineering detection and removal | 2 |
| `/rs-ship` | Session completion — close beads, commit, push, sync | 2 |
| `/rs-explore` | CodeGraph-powered codebase exploration | 2 |
| `/rs-release` | Cross-project release readiness validation | 2 |
| `/rs-contract` | Go struct vs TS interface contract validation | 2 |
| `/rs-deps` | Dependency and import analysis | 3 |
| `/rs-standup` | Standup report from git history + beads | 3 |
| `/rs-onboard` | Codebase onboarding guide generator | 3 |

## Design

- **Pure Markdown** — no build step, no runtime dependencies
- **Symlinks from source** — edit here, changes take effect immediately
- **gstack-inspired patterns** — Iron Laws, Fix-First, phased workflows, structured output
- **Completion status protocol** — DONE / DONE_WITH_CONCERNS / BLOCKED

## Patterns Used

Each skill follows consistent prompt engineering patterns:

1. **Iron Law** — core principle stated upfront
2. **Stop/don't-stop conditions** — clear automation boundaries
3. **Fix-First** — AUTO-FIX trivial issues, ASK for judgment calls
4. **Verification gates** — no completion claims without evidence
5. **Structured output** — consistent report format per skill
