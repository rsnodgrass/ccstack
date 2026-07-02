---
description: |
  Think out of the box on DESIGN: propose the single boldest, highest-taste UI/UX
  direction for the current product — beautiful, intuitive, and genuinely novel,
  not a coat of paint. Deeply explores the existing surfaces, users, and design
  language before answering. Use when asked to "innovate on the design", "push the
  UI/UX", "make this beautiful/bolder", "rs-innovate-design", "raise the taste",
  "what's the out-of-the-box design move?", or "reimagine this interface". Run
  multiple times for a ranked set of distinct directions.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Agent
  - WebSearch
  - WebFetch
---

# rs-innovate-design: Radical Design Innovation Prompt

## Purpose

Surface **one** transformative design direction per run — not a redesign checklist.
The move must raise *taste* and *reduce cognitive load* at the same time, be grounded
in the real product and its users, and produce a "I want to show someone this" moment.
Decoration is not innovation. Novelty that costs clarity is not innovation. The bar is:
**more beautiful AND more intuitive than what exists**, in a way a competitor hasn't shipped.

---

## Phase 1: Design Reconnaissance

Understand the surfaces before touching them. In parallel:

```bash
# find the design surfaces
rg -l "styled|className|StyleSheet|tailwind|oklch|:root|design-token" --type-add 'style:*.{css,scss}' -tstyle -tts -ttsx 2>/dev/null | head -40
```

Read:
- Design system / tokens (`DESIGN-SYSTEM.md`, `tokens/*`, theme files) — the current visual language
- The primary screens/components and their information architecture
- `README.md` — who the real user is and the feeling the product should evoke
- Any mockups (`/design-mockup` output) or existing screenshots

Then a broad Explore agent:

> Explore this product's UI/UX. What are the core surfaces and flows? Where is the
> design generic, safe, or high-cognitive-load? What is the ONE screen or moment that
> matters most to the user, and how good is it today? Note the design tokens, type
> scale, color usage, motion, and interaction patterns already in place.

If the product renders, **see it**: screenshot the key surface (chrome-devtools MCP,
Playwright, or ask the user) — you cannot judge taste from source alone.

---

## Phase 2: Taste & Trend Calibration

Locate the current design's ceiling, and the best-in-class bar:

- What is the emotional target (calm / powerful / playful / precise) and does the UI hit it?
- Who does this *beautifully* today (name real products), and what specifically makes them feel crafted — spacing, type, motion, restraint, a signature interaction?
- What current design language applies (dark-first true-greys, oklch color, liquid-glass depth, bento layout, spatial/motion, progressive disclosure) — and where would it *serve the user*, not just look trendy?
- Where is the product spending "ink" (labels, chrome, edges, chartjunk) that could be removed?

Use WebSearch/WebFetch for current references when the domain is unfamiliar — ground the bar in what shipped, not memory.

---

## Phase 3: The Bold Move

Pick ONE direction that pushes the boundary. Bias toward moves that are simultaneously
bolder AND simpler. Candidate axes (choose the one with the highest taste-per-effort):

- **Information architecture** — a fundamentally clearer way to structure/reveal the data (verdict-first, progressive disclosure, collapse-to-cards, spatial grouping).
- **Signature interaction** — one interaction so good it defines the product (a gesture, a transition, a direct-manipulation moment).
- **Visual language** — a distinctive, coherent, restrained system (color as meaning only, a real type scale, depth via elevation not borders).
- **Preattentive clarity** — encode the thing that matters on the strongest perceptual channel so the eye lands on it in one fixation.
- **Motion with intent** — transitions that explain state change, never decorate.

---

## Phase 4: Proposal (strict format)

Output exactly this — no feature dumps. **One direction.**

```
## The Vision: [Evocative Name]

[2–3 sentences: what the redesigned experience is and what the user feels using it.]

### Why this is the move
[2–4 sentences: what it fixes (taste AND cognitive load), why it's novel vs
competitors, why it fits THIS product and user, and what existing surface it builds on.]

### The signature moment
[One sentence — the "show a friend" beat. The single interaction/view that sells it.]

### Before → After
| Dimension | Today | After |
|-----------|-------|-------|
| First glance (what the eye lands on) | ... | ... |
| Cognitive load | ... | ... |
| Emotional register | ... | ... |

### Design specifics (make it buildable)
- Color: [semantic tokens, oklch, contrast/colorblind notes]
- Type: [scale, weights, mono usage]
- Layout / hierarchy: [structure, spacing rhythm, what's hidden until hover/drill]
- Motion: [what animates, why, prefers-reduced-motion floor]

### Accessibility floor (non-negotiable)
[WCAG contrast, colorblind-safe encoding, reduced-motion, keyboard/focus.]
```

---

## Phase 5: Prototype hook (optional)

Offer to make it real, not just described:
- `/design-mockup` — author a high-fidelity mockup following the design-system conventions.
- `/html-plan` — render the direction as a self-contained visual page for review.
- Or implement directly against the product's tokens.

---

## Constraints

- **One direction per run.** For more, re-run `/rs-innovate-design`.
- Must raise taste **and** lower cognitive load — decoration alone is rejected.
- Grounded in the real product, its tokens, and its actual user — no generic redesigns.
- Accessibility is a floor, not a feature: contrast, colorblind-safe (redundant non-color cues), reduced-motion, focus states.
- Color carries meaning, never decoration; semantic palettes stay semantic.
- Demonstrable in a 30-second show-and-tell.
- Novelty must never cost clarity. If a bold idea confuses, it's not the move.
