# ccstack

My Claude Code stack. 16 slash commands for the workflows I actually use, distilled from 34K+ prompts.

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

## Installation

Skills live in `~/Code/rs-dev/` and are symlinked into `~/.claude/commands/`. Edit here; changes take effect on next session start.

```bash
./setup   # installs all symlinks at once
```

To add a new skill manually:

```bash
ln -sf ~/Code/rs-dev/my-skill.md ~/.claude/commands/my-skill.md
```

Subagents (invoked by the Agent tool, not directly by users) go in `~/.claude/agents/` as plain `.md` files without frontmatter.

**Skill frontmatter** — every skill needs this header:

```markdown
---
description: "trigger phrases — what the user types to invoke this"
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
---
```

## Performance Optimization Workflow

These two skills work as a pair. The golden artifacts corpus is what separates "isomorphic" from "probably fine."

```
Pass 1: /golden-artifacts           ← freeze current behavior as reference (one-time per project)
Pass 2: /extreme-software-optimization [module]   ← profile, optimize, prove equivalence
Pass 3: /extreme-software-optimization [module]   ← next Pareto layer
Pass N: repeat until STATUS: BASELINE_ONLY
```

**Typical gains per pass:**
- Pass 1 — Algorithmic improvements: 10x+
- Pass 2 — Allocation elimination: 2–5x
- Pass 3 — Cache/data layout: 1.5–3x
- Pass N — Diminishing returns

**Three-lock gate** — every optimization must pass all three before it is kept:
1. **Benchmark** — measurably faster (≥5% micro, ≥2% end-to-end)
2. **Oracle** — byte-exact match against golden corpus
3. **Tests** — full test suite green

Auto-reverts anything that fails any lock. The experiment log tracks all attempts — including reverted ones — so repeated runs don't re-try the same dead ends.

**When to re-run `/golden-artifacts`:** after any intentional behavior change, run with `REGENERATE=1` to refreeze the baseline before the next optimization pass.

## Design

- **Pure Markdown** — no build step, no runtime dependencies
- **Symlinks from source** — edit here, changes take effect immediately
- **Iron Laws** — each skill states its core principle upfront
- **Fix-First** — AUTO-FIX trivial issues, ASK for judgment calls
- **Verification gates** — no completion claims without evidence
- **Structured output** — DONE / DONE_WITH_CONCERNS / BLOCKED
- **Search before building** — before designing any non-trivial solution, search for what exists. The 1000x engineer's first instinct is "has someone already solved this?" not "let me design it from scratch." But stay alert for eureka moments — sometimes first-principles reasoning reveals the tried-and-true is wrong, and that's where 11/10 products come from

**Performance optimization**

| Command | What it does |
|---------|-------------|
| `/golden-artifacts [module]` | Freeze current behavior as a verifiable reference corpus. Run once per project before optimizing. |
| `/extreme-software-optimization [module]` | Profile-driven optimization loop with isomorphism proofs. Applies every algorithmic trick; auto-reverts anything that doesn't pass the three-lock gate. Repeat until `BASELINE_ONLY`. |

**Deep audit (blacklight)**

| Command | What it does |
|---------|-------------|
| `/rs-blacklight-audit` | Reveal hidden debt: magic numbers, TODOs, unfinished code, stubs. |
| `/rs-blacklight-plan` | Convert findings to tracked beads with dependencies. |
| `/rs-blacklight-fix` | Team of parallel agents executes beads in optimal order. |
