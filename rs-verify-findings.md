---
description: |
  Senior principal engineer review of debug findings. Cross-correlates each hypothesis
  against actual code, datasheets, and memory maps to catch hallucinations. Built for
  tricky embedded/microprocessor debugging sessions where hardware, memory, and timing
  interact and prior analysis may have invented plausible-sounding but wrong claims.
  Use when asked to "verify findings", "sanity check this analysis", "are these real",
  "review hypotheses", or after a long debug session before acting on conclusions.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Agent
  - WebFetch
---

# Verify Debug Findings (Anti-Hallucination Review)

## Iron Law

**EVERY CLAIM MUST BE GROUNDED IN A FILE:LINE, REGISTER ADDRESS, OR DATASHEET PAGE — OR IT IS DISCARDED.**

A finding that "sounds right" is worse than no finding at all: it burns hours of fix work on a problem that does not exist. Plausibility is not evidence. If the reviewer cannot point at the exact byte of code, the exact register bit, or the exact datasheet section that proves the claim, the claim is a hallucination and gets struck.

---

## When To Use

- After a long debugging session produced a list of hypotheses / root causes
- When findings came from another agent, another session, or a subagent that did not touch code directly
- Embedded / MCU work (Teensy, ESP32, STM32, nRF, RP2040) where claims reference peripherals, DMA, interrupts, memory regions, or cache behavior
- Before implementing fixes based on reasoning chains longer than ~3 hops

---

## Input

The findings to verify. Accept any of:
- A pasted list from a previous turn
- A path to a markdown file / scratchpad
- A bead id (`bd show <id>`)
- "the last analysis" — in which case scroll the conversation and extract every concrete claim

First step is always to **enumerate the claims** as a flat numbered list before verifying anything. No verification starts until the list exists.

---

## Phase 1: Extract Atomic Claims

Break the input into atomic, independently-verifiable claims. One claim = one assertion that can be true or false on its own.

Bad (compound):
> "The DMA ISR races with the SPI flush and corrupts the ring buffer because the tail pointer is updated before the cache is invalidated."

Good (atomic):
1. `spi_flush()` reads `ring.tail` without holding the DMA lock
2. DMA ISR writes `ring.tail` at the end of transfer
3. The ring buffer lives in a non-cache-coherent region (DTCM? OCRAM? SDRAM?)
4. Cache is not invalidated between the ISR write and the flush read

Each atomic claim gets its own row in the verification table. Compound claims hide the one sub-claim that is actually wrong.

---

## Phase 2: Classify Each Claim

Tag each claim with what kind of evidence would prove or disprove it:

| Class | Evidence source |
|-------|-----------------|
| **CODE** | A specific function/line in this repo |
| **HEADER** | A `#define`, struct layout, or linker symbol |
| **HARDWARE** | Datasheet / reference manual / errata page |
| **MEMORY** | Linker script, MPU config, cache region attributes |
| **TIMING** | Measured trace, scope capture, logic analyzer output |
| **BEHAVIOR** | An observed log line or reproduction |
| **INFERENCE** | Derived from other claims — no direct evidence |

**INFERENCE claims cannot stand alone.** They are only valid if every claim they depend on is verified. Mark their dependencies explicitly.

---

## Phase 3: Ground-Truth Each Claim

Work the list top-to-bottom. For each claim, perform the verification that matches its class. Do not skip to the next until the current one is marked.

### CODE claims
1. Open the cited file at the cited line with Read
2. Confirm the symbol exists and does what the claim says
3. If the line number is off by more than ±5, treat as **unverified** — hallucinated line numbers are the most common failure mode
4. If the function was renamed, moved, or deleted: mark **stale**

### HEADER claims
1. Grep for the `#define` or struct. The value must match exactly.
2. For struct layouts, verify `sizeof` and field offsets with a small test or `pahole`-style inspection if available
3. For linker symbols, check the actual `.map` file or linker script — not what "should" be there

### HARDWARE claims
1. Identify the exact chip variant (Teensy 4.1 = IMXRT1062, ESP32-S3 ≠ ESP32, etc.)
2. Fetch the relevant datasheet / reference manual page. Quote the section number.
3. Check errata — silicon bugs invalidate a lot of "correct per datasheet" reasoning
4. If no datasheet access, mark **unverified — needs hardware doc** and stop. Do NOT infer from "how chips usually work."

### MEMORY claims
1. Read the linker script (`*.ld`) to find the region the symbol lives in
2. Check MPU / cache attributes for that region — on Cortex-M7 (Teensy 4.x) DTCM is non-cached, OCRAM2 is cached, etc.
3. Verify with `nm` or map file which region the symbol actually landed in (compilers move things)
4. Never assume cache coherency. Prove it.

### TIMING claims
1. If there is no trace / scope capture / logic analyzer output attached, TIMING claims are **unverified by construction**
2. Reasoning like "the ISR takes ~2µs so it must finish before…" is inference, not evidence. Downgrade to INFERENCE.
3. Cycle-counted claims require the actual cycle counter readout

### BEHAVIOR claims
1. Find the exact log line / failure output in scrollback or a log file
2. If only paraphrased ("it crashes sometimes"), mark **unverified — needs repro**
3. Match the log to the emitting code path — an error string present in two places can mislead

### INFERENCE claims
1. List dependencies. If ANY dependency is unverified or stale, mark the inference **unverified**
2. Walk the logic chain in one direction only — forward from evidence to conclusion. If you have to reason backward ("for the bug to exist, X must be true"), that is a hypothesis, not a verification.

---

## Phase 4: Correlate Across Claims

Hallucinations cluster. A wrong register name in one claim often reappears in three others. After individual verification, cross-check:

- Do any two verified claims contradict each other? (e.g., "runs on core 0" vs "runs in FIQ context")
- Do claims reference the same symbol with different spellings / addresses / sizes?
- Does the story require impossible sequencing (A must finish before B, but B triggers A)?
- Are peripheral register offsets consistent across claims? Off-by-one on a register map is a classic.

Flag any contradiction — even if both claims individually verified, the combination is wrong somewhere and needs a human.

---

## Phase 5: Verdict Per Claim

Each claim ends up in exactly one bucket:

| Verdict | Meaning | Action |
|---------|---------|--------|
| **CONFIRMED** | Evidence directly proves the claim | Safe to act on |
| **REFUTED** | Evidence directly disproves the claim | Strike from the analysis; warn loudly |
| **STALE** | Code/symbol has moved or changed | Re-run the original investigation against current code |
| **UNVERIFIED** | No accessible evidence either way | DO NOT act on; mark as hypothesis only |
| **CONTRADICTS** | Conflicts with another verified claim | Escalate to human — something in the chain is wrong |

**A fix plan may only depend on CONFIRMED claims.** Any UNVERIFIED / STALE / CONTRADICTS claim that the plan rests on must be resolved first, not fixed around.

---

## Phase 6: Parallel Deep-Dives (Optional)

For claims that need heavy verification (datasheet lookup, large code traces, memory map analysis), spawn parallel Agent calls:

- `Explore` agent per CODE claim cluster — "verify these 5 claims about function X against actual source, report file:line confirmation or refutation"
- `teensy-expert` / `embedded-systems` agent for HARDWARE and MEMORY claims on MCU targets
- `debugger` agent for BEHAVIOR claims when a repro exists

Each subagent gets ONLY the claims it owns, plus the Iron Law. Do not let a subagent verify its own inferences.

---

## Phase 7: Report

```
FINDINGS VERIFICATION
Source: <where the claims came from>
Total atomic claims: <N>

CONFIRMED (<count>):
  [C1] <one-line claim> — evidence: <file:line | datasheet §X.Y | log line>
  ...

REFUTED (<count>):
  [R1] <one-line claim> — actual: <what the evidence shows instead>
  ...

STALE (<count>):
  [S1] <one-line claim> — <symbol moved/renamed/deleted at commit ...>
  ...

UNVERIFIED (<count>):
  [U1] <one-line claim> — missing: <datasheet | repro | scope capture | ...>
  ...

CONTRADICTIONS (<count>):
  [X1] claim A vs claim B — <why they cannot both be true>
  ...

SAFE-TO-ACT SUBSET:
  <the minimal set of CONFIRMED claims that actually support the proposed fix>

BLOCKED FIXES:
  <fixes from the original plan that depend on non-CONFIRMED claims, and what evidence
   would be required to unblock each one>

STATUS: CLEAN | NEEDS-EVIDENCE | HALLUCINATION-DETECTED
```

---

## Stop Conditions

- All claims classified into one of the five buckets
- Report emitted
- If HALLUCINATION-DETECTED: stop and surface to the user before any fix work begins
- If NEEDS-EVIDENCE: list the specific artifacts required (scope trace, datasheet section, repro steps) and stop

Never silently "promote" an UNVERIFIED claim to CONFIRMED because it seems likely. That is the exact failure mode this skill exists to prevent.

---

## Anti-Patterns To Reject

- **"The datasheet probably says…"** — fetch it or mark UNVERIFIED
- **"This is standard ARM behavior"** — cite the Architecture Reference Manual section or downgrade
- **"The line number is approximate"** — re-grep and get the exact one, or mark STALE
- **"Both claims are true, they just need to be reconciled"** — no. Contradictions escalate, they do not reconcile themselves.
- **"I verified it in the previous session"** — previous-session verification does not carry over; the code may have changed. Re-verify.
- **"The subagent said it checked"** — confirm the subagent cited a file:line. Unsourced confirmations are hallucinations with extra steps.
