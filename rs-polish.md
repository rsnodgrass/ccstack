---
description: |
  Post-implementation polish. Runs three parallel expert reviews after a feature
  lands: technical writer updates docs and in-code text, principal engineer +
  refactoring specialist audits the implementation, and a test agent ensures
  behavior is consistent with config at every level (project/user/defaults).
  Use when asked to "polish", "review and clean up", "finalize this feature",
  or "make sure this is production-ready". Proactively suggest after any
  substantial feature implementation.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
---

# /rs-polish — Post-Implementation Polish

## Iron Law

**SHIPPING IS NOT THE SAME AS DONE. A feature is finished when its docs are accurate, its code is clean, and its config contracts are verified by tests.**

---

## Stop Conditions

**Only stop for:**
- Contradictory information that requires human judgment to resolve
- Missing context about intended behavior at a config level
- Test failures that reveal a real bug (not a test gap)

**Never stop for:**
- Minor doc rewording (fix it)
- Obvious simplification opportunities (fix them)
- Missing tests for clearly untested paths (write them)

---

## Phase 1: Scope the Feature

Determine what changed. Try in order until one produces output:

```bash
git diff --cached --stat
git diff --stat
git diff main --stat
```

Identify the feature surface:
- What **code** changed (Go/TS/Python files)?
- What **docs** changed or are adjacent to the change (AGENTS.md, README, inline comments, CLAUDE.md)?
- What **config** does the feature touch — and at how many levels?

**Config levels to check (adapt to project):**

| Level | What it looks like |
|-------|--------------------|
| Defaults | Hardcoded values in the binary / struct zero values |
| User | `~/.config/<app>/`, env vars, user-level settings files |
| Project | `.env`, project config files, `sqlc.yaml`, `Makefile` vars |

---

## Phase 2: Launch Three Expert Agents in Parallel

Start all three agents simultaneously — they work on the same repo snapshot and do not conflict.

### Agent A — Technical Writer

**Goal:** Every word that a developer (or AI agent) reads about this feature should be accurate, scannable, and unambiguous.

Scope:
- Inline comments and docstrings in changed files
- AGENTS.md / CLAUDE.md sections adjacent to the feature
- README, docs/, or any markdown that describes the changed behavior

For each piece of text:
1. **Accuracy check** — Does it match what the code actually does today?
2. **Audience check** — Is it in the right place (human docs vs agent instructions)?
3. **Clarity check** — Would a developer unfamiliar with this codebase understand it in one read?
4. **Completeness check** — Does it cover regeneration steps, configuration options, and common pitfalls?

Apply fixes directly. For each change, emit:
```
[DOC] path:line — Was: "..." → Now: "..." (reason)
```

### Agent B — Principal Engineer + Refactoring Specialist

**Goal:** The implementation should be as simple as possible without sacrificing correctness, and no simpler.

Run two passes:

**Pass 1 — Principal Engineer (architecture and correctness):**
- Are the right abstractions in the right layers?
- Is there a simpler model that eliminates a class of bugs?
- Are error paths correct and observable?
- Is any code doing more (or less) than its name suggests?
- Does the implementation match what the docs/comments claim it does?

**Pass 2 — Refactoring Specialist (over-engineering and duplication):**
- Helpers created for a single call site → inline them
- Unnecessary indirection or wrapper functions → collapse them
- Conditional paths that guard against impossible scenarios → remove them
- Config theater (configurability no one will ever exercise) → hardcode it
- Duplication that would be better as a shared function (3+ sites) → extract it

Apply AUTO-FIX items directly. Escalate architectural concerns with a concrete proposed fix.

For each finding, emit:
```
[ENG] path:line — Problem: <what>. Fix: <what was done or proposed>.
```

### Agent C — Config Contract Test Writer

**Goal:** Tests must prove that the feature's behavior at each config level is intentional, not accidental.

Steps:

1. **Map the config surface.** For each config level (defaults, user, project):
   - What value does the feature use when nothing is set?
   - What happens when the user-level config overrides the default?
   - What happens when the project-level config overrides both?
   - What happens with invalid or missing config?

2. **Audit existing tests** for config-level coverage gaps:
   ```bash
   grep -r "default\|user.*config\|project.*config\|env.*=\|os\.Getenv" \
     <relevant test files> --include="*_test.go" --include="*.test.ts"
   ```

3. **Write missing tests** that verify:
   - **Default behavior**: the binary/system works correctly with zero configuration
   - **User override**: a user-level setting overrides the default
   - **Project override**: a project-level setting overrides the user setting
   - **Precedence**: the correct level wins when multiple levels are set
   - **Invalid config**: useful error message, safe fallback (no silent corruption)

4. **Follow project test conventions:**
   - Go: table-driven tests, real SQLite for DB-touching tests (never mock what you own)
   - TS: vitest, real implementations at integration layer
   - Tests must FAIL without the code they verify (no theater)

5. Run the new tests:
   ```bash
   go test ./... -run TestConfig -v 2>&1   # or scoped to the relevant package
   ```

For each test added, emit:
```
[TEST] file — Added: TestConfigDefault, TestConfigUserOverride, TestConfigProjectOverride
```

---

## Phase 3: Consolidate

Collect output from all three agents. Deduplicate overlapping findings (if both Agent A and Agent B flag the same doc comment, apply one fix).

Verify the tests from Agent C pass:
```bash
go test ./... -count=1 -short 2>&1    # or equivalent for project type
```

If any test fails:
- Determine if the test is wrong (wrong expectation) or the code is wrong (real bug)
- Fix whichever is wrong
- Re-run until green

---

## Phase 4: Report

```
POLISH REPORT
────────────────────────────────────────────────────
Feature scope:    N files changed

Documentation:
  Updated:        N docs/comments
  Created:        N new sections
  Status:         CLEAN | UPDATED

Implementation:
  Auto-fixed:     N items (list)
  Escalated:      N items (list, with proposed fixes)
  Status:         CLEAN | FIXED | NEEDS_INPUT

Config Tests:
  Levels covered: defaults | user | project | precedence | invalid
  Tests added:    N
  Tests passing:  N/N
  Status:         COVERED | GAPS_REMAIN (list)
────────────────────────────────────────────────────
STATUS: POLISHED | NEEDS_INPUT | BLOCKED
```

If `STATUS: POLISHED`: the feature is production-ready. Suggest `/rs-check` before merge.

If `STATUS: NEEDS_INPUT`: list the specific decisions that require human judgment. Do not ask open-ended questions — present concrete options (A/B/C) with a recommendation.
