Prompt GPT
```text
Create a prioritized plan for a series of discrete, self-contained projects to
systematically migrate the VistA/M developer experience to a modern VSCode
git CI/CD TDD based one.  Start with generic high yield broad-audience ones
(general M language related, VSCode related) that all the downstream more specific
tooling would re-use as part of the workflow.  also make a prioritized list of those
things that should be done outside the M langauge / outside of VistA so that it can
live in the most modern ecosystem.
```

A practical migration should proceed as a **layered program of small, composable projects**, each delivering standalone value while enabling downstream work. The sequence below prioritizes **broad reuse, minimal coupling to VistA internals, and maximum leverage of modern tooling**.

---

# I. Guiding Structure

## Architectural Layers (build in this order)

```text
1. Language Layer (M tooling, parsing, linting)
2. Editor Layer (VS Code integration)
3. Source Control Layer (Git-native workflows)
4. Sync Layer (filesystem ↔ M runtime)
5. Test Layer (unit + integration)
6. Build/Deploy Layer (CI/CD)
7. Data/Analysis Layer (mirrors, search, indexing)
```

Each project below fits into one of these layers.

---

# II. Phase 1 — Foundational (High Yield, Broad Audience)

These are **language-level and editor-level primitives** that everything else depends on.

---

## Project 1 — M Language Grammar + Parser (Critical Path)

### Goal

Create a **formal, machine-readable grammar + parser for MUMPS**

### Deliverables

* TextMate grammar (for syntax highlighting)
* Tree-sitter parser (preferred for tooling)
* AST generator

### Why first

* Enables linting, refactoring, navigation, indexing
* Eliminates reliance on regex-based tooling

### Scope

* Full routine parsing
* Labels, commands, arguments
* Global references

### Lives outside M

✔ Yes (Node/Rust ecosystem)

---

## Project 2 — VS Code M Language Extension

### Goal

Modern editing experience for M routines

### Features

* Syntax highlighting (from Project 1)
* Snippets (DO, SET, NEW, etc.)
* File association (`*.m`)
* Basic symbol navigation

### Optional (later)

* Hover docs
* Go-to definition
* Cross-reference indexing

### Dependency

* Project 1

### Lives outside M

✔ Yes

---

## Project 3 — M Linter (XINDEX Replacement + Extension)

### Goal

Replace and surpass XINDEX

### Features

* Undefined variable detection
* Naked global detection
* Entry point validation
* Style rules
* Configurable ruleset

### Advanced

* Complexity analysis
* Dead code detection
* Dependency graph

### Integration

* CLI tool
* VS Code diagnostics
* Pre-commit hook

### Dependency

* Project 1

### Lives outside M

✔ Yes

---

## Project 4 — Routine File Format Standard

### Goal

Define canonical mapping:

```text
M routine ↔ filesystem representation
```

### Deliverables

* Naming conventions
* Encoding rules
* Label formatting standards

### Example

```text
src/routines/ABC.m
```

### Why critical

* Enables Git as source of truth
* Required for sync + CI/CD

### Lives outside M

✔ Yes

---

# III. Phase 2 — Developer Workflow Core

---

## Project 5 — M Runtime Bridge CLI

### Goal

Command-line interface to interact with M runtime

### Features

* Load routine from file
* Export routine to file
* Execute entry point
* Capture output

### Example

```bash
mcli load ABC.m
mcli run ABC^TEST
mcli export ABC
```

### Implementation

* Wrapper around YottaDB / GT.M
* Uses subprocess or direct bindings

### Why critical

* Foundation for automation
* Enables CI/CD and testing

### Lives outside M (with thin M layer)

✔ Mostly outside

---

## Project 6 — Filesystem ↔ M Sync Engine

### Goal

Bi-directional synchronization

### Features

* Push routines → M
* Pull routines ← M
* Detect diffs
* Batch operations

### Advanced

* Namespace support
* Conflict detection

### Dependency

* Project 4, 5

### Lives outside M (with M helpers)

✔ Mostly outside

---

## Project 7 — Git Integration Toolkit

### Goal

Make Git the authoritative workflow

### Features

* Pre-commit:

  * Run linter
  * Validate syntax
* Pre-push:

  * Run tests
* Diff tooling:

  * Routine-aware diffs

### Optional

* Semantic diff (label-aware)

### Lives outside M

✔ Yes

---

# IV. Phase 3 — Testing & Quality

---

## Project 8 — M Unit Test Framework

### Goal

Native testing inside M

### Features

* Assertions
* Test discovery
* Isolated execution

### Pattern

```text
TESTABC.m
```

### Must live in M

✔ Yes (execution layer)

---

## Project 9 — External Test Runner

### Goal

Modern test orchestration

### Features

* Run M tests via CLI
* Aggregate results
* Output:

  * JUnit XML
  * JSON

### Integration

* CI pipelines
* VS Code test explorer

### Dependency

* Project 5, 8

### Lives outside M

✔ Yes

---

## Project 10 — Snapshot/Data Testing Framework

### Goal

Validate global state

### Features

* Export globals → JSON
* Compare snapshots
* Detect regressions

### Use cases

* FileMan data validation
* Business logic verification

### Lives outside M (export via M)

✔ Hybrid

---

# V. Phase 4 — Build & Deployment

---

## Project 11 — Declarative Package System (KIDS Replacement)

### Goal

Replace KIDS with modern packaging

### Deliverables

```yaml
package.yml
```

### Features

* Define routines
* Define data migrations
* Define install steps

### Execution

* CLI applies package to M system

### Optional

* Generate KIDS for compatibility

### Lives outside M

✔ Yes

---

## Project 12 — CI/CD Pipeline Templates

### Goal

Standardize automation

### Example (GitHub Actions)

```yaml
- lint
- build
- spin up YottaDB container
- load routines
- run tests
- deploy
```

### Deliverables

* Reusable templates
* Docker images

### Lives outside M

✔ Yes

---

# VI. Phase 5 — Advanced Developer Experience

---

## Project 13 — Language Server (LSP)

### Goal

Full IDE experience

### Features

* Go-to definition
* Find references
* Autocomplete
* Refactoring support

### Dependency

* Project 1

### Lives outside M

✔ Yes

---

## Project 14 — Global/Data Explorer UI

### Goal

Replace % utilities UX

### Features

* Browse globals
* Query data
* Visualize structures

### Implementation

* Web UI or VS Code panel

### Backend

* M → JSON bridge

### Lives outside M

✔ Hybrid

---

## Project 15 — SQLite / Search Mirror

### Goal

Modern data access layer

### Features

* Export FileMan → relational model
* Full-text search
* Vector search (optional)

### Use cases

* Developer search
* Analytics
* Code navigation

### Lives outside M

✔ Yes

---

# VII. Prioritized “Outside M” Investment List

These should be aggressively externalized to maximize leverage:

---

## Tier 1 (Do First — Highest ROI)

1. **Parser + AST (Project 1)**
2. **VS Code extension (Project 2)**
3. **Linter (Project 3)**
4. **Runtime CLI bridge (Project 5)**
5. **Filesystem sync engine (Project 6)**

---

## Tier 2 (Core Workflow)

6. Git hooks + workflow tooling (Project 7)
7. External test runner (Project 9)
8. CI/CD templates (Project 12)

---

## Tier 3 (Modernization Multipliers)

9. Declarative packaging (Project 11)
10. Language server (Project 13)
11. Data explorer UI (Project 14)

---

## Tier 4 (Advanced / Strategic)

12. SQLite/search mirror (Project 15)
13. Snapshot testing (Project 10)

---

# VIII. What Should NOT Be Migrated Out

These remain in M by necessity:

* Routine execution
* FileMan schema operations
* Global storage
* Kernel APIs
* Transaction semantics

---

# IX. Execution Strategy

## Recommended Order (Condensed)

```text
1 → Parser
2 → VS Code extension
3 → Linter
4 → File format standard
5 → Runtime CLI
6 → Sync engine
7 → Git workflow
8 → Testing framework
9 → CI/CD
10 → Packaging
```

---

# X. Key Insight

The most critical early investment is:

> **A real parser + AST for M**

Without it:

* Linting is weak
* Refactoring is impossible
* LSP cannot exist
* Tooling remains fragile

With it:

* Entire modern ecosystem becomes feasible

---

# Bottom Line

* Start with **language + editor primitives** that benefit all users
* Build a **thin bridge to M**, not a replacement
* Move everything else into:

  * VS Code
  * Git
  * CI/CD
  * External tooling ecosystems

This approach yields **incremental value at every step** while converging toward a fully modern developer experience.
