---
description: |
  Convert /rs-blacklight audit findings into structured beads tasks with
  dependencies, priorities, and self-contained descriptions. Use when
  asked to "plan the fixes", "create tasks from audit", "blacklight plan",
  "track findings", or "make beads from audit". Run after /rs-blacklight-audit.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# /rs-blacklight-plan — Convert Findings to Beads

## Iron Law

**EVERY TASK MUST BE SELF-CONTAINED. A coworker picking up a bead with zero prior context must be able to complete it from the description alone.**

No "see above", no "as discussed", no implicit knowledge. Each bead carries its own context.

---

## Phase 1: Load Findings

1. Look for the most recent BLACKLIGHT AUDIT REPORT in conversation history.
2. If no report exists, tell the user to run `/rs-blacklight-audit` first and STOP.
3. Parse all findings by severity and category.

---

## Phase 2: Group into Logical Tasks

Findings are NOT 1:1 with tasks. Group them into coherent work units:

### Grouping Rules

1. **Same file, same category** → single task (e.g., "Extract magic numbers in config/server.go")
2. **Same constant used across files** → single task covering all locations
3. **Related TODOs** → single task if they describe parts of the same feature
4. **Independent findings** → separate tasks

### Task Sizing

- Each task should be completable in **one focused session** (30-90 minutes)
- If a group is too large, split by file or sub-area
- If a group is trivially small (< 5 minutes), merge with a related task

---

## Phase 3: Build Dependency Graph

Determine execution order:

1. **CRITICAL findings first** — always priority 0 or 1
2. **Shared constants/config** before consumers — if you're extracting a config system, do that before individual constant extractions
3. **Core modules before periphery** — fixes in shared libraries before application code
4. **Independent tasks in parallel** — no false dependencies

Map dependencies explicitly:
```
Task A (extract config system) ← no deps
Task B (fix magic numbers in server) ← depends on A
Task C (fix TODOs in auth) ← no deps (independent)
Task D (fix TODOs in handlers) ← depends on C if related
```

---

## Phase 4: Create Beads

For each task, create a bead with a comprehensive, self-contained description.

### Priority Mapping

| Audit Severity | Beads Priority | Rationale |
|---------------|----------------|-----------|
| CRITICAL | 0 (critical) | Security or runtime risk |
| HIGH | 1 (high) | Blocks progress or causes confusion |
| MEDIUM | 2 (medium) | Improves quality when file is touched |
| LOW | 3 (low) | Cleanup opportunity |

### Description Template

Every bead description MUST include:
1. **What:** One sentence summary of the work
2. **Why:** Why this matters (not just "found by audit")
3. **Where:** Exact file paths and line numbers
4. **How:** Specific fix approach (before/after examples where helpful)
5. **Verification:** How to confirm the fix is correct
6. **Context:** Any conventions or patterns from CLAUDE.md/AGENTS.md that apply

### Create Commands

Use parallel subagents when creating many beads to avoid serialization bottleneck.

```bash
bd create \
  --title="Remove hardcoded API key in auth/client.go" \
  --description="**What:** Remove hardcoded API key on line 47 and replace with env var lookup.

**Why:** Hardcoded credentials are a security risk — if this repo becomes public or the key rotates, the code breaks silently.

**Where:** auth/client.go:47

**How:**
- Replace \`apiKey := \"sk-abc123\"\` with \`apiKey := os.Getenv(\"SERVICE_API_KEY\")\`
- Add validation: fail fast if env var is empty
- Update .env.example with the new variable name

**Verification:** \`go build ./...\` passes, \`grep -rn 'sk-abc123' .\` returns nothing, service starts with env var set.

**Context:** Project uses 12-factor env var pattern (see CLAUDE.md)." \
  --type=bug \
  --priority=0
```

For tasks with dependencies:
```bash
bd dep add <child-id> <parent-id>
```

---

## Phase 5: Verify and Report

```bash
bd list --status=open 2>/dev/null
```

```
BLACKLIGHT PLAN REPORT
=======================
Tasks created:        N
  Critical (P0):      X
  High (P1):          Y
  Medium (P2):        Z
  Low (P3):           W

Dependency chains:    [list root → leaf chains]

Execution order (recommended):
  1. <task-id> — <title> (no deps)
  2. <task-id> — <title> (no deps, parallel with #1)
  3. <task-id> — <title> (depends on #1)
  ...

NEXT STEP: /rs-blacklight-fix

STATUS: PLANNED | PARTIALLY_PLANNED (reason)
```

---

## Stop Conditions

**Only stop for:**
- Ambiguous findings that could be grouped multiple ways — ask user for preference
- Findings that require architectural decisions before they can be tasked

**Never stop for:**
- Large number of findings (create tasks systematically)
- Uncertainty about priority (use the mapping table, adjust later)
