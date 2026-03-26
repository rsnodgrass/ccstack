# Agent Guidelines for ccstack Skills

## Prime Directive: Search Before Building

Every agent spawned by ccstack skills MUST follow this principle:

Before designing any solution that involves:
- Concurrency, parallelism, or scheduling
- A pattern you haven't used in this specific runtime/framework before
- Infrastructure (CI, deploy, testing, build tooling)
- Anything where the runtime (bun/node/deno/go) or framework might have a built-in

**Stop and search first:**
1. `"{runtime} {thing you're about to build} built-in"` (e.g., "bun test concurrency built-in")
2. `"{thing} best practice {current year}"` (e.g., "e2e test parallelism best practice 2026")
3. Check the runtime's own docs (bun.com/docs, nodejs.org/api, pkg.go.dev)

**The delta between a hand-rolled semaphore and `test.concurrent()` is the difference between mass and elegance.**

### The Eureka Exception

Sometimes search reveals everyone is doing it wrong. The tried-and-true is deeply in distribution — everyone who tried something different has failed. But occasionally, first-principles reasoning exposes a clear reason why ALL existing approaches are wrong for this specific context.

This is where truly superlative work happens. Zig while others zag.

When this happens: **document the reasoning explicitly.** Why is the conventional approach wrong here? What first-principles insight changes the calculus? This is the 11/10 moment — flag it, don't hide it.

## Agent Behavior

### Fix-First Classification
- **AUTO-FIX**: formatting, unused imports, simple lint, dead code — apply without asking
- **ASK**: security, architecture, design decisions — present with concrete before/after

### Evidence-Based Claims
- Every code reference includes `file:line`
- No summaries without supporting code locations
- "I believe X" is not acceptable — "X is true because file.go:42 shows Y" is

### Structured Output
All agents must end with a clear status:
- **DONE** — task completed, evidence provided
- **DONE_WITH_CONCERNS** — completed but related issues worth noting
- **BLOCKED** — cannot proceed, requires human input

### Stop Conditions
- Only stop for security vulnerabilities needing human judgment or architectural changes
- Never stop for formatting, trivial lint, or unrelated failures
