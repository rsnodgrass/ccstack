---
description: |
  Execute blacklight beads with a team of parallel expert agents. Loads open
  beads, groups by domain expertise, dispatches parallel agents to fix them,
  coordinates results. Use when asked to "fix the findings", "blacklight fix",
  "execute the plan", "work the beads", or "start fixing". Run after
  /rs-blacklight-plan.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
---

# /rs-blacklight-fix — Team-Execute Beads

## Iron Law

**DISPATCH TO SPECIALISTS. COORDINATE, DON'T BOTTLENECK.**

A single agent fixing 30 beads sequentially wastes parallelism. Group by expertise, dispatch in parallel, verify as a coordinator. You are the orchestrator, not the worker.

---

## Phase 1: Load and Queue

```bash
bd ready 2>/dev/null
bd list --status=open 2>/dev/null
bd list --status=in_progress 2>/dev/null
```

If no open beads exist:
- Check if `/rs-blacklight-plan` was run — suggest it if not
- STOP if there's nothing to execute

Build the execution queue:
1. Tasks with no unresolved dependencies, sorted by priority (P0 first)
2. Group into **batches** of independent tasks that can run in parallel

---

## Phase 2: Classify by Domain

Read each bead's description and classify by expertise:

| Domain | File Patterns | Agent Prompt Prefix |
|--------|--------------|---------------------|
| **Go** | `*.go` | "You are a Go expert. Follow Go conventions..." |
| **TypeScript** | `*.ts`, `*.tsx` | "You are a TypeScript expert. Use strict types..." |
| **Python** | `*.py` | "You are a Python expert. Use type hints..." |
| **Config/Infra** | `*.yaml`, `*.toml`, `Makefile`, `Dockerfile` | "You are a DevOps expert. Prefer explicit over implicit..." |
| **Docs** | `*.md`, `*.mdx` | "You are a technical writer. Be concise..." |
| **General** | mixed or unclear | "You are a senior engineer. Follow project conventions..." |

---

## Phase 3: Dispatch Parallel Agents

### Batch Construction

Group independent beads into batches of up to 3 (Agent tool parallel limit):

```
Batch 1: [bead-A (Go), bead-B (TS), bead-C (Go)]     ← all independent
Batch 2: [bead-D (depends on A), bead-E (Py)]          ← D waits for batch 1
Batch 3: [bead-F (Go), bead-G (Config)]                ← independent
```

### Agent Prompt Template

Each dispatched agent gets a self-contained prompt:

```
You are fixing a tracked task from a codebase audit. Work autonomously.

TASK: <bead title>
BEAD ID: <id>

DESCRIPTION:
<full bead description from bd show>

PROJECT CONVENTIONS:
<relevant excerpts from CLAUDE.md — language-specific rules only>

INSTRUCTIONS:
1. Claim the bead: bd update <id> --status=in_progress
2. Read the file(s) listed in the description
3. Implement the fix exactly as described
4. Run the verification command from the description
5. If verification fails, debug and fix — do not leave broken code
6. If you discover NEW issues not in this task, note them but do NOT fix them
7. Close the bead: bd close <id>

OUTPUT FORMAT (required):
BEAD: <id>
STATUS: FIXED | BLOCKED | DISCOVERY
FILES_CHANGED: <file1>, <file2>
VERIFICATION: PASS | FAIL
NOTES: <any discoveries or concerns>
```

### Dispatch

Launch agents in parallel batches. Before dispatching each batch, claim all beads:

```bash
bd update <id1> --status=in_progress
bd update <id2> --status=in_progress
bd update <id3> --status=in_progress
```

Then launch up to 3 Agent tool calls simultaneously, one per bead in the batch. Each agent gets the rendered prompt template above with its specific bead details filled in.

**Trivial beads** (one-line fixes, dead import removal): Fix these inline as coordinator rather than dispatching an agent. The overhead of agent dispatch exceeds the fix time.

---

## Phase 4: Coordinate Results

After each batch completes:

1. **Parse agent output** for STATUS, FILES_CHANGED, VERIFICATION, NOTES
2. **Close successful beads:**
   ```bash
   bd close <id1> <id2> ...
   ```
3. **Handle BLOCKED beads:**
   - Read the agent's notes
   - If it needs a human decision, leave it open and flag it
   - If it hit a dependency issue, re-queue for the next batch
4. **Create discovery beads:**
   ```bash
   bd create \
     --title="<discovered issue>" \
     --description="<agent's notes, expanded to be self-contained>" \
     --type=task \
     --priority=2
   ```
5. **Run quality gates** after each batch:
   ```bash
   [ -f go.mod ] && go build ./... 2>&1 && go vet ./... 2>&1
   [ -f tsconfig.json ] && tsc --go --noEmit 2>&1
   [ -f Makefile ] && make lint 2>&1
   ```
6. **If quality gates fail**, identify which agent's changes broke the build and fix before proceeding.

### Progress Checkpoint (after each batch)

```
BLACKLIGHT FIX PROGRESS
=========================
Batch:       N of M
Completed:   X of Y total beads
In progress: [current batch tasks]
Remaining:   Z beads (W blocked, V ready)
Discoveries: D new beads created
Quality:     PASS | FAIL
```

---

## Phase 5: Unlock and Continue

After closing beads, check for newly unblocked tasks:

```bash
bd ready 2>/dev/null
```

Add newly-unblocked beads to the front of the next batch. Repeat Phase 3-4 until the queue is empty or all remaining beads are BLOCKED.

---

## Phase 6: Final Report

```bash
bd list --status=closed 2>/dev/null | tail -30
bd list --status=open 2>/dev/null
```

```
BLACKLIGHT FIX REPORT
======================
Beads completed:      N of M total
  By parallel agents: X
  By coordinator:     Y (trivial fixes)
Beads remaining:      Z (list ids if any)
Beads discovered:     D new issues

Batches executed:     B
Agents dispatched:    A total

Files modified:       [list]
Quality gates:        PASS | FAIL

STATUS: ALL_FIXED | PARTIALLY_FIXED | BLOCKED

NEXT STEPS:
- If PARTIALLY_FIXED: resume with /rs-blacklight-fix
- If ALL_FIXED: /rs-check to verify, then /rs-ship to land
- If discoveries exist: review with bd list --status=open
```

---

## Stop Conditions

**Only stop for:**
- Quality gate failure that persists after retry — agent conflict needs human resolution
- Beads requiring architectural decisions not covered in the description
- Session resource limits — save progress, report remaining beads

**Never stop for:**
- Discovered issues (create beads, keep going)
- Individual agent failures (retry once, then mark BLOCKED and continue with other beads)
- Trivial beads (fix them inline as coordinator, don't dispatch an agent for a one-liner)
