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
  - Skill
---

# /rs-add-coverage — Drive Code Coverage to 85%+

Invoke the ralph-loop with a test coverage mission. This loop will keep iterating until
coverage exceeds 85%, using expert test agents and making structural changes as needed.

## Execution

Run the following skill invocation:

```
/ralph-loop:ralph-loop "Our code coverage is abysmally low, continue to add meaningful tests that would catch actual bugs, edge cases, failure modes, etc until the code coverage is over 85%. No test theater. Use expert test agents, experts in test harness. Keep running our tests and checking for test coverage. This will likely require some structural changes to make things better testable. Don't just test what we have implemented, test conceptually how various components should work in practice." $ARGUMENTS
```

Pass through any extra arguments (e.g. `--max-iterations`, `--completion-promise`) from the user.
