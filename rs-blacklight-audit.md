---
description: |
  Deep codebase audit — reveal hidden debt. Scans for hardcoded constants,
  TODO comments, unfinished code ("will"/"would"), stale patterns, and
  architectural blind spots. Use when asked to "audit", "scan for debt",
  "what's broken", "find TODOs", "code health check", "blacklight",
  or "reveal hidden issues". Proactively suggest on unfamiliar codebases
  or before major refactors.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
argument-hint: "[scope — module path, or 'all' for full codebase (default: all)]"
---

# /rs-blacklight-audit — Deep Codebase Audit

## Iron Law

**READ EVERYTHING BEFORE YOU JUDGE ANYTHING.**

Scanning without understanding produces false positives. You must understand the project's conventions, architecture, and intentional decisions before flagging violations.

---

## Phase 1: Absorb Context

1. Read ALL project documentation in this order:
   - `CLAUDE.md` (project root and any parent directories)
   - `AGENTS.md` (if exists)
   - `README.md`
   - `CONTRIBUTING.md` (if exists)
   - `docs/` directory listing (scan for architecture docs, ADRs, conventions)

2. Identify the project type and conventions:
   ```bash
   [ -f go.mod ] && echo "GO"
   [ -f package.json ] && echo "NODE"
   [ -f tsconfig.json ] && echo "TYPESCRIPT"
   [ -f pyproject.toml ] && echo "PYTHON"
   [ -f Makefile ] && echo "MAKEFILE"
   [ -f Cargo.toml ] && echo "RUST"
   ```

3. Read key structural files to understand the architecture:
   ```bash
   find . -maxdepth 2 -type d -not -path './.git/*' -not -path './node_modules/*' -not -path './vendor/*' -not -path './.venv/*' | sort
   ```

4. Note conventions that make certain patterns INTENTIONAL (e.g., "we hardcode X because..."). These are NOT findings.

---

## Phase 2: Scan — Hardcoded Constants

Search for values that should be configurable, environment-driven, or defined as named constants.

### Numeric Literals (excluding 0, 1, common defaults)
```bash
# Go: magic numbers in non-test files
grep -rn '[^a-zA-Z0-9_"]\([2-9][0-9]\{2,\}\|[0-9]\{4,\}\)[^a-zA-Z0-9_"]' --include="*.go" --exclude-dir=vendor --exclude="*_test.go" .
# TypeScript/JavaScript
grep -rn '[^a-zA-Z0-9_"]\([2-9][0-9]\{2,\}\|[0-9]\{4,\}\)[^a-zA-Z0-9_"]' --include="*.ts" --include="*.tsx" --include="*.js" --exclude-dir=node_modules .
# Python
grep -rn '[^a-zA-Z0-9_"]\([2-9][0-9]\{2,\}\|[0-9]\{4,\}\)[^a-zA-Z0-9_"]' --include="*.py" --exclude-dir=.venv .
```

### Hardcoded Strings
Look for:
- URLs, hostnames, IP addresses embedded in code (not config)
- File paths that should be configurable
- API keys, secrets, tokens (CRITICAL — flag immediately)
- Email addresses, phone numbers
- Version strings embedded in logic (not version files)

```bash
# URLs and hostnames in source code (exclude tests, docs, configs)
grep -rn 'https\?://[a-zA-Z]' --include="*.go" --include="*.ts" --include="*.py" --exclude-dir=vendor --exclude-dir=node_modules --exclude="*_test.*" --exclude="*.md" .
# hardcoded ports
grep -rn ':[0-9]\{4,5\}[^0-9]' --include="*.go" --include="*.ts" --include="*.py" --exclude-dir=vendor --exclude-dir=node_modules --exclude="*_test.*" .
```

### Timeouts, Retries, Buffer Sizes
```bash
grep -rn 'time\.Duration\|time\.Sleep\|time\.After\|setTimeout\|setInterval\|retry\|Retry\|RETRY\|timeout\|Timeout\|TIMEOUT' --include="*.go" --include="*.ts" --include="*.py" --exclude-dir=vendor --exclude-dir=node_modules .
```

**Filter out false positives:** Constants defined at package/module level with descriptive names are FINE. Only flag inline magic numbers in function bodies.

---

## Phase 3: Scan — TODO / FIXME / HACK Comments

```bash
grep -rn '\(TODO\|FIXME\|HACK\|XXX\|TEMP\|TEMPORARY\|WORKAROUND\)' --include="*.go" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.py" --include="*.rs" --include="*.yaml" --include="*.yml" --exclude-dir=vendor --exclude-dir=node_modules --exclude-dir=.venv .
```

For each finding, read the surrounding context (5 lines above and below) to understand:
- Is this a genuine incomplete item?
- Is it a known tradeoff documented intentionally?
- How stale is it? (`git log -1 -- <file>` for last modification date)

---

## Phase 4: Scan — Unfinished Code Indicators

Search for comments containing future-tense language that signals incomplete implementation:

```bash
# "will", "would", "should", "eventually", "later", "someday", "placeholder"
grep -rn '\(\/\/\|#\|\/\*\).*\b\(will \|would \|should \|eventually\|later\b\|someday\|placeholder\|not yet\|not implemented\|stub\b\)' --include="*.go" --include="*.ts" --include="*.tsx" --include="*.py" --include="*.rs" -i --exclude-dir=vendor --exclude-dir=node_modules --exclude-dir=.venv .
```

Also check for:
- Empty function bodies or `pass`/`panic("not implemented")`/`throw new Error("TODO")`
- Functions that return hardcoded values with a comment about future implementation
- Commented-out code blocks (dead code)

```bash
# empty/stub implementations
grep -rn 'panic("not implemented")\|panic("TODO")\|throw.*not implemented\|throw.*TODO\|raise NotImplementedError\|pass  #' --include="*.go" --include="*.ts" --include="*.py" --exclude-dir=vendor --exclude-dir=node_modules .
# commented-out code (lines that look like code, not prose)
grep -rn '^\s*\/\/\s*[a-zA-Z]*\.\|^\s*\/\/\s*if \|^\s*\/\/\s*for \|^\s*\/\/\s*return \|^\s*#\s*def \|^\s*#\s*class ' --include="*.go" --include="*.ts" --include="*.py" --exclude-dir=vendor --exclude-dir=node_modules .
```

---

## Phase 5: Extended Patterns

Only run these for full-codebase audits (scope = 'all'):

### Stale Dependencies
```bash
[ -f go.mod ] && go list -m -u all 2>/dev/null | grep '\[' | head -20
[ -f package.json ] && npx ncu 2>/dev/null | head -30
```

### Dead Exports
```bash
[ -d .codegraph/ ] && echo "CODEGRAPH_AVAILABLE — use codegraph_search for orphan detection"
```

### Error Swallowing
```bash
# Go: _ = err pattern
grep -rn '_ = .*err\|_ = .*Err\|if err != nil {\s*}' --include="*.go" --exclude-dir=vendor .
# JS/TS: empty catch blocks
grep -rn 'catch.*{[\s]*}' --include="*.ts" --include="*.tsx" --include="*.js" --exclude-dir=node_modules .
```

---

## Phase 6: Classify and Report

For EACH finding, classify severity and actionability:

| Severity | Meaning | Examples |
|----------|---------|---------|
| **CRITICAL** | Security risk or runtime failure | Hardcoded secrets, swallowed errors in critical paths |
| **HIGH** | Technical debt that blocks progress | Unfinished implementations behind active code paths |
| **MEDIUM** | Code quality / maintainability | Magic numbers, stale TODOs, future-tense comments |
| **LOW** | Cleanup opportunity | Commented-out code, minor naming issues |

```
BLACKLIGHT AUDIT REPORT
========================
Project:           <name>
Scope:             <all | module path>
Files scanned:     N

CRITICAL (fix immediately):
- file:line — [category] description

HIGH (fix soon):
- file:line — [category] description

MEDIUM (fix when touching the file):
- file:line — [category] description

LOW (cleanup opportunity):
- file:line — [category] description

SUMMARY:
  Hardcoded constants:     N findings
  TODO/FIXME comments:     N findings
  Unfinished code:         N findings
  Error swallowing:        N findings
  Other:                   N findings
  TOTAL:                   N findings

STATUS: CLEAN | FINDINGS_REPORTED | CRITICAL_ISSUES

NEXT STEP: /rs-blacklight-plan
```

---

## Stop Conditions

**Only stop for:**
- Repositories too large to scan in one pass (>10K files) — ask user to scope to a module

**Never stop for:**
- Files you can't parse (skip and note them)
- False positives (classify and include them — the next stage filters)
