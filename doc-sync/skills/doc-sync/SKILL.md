---
name: doc-sync
description: Verify that all file:line references in markdown documentation point to real code at those exact lines, and that code examples in docs match the current source. Use when the user asks to "review docs against code", "check that line references are accurate", "make sure comments match the code", "verify documentation", or after a refactor that could invalidate line numbers. Also use proactively when inline comments or code examples in docs look stale. Repeat rounds until a full pass finds no further changes.
argument-hint: "[path to docs, default all *.md]"
allowed-tools: Bash(grep:*), Bash(find:*), Bash(wc:*), Bash(go:*), Bash(npm:*), Bash(pytest:*), Read, Edit
---

# Documentation Sync

Ensure every file:line reference in markdown docs points to current code, and every code example matches actual source.

## Procedure

### Step 0 — Detect project stack

```bash
ls go.mod package.json pyproject.toml Cargo.toml setup.py 2>/dev/null
# Bash has no single marker file — check for .sh files instead
find . -maxdepth 2 -name "*.sh" -o -name "*.bash" 2>/dev/null | head -1
```

Use the result throughout this procedure for extensions, test commands, and comment style:

| Marker file | Source extensions | Test command | Comment prefix |
|-------------|-------------------|--------------|----------------|
| `go.mod` | `.go` | `go test ./...` | `//` |
| `package.json` | `.ts`, `.tsx`, `.js`, `.jsx` | `npm test` | `//` |
| `pyproject.toml` / `setup.py` | `.py` | `pytest` | `#` |
| `*.sh` / `*.bash` files present | `.sh`, `.bash` | `bats .` | `#` |

If multiple markers exist, apply all relevant extension patterns.

### Step 1 — Collect all markdown files

```bash
find . -name "*.md" -not -path "*/.git/*" -not -path "*/node_modules/*" | sort
```

If the user passed a path argument, restrict to that path.

### Step 2 — Extract all file:line references

Look for these patterns in markdown files:
- `**Location**: \`file.go:N\`` or `**Location**: \`file.go:N-M\``
- `` `file.go:N` `` inline in text
- `// file.go:N-M` comments inside fenced code blocks

```bash
# File:line references in backticks — matches any source extension
grep -rn "\`[a-z_./-]*\.[a-z]*:[0-9]" docs/ README.md QUICKSTART.md 2>/dev/null

# Cross-file references embedded in comment lines (// or # style)
grep -rn "// .*\.[a-z]*:[0-9]\|# .*\.[a-z]*:[0-9]" docs/ 2>/dev/null
```

### Step 3 — Verify each reference

For every reference found, Read the source file starting at the claimed line number and check:

1. **Line exists** — file has at least that many lines
2. **Content matches** — the function/struct/variable described is actually at that line
3. **Code examples match** — if the doc shows a code block, compare it to the actual source (simplified examples are OK; inverted logic or removed fields are not)
4. **Description is accurate** — text describing what the code does reflects current behavior

### Step 4 — Check inline comments in source files

```bash
grep -rn "// .*[0-9]+ lines\|// .*TODO\|// .*FIXME" internal/ main.go 2>/dev/null
```

Spot-check any comments that describe counts, behavior, or reference other files.

### Step 5 — Fix all stale items

| Issue | Fix |
|-------|-----|
| Line number off by N | Update to actual line number |
| Code example shows old logic | Rewrite to match current source |
| Stale field/function name | Correct to current name, or remove if deleted |
| Wrong count ("~X lines") | Update if significantly off; leave if still approximate |
| Stale behavioral description | Rewrite to match current behavior |

### Step 6 — Run tests

Run the project's test suite to confirm no regressions from comment/doc edits:

```bash
# Go
go test ./...

# Node / Python / other — substitute the project's test command
npm test
pytest
```

Skip if the project has no test suite or if only markdown files were changed.

### Step 7 — Repeat

Go back to Step 2. Keep iterating until a complete pass over all references finds **zero issues**.

> **Termination**: Stop only when a full round produces no changes at all. A round that fixes items must be followed by another full round to confirm the fixes didn't introduce new stale content.

## Common stale patterns

- Line numbers drift when code is inserted above a referenced function
- Code examples go stale when error handling logic is refactored (e.g., condition flipped, new branch added)
- Struct field references in docs survive after fields are removed or renamed
- "~N lines" file size claims drift as files grow
- Inline comments describing behavior that was later refactored (e.g. "skips X" when the code now always processes X)

## Quick check script

Run this to get a starting list of all references to verify:

```bash
grep -rn "\`[a-z_./-]*\.[a-z]*:[0-9]" docs/ README.md QUICKSTART.md 2>/dev/null | \
  grep -v "^Binary" | \
  sed 's/.*`\([a-z_./-]*\.[a-z]*:[0-9-]*\)`.*/\1/' | \
  sort -u
```
