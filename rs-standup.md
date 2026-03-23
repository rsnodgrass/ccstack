---
description: |
  Generate standup report from git history and beads status. Use when asked to
  "standup", "what did I do", "daily report", "status update", "what happened
  yesterday", or "summarize my work".
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# Standup Report

## Iron Law

**FACTS, NOT NARRATIVES. Commits and closed issues — not what you remember doing.**

---

## Phase 1: Gather Data

### Git History (last 24 hours or since last workday)
```bash
# commits since yesterday
git log --oneline --since="24 hours ago" --author="$(git config user.name)" 2>/dev/null
# or since last Friday if Monday
git log --oneline --since="3 days ago" --author="$(git config user.name)" 2>/dev/null
```

### Beads Activity
```bash
# recently closed issues
bd list --status=closed 2>/dev/null | head -20
# currently in progress
bd list --status=in_progress 2>/dev/null
# blocked items
bd blocked 2>/dev/null
```

### Files Changed
```bash
git diff --stat HEAD~5 2>/dev/null || git diff --stat main 2>/dev/null
```

---

## Phase 2: Categorize

Group work into:
- **Completed**: closed beads + merged commits
- **In Progress**: open beads marked in_progress
- **Blocked**: items with unresolved blockers
- **Discovered**: new issues found during work

---

## Phase 3: Generate Report

```
STANDUP — <date>

DONE:
- <commit/issue summary> (files: X changed)
- <commit/issue summary>

IN PROGRESS:
- <issue title> — <brief status>

BLOCKED:
- <issue title> — blocked by: <blocker>

DISCOVERED:
- <new issue> — created as <beads-id>
```

Keep it scannable. One line per item. No paragraphs.
