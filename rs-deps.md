---
description: |
  Dependency and import analysis. Checks for circular imports, stale packages,
  layer violations, and unused dependencies. Use when asked to "check dependencies",
  "find circular imports", "update packages", "dependency audit", or "are our deps
  up to date?". Proactively suggest during major version upgrades or security audits.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# Dependency & Import Analysis

## Iron Law

**EVERY DEPENDENCY IS A LIABILITY. Justify its existence or remove it.**

---

## Phase 1: Detect Package Managers

```bash
[ -f go.mod ] && echo "GO"
[ -f package.json ] && echo "NODE"
[ -f pyproject.toml ] && echo "PYTHON"
```

---

## Phase 2: Staleness Check

### Node.js
```bash
npx npm-check-updates 2>/dev/null || ncu 2>/dev/null
```

### Go
```bash
go list -m -u all 2>/dev/null | grep '\[' | head -20
```

### Python
```bash
pip list --outdated 2>/dev/null | head -20
```

Report packages with major version gaps (potential breaking changes) vs minor/patch updates.

---

## Phase 3: Circular Import Detection

### Go
```bash
go vet ./... 2>&1 | grep -i "import cycle"
```

### TypeScript
```bash
# trace import chains
grep -rn "^import" --include="*.ts" --include="*.tsx" | grep -v node_modules | grep -v '.d.ts'
```

For TypeScript, build an import graph and detect cycles by tracing A→B→C→A patterns.

---

## Phase 4: Layer Violation Detection

Check that imports follow the dependency direction:
- `handlers/` → `services/` → `repo/` → DB (never reverse)
- `apps/` → `packages/` (never reverse)
- `internal/` never imported from outside its module

```bash
# Go: check for reverse dependencies
grep -rn 'import.*handlers' --include="*.go" internal/services/ internal/repo/
grep -rn 'import.*services' --include="*.go" internal/repo/
```

---

## Phase 5: Unused Dependency Detection

### Node.js
```bash
npx depcheck 2>/dev/null || echo "depcheck not available"
```

### Go
```bash
# unused imports are compile errors in Go, but check go.mod for unused modules
go mod tidy -v 2>&1
```

---

## Phase 6: Report

```
DEPENDENCY REPORT
Package Manager: [Go | Node | Python | Multiple]

STALENESS:
| Package        | Current | Latest | Gap    | Risk   |
|----------------|---------|--------|--------|--------|
| typescript     | 5.3.0   | 5.7.0  | minor  | LOW    |
| drizzle-orm    | 0.28.0  | 0.35.0 | major  | HIGH   |

Circular Imports: NONE | [list with file paths]
Layer Violations: NONE | [list with details]
Unused Dependencies: NONE | [list]
STATUS: HEALTHY | NEEDS_ATTENTION | CRITICAL
```
