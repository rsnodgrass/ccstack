---
description: |
  Iterative spot-check review loop that stops correctly. Samples random files/sections,
  triages findings by concrete severity (not theoretical risk), and only counts a round
  as "clean" when no Tier-1 or Tier-2 bugs are found. Use instead of /loop for ongoing
  code health checks. Stops at 3 consecutive clean rounds by default.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Agent
  - Edit
  - Write
---

# Iterative Spot-Check Review

## Iron Law

**NEVER FIX A BUG YOU CANNOT DESCRIBE AS A CONCRETE OBSERVABLE FAILURE.**

"Could corrupt data under concurrent access" is not a concrete failure. "Returns stale team name for one render frame" is. "Causes REC_ERROR state after a zero-byte recording" is. The test: if you cannot write a one-sentence description of what the user or serial monitor would see when this bug triggers, it is not a bug — it is a hypothesis.

---

## Input Parsing

Parse optional args: `/rs-spot-check [--rounds N] [--path dir]`
- `--rounds N`: clean-round threshold before stopping (default: 3)
- `--path dir`: restrict sampling to this directory (default: cwd)

---

## Persistent State

Track across rounds in a local variable (not written to disk):
```
clean_rounds = 0        # consecutive rounds with NO T1/T2 findings
total_rounds = 0
all_t1_t2_fixed = []    # accumulates across rounds
all_t3_skipped = []     # theoretical findings we explicitly skipped
```

---

## Phase 1: Sample Files

Each round, pick **3 files** that were NOT the primary focus of the previous round. Prefer:
- Files with the most recent git changes: `git log --oneline -20 --name-only | grep -v '^[0-9a-f]' | sort | uniq -c | sort -rn | head -10`
- Files with no test coverage (no corresponding file in `test/`)
- Files you haven't reviewed this session

Within each file, sample a **random section** (don't always read line 1):
```bash
# Pick a random offset to start reading within the file
wc -l <file>   # get line count
# start at a random 1/3 or 2/3 point, read 100-150 lines
```

---

## Phase 2: Triage Every Finding

For each potential issue found, answer all four questions before proceeding:

### Question 1: Concrete failure
> What does the user or serial monitor see when this triggers?

If you cannot answer this in one sentence, discard the finding.

### Question 2: Trigger frequency
> In normal use, how often does the triggering condition occur?

- **Always / daily**: proceed
- **Rare but plausible** (specific user action, specific network state): proceed
- **Requires microsecond-precision interleaving** of independently-scheduled paths: discard

### Question 3: Worst-case impact
Score from these options (highest that applies):
- **CRASH** — device panics, hangs permanently, data loss
- **WRONG** — incorrect behavior a user would notice (wrong screen, failed recording, broken auth)
- **GLITCH** — transient incorrect state that self-corrects on next cycle/reboot
- **NOISE** — log spam, minor inefficiency

### Question 4: Fix cost
- **TRIVIAL** — single line, no new API surface, no new call sites
- **LOCAL** — <10 lines, stays within one function/file
- **INVASIVE** — new public API, multiple call sites, changes data ownership semantics

### Severity Tier Assignment

| Impact | Trigger Frequency | Fix Cost | Tier |
|--------|-------------------|----------|------|
| CRASH or WRONG | Any | Any | **T1 — Fix now** |
| GLITCH | Always/daily | TRIVIAL or LOCAL | **T2 — Fix now** |
| GLITCH | Rare/plausible | TRIVIAL | **T2 — Fix now** |
| GLITCH | Rare/plausible | INVASIVE | **T3 — Skip** |
| GLITCH | Microsecond-race | Any | **T3 — Skip** |
| NOISE | Any | Any | **T3 — Skip** |

**T3 findings are explicitly logged and skipped.** Do not fix them. Add a comment in the
source only if the code has no existing comment about the limitation.

---

## Phase 3: Fix T1 and T2 Only

For each T1/T2 finding:

1. Read the actual code at the cited location (do not fix from memory)
2. Verify the code path matches the description — if it doesn't, reclassify as T3
3. Apply the minimal fix: one targeted change, no refactoring the surrounding code
4. Run the project's test suite to confirm no regressions
5. Ask: **"Could a test have caught this?"** If yes, write it (one focused test function)

---

## Phase 4: Update Round Counter

```
total_rounds += 1

if this round had NO T1 or T2 findings:
    clean_rounds += 1
else:
    clean_rounds = 0   # reset — a bug found means start the count over
```

**T3-only rounds do NOT increment clean_rounds.** Finding ten theoretical races
in a row is not the same as the code being healthy.

---

## Phase 5: Stop Decision

```
if clean_rounds >= N:
    DONE — stop
else:
    run another round (go to Phase 1)
```

---

## Phase 6: Final Report

```
SPOT-CHECK COMPLETE
Rounds: <total>  |  Clean (T1/T2-free): <clean_rounds>/<threshold>

Bugs fixed (<count>):
  [T1] file:line — one-line description
  [T2] file:line — one-line description
  ...

Theoretical findings skipped (<count>):
  [T3] file:line — one-line description (why skipped: microsecond-race / low-impact glitch)
  ...

Tests added: <count>

STATUS: CLEAN (N consecutive rounds with no T1/T2 findings)
```

---

## Anti-Patterns to Reject

These are NOT bugs under this skill's Iron Law:

- **"Could cause a race if X is called while Y is running"** — requires proving both
  actually execute concurrently (same-core sequential code cannot race with itself)
- **"Pointer returned after mutex release"** — only a UAF if the pointed-to memory
  can actually be freed before the caller dereferences it; prove the free path
- **"Short-circuits on first NVS failure"** — only a bug if partial NVS failure
  with `&&` produces a worse outcome than attempting all writes first
- **"Buffer could truncate if URL exceeds N chars"** — only a bug if the system
  that generates those strings can actually produce strings that long
