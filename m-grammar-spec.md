# tree-sitter-m — Specification v0.1

**Status:** draft for review
**License of this document:** AGPL-3.0 (matches the artifact it specifies)
**Validation corpus:** VEHU VistA distribution, 39,330 routines, running on YottaDB in vista-meta's existing Docker container

---

## Table of contents

* [1. Project identity](#1-project-identity)
* [2. Scope and non-goals](#2-scope-and-non-goals)
* [3. Architectural decisions](#3-architectural-decisions)
* [4. Grammar scope — ANSI Standard M](#4-grammar-scope--ansi-standard-m)
* [5. Output specification](#5-output-specification)
* [6. Validation methodology](#6-validation-methodology)
* [7. Repository layout](#7-repository-layout)
* [8. Toolchain and dependencies](#8-toolchain-and-dependencies)
* [9. Binding layer — vista-meta-ast-bind](#9-binding-layer--vista-meta-ast-bind)
* [10. Milestones and roadmap](#10-milestones-and-roadmap)
* [11. Risks and open questions](#11-risks-and-open-questions)
* [12. Success criteria for v1.0](#12-success-criteria-for-v10)

---

## 1. Project identity

**Name.** `tree-sitter-m`.

**Purpose.** A portable, generic, machine-readable grammar for the M
(MUMPS) programming language, implemented as a tree-sitter parser.
The grammar is the artifact; the 39,330-routine VistA corpus
running on YottaDB is the validation set, not the target. The
grammar must be reusable on RPMS, WorldVistA, OSEHRA, FOIA-VistA,
banking and insurance MUMPS shops, and any other M codebase with no
VistA-specific assumptions.

**License.** AGPL-3.0, matching YottaDB. M shops running on
AGPL-licensed engines (YottaDB, GT.M open-source line) inherit the
same posture; adopters on InterSystems IRIS are not blocked from
parsing their own code at build time, but are bound by AGPL terms if
they distribute modified versions of the parser.

**Relationship to vista-meta.** This grammar is an upstream
component, not a vista-meta sub-project. It lives in its own
repository (`tree-sitter-m`). vista-meta consumes it through a
separate, optional binding layer (`vista-meta-ast-bind`, also a
distinct repository) that joins parser output to vista-meta's
existing 19-TSV code model. Within vista-meta, the grammar replaces
the regex-based extractors in `build_routine_calls.py` and
`build_routine_globals.py`, and powers the next generation of the
VSCode extension's semantic features.

**Two driving use cases.** Per design, the grammar serves:

1. **Replace the regex extractor.** Close vista-meta's documented
   1.25% XINDEX disagreement gap and produce a defensibly correct
   call graph and global-reference graph, with every edge traceable
   to a precise byte range in source.
2. **Power IDE semantics.** Provide go-to-definition,
   find-references, hover, and semantic highlighting in the
   vista-meta VSCode extension, plus generic editor support
   (Neovim, Helix, Emacs, Zed) via the standard tree-sitter
   ecosystem.

Refactoring/transformation tooling and AI-agent backends are
plausible downstream consumers but are not in scope for v1.0.

---

## 2. Scope and non-goals

**In scope for v1.0:**

* ANSI Standard M (X11.1-1995 / ISO 11756:1999) — lexical grammar
  and concrete syntax
* All standard commands, intrinsic functions, intrinsic special
  variables, and operators
* Routine structure: column-zero tags, formal parameter lists,
  line-of-code blocks, dot-level DO blocks
* Postconditionals on commands and on individual arguments
* Indirection in all three forms (name, subscript, pattern),
  captured syntactically
* Pattern-match syntax (the `?` operator and pattern-element
  grammar)
* Comments (`;` to end of line)
* Concrete Syntax Tree as canonical output
* JSON CST serialization with versioned schema
* TSV adjacency-list emitter (lives in the binding layer, but
  schema specified here)

**Deferred to v0.2 or later:**

* YottaDB-specific extensions (`$ZTRNLNM`, `$ZSEARCH`, MERGE
  extensions, `%YDB*` intrinsics)
* InterSystems Caché / IRIS extensions (`&sql(...)`, `##class(...)`,
  `$ZF` family)
* DD-embedded MUMPS (the MUMPS fragments living inside FileMan data
  dictionaries; different lexical context — single-line, no tag
  structure, often bare expressions — and deserves its own
  sub-grammar)
* AST layer (separate downstream walker, not part of this grammar)
* Semantic analysis (symbol tables, call resolution, indirection
  resolution, type inference)
* Refactoring or transformation tooling
* Pre-ANSI dialects (DSM-11, MUMPS-11)

**Non-goals:**

* Replacing mfmt (mfmt v2 may consume the CST, but this grammar
  does not ship a formatter)
* A full language server (LSP) — the grammar is a building block
  for one, not the server itself
* Runtime analysis or M-to-X transpilation
* Anything VistA-specific in the grammar itself (see AD-02)

---

## 3. Architectural decisions

Four decisions drive everything else in this spec. They are listed
once here so downstream sections can reference them by number.

### AD-01: CST is canonical; AST is a derived layer maintained separately

The parser emits a Concrete Syntax Tree that preserves every token,
comment, whitespace span, and source position byte-exactly. This is
tree-sitter's native model. It is required by the IDE use case
(semantic features need exact source positions and round-trip
fidelity) and by the regex-replacement use case (defensible XINDEX
cross-validation requires that every extracted edge be traceable to
a precise byte range). An AST — a normalized, trivia-free semantic
view — is built downstream by walking the CST. The AST walker is
not part of this grammar project; it lives in whichever consumer
needs it. This avoids the historical failure mode of M parser
projects that conflated the two and were rewritten when refactoring
tooling appeared.

### AD-02: The grammar knows nothing about FileMan, PIKS, globals' semantic meaning, or VistA conventions

The parser sees `^DPT(DFN,0)` as a global reference with subscripts.
It does not know that `^DPT` is FileMan File 2, that File 2 is
PATIENT, or that PATIENT data is PHI. All such fusion happens in
`vista-meta-ast-bind`, a separate, optional module. This is the
only way the grammar can be reused on non-VistA M codebases, the
only way it can be tested in isolation against synthetic fixtures,
and the only way other groups can adopt it without inheriting
vista-meta's data model.

### AD-03: tree-sitter is the parser generator, with a custom external scanner in C for context-sensitive lexing

Tree-sitter wins on the IDE side (incremental parsing, mature error
recovery, broad editor integrations) and is the de facto standard
for new language grammars in 2026. M's column-zero tag rule,
dot-level DO block indentation, and command-abbreviation token
disambiguation require an external scanner — approximately 200–400
lines of C. This is normal tree-sitter practice (compare
tree-sitter-bash, tree-sitter-fortran, tree-sitter-python). The
grammar itself is `grammar.js`; the external scanner is
`src/scanner.c`. The generated `parser.c` is also committed, per
tree-sitter convention.

### AD-04: Three output formats, layered

The canonical artifact is the tree-sitter native parser
(`grammar.js` + `scanner.c` + generated `parser.c`). For consumers
outside the tree-sitter ecosystem, a **JSON CST dump** provides a
portable, language-agnostic interchange format with a versioned
schema. For analytics consumers wanting to join parser output
against vista-meta's existing 19 code-model TSVs, a **TSV
adjacency-list emitter** produces `ast-nodes.tsv`,
`ast-edges.tsv`, and `ast-globals.tsv`. The TSV emitter is part of
the vista-meta-ast-bind layer, not this repo.

---

## 4. Grammar scope — ANSI Standard M

This section enumerates what the grammar must recognize. It is
descriptive of the language surface, not prescriptive about
implementation choices in `grammar.js`.

### 4.1 Lexical layer

Whitespace is significant in M to a degree most languages do not
require. The grammar must distinguish:

* The single space between command and argument (required
  syntactically)
* The single space between consecutive commands on the same line
  (the command separator)
* Two-or-more spaces, which in some contexts terminate the
  parseable line and begin trailing comment territory
* Tab versus space (the SAC requires no tabs; the grammar must
  recognize both for tolerance)
* Line endings (`\r\n` and `\n` both legal)

### 4.2 Routine structure

A routine is a sequence of lines. Each line is one of:

* A blank line
* A comment-only line (`;` plus text)
* A tag line (label at column 0, optional formal parameter list,
  optional code on the same line)
* A code line (whitespace at column 0, then commands)

The first line of every routine is conventionally
`RoutineName ;comment` (the routine header line). The second line
is conventionally `;;version;package;**patches**;date` (the version
line that kids-vc canonicalizes). Neither is enforced by the
grammar; both are recognized as ordinary tag/comment lines.

The external scanner enforces the column-zero rule for tag
detection: a label-like token starting at column 0 begins a tag
definition; the same token preceded by whitespace is a reference,
not a definition.

### 4.3 Commands

ANSI M defines approximately 25 standard commands: `BREAK`,
`CLOSE`, `DO`, `ELSE`, `FOR`, `GOTO`, `HALT`, `HANG`, `IF`, `JOB`,
`KILL`, `LOCK`, `MERGE`, `NEW`, `OPEN`, `QUIT`, `READ`, `SET`,
`TCOMMIT`, `TROLLBACK`, `TSTART`, `USE`, `VIEW`, `WRITE`,
`XECUTE`. Every command may be abbreviated to its first 1–4 letters
per a fixed table (`SET` → `S`, `WRITE` → `W`, `XECUTE` → `X`). The
grammar must accept every valid abbreviation as the same token
kind. Disambiguation between `H` as `HANG` versus `H` as `HALT`
depends on argument presence and is a parsing concern, not a
lexing one.

Every command may carry a postconditional:
`command:expression argument`. Postconditionals also attach to
individual arguments of multi-argument commands:
`SET:cond x=1,y:cond2=2`. Both forms are first-class in the CST.

Multiple commands per line are separated by a single space;
multiple arguments per command are separated by commas. Both are
captured distinctly.

`LOCK` has its own argument syntax with `+` (incremental) and `-`
(decremental) prefixes and optional timeout. `FOR` has three
distinct forms (no argument; single value; range with increment).
`DO` and `XECUTE` accept both label references and quoted code
strings. All three are explicitly within v1.0 scope.

### 4.4 Expressions and operators

M evaluates strictly left-to-right with no operator precedence
(other than unary operators and parentheses). `2+3*4` is `20`, not
`14`. The grammar therefore needs no precedence climbing — a flat
left-fold over `binary_expression` nodes is correct. This is a
simplification relative to most languages.

Operators include arithmetic (`+ - * / \ # **`), string
concatenation (`_`), comparison (`= '= < > '< '> [ ] ]] '[ '] ']]`),
logical (`& ! '`), and the pattern-match operator `?`. Unary `+`,
`-`, `'` (NOT) are right-associated.

### 4.5 Intrinsic functions and special variables

ANSI M defines approximately 22 intrinsic functions (`$ASCII`,
`$CHAR`, `$DATA`, `$EXTRACT`, `$FIND`, `$FNUMBER`, `$GET`,
`$JUSTIFY`, `$LENGTH`, `$NAME`, `$NEXT`, `$ORDER`, `$PIECE`,
`$QLENGTH`, `$QSUBSCRIPT`, `$QUERY`, `$RANDOM`, `$REVERSE`,
`$SELECT`, `$STACK`, `$TEXT`, `$TRANSLATE`) and approximately 21
intrinsic special variables (`$DEVICE`, `$ECODE`, `$ESTACK`,
`$ETRAP`, `$HOROLOG`, `$IO`, `$JOB`, `$KEY`, `$PRINCIPAL`, `$QUIT`,
`$STORAGE`, `$SYSTEM`, `$TEST`, `$TLEVEL`, `$X`, `$Y`, etc.). Both
classes are abbreviable to one or two characters (`$E` =
`$EXTRACT`, `$P` = `$PIECE`). The grammar recognizes all of them,
with abbreviations.

`$Z*` names are the ANSI escape hatch for vendor extensions. The
grammar parses them as syntactically well-formed intrinsic-function
or intrinsic-variable nodes without interpreting them; YottaDB and
IRIS attach their own meanings, which the grammar does not encode.

### 4.6 Indirection

M has three forms of indirection, all written with `@`:

* **Name indirection**: `S @X=1` — `X` evaluates at runtime to a
  variable name
* **Subscript indirection**: `S ^GLO@X@(sub)=1` — `X` evaluates to
  a partial subscript path
* **Pattern indirection**: `I X?@PAT` — `PAT` evaluates to a
  pattern string

The grammar captures the `@` operator and its operand. Resolution
(what the indirected name actually refers to) is a runtime concern
and is out of scope. Static analyzers consuming the CST will treat
indirected references as opaque, with a flag.

### 4.7 Globals, locals, naked references

A global reference begins with `^`; a local does not. Subscripts
are parenthesized comma-separated expressions. A naked reference
`^(subs)` reuses the most recently referenced global name and
all-but-last subscript. The grammar captures this distinctly: a
`naked_global_reference` node has no name token, only subscripts;
resolution is downstream.

### 4.8 Pattern grammar

The right-hand side of `?` is its own mini-grammar: repetition
counts (`1`, `1.3`, `1.`, `.4`), pattern codes (`A`, `C`, `E`, `L`,
`N`, `P`, `U`), literal strings, alternation `(p1,p2)`, and
indirection. This is grammatically self-contained and is a
sub-grammar in `grammar.js`.

### 4.9 Dot-level DO blocks

```
 D
 . S X=1
 . S Y=2
 . D
 . . S Z=3
```

Block nesting is by leading dots, not whitespace. The number of
dots determines depth. The external scanner produces `INDENT` and
`DEDENT` tokens analogous to Python's, but counting dots rather
than spaces. This is the second-largest external scanner
responsibility after column-zero tag detection.

### 4.10 Comments

`;` begins a comment that extends to end of line. The grammar
preserves the comment text in the CST as trivia attached to the
surrounding node. Doc-comment conventions (`;;` for line-2 version
data, `;@summary` and `;@test` for vista-meta lint compatibility)
are not interpreted by the grammar; they are ordinary comments at
the syntactic level. Interpretation is a downstream concern for
the lint tool.

---

## 5. Output specification

### 5.1 Layer 1 — Tree-sitter native

The repository ships `grammar.js`, `src/scanner.c`, and the
generated `src/parser.c`. Editor integrations (Neovim, Helix,
Emacs, Zed, VS Code via the tree-sitter extension, GitHub's
semantic indexer) consume these directly. Standard tree-sitter
conventions apply:

* `queries/highlights.scm` — syntax highlighting
* `queries/locals.scm` — local-scope queries for IDE features
* `queries/tags.scm` — tag/symbol queries for code navigation

These query files are part of the grammar repo and are versioned
with it.

### 5.2 Layer 2 — JSON CST dump

A small Python utility (`m-cst-dump`) traverses the tree-sitter
parse tree and emits one JSON file per source routine. The schema
is versioned (`schema_version: "1"`) and stable. Each node carries:

| Field | Type | Description |
| --- | --- | --- |
| `kind` | string | Node kind from `grammar.js` (e.g. `set_command`, `global_reference`) |
| `start_byte` | integer | UTF-8 byte offset of node start |
| `end_byte` | integer | UTF-8 byte offset of node end |
| `start_point` | `[row, col]` | Zero-indexed line and column (column in UTF-8 bytes) |
| `end_point` | `[row, col]` | Zero-indexed end line and column |
| `is_named` | bool | Tree-sitter named-node flag |
| `text` | string | Source text — only for leaf nodes; omitted for internal nodes to save space |
| `children` | array | Child nodes |

Trivia (whitespace, comments) is preserved as named nodes when
significant (comments) and as `extra` nodes otherwise.

**Round-trip stability is a guarantee.** Parsing a routine, dumping
it, and re-parsing the dump must produce an identical CST. This is
enforced in CI (see §6.5).

The dumper uses `orjson` if available for performance, falling back
to stdlib `json`. No other dependencies.

### 5.3 Layer 3 — TSV adjacency

The TSV emitter is part of the `vista-meta-ast-bind` repo, not this
one, but its schema is specified here so the grammar repo's CI can
verify the JSON dump contains all necessary information.

* **`ast-nodes.tsv`**: one row per significant CST node — stable
  ID, parent ID, kind, source position, routine name. Joinable to
  vista-meta's `routines.tsv` on `routine_name`.
* **`ast-edges.tsv`**: one row per call/reference edge. Drop-in
  superset of vista-meta's existing `routine-calls.tsv`, with new
  columns for `caller_tag`, `callee_tag`, exact byte range, and
  `indirection_flag`.
* **`ast-globals.tsv`**: one row per global reference — routine,
  tag, line, global root, subscript depth, read-or-write kind.
  Drop-in superset of `routine-globals.tsv`.

**Schema-compatibility invariant.** The new TSVs must contain every
column the old ones did, in the same order, so existing consumers
(the VSCode extension, the CLI, downstream queries) continue to
work unmodified during the transition.

---

## 6. Validation methodology

### 6.1 Corpus

The validation corpus is the 39,330 routines of the VEHU VistA
distribution running in vista-meta's existing YottaDB Docker
container. Routines are pulled from `/opt/VistA-M/r/` via the
existing container mount. The corpus is augmented with synthetic
fixtures (§6.3) to cover constructs that VEHU does not exercise.

### 6.2 Pass-rate target — v1.0

The grammar must parse **≥99.5%** of the 39,330 VEHU routines with
**zero error nodes** in the resulting CST. The remaining ≤0.5%
(≤197 routines) is the budgeted failure tail and is logged in
`validation/known-failures.tsv` with one row per failed routine,
error position, and triage notes (typically: vendor-specific
Z-extensions, deliberately malformed test fixtures, or genuine
grammar gaps to address in v0.2).

### 6.3 Synthetic fixtures

A `test/corpus/` directory contains hand-written fixtures covering
each grammar production with both positive (well-formed) and
negative (error-recovery) cases. tree-sitter's standard
`tree-sitter test` runs them in CI. Required coverage:

* All 25 standard commands with all valid abbreviations
* All postconditional positions (command-level and argument-level)
* All three indirection forms
* Naked global references in valid and invalid contexts
* Dot-level DO blocks at depths 1, 2, 3, and 5
* Pattern-match expressions covering all pattern codes and
  repetition counts
* `LOCK` with `+` / `-` and timeout
* All three `FOR` forms
* Z-prefix vendor functions and special variables (parsed as
  generic intrinsic nodes)
* Pathological whitespace cases (tabs, multi-space, trailing
  whitespace)
* Maximum-line-length cases (M standard caps lines at 255
  characters; VistA SAC at 245)

### 6.4 XINDEX cross-validation

vista-meta already drives XINDEX through the VMXIDX bridge and
produces `xindex-xrefs.tsv` (214,011 call edges, validated against
today's regex extractor at 98.75% agreement). The new grammar must
achieve **≥99.0% agreement** with XINDEX on call-graph edges and
**≥99.0% agreement** on global-reference extraction.

Agreement is computed by `validate_against_xindex.py` (existing
script, lightly modified to consume AST output instead of regex
output). Disagreements are logged with side-by-side reasoning to
`validation/xindex-deltas.tsv`. The expectation is that most
disagreements will be cases where the grammar correctly excludes
edges that the regex falsely included (commented-out code is the
known regex over-extraction); these count as correct grammar
behavior but are still recorded for transparency.

### 6.5 Round-trip stability

For every routine in the validation corpus,
`parse → JSON dump → re-parse` must produce a CST identical to the
original. Mismatches are CI failures. This is the strongest
statement of grammar correctness: the JSON dump is a lossless
serialization of the CST.

### 6.6 Continuous integration

GitHub Actions runs three checks on every PR:

1. `tree-sitter test` — synthetic fixtures, must pass 100%
2. `tree-sitter parse --quiet` over the full VEHU corpus — fail if
   pass rate drops below 99.5%
3. `validate_against_xindex.py` — XINDEX agreement, fail if either
   metric drops below 99.0%

CI run time is budgeted at <5 minutes for the full validation pass.
If the corpus parse takes longer in practice, the pass-rate check
is sampled (1,000 routines) on PR runs and run full-corpus only on
`main` and on tag pushes.

---

## 7. Repository layout

```
tree-sitter-m/
├── grammar.js                  # The grammar
├── src/
│   ├── scanner.c               # External scanner (column-zero, dot blocks, abbrevs)
│   ├── parser.c                # Generated; committed per tree-sitter convention
│   └── tree_sitter/parser.h
├── queries/
│   ├── highlights.scm          # Syntax highlighting
│   ├── locals.scm              # Local-scope queries for IDE features
│   └── tags.scm                # Tag/symbol queries
├── bindings/
│   ├── node/                   # Node.js binding (tree-sitter standard)
│   ├── python/                 # Python binding
│   └── rust/                   # Rust binding
├── test/
│   └── corpus/                 # Synthetic fixtures, one .txt file per production
├── validation/
│   ├── run-vehu-corpus.sh      # Pulls 39,330 routines from container, parses each
│   ├── known-failures.tsv      # Budgeted ≤0.5% tail with triage notes
│   ├── xindex-deltas.tsv       # Disagreements with XINDEX, with reasoning
│   └── validate_against_xindex.py
├── tools/
│   └── m-cst-dump              # JSON CST dumper (Python, stdlib + optional orjson)
├── docs/
│   ├── spec.md                 # This document
│   ├── grammar-walkthrough.md  # Per-production rationale
│   └── m-language-notes.md     # M peculiarities for grammar-implementer onboarding
├── .github/workflows/
│   └── ci.yml                  # tree-sitter test + corpus + XINDEX agreement
├── LICENSE                     # AGPL-3.0
├── README.md
└── package.json                # tree-sitter conventions
```

---

## 8. Toolchain and dependencies

* **tree-sitter CLI** ≥0.24, for grammar generation, testing, and
  parsing
* **C compiler** (gcc or clang) for the external scanner
* **Node.js** ≥20 for `tree-sitter generate` and the Node binding
* **Python** ≥3.11 for `m-cst-dump` and the validation scripts
* **Docker + YottaDB** — vista-meta's existing container, used only
  at validation time
* **GitHub Actions** for CI

No runtime dependencies for the parser itself beyond a C compiler.
The Python tooling uses stdlib plus optional `orjson`. This honors
vista-meta's "no runtime dependencies" discipline.

---

## 9. Binding layer — vista-meta-ast-bind

A separate, optional repository. The grammar repo does not depend
on it; vista-meta depends on both.

**Purpose.** Walk JSON CST dumps from `m-cst-dump` and produce the
analytics TSVs (`ast-nodes.tsv`, `ast-edges.tsv`,
`ast-globals.tsv`) that join to vista-meta's existing 19
code-model TSVs. Replace the regex extractors
(`build_routine_calls.py`, `build_routine_globals.py`) in
vista-meta's pipeline. Annotate AST nodes with PIKS classification
by joining global roots to `files.tsv`.

**Outputs.** The three TSVs specified in §5.3.

**Schema-compatibility invariant.** New TSVs are drop-in supersets
of the old ones — every existing column preserved, new columns
appended. Existing consumers (VSCode extension, CLI, downstream
queries) continue to work unmodified during the transition.

**Migration plan.** vista-meta's `make` pipeline introduces a new
target `make ast-extract` that runs the grammar + binding layer.
For one release cycle, both `make routine-calls` (regex) and
`make ast-extract` run in parallel; outputs are diffed by
`validate_against_ast.py` to surface any regressions before the
regex extractor is removed.

---

## 10. Milestones and roadmap

| Milestone | Scope | Exit criterion |
| --- | --- | --- |
| **M0** | Grammar skeleton: routine structure, comments, blank/tag/code line distinction, external scanner for column-zero tags | `tree-sitter test` passes on ~20 hand-written line-type fixtures |
| **M1** | All 25 commands with abbreviations, postconditionals, multi-command lines | `tree-sitter test` passes per-command; `tree-sitter parse` succeeds on a hand-picked sample of 50 simple VEHU routines |
| **M2** | Expressions, operators, full ANSI intrinsic functions/variables, local and global references, naked references | Sample expands to 500 routines; ≥95% pass rate |
| **M3** | Indirection (all three forms), pattern grammar, dot-level DO blocks (external scanner extended) | Sample expands to 5,000 routines; ≥98% pass rate |
| **M4** | Validation harness: full VEHU corpus run, known-failures triage, XINDEX cross-validation | Full corpus ≥99.0% pass; XINDEX agreement ≥98.5% |
| **M5** | JSON CST dumper, round-trip stability test in CI, query files (`highlights.scm`, `locals.scm`, `tags.scm`) | Round-trip stable on full corpus; basic syntax highlighting works in Neovim and VS Code |
| **M6** | Hardening: close gap to 99.5% by triaging each `known-failures.tsv` entry; XINDEX agreement to ≥99.0%; documentation | All v1.0 success criteria met (§12) |
| **M7** *(separate repo)* | `vista-meta-ast-bind`: TSV emitter, regex-extractor replacement, PIKS annotation | vista-meta pipeline runs end-to-end against AST output; produces TSVs schema-compatible with existing code model |
| **M8** *(separate repo)* | vista-meta VSCode extension upgrade: consume tree-sitter-m for go-to-definition, find-references, semantic highlight | Sidebar adds "Definitions" and "References" sections backed by the parser |

M0 through M6 deliver v1.0 of `tree-sitter-m`. M7 and M8 are
vista-meta consumers and ship on their own cadence.

---

## 11. Risks and open questions

**Tree-sitter expressivity for indirection.** Tree-sitter is an
LR-style parser. M's indirection is fully runtime-resolved; the
parser captures the syntax but cannot resolve what is being
indirected. This is fine for the IDE use case (greyed-out
indirected references with a tooltip) but is a known limitation
for downstream call-graph extraction. Mitigation: flag indirected
calls in `ast-edges.tsv` and accept them as an unresolvable tail.
Estimate from the existing regex extractor: <2% of call sites
involve indirection.

**Performance at corpus scale.** Tree-sitter typically parses at
10–50 MB/s. The VEHU corpus is approximately 200 MB of M source;
full-corpus parse should fit in 5–20 seconds on a modern laptop.
The JSON dump is ~10× the parse time. If CI run time becomes a
concern, JSON dump is sampled in CI and run in full only nightly.

**External scanner complexity.** `scanner.c` is the highest-risk
component. M's column-zero rule, dot-level DO indentation, and
command-abbreviation token disambiguation interact in non-obvious
ways. Budget: 400 LOC of C with extensive unit tests. If the
scanner exceeds 800 LOC, that is a signal to reconsider grammar
structure before pushing further.

**YottaDB extensions in VEHU.** The VEHU corpus is largely ANSI M
but contains some YottaDB-specific intrinsics (`$ZTRNLNM`,
`$ZSEARCH`, etc.) and may use YottaDB's `MERGE` extensions. A
spot-check of the failure tail at M4 will quantify how many of the
budgeted ≤0.5% failures are YDB-specific. If the number is large,
v0.2 prioritizes a YottaDB extension layer.

**DD-embedded MUMPS.** The MUMPS expressions embedded in FileMan
data dictionaries (extracted to `DD-code/*.m` by kids-vc) are a
different parsing context: typically single-line, no tag structure,
often bare expressions rather than command sequences. v1.0 does
not handle this. v0.2 introduces a `dd_expression` entry rule that
shares the expression sub-grammar with the main grammar.

**License compatibility for IRIS adopters.** AGPL-3.0 matches
YottaDB. InterSystems IRIS shops parsing their own M code at build
time are not licensing-blocked (they possess the source they
parse), but they are bound by AGPL terms if they distribute
modified versions of the parser. This is the standard AGPL position
and is intentional.

**Disagreement direction with XINDEX.** Per vista-meta's existing
analysis, disagreements between the regex extractor and XINDEX are
predominantly cases where regex over-extracts (e.g., calls inside
comments). The grammar should also out-perform XINDEX in some
cases (XINDEX has known limitations around indirection and dynamic
DO targets). The CI gate is bidirectional: large divergence in
*either* direction triggers a manual review, since silent
regressions in either tool are equally bad.

---

## 12. Success criteria for v1.0

The following must all be true for v1.0 release:

1. **Pass rate**: ≥99.5% of VEHU's 39,330 routines parse with zero
   error nodes.
2. **Round-trip stability**: 100% of parsed routines round-trip
   through JSON dump and re-parse to identical CSTs.
3. **XINDEX agreement**: ≥99.0% on call-graph edges; ≥99.0% on
   global-reference extraction.
4. **CI gating**: All three checks (synthetic fixtures, corpus
   pass rate, XINDEX agreement) gate every PR on `main`.
5. **Editor integration**: `tree-sitter-m` is installable in
   Neovim and VS Code via standard tree-sitter mechanisms, with
   working syntax highlighting.
6. **Documentation**: `docs/spec.md`,
   `docs/grammar-walkthrough.md`, and `docs/m-language-notes.md`
   are complete and reviewed.
7. **Known failures triaged**: `validation/known-failures.tsv`
   has one row per failed routine in the ≤0.5% tail, each tagged
   with category (vendor-extension / test-routine /
   grammar-gap-deferred-to-v0.2 / other).
8. **License**: AGPL-3.0 stamped in `LICENSE` and in every source
   file's header.
9. **Reproducibility**: A fresh clone, `npm install &&
   tree-sitter generate && tree-sitter test`, succeeds on Linux
   and macOS.
