---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This should be run in a dedicated worktree (created by brainstorming skill or manually).

**Save plans to:** `docs/plans/YYYY-MM-DD-<feature-name>.md`

## Context Recovery

On startup, recover context from breadcrumb files:

1. Read `CODEX_THREAD_ID` from `/tmp/codex_thread_id`
2. Read the design doc path from `/tmp/current_design_doc`
3. Read the design doc itself to rebuild full context

If `/tmp/codex_thread_id` exists and is valid, reuse the existing Codex thread. If missing or Codex is unavailable, proceed without Codex review and inform the user.

If `/tmp/current_design_doc` exists, read the design doc it points to. If missing, look for the most recent `*-design.md` in `docs/plans/`. If neither exists, ask the user for the design doc path.

The design has already been reviewed and approved by Codex during brainstorming. Do not re-send it. The existing Codex thread retains that context.

## Worktree Verification

After recovering context, verify you are working inside a git worktree:

```bash
git rev-parse --show-toplevel 2>/dev/null
git worktree list
```

**If already inside a worktree:** Proceed. Read the worktree path from `pwd` or `git rev-parse --show-toplevel`.

**If NOT inside a worktree but `/tmp/current_worktree` exists:** Read the path and `cd` into it. Verify it is a valid worktree:

```bash
WORKTREE_PATH=$(cat /tmp/current_worktree)
cd "$WORKTREE_PATH"
git rev-parse --show-toplevel
```

**If NOT inside a worktree and no breadcrumb exists:** Dispatch a subagent (lower model, e.g., Sonnet) using the `git-worktree-agent` prompt to create one. Wait for completion, then read `/tmp/current_worktree` and `cd` into it.

**After entering the worktree, verify Claude Code is operating there:**

```bash
pwd
git status
git branch --show-current
```

If `pwd` does not match the worktree path, something went wrong. Report to user and stop.

## Codex Integration

**Codex availability:**
If Codex is unavailable (MCP not connected, usage limit hit, or any error from `codex-reply`), skip all Codex steps and proceed without Codex review. Inform the user that Codex review was skipped and why.

**Working directory awareness:**
Codex's MCP server uses the CWD from when it was started (the main repo), NOT the worktree. `codex-reply` does not change this. Since Codex is a reviewer and does not touch files, this is fine — but all messages to Codex via `codex-reply` MUST include:

```
NOTE: Implementation is in worktree at <worktree-absolute-path>.
All file paths in this plan are relative to the worktree root.
```

This ensures Codex review comments reference the correct paths.

**Codex is consulted once:**
1. **Before presenting to user** — Send the full plan to Codex via `codex-reply` for review. Codex must pass the plan. See "Codex Review Gate" below.

### Codex Review Gate

Before presenting the plan to the user, send the full plan to Codex for review via `codex-reply`.

- If Codex passes: proceed to present plan to user.
- If Codex flags issues: fix the plan and resubmit for review.
- Maximum 5 review rounds. If the plan still has unresolved issues after 5 rounds, present the plan to the user and clearly state what Codex still flagged as unresolved.

## Checklist

You MUST complete these steps in order:

1. **Recover context** — read `/tmp/codex_thread_id` and `/tmp/current_design_doc`, load the design doc
2. **Verify worktree** — ensure working inside a worktree, create one if needed, confirm `pwd` is correct
3. **Draft the implementation plan** — following the task structure and granularity rules below
4. **Codex review gate** — send plan to Codex via `codex-reply` (include worktree path note), iterate up to 5 times until pass
5. **Present plan to user** — include any unresolved Codex flags if review gate did not fully pass
6. **Save plan** — write to `docs/plans/YYYY-MM-DD-<feature-name>.md` and commit
7. **Offer execution handoff**

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

**Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

**Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## Remember
- Exact file paths always
- Complete code in plan (not "add validation")
- Exact commands with expected output
- Reference relevant skills with @ syntax
- DRY, YAGNI, TDD, frequent commits

## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Stay in this session
- Fresh subagent per task + code review

**If Parallel Session chosen:**
- Guide them to open new session in worktree
- **REQUIRED SUB-SKILL:** New session uses superpowers:executing-plans
