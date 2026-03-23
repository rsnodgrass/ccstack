---
description: |
  Session completion — landing the plane. Closes beads, commits, pushes, syncs.
  Use when asked to "wrap up", "ship it", "land the plane", "end session",
  "push everything", or "are we done?". Proactively suggest when work appears
  complete and changes are uncommitted.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
---

# Landing the Plane

## Iron Law

**WORK IS NOT COMPLETE UNTIL `git push` SUCCEEDS.**

Local commits are local. Unpushed code is unshipped code. Never say "done" with unpushed changes.

---

## Stop Conditions

**Only stop for:**
- Merge conflicts that need human judgment
- Quality gate failures (from /rs-check) that need fixes
- Push failures (auth, permissions, protected branch)

**Never stop for:**
- Uncommitted changes (commit them)
- Beads sync issues (warn and continue)
- Missing commit message (generate one from the diff)

---

## Phase 1: Close Completed Work

Check beads for in-progress work:

```bash
bd list --status=in_progress 2>/dev/null || echo "BEADS_UNAVAILABLE"
```

If beads are available:
- Review each in-progress issue against the current changes
- Close issues that are complete: `bd close <id1> <id2> ...`
- For issues that need follow-up: leave in-progress, note them in the report

---

## Phase 2: Check for Uncommitted Changes

```bash
git status --short
```

If there are uncommitted changes:
1. Review what changed
2. Stage files explicitly (NEVER `git add .` — always name files):
   ```bash
   git add <file1> <file2> ...
   ```
3. Generate a commit message from the diff:
   - Format: `type(scope): summary` or plain imperative sentence
   - One line only, max ~72 chars
   - Never reference Claude or AI in the message
   ```bash
   git commit -m "<generated message>"
   ```

If no changes: skip to Phase 4.

---

## Phase 3: Quality Gates (if code changed)

Run quality checks on the committed code:

```bash
# detect and run appropriate checks
[ -f Makefile ] && make typecheck 2>&1 && make lint 2>&1 && make build 2>&1
[ ! -f Makefile ] && [ -f go.mod ] && go vet ./... 2>&1 && go build ./... 2>&1
[ ! -f Makefile ] && [ -f tsconfig.json ] && npx tsc --noEmit 2>&1
```

If quality gates fail:
- For auto-fixable issues: fix, re-commit, continue
- For real failures: STOP and report what needs fixing

---

## Phase 4: Sync and Push

```bash
# pull latest, rebase on top
git pull --rebase 2>&1

# push to remote
git push 2>&1

# sync beads
bd dolt pull 2>/dev/null || true
bd dolt push 2>/dev/null || true

# verify
git status
```

If push fails: diagnose and report. Do NOT force push.

---

## Phase 5: Create Follow-Up Issues

For any remaining work discovered during this session:

```bash
bd create --title="<description>" --description="<context>" --type=task --priority=2
```

---

## Phase 6: Report

```
SHIP REPORT
Beads closed: [list or "none"]
Files committed: [list]
Commit: <hash> <message>
Pushed to: <branch>
Beads synced: YES | NO (reason)
Follow-up issues: [list or "none"]
STATUS: SHIPPED | BLOCKED
```
