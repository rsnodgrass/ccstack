---
description: |
  CodeGraph-powered codebase exploration. Traces call chains, maps dependencies,
  answers "where is X used?" and "what calls Y?". Use when asked to "find",
  "trace", "where is", "what calls", "who uses", "impact of changing",
  "how does X work", or "show me the code path". Proactively suggest when
  the user needs to understand unfamiliar code before making changes.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# Codebase Exploration

## Iron Law

**ANSWER WITH EVIDENCE. File paths, line numbers, and code snippets — not summaries.**

Every answer must include specific file:line references the user can navigate to.

---

## Phase 1: Choose Strategy

Check if CodeGraph is available:

```bash
[ -d .codegraph/ ] && echo "CODEGRAPH_AVAILABLE" || echo "CODEGRAPH_UNAVAILABLE"
```

### With CodeGraph (preferred — instant lookups)

Use CodeGraph MCP tools for faster exploration:

| Question Type | Tool | Example |
|--------------|------|---------|
| "Where is X defined?" | `codegraph_search` | Find symbol by name |
| "What calls X?" | `codegraph_callers` | Trace callers of a function |
| "What does X call?" | `codegraph_callees` | Trace callees of a function |
| "What breaks if I change X?" | `codegraph_impact` | Impact analysis |
| "Show me X's code" | `codegraph_node` | Get source code + details |
| "What's relevant to task Y?" | `codegraph_context` | Context for a task |

### Without CodeGraph (fallback — still effective)

Use Grep + Glob + Read:

| Question Type | Approach |
|--------------|----------|
| "Where is X defined?" | `grep -rn "func X\|class X\|const X\|export.*X" --include="*.go" --include="*.ts"` |
| "What calls X?" | `grep -rn "X(" --include="*.go" --include="*.ts"` |
| "What files match pattern?" | `find . -name "*.go" -path "*/handlers/*"` |

---

## Phase 2: Investigate

Execute the appropriate searches based on the user's question. Chain multiple lookups when needed:

1. **Find the symbol** — locate the definition
2. **Read the code** — understand what it does (Read tool with specific line ranges)
3. **Trace connections** — find callers, callees, dependencies
4. **Map the flow** — if data flow question, trace through all layers

---

## Phase 3: Answer

Present findings with:

1. **Direct answer** — one sentence answering the question
2. **File references** — `file_path:line_number` for each relevant location
3. **Code snippets** — key excerpts (not full files)
4. **Call chain** — if tracing a flow, show the chain:
   ```
   handler (api/handlers/recording.go:45)
     → service (internal/services/recording.go:112)
       → repo (internal/repo/recording.go:78)
         → DB query (internal/repo/queries/recording.sql:23)
   ```
5. **Mermaid diagram** — for complex dependency graphs or data flows:
   ```mermaid
   graph LR
     A[Handler] --> B[Service]
     B --> C[Repository]
     C --> D[Database]
   ```

```
STATUS: ANSWERED | PARTIAL (explain what's missing) | UNABLE (explain why)
```
