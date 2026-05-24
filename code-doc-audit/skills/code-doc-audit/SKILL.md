---
name: code-doc-audit
description: Audit documentation and inline comments against the current code for **concept, design, and implementation accuracy**. Verifies that the behaviors, business rules, file/test inventories, and CLI flags described in markdown and source comments still match what the code does — and surfaces key concepts present in the code but missing from the docs. Use when the user asks for a "thorough code base analysis against documentation", a "doc audit", to "make sure docs align with the code conceptually", or after a feature/refactor that changes behavior. For pure file:line / code-snippet drift, use **doc-sync** instead (this skill explicitly does NOT re-check line numbers). Repeat rounds until a full pass finds no further changes.
argument-hint: "[optional path to scope the audit]"
allowed-tools: Bash(grep:*), Bash(find:*), Bash(wc:*), Bash(ls:*), Bash(jq:*), Bash(go:*), Bash(npm:*), Bash(pytest:*), Read, Edit, Write
---

# Code/Doc Audit (Concepts, Design, Implementation)

Verify that every markdown file and every inline source comment describes the codebase **accurately and comprehensively** at the level of **concepts, design, and behavior** — and add documentation for key behaviors that exist in code but aren't documented.

> **Scope boundary.** This skill deliberately does **not** check file:line references or whether quoted code snippets match the current source character-for-character. Those checks belong to **doc-sync**. If a stale line number is the only problem, this skill should leave it alone and (optionally) note that doc-sync should be re-run. Conversely, if a code snippet's *meaning* has drifted from current behavior (inverted condition, removed branch, renamed concept), that **is** in scope here — fix the description regardless of whether the line number is right.

## When to invoke

- "Review the codebase and make sure the docs and inline comments reflect it."
- "Do a thorough doc audit of concepts and behavior."
- "Are there any business rules in the code that aren't documented?"
- After a behavior-changing refactor or new feature.
- Before cutting a release.
- After a long stretch of work where docs were added in passing.

Pair with **doc-sync** afterward for the file:line / snippet check.

> **Why the audit needs an upfront plan, not just a checklist.** The procedure below is generic. Real projects have *specific* drift patterns the generic procedure does not enumerate (e.g. a stale `go test -cover` figure in the README table, a test-inventory table missing one new `*_test.go`, an inline comment pointing at a renamed doc section). If you skip Step 0 — "load any project-specific patterns and enumerate categories upfront" — you will catch *some* drift on each run but miss other categories, and a second invocation will then find them. Step 0 + Step 12 together close that loop: Step 0 enumerates what you will check (including memory-loaded project-specific patterns) and Step 12 captures any *new* categories you found this run, so the next audit converges in one pass.

## Category Catalog

These are the drift categories the audit checks. Use this catalog to seed your Step 0 enumeration; load any additional project-specific categories from memory at the start of Step 0. Each item lists a one-line check.

1. **File / test inventories** — `find` for actual files; diff against trees/tables in docs. Both directions: stale (in doc, not on disk) and missing (on disk, not in doc).
2. **Source directory trees** — every production source file appears in the docs' tree (`README.md`, `ARCHITECTURE.md`, etc.); no entries reference deleted files.
3. **Test coverage tables** — every percentage in a "Coverage" table in docs matches the test runner's current output (`go test -cover`, `pytest --cov`, `jest --coverage`, etc.). Tolerance ≈2pp.
4. **Behavioral claims** — caching policies, rate-limit handling, error fallbacks, query-mode / state-machine rules. Trace one execution path through the code for each claim.
5. **CLI / flag parity** — every flag defined in source appears in the README's flag list; every flag in the docs still exists in source.
6. **Config schema parity** — every field in `config.example.yaml` (or equivalent) exists in the schema struct, and vice versa. Watch for `omitempty` fields that may legitimately be commented out.
7. **Renamed / removed concepts** — a glossary's "_Avoid_:" list (or commit history `rename`/`remove` log) names the old terms; grep code + docs for survivors.
8. **Magnitude claims** — counts and sizes drift fast. "45+ constants", "~N lines", "300-line file". Either keep within tolerance or replace with magnitude-free phrasing.
9. **Cross-doc section references** — every `"Section Name"` cited in a source comment or another doc must exist as a heading in the target doc.
10. **Inline cross-file references** — every `// See foo.go` comment must point to a current path.
11. **Package / module doc comments** — each package's doc comment should describe current files and behaviors, not the pre-refactor state.
12. **Missing concept coverage** — exported symbols that encode non-obvious business rules (retry budgets, TTLs, quotas, environment variables) and have no entry in README/ARCHITECTURE.

This catalog is intentionally generic. **Project-specific categories** (e.g. "battery scope must specify today's live Day Mode") live in project memory (Step 0) and supplement the catalog. When a new category is discovered during the audit, capture it in Step 12.

## Procedure

### Step 0 — Plan the audit (load memory, enumerate categories)

Before running any audit step, do these in order:

1. **Load project memory.** Look for project-specific drift checklists in (in priority order):
   - `~/.claude/projects/<project-slug>/memory/` (any `*audit*.md` or `*doc*.md` feedback-type entries)
   - `.audit-checklist.md` or `docs/AUDIT_CHECKLIST.md` in the project root
   - `CLAUDE.md` in the project root (look for a "code-doc-audit" section)

   If any source enumerates project-specific drift patterns, treat them as additions to the Category Catalog above. If none exists, continue with just the standard catalog.

2. **Detect project stack.** Run:
   ```bash
   ls go.mod package.json pyproject.toml Cargo.toml setup.py 2>/dev/null
   ```
   Use the result to fill in extensions, test patterns, and comment style for Steps 1–9:

   | Marker | Source ext | Test file pattern | Test command | Comment prefix |
   |--------|-----------|-------------------|-------------|----------------|
   | `go.mod` | `.go` | `_test.go` suffix | `go test ./...` | `//` |
   | `package.json` | `.ts`, `.tsx`, `.js`, `.jsx` | `.test.*` / `.spec.*` | `npm test` | `//` |
   | `pyproject.toml` / `setup.py` | `.py` | `test_*.py` / `*_test.py` | `pytest` | `#` |
   | `Cargo.toml` | `.rs` | `mod tests` blocks | `cargo test` | `//` |

3. **Enumerate the categories you will check this run.** Output the list to the user before starting. The list = the standard 12 categories above **plus** anything loaded from memory **minus** anything explicitly out of scope for this run.

   Example output:
   > "I will check these 14 categories this audit: [standard 1-12] + [project memory: 'coverage table' (already in catalog), 'battery-scope phrasing must specify today's live Day Mode']."

4. **Track per-category status.** As you proceed through Steps 1-9, mark each enumerated category as `clean`, `fixed-N-things`, or `flagged-for-user`. This per-category accounting is what Step 10's termination check uses (not just "issues array empty").

Skipping Step 0 produces audits that find different drift on every invocation. **Do not skip it** — even if the catalog above seems exhaustive, enumerating it explicitly forces blind spots into the open before you start auditing.

### Step 1 — Inventory the codebase

```bash
# Production source — use extension(s) from Step 0 (example: Go)
find . -name "*.go" -not -path "*/.git/*" -not -name "*_test.go" | sort

# Test source — use test-file pattern from Step 0 (example: Go)
find . -name "*_test.go" -not -path "*/.git/*" | sort

# All markdown
find . -name "*.md" -not -path "*/.git/*" -not -path "*/.claude/*" | sort
```

Map the result. You should know: every package, every test file, every markdown file, and which exported symbols exist in each.

### Step 2 — Identify recent behavior changes

If the user implies "since the last audit", inspect recent commits and `git status` for modified files. Pay special attention to changes that move semantics:
- New exported functions / types / constants that change a user-visible contract.
- New CLI flags.
- Behavior changes in flow-control functions (e.g. caching, error handling, retry policies, rate limiting).
- Renamed, deleted, or relocated concepts (a renamed function often leaves its old name in docs).
- New test files that cover a new behavior the docs don't mention.

Run `git log --since="2 weeks ago" --oneline` and skim recent commits for "fix", "refactor", "replace", "rename", "remove" — those are the high-signal signals for doc drift. When filtering by file extension, use the extension(s) detected in Step 0 rather than hardcoding `*.go`.

### Step 3 — Verify file / test inventories

Many docs include directory trees or test-file tables. For each tree/table entry, confirm the file exists and the description still matches.

```bash
# Find inventory-style mentions (directory trees and test tables)
grep -rn "├──\s*[a-z_.-]*\.go\|^\s*-\s*\`[a-z_-]*\.go\`" docs/ README.md QUICKSTART.md

# For each claimed file, verify
ls path/to/claimed/file
```

Common drifts:
- A test file is renamed / removed but the tree still lists it.
- A file's stated *purpose* (the comment to the right of `├──`) drifted because the file's role changed.

Fix by removing stale rows, updating descriptions, or adding new rows for new files. If coverage moved to a different file, add a one-sentence pointer rather than just removing.

### Step 4 — Verify behavioral claims (the core of this skill)

For each markdown paragraph that describes program behavior — caching policy, rate-limit handling, error fallbacks, timezone rules, query types, etc. — open the source and confirm the claim is still true. Watch for:

- **Inverted conditions.** "X happens when Y" but the code now does the opposite.
- **Removed branches.** "For past dates, …" but past-date handling has been folded into a single rule.
- **Renamed concepts.** Old name (e.g. `ProcessItem`) cited where the code now uses the new name (e.g. `HandleItem`).
- **Stale magnitudes.** "60-second cache lifetime" when the rule is now per-resource. "20+ constants" when there are 56 — significantly off should be updated.
- **Obsolete files / markers.** "a sentinel file holds the state" when the implementation has moved to an in-memory or database-backed approach.
- **Promises the code no longer keeps.** "Cache is never deleted" when the test cleanup wipes it.

Be skeptical of behavioral examples in docs. Trace at least one execution path through the code for each behavior the docs claim. If the *meaning* has drifted, rewrite the description — even if the line number reference happens to still be correct.

### Step 5 — Verify CLI / flag documentation

Use the grep pattern that matches your stack's argument-parsing library:

```bash
# Go (flag / pflag / cobra)
grep -rn "BoolVar\|StringVar\|IntVar\|Float64Var" --include="*.go" . 2>/dev/null

# Python (argparse / click / typer)
grep -rn "add_argument\|@click\.\(option\|argument\)\|Annotated\[" --include="*.py" . 2>/dev/null

# TypeScript / Node (commander / yargs / meow)
grep -rn "\.option\|\.argument\|\.command" --include="*.ts" --include="*.js" . 2>/dev/null

# Rust (clap)
grep -rn "#\[arg\]\|Arg::new\|\.arg(" --include="*.rs" . 2>/dev/null
```

For each flag, confirm:
- README's CLI flag list mentions it (correct name, correct description).
- Help text on the flag matches the documented description.
- Any flag with non-trivial output (e.g. `--debug`, `--verbose`, `--dry-run`) has a dedicated subsection in README or ARCHITECTURE showing what it does and when to use it.
- Removed flags are not still listed.

### Step 6 — Find missing concept coverage

A concept counts as "key" if **at least one** of these is true:
- It is exported and is a non-trivial behavior contract (e.g. a retry budget, a cache TTL, a quota limit).
- It changes user-visible behavior (CLI flags, environment variables, output format).
- It is referenced in another doc but defined nowhere readable.
- It encodes a business rule that isn't obvious from the type signature (e.g. "records older than X days are treated as immutable and cached indefinitely").
- It has multiple call sites and a non-obvious invariant.

For each key concept, check if it appears in:
- `README.md` (user-facing)
- `docs/ARCHITECTURE.md` (designer-facing)
- The package doc comment of the package that owns it
- Inline doc comments on the exported symbol

If the concept is undocumented or only mentioned in passing, **add a section** in the most appropriate doc explaining:
- **What** the concept is (one sentence)
- **When** it applies (the trigger or branch condition)
- **Why** it exists (the business rule or design pressure that forced it)

Prefer adding to the doc that already discusses adjacent concepts (e.g. add a new retry rule to the existing "Error Handling" section, not to a fresh section). Don't create new top-level files unless absolutely necessary.

### Step 7 — Verify inline source comments

```bash
# Inline comments describing behavior, magnitudes, or cross-references
# Adapt --include and comment prefix (// or #) from Step 0

# Go / TypeScript / Rust (// comments)
grep -rn "// .*[0-9]\+ lines\|// .*TODO\|// .*FIXME\|// .*\.[a-z]*:[0-9]" --include="*.go" . 2>/dev/null

# Python (# comments)
grep -rn "# .*[0-9]\+ lines\|# .*TODO\|# .*FIXME\|# .*\.[a-z]*:[0-9]" --include="*.py" . 2>/dev/null

# Package / module doc comments — skim for stale claims about what the package owns
grep -rn "^// Package " --include="*.go" . 2>/dev/null        # Go
grep -rn "^\"\"\"" --include="*.py" . 2>/dev/null | head -20  # Python module docstrings
```

Spot-check comments that:
- Cite a line count or file size (these rot).
- Cross-reference another file by purpose ("see foo.go for the X handler").
- Describe a multi-step algorithm — does the code still match the steps?
- Reference a removed helper by name.
- Describe a "why" that has since been superseded.

Comments that simply restate the code below (`// Set x to 5` above `x = 5`) are not in scope — flag those for removal separately if egregious, but the focus here is *meaning drift*, not redundancy.

### Step 8 — Apply fixes

Edit in priority order, smallest blast radius first:

| Issue | Action |
|-------|--------|
| Stale file in inventory | Remove the entry; add prose pointing to where the coverage now lives if appropriate |
| Stale concept description | Rewrite to current behavior |
| Stale magnitude / count | Update if significantly off; leave if still approximate |
| Stale inline comment | Update or delete |
| Missing concept | Add a subsection in the most appropriate doc |
| Stale CLI flag | Sync README's CLI list with the project's flag definitions |

When rewriting a paragraph, preserve the *structure* of the surrounding doc (headings, table format, bullet style) so the diff stays minimal and reviewable.

### Step 9 — Verify with tests

Run the project's build and test suite to confirm no regressions. Use the test command from Step 0:

```bash
go test ./...           # Go (also run: go build ./...)
npm run build && npm test   # TypeScript / Node
pytest                  # Python
cargo test              # Rust
```

The build step (where it exists) catches stale exported names; tests catch behavioral regressions from any accidental code edits.

### Step 10 — Repeat

Go back to Step 3. Keep iterating until a full round produces zero changes.

> **Termination**: stop only when (a) every category from your Step 0 enumeration reports `clean` on the final pass, AND (b) the project's test suite is green. The bar is per-category cleanliness, not just "issues array empty" — a category can be empty *because you forgot to check it*, which is exactly the failure mode Step 0 is designed to prevent.

### Step 11 — (Optional) hand off to doc-sync

After this skill finishes, line numbers and code snippets may now be drifted because doc text shifted around. Suggest the user run `/doc-sync` next to mop up file:line and snippet drift.

### Step 12 — Capture new drift patterns for next time

Walk back through what you fixed this run. For each fix, ask: **"Was this category already in the standard catalog or in project memory before I started?"**

- **Yes** → no action; the existing checklist already covers it.
- **No** → this is a *new* drift category for this project. Without capturing it, the next audit will re-discover it the slow way. Write a project memory entry (feedback-type) so the next `/code-doc-audit` invocation loads it in Step 0 and checks it first.

Memory entry shape (feedback-type, one per pattern or one combined entry per project):

```markdown
---
name: feedback-code-doc-audit-<project>-checklist
description: Project-specific drift patterns to check on every /code-doc-audit run for <project>
metadata:
  type: feedback
---

When running `/code-doc-audit` on <project>, also check these categories in addition to the SKILL's standard catalog:

1. **<Category name>** — <one-line check, e.g. "every doc mention of <concept> must say <exact phrasing>; the code gate is at <file>:<symbol>">
2. ...

**Why:** <one sentence — usually the past incident that caused this drift>
**How to apply:** Run as an explicit step during Step 0; treat as a standard category from then on.
```

If the user does not maintain a project-memory directory, propose the equivalent content as a `docs/AUDIT_CHECKLIST.md` patch instead — Step 0 already looks for that file. The point is to make the discovery durable across runs.

**Why this step matters.** The audit's value compounds across runs only if discoveries are captured. Otherwise every audit redoes the same surface-scan and finds *different* missing categories. With Step 0 + Step 12 in place, the audit converges in one pass after the first time each pattern is seen.

## Output format

When done, summarize for the user in this shape:

```
**Stale concepts fixed:**
| File | Was | Now |
| ... | ... | ... |

**Missing concepts added:**
| File | Concept | Section |
| ... | ... | ... |

**Inventory fixes:**
| File | Change |
| ... | ... |

**Tests:** clean.

Recommend running /doc-sync next to verify file:line refs.
```

Keep the summary terse — the diff is the proof of work. Do NOT include line-number changes in this summary; if line numbers happen to shift, that's doc-sync's territory.

## Concrete examples of stale patterns

These are real-world instances of the categories above — useful for recognizing drift in the wild. Each maps to a Category Catalog entry.

- **Renamed terminology survivors** (Category 7) — old names like `ProcessItem` / `QueryType` / `--setup-oauth` survive in docs long after the code uses the new name.
- **Missing CLI flag entries** (Category 5) — a flag added in source with no corresponding entry in README's flag list, or no dedicated subsection for non-trivial flags like `--debug` / `--verbose` / `--dry-run`.
- **Stale debug / log-format samples** (Category 4) — README shows `[DEBUG] foo=N` but the code now prints `[DEBUG] foo: N`.
- **Stale cache / rate-limit magnitudes** (Category 4 + 8) — "60-second cache" claim when the rule is now per-resource (1h for one tier, 24h for another).
- **Test inventory rows for renamed / deleted files** (Category 1).
- **"X test file covers Y" pointing at the old file** (Category 1) — coverage moved to a different test file.
- **Package doc comments listing features the package no longer owns** (Category 11).
- **Brittle "Used in `<file list>`" comments** (Category 10) — enumerate caller sites; rot whenever a new caller is added.
- **Frozen magnitude claims** (Category 8) — "45+ constants" when the count is now 43 and the +N suggests "≥45".

## Quick first-pass commands

Adapt extension filters and patterns to your stack (detected in Step 0). Examples shown for Go.

```bash
# Recently modified production code (likely sources of doc drift)
# Replace '*.go' with your stack's extension; replace grep -v _test.go with your test-file pattern
git log --since="2 weeks ago" --name-only --pretty=format: -- '*.go' | grep -v _test.go | sort -u | grep -v '^$'

# Inventory-style file mentions in docs — replace .go with your extension
grep -rn "├──\s*[a-z_.-]*\.go\|^\s*-\s*\`[a-z_-]*\.go\`" docs/ README.md

# Exported symbols added recently (potential undocumented concepts)
# Go: uppercase function/type/const/var names
git log --since="2 weeks ago" -p -- '*.go' | grep -E "^\+func [A-Z]|^\+type [A-Z]|^\+const [A-Z]|^\+var [A-Z]" | sort -u
# Python: public functions/classes (no leading underscore)
# git log --since="2 weeks ago" -p -- '*.py' | grep -E "^\+def [a-z]|^\+class [A-Z]" | sort -u
# TypeScript: exported symbols
# git log --since="2 weeks ago" -p -- '*.ts' | grep -E "^\+export (function|class|const|type)" | sort -u

# CLI flags currently defined — use the pattern for your stack's arg-parsing library
grep -rn "BoolVar\|StringVar\|IntVar" --include="*.go" .       # Go
# grep -rn "add_argument\|@click" --include="*.py" .           # Python
# grep -rn "\.option\|\.argument" --include="*.ts" .           # TypeScript
```
