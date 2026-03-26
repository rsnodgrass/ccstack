# ccstack — Claude Code Skills

## What This Is

A curated set of Claude Code slash commands (`/rs-*`) for engineering workflows. Pure markdown, no build step, installed via symlinks.

## Editing Skills

- Each skill is a single `.md` file in the repo root
- Changes take effect immediately (symlinked to `~/.claude/commands/`)
- Run `./setup` after adding new skill files

## Skill Structure

Every skill follows this pattern:

```
---
description: "..."
allowed-tools: [...]
---

# Title

## Iron Law
**NON-NEGOTIABLE PRINCIPLE IN ALL CAPS.**

## Phase N: ...
## Stop Conditions
## Report
```

## Core Principles

### Search Before Building

Before designing any solution that involves concurrency, parallelism, scheduling, infrastructure, or a pattern you haven't used in this runtime/framework before:

1. **Search first** — "has someone already solved this?" Check runtime built-ins, framework features, and established best practices.
2. **Understand the landscape** — know what the tried-and-true approach is and why people use it.
3. **Apply first-principles reasoning** — sometimes the conventional approach is deeply wrong. When first-principles thinking reveals a clear reason why the existing solutions fail, that's a eureka moment. This is where 11/10 products come from. Be on the lookout for it.

The 1000x engineer's instinct: search first, understand what exists, then decide whether to use it or intentionally diverge with reasoning.

**Rule of thumb**: if you're about to write >20 lines of infrastructure code, search first.

### Iron Laws

Each skill states its non-negotiable principle upfront. These are not suggestions — they define the skill's identity and cannot be violated.

### Fix-First

AUTO-FIX trivial issues (formatting, unused imports, simple lint). ASK for judgment calls (security, architecture, design decisions). Never block on something you can fix yourself.

### Verification Gates

No completion claims without evidence. Tests must pass, builds must succeed, reproduction must confirm the fix.

## Conventions

- Beads (`bd`) for task tracking, never TodoWrite or TaskCreate
- Structured output: DONE / DONE_WITH_CONCERNS / BLOCKED
- Every finding must be actionable — no vague "consider doing X"
- File:line references for all code claims
