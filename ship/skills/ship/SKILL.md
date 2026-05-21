---
name: ship
description: Commit any pending changes, push the current branch, and open a GitHub PR whose body summarizes all commits since main diverged. Use when the user says "ship it", "open a PR", "push and PR", or "/ship".
argument-hint: [optional PR title override]
allowed-tools: Bash(git:*), Bash(gh:*)
---

# Ship: Commit → Push → PR

Open a pull request for the current branch. Summarize all work since `main` diverged.

## Step 1 — Understand the current state

Run these in parallel:

```bash
git status
git diff
git log --oneline main...HEAD
```

## Step 2 — Commit any pending changes

If `git status` shows staged or unstaged modifications:

1. Stage all modified tracked files (do NOT use `git add -A`; add specific files by name from `git status` output to avoid accidentally including secrets or binaries).
2. Draft a concise commit message (1–2 sentences) summarizing only the pending diff — not the whole branch.
3. Commit using a HEREDOC:

```bash
git commit -m "$(cat <<'EOF'
<message here>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

If there are no pending changes, skip this step.

## Step 3 — Build the PR summary

```bash
git log --oneline main...HEAD
git diff main...HEAD --stat
```

From the commit list, synthesize:

- **PR title**: ≤70 characters; active voice; describes *what* changed (e.g. "add cache-backup and cache-restore commands")
- **PR body bullets**: one bullet per logical chunk of work (group related commits). Focus on *why* each change matters, not just what files changed.
- **Test plan**: a short checklist of things a reviewer should manually verify.

If the user passed a title argument, use it verbatim as the PR title.

## Step 4 — Push

```bash
git push -u origin HEAD
```

If push is rejected (non-fast-forward), stop and report — do NOT force push.

## Step 5 — Create the PR

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<bullets>

## Test plan
<checklist>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

After the PR is created, output the PR URL so the user can open it.

## Rules

- Never force-push (`--force`) or skip hooks (`--no-verify`).
- Never commit files that look like secrets: `.env`, `*.key`, `*credentials*`, `config.yaml` containing real tokens.
- If `gh` is not authenticated, stop and tell the user to run `gh auth login`.
- If the branch is already `main`, stop and ask the user to create a feature branch first.
