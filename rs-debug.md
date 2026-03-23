---
description: "Structured debugging workflow. Triggers: debug, fix this bug, why is this broken, root cause, investigate error. Proactively suggest when errors appear in build output, test failures, or runtime exceptions."
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
argument-hint: "<error message or symptom description>"
---

# Structured Debugging

## Iron Law

**NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST.**

Fixing symptoms creates whack-a-mole debugging. Every "quick patch" that skips root cause analysis costs 3x more time when the real bug resurfaces.

---

## Phase 1: Collect Symptoms

1. Read the full error message, stack trace, and reproduction steps provided by the user.
2. Search for related errors in logs, test output, or build output.
3. Identify the affected files and code paths.

**If context is insufficient, ask ONE focused question** -- not a checklist. Example: "Does this happen on every request or only intermittently?" not "Can you provide the full stack trace, environment, OS version, browser, and steps to reproduce?"

---

## Phase 2: Root Cause Investigation

1. **Trace the code path** from the symptom backward to potential causes. Read the failing function, then its callers, then the data sources.
2. **Check recent changes** to affected files:
   ```bash
   git log --oneline -20 -- <affected-files>
   ```
3. **Regression check**: Was this working before? If yes, the root cause is in the diff. Use `git log --oneline -20` and `git diff <last-known-good>..HEAD -- <affected-files>` to narrow it down.
4. **Reproduce deterministically** if possible. A bug you can't reproduce is a bug you can't verify as fixed.

**Output a specific, testable claim:**

> Root cause hypothesis: [concrete statement about what is wrong and why]

---

## Phase 3: Pattern Matching

Check the symptom against these known patterns before forming a hypothesis:

| Pattern | Signature | Where to Look |
|---------|-----------|---------------|
| **Data flow break** | Field exists at source, missing at destination | Trace field through frontend, API handler, service, DB query. VERY COMMON. |
| **Race condition** | Intermittent, timing-dependent, works under debugger | Shared state, goroutines/promises without synchronization |
| **Nil/null propagation** | Nil pointer dereference, "cannot read property of undefined" | Missing nil guards, optional fields assumed present |
| **Configuration drift** | Works locally, fails in staging/prod | Env vars, feature flags, service URLs, secrets |
| **Missing env variable** | Silent failure or empty string where value expected | `turbo.json` globalEnv, `.devcontainer/setup-env.sh`, task-specific env arrays |
| **pgroll migration state** | Column not found, type mismatch, constraint violation | Migration files in `migrations/pgroll/`, schema version drift between environments |
| **Stale cache** | Shows old data, fix doesn't take effect | Build cache, CDN, browser cache, query cache, module cache |

---

## Phase 4: Hypothesis Testing

1. **Add a temporary log or assertion** at the suspected root cause location. Do not guess -- instrument.
2. **Run the reproduction** and check the output.
3. **Confirm or reject** the hypothesis based on evidence.

### 3-Strike Rule

If 3 hypotheses fail in sequence, **STOP and escalate**. You are likely looking at the wrong layer.

### Red Flags -- Stop Immediately If You Catch Yourself Doing These

- **"Quick fix for now"** -- No. Find the root cause.
- **Proposing a fix before tracing the data flow** -- You are guessing, not debugging.
- **Each fix reveals a new problem elsewhere** -- You are at the wrong layer. Step back.

---

## Phase 5: Implementation

1. **Fix the root cause**, not the symptom. If the symptom is a nil pointer but the root cause is a missing validation upstream, fix the validation.
2. **Minimal diff** -- fewest files, fewest lines changed. Do not refactor while debugging.
3. **Write a regression test** that:
   - FAILS without the fix
   - PASSES with the fix
4. **Run the full test suite** -- confirm no regressions.
5. **Blast radius check**: if the fix touches more than 5 files, flag it explicitly. Large fixes deserve extra scrutiny.

---

## Phase 6: Verification & Report

1. **Fresh reproduction** from scratch confirming the fix resolves the original symptom.
2. **Produce a structured report:**

```
DEBUG REPORT
Symptom:         [what was observed]
Root cause:      [what was actually wrong]
Fix:             [what was changed, file:line refs]
Evidence:        [test output showing fix works]
Regression test: [file:line of new test]
STATUS:          DONE | DONE_WITH_CONCERNS | BLOCKED
```

### Status Definitions

- **DONE** -- Root cause identified, fix applied, regression test passing, no side effects.
- **DONE_WITH_CONCERNS** -- Fix applied but there are related issues worth noting (e.g., similar patterns elsewhere, tech debt exposed).
- **BLOCKED** -- Cannot proceed. Requires human input, access, or a decision.

---

## Stop Conditions

**Only stop for:**
- 3 failed hypotheses in sequence (escalate with what was tried and ruled out)
- Security-sensitive changes that require human review before applying

**Never stop for:**
- Unrelated test failures (note them, keep going)
- Formatting issues in affected files (fix after the bug is resolved)
