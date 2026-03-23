---
description: |
  Cross-project release readiness validation. Runs quality gates, checks for
  blockers, validates migrations, generates release notes. Use when asked to
  "prepare release", "release readiness", "pre-release check", "are we ready
  to ship?", or "create release PR". Proactively suggest when merging to main.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
---

# Release Readiness

## Iron Law

**NO RELEASE WITHOUT EVIDENCE THAT EVERYTHING WORKS.**

A release is not "probably fine." Every gate must pass with proof.

---

## Stop Conditions

**Only stop for:**
- Quality gate failures
- Open P0/P1 beads issues
- Invalid migration structure
- Missing environment variable defaults

**Never stop for:**
- P3/P4 beads issues (note in release notes)
- TODO/FIXME comments (note in report, don't block)
- Documentation gaps (flag, don't block)

---

## Phase 1: Quality Gates

Run all quality checks (same as /rs-check):

```bash
[ -f Makefile ] && make typecheck 2>&1 && make lint 2>&1 && make build 2>&1
```

If any gate fails: STOP. Fix before proceeding.

---

## Phase 2: Blocker Check

### Beads Issues
```bash
bd list --status=open --priority=0 2>/dev/null || echo "NO_P0"
bd list --status=open --priority=1 2>/dev/null || echo "NO_P1"
```
If P0/P1 issues exist: STOP. These must be resolved before release.

### Draft PRs
```bash
gh pr list --state open --draft 2>/dev/null || echo "NO_DRAFT_PRS"
```
Flag any draft PRs that might be blocking.

---

## Phase 3: Migration Validation

Check for pgroll migrations in the release:

```bash
git log main..HEAD --name-only -- 'migrations/pgroll/*.json'
```

For each migration file:
- Verify no mixed `sql` + other op types (CRITICAL — causes deploy rollback)
- Verify correct sequencing (earlier migrations don't depend on later ones)
- Check for zero-downtime compatibility (expand-then-contract)

---

## Phase 4: Environment Variable Check

```bash
# check setup-env.sh for defaults
[ -f .devcontainer/setup-env.sh ] && echo "HAS_SETUP_ENV"
```

Cross-reference any new env vars in the diff against:
- `.devcontainer/setup-env.sh` (must have defaults)
- `turbo.json` (must be listed in globalEnv or task env)
- `apps/api-go/internal/config/config.go` (for Go feature flags)
- `packages/env/src/feature-flags.ts` (for TS feature flags)

Flag any env var without a default.

---

## Phase 5: Code Quality Scan

```bash
# check for leftover markers in changed files
git diff main --name-only | xargs grep -n 'TODO\|FIXME\|HACK\|XXX\|TEMPORARY' 2>/dev/null || echo "CLEAN"
```

Report findings but don't block (informational).

---

## Phase 6: Generate Release Notes

```bash
git log main..HEAD --oneline --no-merges
```

Categorize commits into:
- **Added**: new features
- **Changed**: modifications to existing functionality
- **Fixed**: bug fixes
- **Infrastructure**: deployment, config, CI changes

---

## Phase 7: Create Release PR (if all gates pass)

```bash
gh pr create --base main --draft \
  --title "release: <version summary>" \
  --body "<generated release notes with Mermaid diagrams where helpful>"
```

Always create as DRAFT (per CLAUDE.md rules).

---

## Phase 8: Report

```
RELEASE READINESS REPORT
Quality Gates:     PASS | FAIL
Blockers (P0/P1):  NONE | [list]
Migrations:        VALID | INVALID (details)
Environment Vars:  CLEAN | MISSING_DEFAULTS (list)
Code Markers:      N TODO/FIXME found (informational)
Draft PRs:         NONE | [list]
Release PR:        <URL> (draft)
STATUS: READY | NOT_READY (reasons)
```
