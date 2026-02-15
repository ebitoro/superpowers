---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees with smart directory selection and safety verification
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching.

**Core principle:** Always create worktrees inside the project at `.worktrees/`.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

**Cost optimization:** This skill is procedural (no creative decisions). Other skills should dispatch the `worktree-setup` agent (see `agents/worktree-setup.md`) instead of running this skill inline. The agent runs on Sonnet and keeps worktree setup out of the caller's context window.

## Worktree Location

**Fixed:** All worktrees go in `<project-root>/.worktrees/<branch-name>`.

```bash
PROJECT_ROOT="$(git rev-parse --show-toplevel)"
WORKTREE_DIR="$PROJECT_ROOT/.worktrees"
```

No other locations. No asking the user. No checking CLAUDE.md for preferences. Always `.worktrees/` inside the project.

## The Process

### Step 1: Verify .worktrees/ is gitignored

```bash
git check-ignore -q .worktrees 2>/dev/null
```

If NOT ignored, fix it immediately:
```bash
echo '.worktrees/' >> .gitignore
git add .gitignore && git commit -m "chore: gitignore .worktrees/"
```

### Step 2: Create worktree

```bash
mkdir -p .worktrees
git worktree add ".worktrees/$BRANCH_NAME" -b "$BRANCH_NAME"
cd ".worktrees/$BRANCH_NAME"
```

### Step 3: Run project setup

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

### Step 4: Verify clean baseline

Run tests to ensure worktree starts clean:

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Continue.

### Step 5: Report location

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Red Flags

**Never:**
- Create worktrees outside the project directory
- Create worktrees in `~/.config/` or any global location
- Skip gitignore verification
- Skip baseline test verification
- Proceed with failing tests without asking

**Always:**
- Use `.worktrees/` inside the project root
- Verify `.worktrees/` is gitignored before creating
- Auto-detect and run project setup
- Verify clean test baseline

## Integration

**Called by:**
- **writing-plans** (Step 1) - REQUIRED before drafting a plan
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
- Any skill needing isolated workspace

**Pairs with:**
- **finishing-a-development-branch** - REQUIRED for cleanup after work complete
