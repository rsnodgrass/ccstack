---
description: "Over-engineering detection and simplification. Triggers: simplify, over-engineered, too complex, can this be simpler. Proactively suggest after implementation to catch unnecessary complexity before it ships."
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
---

# Over-Engineering Detection & Simplification

## Iron Law

**THE SIMPLEST SOLUTION THAT WORKS IS THE BEST SOLUTION.**

Three similar lines of code are better than one premature abstraction. Abstractions should be discovered through duplication, not invented in anticipation of it.

---

## Phase 1: Scope

Determine what changed. Try in order until one produces output:

```bash
git diff --cached --stat          # staged changes
git diff --stat                   # unstaged changes
git diff main --stat              # full branch diff
```

Read the full diff for all changed files. This is the review surface.

---

## Phase 2: Anti-Pattern Detection

Check **every changed file** for these specific patterns:

### 1. Premature Abstraction
A helper, utility, or shared function created for a **single call site**. If it's only used once, inline it. Abstractions earn their existence through reuse.

### 2. Unnecessary Indirection
A wrapper function whose only job is calling another function, possibly with slight argument reordering. The caller should call the target directly.

### 3. Over-Generic Interfaces
Accepts `interface{}`, `any`, `unknown`, or a broad union type when a **concrete type** covers all actual usage. Generics are warranted only when multiple concrete types exist today, not "might exist someday."

### 4. Feature Flags for Non-Features
Backwards-compatibility shims, migration toggles, or conditional paths added when you could just **change the code**. If there is no rollback scenario, there is no need for a flag.

### 5. Defensive Excess
Error handling for scenarios that cannot occur. Examples: nil-checking the return value of `make()`, catching exceptions from pure arithmetic, validating enum values that are already compiler-checked.

### 6. Configuration Theater
Making something configurable (env var, config file entry, constructor parameter) that will **never be configured** differently. Hardcode it. You can extract it later if a real need emerges.

### 7. Comment Bloat
- Docstrings or comments added to **unchanged** code (scope creep)
- Comments that restate what the code obviously does (`// increment counter` above `counter++`)
- Comments that should be variable/function names instead

### 8. Abstract Naming
`Manager`, `Handler`, `Service`, `Processor`, `Orchestrator` when a verb-noun name is clearer. `sendEmail` beats `EmailService.process`. Name things after what they **do**, not what they **are**.

### 9. Unnecessary Layers
service -> repository -> DB when service -> DB is sufficient. Each layer must justify its existence with distinct responsibility. If the "repository" is just forwarding queries, remove it.

### 10. Import Bloat
Pulling in a library dependency for one function that is 5-10 lines to write inline. The dependency carries update burden, security surface, and bundle size for minimal value.

---

## Phase 3: Simplification Suggestions

For each finding, classify it:

### AUTO-FIX (apply directly, no confirmation needed)
- Dead code removal (unused functions, unreachable branches)
- Unused imports
- Redundant comments that restate obvious code
- Unused variables or parameters

### ASK (present with before/after, wait for confirmation)
- Structural simplifications (removing a layer, collapsing an abstraction)
- Inlining a helper back to its single call site
- Replacing a generic interface with a concrete type
- Removing a configuration parameter and hardcoding

For ASK items, show a **concrete before/after** code comparison. Do not describe the change abstractly -- show the exact code.

---

## Phase 4: Apply and Report

1. Apply all AUTO-FIX items immediately.
2. Present all ASK items with their before/after diffs.
3. After the user responds to ASK items, apply approved changes.
4. Produce the report:

```
SIMPLIFICATION REPORT
Files reviewed:        N
Findings:              M (X auto-fixed, Y for your review)
Lines removed:         Z
Abstractions inlined:  W
STATUS:                CLEAN | SIMPLIFIED | NEEDS_REVIEW
```

### Status Definitions

- **CLEAN** -- No over-engineering detected. Code is appropriately simple.
- **SIMPLIFIED** -- Issues found and resolved (auto-fixed or approved).
- **NEEDS_REVIEW** -- ASK items pending your decision.

---

## Stop Conditions

**Only stop for:**
- Changes that alter public API surface (exported functions, REST endpoints, CLI flags, shared package interfaces) -- these require explicit approval.

**Never stop for:**
- Internal simplifications (private functions, unexported types, internal module structure)
- Removing dead code
- Inlining single-use abstractions
