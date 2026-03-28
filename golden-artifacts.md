---
description: "Build verifiable golden artifact corpus for a project. Use before /extreme-software-optimization to establish the reference baseline that makes isomorphism claims provable."
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
---

# /golden-artifacts — Golden Artifacts Corpus Builder

**Iron Law: AN OPTIMIZATION WITHOUT A REFERENCE CORPUS IS JUST GAMBLING. FREEZE BEHAVIOR FIRST, CHANGE IT SECOND.**

**argument-hint:** `[target — module path or function name]` (default: full project survey)

---

## Phase 1: Survey

1. Read `CLAUDE.md`, `README.md`, and any `docs/` for project context.
2. Identify the language(s) and testing frameworks in use.
3. Map the current test coverage:
   - Unit tests: what functions are tested?
   - Integration tests: what workflows are covered?
   - Existing golden files: any `testdata/`, `.golden/`, `fixtures/`, `snapshots/` directories?
   - Fuzz targets: any existing fuzz corpus?
4. Identify **pure functions** (deterministic, no side effects) — best candidates for golden output testing.
5. Identify **critical paths** — functions optimizations are most likely to target (serialization, hashing, encoding, parsing, hot loops).

Report findings before proceeding. If golden files already exist, check if they are stale (compare against current output) before adding new ones.

---

## Phase 2: Generate Golden Outputs

For each critical pure function identified in Phase 1:

### 2a. Design the Input Set

Cover these categories for every function:

| Category | Examples |
|----------|---------|
| Empty / zero | `""`, `[]`, `0`, `nil` |
| Single element | `["x"]`, `1`, one-byte slice |
| Typical small | Representative real-world small input |
| Typical medium | Representative real-world medium input |
| Large / stress | Near-maximum size the function is expected to handle |
| Boundary | Off-by-one from limits, max integer values |
| Special characters | Unicode, null bytes, escape sequences (string-processing functions) |
| Adversarial | Inputs that historically caused bugs; hash collision inputs if relevant |

### 2b. Capture and Freeze

Run the current implementation against the input set and write outputs to disk:

```
.golden/
  <module>/
    <function>/
      case_empty.golden
      case_single.golden
      case_typical_small.golden
      case_typical_medium.golden
      case_large.golden
      case_boundary.golden
      case_adversarial_01.golden
      README.md   ← documents what each case tests and why
```

Binary outputs: store as hex dumps with a `.hex` extension alongside raw `.golden` files.

### 2c. Write the Golden Test Harness

Write a test that:
1. Reads each `.golden` file
2. Runs the function with the corresponding input
3. Asserts byte-exact equality with the stored output
4. Reports which specific case failed (not just "golden test failed")

Must support a `REGENERATE=1` or `--update-goldens` flag to refresh corpus when behavior is intentionally changed.

**Language-specific patterns:**

```rust
// Rust: include golden files at compile time for zero-fs-dep tests
#[test]
fn golden_encode_empty() {
    let got = encode(&[]);
    let want = include_bytes!("../testdata/encode/case_empty.golden");
    assert_eq!(got, want, "encode(empty) diverges from golden");
}
```

```go
// Go: table-driven golden tests
func TestGoldenEncode(t *testing.T) {
    for _, tc := range goldenCases {
        t.Run(tc.name, func(t *testing.T) {
            got := Encode(tc.input)
            want := readGolden(t, tc.name)
            if !bytes.Equal(got, want) {
                t.Errorf("encode(%s): got %x, want %x", tc.name, got, want)
            }
        })
    }
}
```

```python
# Python: pytest with parametrize
@pytest.mark.parametrize("case_name,input_val", golden_cases())
def test_golden_encode(case_name, input_val, golden_dir):
    result = encode(input_val)
    golden_path = golden_dir / f"{case_name}.golden"
    if os.getenv("REGENERATE"):
        golden_path.write_bytes(result)
    assert result == golden_path.read_bytes(), f"diverges from golden: {case_name}"
```

---

## Phase 3: Property-Based Oracle Tests

Golden files prove specific inputs. Property-based tests prove *all* inputs (within the generator's distribution).

### 3a. Write Oracle Tests

Run the **current implementation** as the reference oracle against randomized inputs:

```python
# Python: Hypothesis
from hypothesis import given, strategies as st

@given(st.binary(max_size=65536))
def test_encode_oracle(data):
    """Encode output must be deterministic and round-trip stable."""
    result1 = encode(data)
    result2 = encode(data)
    assert result1 == result2, "encode is non-deterministic"
    assert decode(result1) == data, "round-trip failed"
```

```rust
// Rust: proptest
proptest! {
    #[test]
    fn encode_roundtrip(data: Vec<u8>) {
        let encoded = encode(&data);
        let decoded = decode(&encoded).unwrap();
        prop_assert_eq!(data, decoded);
    }
}
```

```go
// Go: rapid
func TestEncodeRoundTrip(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        data := rapid.SliceOf(rapid.Byte()).Draw(t, "data")
        encoded := Encode(data)
        decoded, err := Decode(encoded)
        if err != nil || !bytes.Equal(data, decoded) {
            t.Fatalf("round-trip failed for input %x", data)
        }
    })
}
```

### 3b. Mutation Oracle Tests

For functions likely to be optimized, add an explicit reference vs optimized comparison test:

```python
# Keep original as `_reference` until optimized version has 1,000+ runs with zero divergences
@given(st.binary(max_size=65536))
def test_optimized_matches_reference(data):
    assert encode_optimized(data) == encode_reference(data)
```

---

## Phase 4: Fuzz Corpus Seeds

If the project has fuzz targets, generate a starter corpus from golden inputs:

```bash
# Rust: cargo-fuzz
mkdir -p fuzz/corpus/fuzz_encode
for f in .golden/encode/*.golden; do
    cp "$f" "fuzz/corpus/fuzz_encode/$(basename $f .golden)"
done

# Go: native fuzzing
mkdir -p testdata/fuzz/FuzzEncode/corpus

# Python: atheris
mkdir -p fuzz_corpus/encode
```

Seed with: all golden input files + edge case inputs + any historically problematic inputs.

---

## Phase 5: Snapshot Tests (APIs and CLI tools)

For APIs and CLIs where output is structured (JSON, text, HTML):

- Write snapshot tests for all API endpoints with representative payloads
- Write snapshot tests for all CLI command outputs
- Commit snapshots to the repo — they fail loudly when output changes

```bash
# Node: vitest --update / jest --updateSnapshot
# Python: syrupy (pytest plugin)
# Go: cupaloy
```

---

## Phase 6: Corpus Documentation

Create `.golden/README.md`:

```markdown
# Golden Artifacts Corpus

## Purpose
Reference corpus for isomorphism verification during optimization passes.
Do not regenerate except when intentional behavior changes are made.

## Regenerating
Run: `REGENERATE=1 pytest tests/golden/`  (or language equivalent)
Commit regenerated files with a message explaining the intentional behavior change.

## Coverage
| Module | Functions | Input Cases | Property Tests | Fuzz Seeds |
|--------|-----------|-------------|----------------|------------|
| encode | 3         | 42          | 2 (1,000 runs) | 42         |
| hash   | 2         | 18          | 1 (1,000 runs) | 18         |

## Adding New Cases
1. Add input to the test file
2. Run with `REGENERATE=1` to capture initial output
3. Review output manually — confirm it is correct
4. Commit test + golden file together
```

---

## Phase 7: Report

```
GOLDEN ARTIFACTS REPORT
────────────────────────
Functions covered:     N
Golden files created:  M
Property test suites:  K (total fuzz runs: 1,000 each)
Fuzz corpus seeds:     P
Snapshot tests:        Q

Corpus location: .golden/
Regeneration:    REGENERATE=1 <test command>

STATUS: READY_FOR_OPTIMIZATION | PARTIAL_COVERAGE | NEEDS_MANUAL_REVIEW
```

### Status Definitions

- **READY_FOR_OPTIMIZATION** — All critical functions have golden coverage, property tests, and oracle tests. Safe to run `/extreme-software-optimization`.
- **PARTIAL_COVERAGE** — Core functions covered; secondary functions lack full corpus. Optimization can proceed with care.
- **NEEDS_MANUAL_REVIEW** — Non-deterministic outputs found, or side-effecting functions require manual validation before freezing.
