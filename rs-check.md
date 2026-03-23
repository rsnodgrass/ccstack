---
description: |
  Pre-merge quality gates. Runs typecheck, lint, build, and tests in parallel.
  Use when asked to "check before merge", "run quality gates", "pre-PR check",
  "make sure this is ready", or "run checks". Proactively suggest before any
  PR creation or merge to main.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# Pre-Merge Quality Gates

## Iron Law

**NO PR WITHOUT PASSING QUALITY GATES.**

Every merge to main triggers a real deployment. A failing quality gate caught locally saves 10 minutes of CI wait and a blocked deploy pipeline.

---

## Stop Conditions

**Only stop for:**
- Build failures that require architectural changes
- Test failures that need human judgment (flaky vs real)

**Never stop for:**
- Auto-fixable lint errors (fix them with `--write` / `--fix`)
- Formatting issues (auto-format and report)
- Missing imports (suggest the fix)

---

## Phase 1: Detect Project Type

Scan the working directory to identify what quality gates apply:

```bash
# detect project markers
[ -f Makefile ] && echo "HAS_MAKEFILE"
[ -f go.mod ] && echo "HAS_GO"
[ -f package.json ] && echo "HAS_NODE"
[ -f tsconfig.json ] && echo "HAS_TYPESCRIPT"
[ -f biome.json ] || [ -f biome.jsonc ] && echo "HAS_BIOME"
[ -f turbo.json ] && echo "HAS_TURBO"
[ -f pyproject.toml ] && echo "HAS_PYTHON"
```

Build the gate list based on markers found.

---

## Phase 2: Run Quality Gates in Parallel

Run ALL applicable gates simultaneously. Capture output from each.

### Monorepo (Makefile + Turbo)
```bash
make typecheck 2>&1 | tee /tmp/rs-check-typecheck.txt &
make lint 2>&1 | tee /tmp/rs-check-lint.txt &
make build 2>&1 | tee /tmp/rs-check-build.txt &
wait
```

### Go Project
```bash
go vet ./... 2>&1 | tee /tmp/rs-check-vet.txt &
golangci-lint run ./... 2>&1 | tee /tmp/rs-check-lint.txt &
go build ./... 2>&1 | tee /tmp/rs-check-build.txt &
go test ./... -count=1 -short 2>&1 | tee /tmp/rs-check-test.txt &
wait
```

### TypeScript Project
```bash
tsc --go --noEmit 2>&1 | tee /tmp/rs-check-typecheck.txt &
npx @biomejs/biome check . 2>&1 | tee /tmp/rs-check-lint.txt &
wait
```

### Python Project
```bash
ruff check . 2>&1 | tee /tmp/rs-check-lint.txt &
mypy . 2>&1 | tee /tmp/rs-check-typecheck.txt &
wait
```

---

## Phase 3: Auto-Fix What's Fixable

For lint/format failures, attempt auto-fix before reporting:

- **Biome:** `npx @biomejs/biome check --write .` (fixes formatting + safe lint rules)
- **Go:** `golangci-lint run --fix ./...`
- **Ruff:** `ruff check --fix .`

After auto-fix, re-run the failing gate to verify it passes now.

Report each auto-fix: `[AUTO-FIXED] <tool>: <what was fixed>`

---

## Phase 4: Report

Output a summary table:

```
QUALITY GATES
+------------------+--------+------------------------------------------+
| Gate             | Status | Details                                  |
+------------------+--------+------------------------------------------+
| Typecheck        | PASS   |                                          |
| Lint             | PASS   | 3 issues auto-fixed                      |
| Build            | FAIL   | missing import: pkg/foo                  |
| Tests (unit)     | PASS   | 142 passed, 0 failed                     |
+------------------+--------+------------------------------------------+
```

For each FAIL:
1. Show the first error (not the full output — just the actionable part)
2. Suggest a fix
3. Offer to apply the fix

```
STATUS: PASS | FAIL | PASS_WITH_FIXES
```

If all gates pass (including after auto-fixes): `STATUS: PASS`
If any gate fails and cannot be auto-fixed: `STATUS: FAIL`
