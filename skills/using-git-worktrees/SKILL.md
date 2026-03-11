---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees with smart directory selection and safety verification
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching.

**Core principle:** Detect the project's worktree layout, then create worktrees in the right place.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Layout Detection

Projects use one of two layouts. Detect which one **before** choosing a worktree directory.

### Sibling Layout (preferred)

The git repo lives in a subdirectory (commonly `main/`) and worktrees are siblings:

```
project/
  main/          # git repo (.git directory lives here)
  worktrees/     # worktrees go here, beside main
    feature-a/
    bugfix-b/
```

**How to detect:** Check if the parent of the git toplevel has a `worktrees/` sibling directory:

```bash
git_root=$(git rev-parse --show-toplevel)
parent=$(dirname "$git_root")
ls -d "$parent/worktrees" 2>/dev/null
```

**If found:** Use `$parent/worktrees/$BRANCH_NAME`. No gitignore verification needed — worktrees are outside the repo.

### In-Repo Layout

Worktrees live inside the git repo itself:

```
project/               # git repo root
  .worktrees/          # or worktrees/
    feature-a/
```

**How to detect:** Check for `.worktrees/` or `worktrees/` inside the git toplevel:

```bash
git_root=$(git rev-parse --show-toplevel)
ls -d "$git_root/.worktrees" "$git_root/worktrees" 2>/dev/null
```

**If found:** Use that directory. If both exist, `.worktrees` wins.

## Directory Selection Process

Follow this priority order:

### 1. Detect Sibling Layout

```bash
git_root=$(git rev-parse --show-toplevel)
parent=$(dirname "$git_root")
if [ -d "$parent/worktrees" ]; then
  WORKTREE_DIR="$parent/worktrees"
  LAYOUT="sibling"
fi
```

### 2. Check In-Repo Directories

```bash
if [ -d "$git_root/.worktrees" ]; then
  WORKTREE_DIR="$git_root/.worktrees"
  LAYOUT="in-repo"
elif [ -d "$git_root/worktrees" ]; then
  WORKTREE_DIR="$git_root/worktrees"
  LAYOUT="in-repo"
fi
```

### 3. Check CLAUDE.md

```bash
grep -i "worktree.*director" "$git_root/CLAUDE.md" 2>/dev/null
```

**If preference specified:** Use it without asking.

### 4. Ask User

If no directory exists and no CLAUDE.md preference:

```
No worktree directory found. Where should I create worktrees?

1. worktrees/ (sibling layout — beside repo checkout)
2. .worktrees/ (in-repo, hidden)
3. Other location

Which would you prefer?
```

## Safety Verification

### Sibling Layout

No gitignore verification needed — worktrees directory is outside the git repo.

### In-Repo Layout

**MUST verify directory is ignored before creating worktree:**

```bash
git check-ignore -q "$WORKTREE_DIR" 2>/dev/null
```

**If NOT ignored:**
1. Add appropriate line to .gitignore
2. Commit the change
3. Proceed with worktree creation

## Creation Steps

### 1. Detect Project Name

```bash
# For sibling layout, use the parent directory name
# For in-repo layout, use the git toplevel name
if [ "$LAYOUT" = "sibling" ]; then
  project=$(basename "$parent")
else
  project=$(basename "$git_root")
fi
```

### 2. Create Worktree

```bash
path="$WORKTREE_DIR/$BRANCH_NAME"

# Create worktree with new branch
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

### 3. Run Project Setup

Auto-detect and run appropriate setup:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

### 4. Verify Clean Baseline

Run tests to ensure worktree starts clean:

```bash
# Examples - use project-appropriate command
npm test
cargo test
pytest
go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### 5. Report Location

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Sibling `worktrees/` exists beside repo | Use it (no gitignore needed) |
| In-repo `.worktrees/` exists | Use it (verify ignored) |
| In-repo `worktrees/` exists | Use it (verify ignored) |
| Neither layout found | Check CLAUDE.md → Ask user |
| In-repo directory not ignored | Add to .gitignore + commit |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## Common Mistakes

### Confusing sibling and in-repo layouts

- **Problem:** Running gitignore checks on a sibling layout (unnecessary) or skipping them on an in-repo layout (dangerous)
- **Fix:** Always detect the layout first, then apply the right safety checks

### Assuming git toplevel is the project root

- **Problem:** In sibling layout, `git rev-parse --show-toplevel` returns `project/main`, not `project/`
- **Fix:** Check the parent directory for sibling worktrees before assuming in-repo layout

### Skipping ignore verification (in-repo layout)

- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always use `git check-ignore` before creating in-repo worktrees

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

### Hardcoding setup commands

- **Problem:** Breaks on projects using different tools
- **Fix:** Auto-detect from project files (package.json, etc.)

## Example Workflows

### Sibling Layout

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Detect git root: /Users/chris/dev/github/genomenon/dex-sdk/main]
[Check parent for sibling worktrees: ../worktrees/ exists — sibling layout]
[Create worktree: git worktree add ../worktrees/PLAT-1234 -b PLAT-1234]
[Run project setup]
[Run tests - 120 passing]

Worktree ready at /Users/chris/dev/github/genomenon/dex-sdk/worktrees/PLAT-1234
Tests passing (120 tests, 0 failures)
Ready to implement PLAT-1234
```

### In-Repo Layout

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Detect git root: /Users/chris/projects/myapp]
[No sibling worktrees/ found]
[Check in-repo: .worktrees/ exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[Create worktree: git worktree add .worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/chris/projects/myapp/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

## Red Flags

**Never:**
- Create in-repo worktree without verifying it's ignored
- Skip baseline test verification
- Proceed with failing tests without asking
- Assume layout without checking
- Skip CLAUDE.md check

**Always:**
- Detect layout first: sibling > in-repo > CLAUDE.md > ask
- Verify ignored for in-repo layout only
- Auto-detect and run project setup
- Verify clean test baseline

## Integration

**Called by:**
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
- Any skill needing isolated workspace

**Pairs with:**
- **finishing-a-development-branch** - REQUIRED for cleanup after work complete
