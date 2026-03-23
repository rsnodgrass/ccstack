---
description: |
  Codebase onboarding guide generator. Creates a focused guide for a specific
  module or the entire project. Use when asked to "explain this code", "how
  does this work", "onboard me", "give me context on", "what does this module do",
  or "help me understand". Proactively suggest when the user is new to a part
  of the codebase.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# Codebase Onboarding

## Iron Law

**SHOW THE MAP BEFORE THE TERRITORY. Start with purpose, then structure, then details.**

---

## argument-hint

<module path or 'all' for full project overview>

---

## Phase 1: Identify Scope

If a specific module/path is given, scope to that. Otherwise, generate a project-level overview.

```bash
# project-level markers
[ -f README.md ] && echo "HAS_README"
[ -f CLAUDE.md ] && echo "HAS_CLAUDE_MD"
[ -f go.mod ] && echo "GO_PROJECT"
[ -f package.json ] && echo "NODE_PROJECT"
[ -f turbo.json ] && echo "MONOREPO"
ls -d apps/*/ 2>/dev/null && echo "HAS_APPS"
ls -d packages/*/ 2>/dev/null && echo "HAS_PACKAGES"
```

---

## Phase 2: Understand Purpose

Read key files to understand what this code does:
1. README.md (if exists)
2. CLAUDE.md (project conventions)
3. Main entry points (main.go, index.ts, app.ts)
4. Package.json or go.mod (dependencies reveal purpose)

---

## Phase 3: Map Structure

Generate a structural overview:

```bash
# directory tree (depth 2-3)
find <scope> -type f -name "*.go" -o -name "*.ts" -o -name "*.tsx" | head -50
```

Categorize files by role:
- Entry points
- Handlers / Routes
- Services / Business logic
- Data access / Repositories
- Shared utilities
- Tests
- Configuration

---

## Phase 4: Identify Patterns

Read 2-3 representative files to identify:
- Coding patterns used (dependency injection, middleware chains, etc.)
- Error handling approach
- Testing patterns
- Configuration approach (env vars, config files)

---

## Phase 5: Generate Guide

```
ONBOARDING: <module or project name>

PURPOSE:
<1-2 sentences: what this does and why it exists>

STRUCTURE:
<directory tree with annotations>
  apps/api-go/           # Go backend API
    internal/handlers/    # HTTP handlers (thin — delegate to services)
    internal/services/    # Business logic
    internal/repo/        # Database access (sqlc-generated)
  packages/ids-go/       # Shared ID types with prefixes

KEY FILES:
- <file>:<line> — <what it does and why it matters>
- <file>:<line> — <what it does and why it matters>

PATTERNS:
- <pattern name>: <brief explanation with file:line example>

HOW TO RUN:
  <specific commands>

HOW TO TEST:
  <specific commands>

COMMON PITFALLS:
- <thing that trips people up> — <how to avoid it>

RELATED DOCS:
- <path to relevant documentation>
```
