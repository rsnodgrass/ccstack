---
description: "fix tests, add tests, improve coverage, write tests — proactively suggest after implementation changes"
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
---

# /rs-test — Test Improvement

**Iron Law: EVERY TEST MUST PROVE SOMETHING. No test theater — tests that always pass prove nothing.**

**argument-hint:** `[fix|add|improve]` (default: `improve`)

## Modes

- **`fix`** — Diagnose failing tests, find root cause, fix them, re-run to verify.
- **`add`** — Analyze git diff for untested code paths, write new tests following the test pyramid.
- **`improve`** — Both `fix` + `add`.

---

## Phase 1: Detect Test Framework

Identify the project's test stack before doing anything:

| Stack | Framework | Style |
|-------|-----------|-------|
| Go | `go test` | Table-driven tests |
| TypeScript | vitest | Standard vitest conventions |
| React | `@testing-library/react-native` | Component + hook testing |
| Python | pytest | Fixtures, parametrize |

Look for config files (`vitest.config.*`, `pytest.ini`, `go.mod`, etc.) and existing test files to confirm.

## Phase 2: Run Existing Tests

Run the full test suite (or scoped to the relevant module) and capture all output. Record:
- Total tests, passing, failing, skipped
- Failure messages and stack traces

## Phase 3: Fix Mode

For each failing test:

1. **Diagnose** — Read the failing test and the code it exercises. Trace the failure to its root cause (is the test wrong, or is the code wrong?).
2. **Fix** — Apply the minimal fix. Prefer fixing the test if the code behavior is intentionally changed; fix the code if the test expectation is correct.
3. **Re-run** — Verify the fix resolves the failure.

**3-Strike Rule:** If 3 fix attempts for the same test fail, STOP and escalate to the human. Do not keep guessing.

## Phase 4: Add Mode

1. **Analyze git diff** — Run `git diff main` (or `git diff HEAD~1` if on main) to identify changed code paths.
2. **Identify untested paths** — Look for new functions, branches, error handling, edge cases without corresponding tests.
3. **Generate tests following the pyramid:**
   - **Unit tests first** (most tests here) — Pure logic, single functions, mocked external dependencies.
   - **Integration tests second** — Real DB queries, API handlers, service interactions.
   - **E2E tests sparingly** — Only for critical user paths.

## Phase 5: Verify

Run the full test suite. All tests (new + existing) must pass.

---

## Key Nuances

### Go
- Use **table-driven tests** with descriptive subtest names.
- Integration tests: use real DB connections. **Do NOT mock what you own** at the integration layer.
- Set git identity in tests to avoid polluting user config:
  ```go
  cmd.Env = append(os.Environ(),
      "GIT_AUTHOR_NAME=Test User",
      "GIT_AUTHOR_EMAIL=test@example.com",
      "GIT_COMMITTER_NAME=Test User",
      "GIT_COMMITTER_EMAIL=test@example.com",
  )
  ```

### TypeScript
- Use **vitest**. Mock external dependencies only (third-party APIs, services you don't own).
- **Never mock what you own** at the integration layer — test real HTTP handlers, real DB queries.

### General
- A failing unit test should tell you **exactly** what broke.
- A failing E2E test tells you **something** broke. Prefer the former.
- Follow the test pyramid: more unit, fewer E2E.

---

## Stop Conditions

**Only stop for:**
- Test framework not installed (report which framework and how to install).
- Architectural issues making code fundamentally untestable (report what needs to change).

**Never stop for:**
- Existing test failures unrelated to current changes — note them, skip them, continue.

---

## Verification Gate

Every new test must satisfy:
1. **FAIL without the code it tests** (proves the test is meaningful).
2. **PASS with the code** (proves correctness).

If a test passes without the code it claims to cover, it is theater. Delete it and write a real one.

---

## Structured Output

End with a TEST REPORT:

```
## TEST REPORT

- **Before:** X passing, Y failing, Z skipped
- **After:** X passing, Y failing, Z skipped
- **New tests written:** N
- **Failures fixed:** N
- **Escalated (3-strike):** N (list if any)
- **Skipped (unrelated failures):** N (list if any)
```
