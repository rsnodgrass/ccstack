---
description: |
  Deep code review with automatic expert routing. Analyzes changed files and
  dispatches to specialist agents in parallel. Use when asked to "review this",
  "check my code", "have someone review", or "code review". Proactively suggest
  after implementing a feature, before committing.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Agent
  - Edit
  - Write
---

# Expert-Routed Code Review

## Iron Law

**EVERY FINDING MUST BE ACTIONABLE. No vague warnings, no "consider doing X".**

Each finding is either AUTO-FIX (apply it now) or ASK (present to user with a concrete fix).

---

## Stop Conditions

**Only stop for:**
- Security vulnerabilities that need human judgment
- Architectural concerns that change the approach

**Never stop for:**
- Formatting issues (auto-fix)
- Unused imports (auto-fix)
- Simple naming improvements (auto-fix)

---

## Phase 1: Scope the Review

Determine what changed:

```bash
# prefer staged changes, fall back to unstaged, fall back to branch diff
DIFF=$(git diff --cached --stat 2>/dev/null)
[ -z "$DIFF" ] && DIFF=$(git diff --stat 2>/dev/null)
[ -z "$DIFF" ] && DIFF=$(git diff main --stat 2>/dev/null)
```

List changed files and categorize by type:
- `.go` files → Go review
- `.ts`, `.tsx` files → TypeScript review
- `.sql`, `.json` in `migrations/` → Migration review
- `.tf` files → Terraform review
- `.md` files → Documentation review
- `.yaml`, `.yml` in `infra/` → Infrastructure review

---

## Phase 2: Route to Specialists

Launch specialist agents IN PARALLEL based on file types found. Each agent gets:
1. The relevant subset of the diff
2. Specific review criteria for their domain

### Go Review (agent: code-reviewer)
Check for:
- Security: SQL injection, command injection, path traversal
- Prefixed IDs: never raw UUIDs outside DB boundary (use `ids.ParseXXX()`)
- Error handling: no swallowed errors, proper `error` wrapping
- Function length: flag >50 statements
- Cognitive complexity: flag >25
- Defensive coding: nil checks, bounds checks
- Over-engineering: flag unnecessary abstractions

### TypeScript Review (agent: typescript-pro + simplifier)
Check for:
- Type safety: no `any`, no type assertions without justification
- Biome compliance: formatting, lint rules
- Over-engineering: unnecessary abstractions, premature optimization
- React patterns: proper hooks usage, no inline styles
- Import hygiene: no circular imports, logical ordering

### Migration Review (agent: postgres-pro)
Check for:
- pgroll rules: NO mixing `sql` ops with other op types in same file
- Zero-downtime: expand-then-contract pattern
- Index safety: concurrent index creation for large tables
- JSONB preference: use metadata column over new columns when appropriate
- timestamptz: never plain timestamp

### Terraform Review (agent: terraform-pro)
Check for:
- State safety: no resources managed by both Terraform and pipeline
- Naming conventions: consistent resource naming
- Security: no hardcoded secrets, proper IAM scoping

### Documentation Review (agent: technical-writer)
Check for:
- Accuracy: does the doc match the code?
- Audience: is it in the right directory (docs/human/ vs docs/ai/)?
- Clarity: human-readable, scannable, progressive disclosure

---

## Phase 3: Consolidate Findings

Merge all specialist reports. Deduplicate. Sort by severity.

### Fix-First Classification

For each finding:
- **AUTO-FIX**: formatting, unused imports, simple lint, missing error wraps → apply directly
- **ASK**: security issues, architectural concerns, design decisions → present to user

### Reinvention Check (ALWAYS RUN)

Before approving any new infrastructure, patterns, or non-trivial implementations, verify the author searched first:
- **Runtime/framework built-ins** — is there a built-in that already does this? (e.g., hand-rolled semaphore vs `test.concurrent()`)
- **Battle-tested libraries** — did someone hand-roll 50 lines when a well-maintained package handles this?
- **Standard patterns** — is this a known solved problem (auth, caching, deploy pipelines, error handling) with an established best practice?
- **>20 lines of infrastructure code** without evidence of prior search is a red flag

**The exception**: if first-principles reasoning reveals the conventional approach is fundamentally wrong for this use case, that's a eureka moment — flag it as such with the reasoning, don't reject it as reinvention.

### Over-Engineering Check (ALWAYS RUN)

Regardless of file types, check ALL changed code for:
- Helpers/utilities created for one-time use
- Premature abstractions (3 similar lines > unnecessary abstraction)
- Feature flags or backwards-compat shims when you can just change the code
- Error handling for impossible scenarios
- Extra configurability nobody asked for
- Comments/docstrings added to unchanged code
- Overly generic interfaces when a concrete type suffices

---

## Phase 4: Apply and Report

1. Apply all AUTO-FIX items. Report each:
   `[AUTO-FIXED] file:line — Problem: what was wrong. Fix: what was done.`

2. Present ASK items (if any) in a batch:
   ```
   Review found N issues. M auto-fixed. K need your input:

   1. [CRITICAL] file:line — Description
      Fix: specific code change
      A) Fix  B) Skip

   2. [IMPORTANT] file:line — Description
      Fix: specific code change
      A) Fix  B) Skip

   RECOMMENDATION: Fix #1 (security), skip #2 (low risk).
   ```

3. Summary:
   ```
   REVIEW REPORT
   Files reviewed: N (Go: X, TS: Y, Other: Z)
   Specialists consulted: [list]
   Findings: N total (M auto-fixed, K asked, J clean)
   Over-engineering: [CLEAN | N issues found]
   STATUS: CLEAN | ISSUES_FOUND | CRITICAL
   ```
