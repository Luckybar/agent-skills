---
name: using-git-worktrees
description: Creates isolated git worktrees for safe development. Use when starting implementation work, when the autonomous pipeline begins its build phase, or when you need to isolate changes on a new branch without affecting the main working directory.
---

# Using Git Worktrees

## Overview

Git worktrees let you work on a separate branch in a separate directory without touching the main working tree. This skill creates an isolated workspace for implementation: new branch, fresh setup, verified test baseline. If something goes wrong, the main branch is untouched. When the work is done, changes merge back cleanly.

## When to Use

- Before starting `subagent-driven-development` or `autonomous-pipeline` build phase
- When implementing a feature that should be isolated from in-progress work
- When you want a clean rollback point — discard the worktree and everything is undone

**NOT for:**
- Single-file hotfixes (overkill)
- Repos without a test suite (no baseline to verify)
- When the user explicitly wants to work in the current branch

## Process

### Step 1: Choose Worktree Location

Check in this order:
1. Does a `.worktrees/` or `worktrees/` directory already exist? → Use it
2. Does CLAUDE.md specify a worktree preference? → Follow it
3. Default: create `.worktrees/` in the project root

### Step 2: Verify Git Ignore

The worktree directory MUST be git-ignored. Check:

```bash
git check-ignore .worktrees/
```

If NOT ignored:
1. Add `.worktrees/` to `.gitignore`
2. Commit the `.gitignore` change before proceeding

This prevents worktree contents from being accidentally committed to the main branch.

### Step 3: Create Worktree

```bash
# Branch name: descriptive, based on the feature/task
git worktree add .worktrees/<feature-name> -b <feature-branch-name>
```

Naming conventions:
- Branch: `feat/<feature-name>`, `fix/<bug-name>`, or `task/<task-name>`
- Worktree directory: matches the branch purpose

### Step 4: Project Setup

In the new worktree directory, detect and run the appropriate setup:

| Detection | Setup Command |
|---|---|
| `package.json` + `package-lock.json` | `npm ci` |
| `package.json` + `yarn.lock` | `yarn install --frozen-lockfile` |
| `package.json` + `pnpm-lock.yaml` | `pnpm install --frozen-lockfile` |
| `Cargo.toml` | `cargo build` |
| `requirements.txt` | `pip install -r requirements.txt` |
| `go.mod` | `go mod download` |
| `Gemfile` | `bundle install` |
| `pyproject.toml` | `pip install -e .` or `poetry install` |

If the project has a documented setup process (in README or CLAUDE.md), follow that instead.

### Step 5: Verify Test Baseline

Run the full test suite in the worktree:

```bash
cd .worktrees/<feature-name>
# Run the project's test command
```

**All tests must pass.** If tests fail on a fresh worktree from the base branch, that's a pre-existing issue — report it to the human before proceeding.

### Step 6: Report Ready

```
WORKTREE READY

Location: .worktrees/<feature-name>
Branch: <feature-branch-name>
Base: <base-branch> @ <commit-hash>
Setup: <setup command ran>
Tests: <N> passing, 0 failing
Status: Ready for implementation
```

## After Implementation: Finishing Up

When all work in the worktree is complete:

### Option A: Merge to Base Branch
```bash
# From main working directory
git merge <feature-branch-name>
# Or create a PR
```

### Option B: Create Pull Request
```bash
git push -u origin <feature-branch-name>
gh pr create --title "<title>" --body "<body>"
```

### Option C: Discard
```bash
# Remove the worktree
git worktree remove .worktrees/<feature-name>
# Delete the branch
git branch -D <feature-branch-name>
```

**Always ask the human** which option to use. Never auto-merge or auto-discard.

### Cleanup

After merge or discard:
```bash
git worktree remove .worktrees/<feature-name>
```

If all worktrees are removed, the `.worktrees/` directory can stay (it's git-ignored).

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll just work on the current branch, worktrees are overhead" | Worktree setup takes 30 seconds. Untangling a broken main branch takes 30 minutes. The overhead pays for itself on the first mistake. |
| "The test suite is slow, I'll skip the baseline check" | If you skip baseline verification, you won't know whether a test failure is yours or pre-existing. Run the tests. |
| "I'll create the worktree but skip the setup step" | Missing dependencies cause mysterious failures later. Run the setup. |
| "I'll merge directly without asking" | Merging is a destructive action on the main branch. Always confirm with the human. |

## Red Flags

- Working directly on main/master when a worktree was intended
- Worktree directory not in .gitignore
- Skipping test baseline verification
- Auto-merging without human confirmation
- Leaving orphaned worktrees after work is complete

## Verification

After worktree setup, confirm:
- [ ] Worktree directory exists and is git-ignored
- [ ] Branch was created from the correct base
- [ ] Project setup completed without errors
- [ ] Full test suite passes (baseline verified)
- [ ] Report delivered with location, branch, and test count
