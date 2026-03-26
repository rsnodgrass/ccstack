---
description: |
  Drive code coverage from abysmally low to 85%+ using ralph-loop with expert test agents.
  Adds meaningful tests that catch real bugs, edge cases, and failure modes — no test theater.
  May make structural changes to improve testability. Use when asked to "add coverage",
  "improve coverage", "we need more tests", or "coverage is too low".
argument-hint: "[--max-iterations N] [--completion-promise TEXT]"
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
---

# /rs-add-coverage — Drive Code Coverage to 85%+

Invoke the ralph-loop with a test coverage mission. This loop will keep iterating until
coverage exceeds 85%, using expert test agents and making structural changes as needed.

## Execution

Run the ralph-loop setup script directly (it's hidden from the Skill tool):

```bash
RALPH_LOOP_SCRIPT=$(find ~/.claude/plugins/cache/claude-plugins-official/ralph-loop -name "setup-ralph-loop.sh" -path "*/scripts/*" 2>/dev/null | head -1)
```

Then invoke it:

```bash
"$RALPH_LOOP_SCRIPT" "Our code coverage is abysmally low, continue to add meaningful tests that would catch actual bugs, edge cases, failure modes, etc until the code coverage is over 85%. No test theater. Use expert test agents, experts in test harness. Keep running our tests and checking for test coverage. This will likely require some structural changes to make things better testable. Don't just test what we have implemented, test conceptually how various components should work in practice." --completion-promise "Coverage exceeds 85% across all packages" $ARGUMENTS
```

Then work on the task as described in the ralph-loop output.

Pass through any extra arguments (e.g. `--max-iterations`, `--completion-promise`) from the user.
