---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Reasoning Effort

Use the highest reasoning effort (ultrathink) for ALL steps throughout this entire planning process. Task decomposition, dependency ordering, test design, and code scaffolding all benefit from deep analytical thinking. A poorly decomposed plan creates cascading problems during implementation — the cost of thinking deeply here is repaid many times over.

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This MUST be run in a dedicated worktree. If not already in one, invoke `superpowers:using-git-worktrees` first.

**Save plans to:** `docs/plans/YYYY-MM-DD-<feature-name>.md`

## Codex Integration

> See `lib/codex-integration.md` for Codex patterns. All interactions go through the codex-agent (`skills/codex-agent/SKILL.md`).

**Context recovery** — read the design doc from the state directory at the **main repo root**:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
cat "$MAIN_REPO/.codex-state/current_design_doc"
```

Read the design doc path and load it. If missing, look for the most recent `*-design.md` in `docs/plans/`. If neither exists, ask the user.

<HARD-GATE>
**Codex review gate is MANDATORY.** You MUST run the tiered review gate before presenting the plan to the user. Do NOT skip this step. Do NOT present the plan without running the review gate first (unless Codex is confirmed unavailable).
</HARD-GATE>

**Tiered review gate (3+3 pattern):**

The review gate is split across a subagent (first 3 rounds) and the main session (3 more if needed). This preserves main session context — see `lib/codex-integration.md` "Tiered Review Gate" for the full pattern.

- **Tier 1 (subagent):** Dispatch `plan-review-gate` agent (see `agents/plan-review-gate.md`). It handles codex-agent dispatch, independent verification of every finding, plan fixes, and up to 3 rounds autonomously.
- **Tier 2 (main session escalation):** Only if the subagent returns `verdict: fail` + `severity: must_fix`. The main session takes over with the returned `thread_id`, dispatches codex-agent directly, verifies + fixes for up to 3 more rounds.
- **Escalation to user:** If Tier 2 still results in `verdict: fail` + `severity: must_fix` after 3 rounds, present the plan with unresolved issues and let the user decide.

**Profile inheritance:** The Codex thread was created during brainstorming with `profile: "xhigheffort"`. All dispatches here reuse that thread and inherit the `xhigheffort` profile automatically.

## Checklist

You MUST complete these steps in order:

1. **Verify worktree** — check if inside a git worktree (`git worktree list`). If NOT in a worktree, dispatch the `worktree-setup` agent (see `agents/worktree-setup.md`) with the branch name. The agent runs on Sonnet and handles the full setup.
2. **Recover context** — run `MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"` to find the main repo root, then read `$MAIN_REPO/.codex-state/current_design_doc`, load the design doc
3. **Draft the implementation plan** — following the task structure and granularity rules below
4. **Save draft plan** — write to `docs/plans/YYYY-MM-DD-<feature-name>.md` (needed before subagent can read it)
<HARD-GATE>
**Dispatch the plan-review-gate in FOREGROUND.** Do NOT set `run_in_background: true`. The main session has nothing to do while waiting — it needs the result to proceed. Running it in background wastes time and confuses the user.
</HARD-GATE>

5. **Tier 1: Subagent review gate** — dispatch `plan-review-gate` agent in **foreground**:
   ```
   Agent tool:
     name: "plan-review-gate"
     description: "Codex plan review gate"
     run_in_background: false
     prompt: |
       plan_path: <absolute path to saved plan>
       design_doc_path: <absolute path to design doc>
       worktree_path: <absolute worktree path>
       thread_id: <from .codex-state/codex_thread_id, or omit>
   ```
   Read the agent template at `agents/plan-review-gate.md` and include its full content in the prompt.
6. **Handle subagent result:**
   - `pass` → go to step 8
   - `pass-with-flags` or (`fail` + `can_proceed`) → flags already written to `docs/unresolved-flags.md` by subagent, go to step 8
   - `fail` + `must_fix` → go to step 7
7. **Tier 2: Main session escalation** — take over with the returned `thread_id`. Dispatch codex-agent directly with `mode: review-gate`, verify each finding independently against the plan and codebase, fix verified issues. Up to 3 rounds. If still `fail` + `must_fix`, present to user with unresolved issues and let them decide. If `can_proceed`, append flags to `docs/unresolved-flags.md` and commit.
8. **Present plan to user** — only after review gate passes (or escalation resolves, or Codex unavailable). Include any unresolved Codex flags if review gate did not fully pass
9. **Commit plan** — the plan file was saved in step 4 and may have been edited by the review gate. Stage and commit the final version.
10. **Write session breadcrumbs** — persist plan path and worktree path so the next session can recover context after `/clear`:
    ```bash
    MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
    echo "docs/plans/YYYY-MM-DD-<feature-name>.md" > "$MAIN_REPO/.codex-state/current_plan"
    pwd > "$MAIN_REPO/.codex-state/current_worktree"
    ```
11. **Offer execution handoff**

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

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (same session), superpowers:team-driven-development (5+ tasks), or superpowers:executing-plans (separate session) to implement this plan task-by-task.

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

After saving the plan and writing breadcrumbs, offer execution choice:

**"Plan complete and saved to `docs/plans/<filename>.md`. Four execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Subagent-Driven (fresh context)** - `/clear` then call `superpowers:subagent-driven-development`. Breadcrumbs are saved so I'll recover the plan path and worktree automatically.

**3. Team-Driven (recommended for 5+ tasks)** - Leader agent orchestrates via AgentTeam, fresh implementer per task, minimal context usage

**4. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?"**

**If option 1 (same session):**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Stay in this session
- Fresh subagent per task + code review

**If option 2 (clear + continue):**
- User runs `/clear`
- User invokes `superpowers:subagent-driven-development`
- Skill reads breadcrumbs from `.codex-state/current_plan` and `.codex-state/current_worktree`, recovers context, and cleans up

**If option 3 (team-driven):**
- **REQUIRED SUB-SKILL:** Use superpowers:team-driven-development
- Stay in this session or `/clear` first (breadcrumbs supported)
- Recommended when plan has 5+ tasks to minimize context usage

**If option 4 (parallel session):**
- Guide them to open new session in worktree
- **REQUIRED SUB-SKILL:** New session uses superpowers:executing-plans
