---
description: |
  Think out of the box: propose the single smartest, most radically innovative,
  creative, useful, and compelling addition to the current project. Deeply
  explores project state before answering. Use when asked "what's the best next
  feature?", "innovate", "think outside the box", "what would you add?", or
  "what's the smartest addition?". Run multiple times to get a ranked set of
  distinct ideas.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Agent
---

# rs-innovate: Radical Innovation Prompt

## Purpose

Surface **one** transformative idea per run — not a laundry list. The answer
must be grounded in the current project state, not generic. It should be
something a competitor hasn't shipped, that leverages what already exists, and
that would make someone show the product to a friend.

---

## Phase 1: Deep Project Reconnaissance

Before proposing anything, fully understand where the project stands:

```bash
# beads: open issues and in-progress work
bd list --status=open 2>/dev/null | head -40 || echo "BEADS_UNAVAILABLE"
bd list --status=in_progress 2>/dev/null || echo ""
```

Read (in parallel if possible):
- `PLAN.md` or `ROADMAP.md` — intended direction
- `README.md` — stated purpose and audience
- `DESIGN-SYSTEM.md` or `ARCHITECTURE.md` — technical constraints
- Top-level `package.json` / `go.mod` / `pyproject.toml` — ecosystem
- Recent git log: `git log --oneline -20`

Also run a broad Explore agent:

> Explore this project. What does it currently do well? What is obviously
> missing? What is the most painful gap between what users want and what
> exists? Check key source files, existing features, and any open beads.

---

## Phase 2: Competitive Gap Analysis

Think about the problem space the project operates in:

- Who are the real-world users and what do they struggle with daily?
- What do competing products or tools do that this doesn't?
- What would make a power user evangelize this to their peers?
- What existing building block (already in the codebase) is under-leveraged?
- What OS/platform capability (widgets, intents, shortcuts, background services,
  push, spatial audio, voice, watch, etc.) is untapped?

---

## Phase 3: Proposal (strict format)

Output exactly this structure — no bullet dumps, no feature lists. **One idea.**

```
## The Answer: [Punchy Name]

[2–3 sentence description of what it is and what the user experiences.]

### Why this is the move

[2–4 sentences: what gap it fills, why no competitor has it, why NOW is the
right time to build it, and which existing code/data makes it achievable.]

### Leverage points (what already exists that this builds on)

| Existing asset | How it's used |
|----------------|---------------|
| ...            | ...           |

### What users will say when they see it

[One sentence. The "show a friend" moment.]

### Architecture sketch (optional — only if non-obvious)

[Only include if the implementation path needs clarification.]
```

---

## Phase 4: Beads capture (optional)

If the user wants to track the idea, create a high-level beads epic:

```bash
bd create \
  --title="[feature name]" \
  --description="[proposal text]" \
  --type=feature \
  --priority=2
```

---

## Constraints

- **One idea per run.** If asked for more, re-run: `/rs-innovate` again.
- The idea must be buildable with the current tech stack — no platform pivots.
- It must leverage something already in the codebase (no greenfield rewrites).
- It must be demonstrable in a 30-second show-and-tell.
- Never propose something already tracked in open beads.
