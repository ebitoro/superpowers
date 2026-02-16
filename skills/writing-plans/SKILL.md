---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This MUST be run in a dedicated worktree. If not already in one, invoke `superpowers:using-git-worktrees` first.

**Save plans to:** `docs/plans/YYYY-MM-DD-<feature-name>.md`

## Codex Integration

> See `lib/codex-integration.md` for Codex patterns. All interactions go through the codex-agent (`agents/codex-agent.md`).

**Context recovery** — read the design doc from the state directory at the **main repo root**:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
cat "$MAIN_REPO/.codex-state/current_design_doc"
```

Read the design doc path and load it. If missing, look for the most recent `*-design.md` in `docs/plans/`. If neither exists, ask the user.

**Codex is consulted once:**
- **Before presenting to user** — Dispatch codex-agent with `mode: review-gate` containing the full plan and worktree path. If verdict is `fail`, fix verified issues and redispatch (max 5 rounds).

## Checklist

You MUST complete these steps in order:

1. **Verify worktree** — check if inside a git worktree (`git worktree list`). If NOT in a worktree, dispatch the `worktree-setup` agent (see `agents/worktree-setup.md`) with the branch name. The agent runs on Sonnet and handles the full setup.
2. **Recover context** — run `MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"` to find the main repo root, then read `$MAIN_REPO/.codex-state/current_design_doc`, load the design doc
3. **Draft the implementation plan** — following the task structure and granularity rules below
4. **Codex review gate** — dispatch codex-agent with `mode: review-gate` (include worktree path), iterate up to 5 rounds (see `lib/codex-integration.md`)
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

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (same session) or superpowers:executing-plans (separate session) to implement this plan task-by-task.

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
