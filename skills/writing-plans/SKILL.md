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

> **Reference:** See `lib/codex-integration.md` for shared patterns (state directory, availability, review gate logic, working directory awareness, cleanup).

**Context recovery** — on startup:
1. Read `CODEX_THREAD_ID` from `.codex-state/codex_thread_id`
2. Read the design doc path from `.codex-state/current_design_doc`
3. Read the design doc itself to rebuild full context

Validate the thread ID by testing a `codex-reply` call. Follow the validation and recovery steps in `lib/codex-integration.md`:
- If the thread works: reuse it. The design context from brainstorming is retained.
- If "Session not found": create a new thread via `codex`, save the new ID, and send the design doc as context so Codex is caught up.
- If Codex is unavailable entirely: proceed without Codex and inform the user.

If `.codex-state/current_design_doc` exists, read the design doc it points to. If missing, look for the most recent `*-design.md` in `docs/plans/`. If neither exists, ask the user for the design doc path.

**Codex is consulted once:**
- **Before presenting to user** — Send the full plan to Codex via `codex-reply` for review. Follow the review gate pattern from `lib/codex-integration.md` (max 5 rounds). Include the worktree path note.

## Checklist

You MUST complete these steps in order:

1. **Verify worktree** — check if inside a git worktree (`git worktree list`). If NOT in a worktree, **REQUIRED SUB-SKILL:** invoke `superpowers:using-git-worktrees` to create one before proceeding.
2. **Recover context** — read `.codex-state/codex_thread_id` and `.codex-state/current_design_doc`, load the design doc
3. **Draft the implementation plan** — following the task structure and granularity rules below
4. **Codex review gate** — send plan to Codex via `codex-reply` (include worktree path note), iterate up to 5 rounds (see `lib/codex-integration.md`)
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
