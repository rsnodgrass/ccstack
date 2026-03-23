---
description: |
  API contract validation between Go backend and TypeScript frontend. Finds
  mismatches in response structs vs interfaces. Use when asked to "check API
  contract", "validate types match", "Go struct vs TS interface", "find type
  mismatches", or "why is the frontend getting wrong data". Proactively suggest
  when modifying API response types in either Go or TypeScript.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# API Contract Validation

## Iron Law

**THE BACKEND IS THE SOURCE OF TRUTH. The frontend must match it exactly.**

If Go says `snake_case` and TypeScript says `camelCase`, the mismatch is a bug waiting to happen. If Go omits a field, TypeScript won't magically receive it.

---

## Phase 1: Identify Contracts

Find Go response structs and their TypeScript counterparts:

### Go Side
```bash
# find response structs in handlers
grep -rn 'type.*Response struct' --include="*.go" apps/ internal/ packages/
grep -rn 'json:"' --include="*.go" apps/ internal/ packages/ | head -50
```

### TypeScript Side
```bash
# find corresponding interfaces/types
grep -rn 'interface.*Response' --include="*.ts" --include="*.tsx" apps/ packages/
grep -rn 'type.*Response' --include="*.ts" --include="*.tsx" apps/ packages/
```

Match Go structs to TS interfaces by name pattern (e.g., `RecordingResponse` in Go ↔ `RecordingResponse` or `Recording` in TS).

---

## Phase 2: Compare Field-by-Field

For each matched pair, read both definitions and compare:

| Check | What to Look For |
|-------|-----------------|
| **Missing fields** | Field in TS but not in Go (will always be undefined) |
| **Extra fields** | Field in Go but not in TS (data loss on client) |
| **Type mismatches** | `string` vs `number`, `boolean` vs `string` |
| **Nullability** | Go `*string` (nullable) vs TS `string` (required) |
| **Naming** | Go `json:"user_id"` vs TS `userId` (camelCase conversion) |
| **Nested types** | Embedded structs vs nested interfaces |
| **Enum values** | Go constants vs TS union types |

---

## Phase 3: PII Boundary Check

For endpoints under `/api/v1/public/*`:

```bash
grep -rn 'public' --include="*.go" apps/api-go/internal/handlers/ | grep -i 'route\|handler\|register'
```

Check response types for PII fields:
- Email addresses
- User IDs (raw, not prefixed)
- Phone numbers
- IP addresses
- Session tokens

These MUST NOT appear in public endpoint responses.

---

## Phase 4: Prefix ID Check

Verify all ID fields use type prefixes:

```bash
# find ID fields in response structs
grep -n 'ID.*string.*json:' --include="*.go" apps/ internal/
```

Check each ID field:
- Must be prefixed (e.g., `rec_`, `team_`, `usr_`, `repo_`, `spk_`)
- Raw UUIDs only at DB boundary
- Use `ids.ParseXXX()` functions, never raw UUID parsing

---

## Phase 5: Report

```
CONTRACT VALIDATION REPORT
Contracts checked: N (matched Go struct ↔ TS interface pairs)

MISMATCHES:
| Go Struct          | TS Interface       | Issue                              | Severity |
|--------------------|--------------------|-------------------------------------|----------|
| RecordingResponse  | Recording          | `speaker_id` missing in TS         | HIGH     |
| TeamResponse       | TeamData           | `email` in public endpoint (PII!)  | CRITICAL |
| RepoResponse       | RepoInfo           | `id` is raw UUID, needs `repo_` prefix | HIGH  |

PII Check: CLEAN | VIOLATIONS (list)
Prefix IDs: CLEAN | VIOLATIONS (list)
STATUS: CLEAN | MISMATCHES_FOUND | CRITICAL (PII leak)
```
