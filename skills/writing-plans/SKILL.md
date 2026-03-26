---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This should be run in a dedicated worktree (created by brainstorming skill).

**Save plans to:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- (User preferences for plan location override this default)

## Input Discovery

Find the design doc to plan from. The user may have cleared the session, so conversation context may be absent.

1. **Check breadcrumb first:**
   ```bash
   MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
   DESIGN_DOC="$MAIN_REPO/$(cat "$MAIN_REPO/.dev-state/current_design_doc" 2>/dev/null)"
   ```
2. **If breadcrumb missing or file not found:** scan `docs/superpowers/specs/` for the most recent spec file (by filename date prefix or modification time)
3. **If multiple candidates:** ask the user which one
4. **Read the design doc** before proceeding — this is the spec you are planning from

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

**Dispatch an Explore subagent** to scan the codebase and return a structured summary. This keeps raw file contents out of the main session's context.

```
Agent tool:
  subagent_type: "Explore"
  description: "Explore codebase for plan"
  prompt: |
    Explore this project to inform an implementation plan. Return a concise summary covering:
    - **Project structure**: directory layout, key directories, file organization patterns
    - **Existing files relevant to <feature>**: paths, responsibilities, approximate sizes
    - **Patterns and conventions**: naming, module structure, import style, test organization
    - **Test setup**: framework, test locations, how tests are run, example test structure
    - **Dependencies**: key libraries, build tools, configuration files
    - **Files that would need modification for <feature>**: paths with line counts and current responsibilities

    Keep the summary under 500 words. Focus on what someone planning file structure changes would need to know.
    Do NOT include raw file contents — summarize.
```

Use the returned summary to inform file structure decisions:

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.
- If you need specific file details during planning, read individual files directly — but avoid bulk exploration in the main session.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

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

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

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

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## No Placeholders

Every step must contain the actual content an engineer needs. These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code — the engineer may be reading tasks out of order)
- Steps that describe what to do without showing how (code blocks required for code steps)
- References to types, functions, or methods not defined in any task

## Remember
- Exact file paths always
- Complete code in every step — if a step changes code, show the code
- Exact commands with expected output
- DRY, YAGNI, TDD, frequent commits

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch.

**1. Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

**2. Placeholder scan:** Search your plan for red flags — any of the patterns from the "No Placeholders" section above. Fix them.

**3. Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

If you find issues, fix them inline. No need to re-review — just fix and move on. If you find a spec requirement with no task, add the task.

## Commit the Plan

After the plan review loop passes, commit the plan document:

```bash
git add <plan-file-path>
git commit -m "docs(plan): add implementation plan for <feature-name>"
```

## Codex Plan Review Gate

After committing, run a Codex review gate on the complete plan. This uses the Tiered Review Gate (3+3 pattern) from `lib/codex-integration.md` to preserve context.

**Save breadcrumbs before review:**
```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.dev-state"
mkdir -p "$STATE_DIR"
echo "<relative-path-to-plan>" > "$STATE_DIR/current_plan"
echo "$(pwd)" > "$STATE_DIR/current_worktree"
```

<HARD-GATE>
**You MUST attempt the Codex init step below.** The init step CREATES a new thread — do NOT check for an existing thread or skip because "no thread is available." The only valid reason to skip the Codex review gate is if the init step itself returns `status: unavailable`.
</HARD-GATE>

**Step 1: Create Codex thread (foreground).** This is mandatory — dispatch the codex-agent to create a thread:
```
Agent tool:
  subagent_type: "superpowers:codex-agent"
  description: "Init Codex thread for plan review"
  prompt: |
    mode: init
    profile: xhigheffort
```
**Wait for the result.** Save the returned `thread_id`. If the agent returns `status: unavailable`, skip to CC Fallback Plan Review below.

**Step 2: Dispatch plan-review-gate subagent (foreground, 3 rounds max):**

**IMPORTANT: Dispatch in foreground and wait for the result.** Do NOT background this agent. The review gate must complete before proceeding to execution handoff. You MUST pass the `thread_id` returned from Step 1.

```
Agent tool:
  subagent_type: "superpowers:plan-review-gate"
  description: "Codex plan review gate"
  run_in_background: false
  prompt: |
    plan_path: <absolute-path-to-plan>
    design_doc_path: <absolute-path-to-design-doc>
    worktree_path: <absolute-path-to-worktree>
    thread_id: <thread_id from Step 1>
```

**Handle Tier 1 result:**
- `pass` or `pass-with-flags` (can_proceed): proceed to execution handoff
- `fail` + `can_proceed`: append flags to `docs/unresolved-flags.md`, proceed
- `fail` + `must_fix`: escalate to Tier 2

**Tier 2 — Main session escalation (3 rounds max):**
Only if Tier 1 returned `fail` + `must_fix`:
1. Dispatch codex-agent directly with the `thread_id` returned from Tier 1
2. **Independently verify each finding** — read the actual plan section and design doc, confirm the issue exists. Dismiss false positives. Fix confirmed issues.
3. Redispatch until pass or 3 rounds exhausted
4. If still unresolved → escalate to user

**If codex-agent reports `status: unavailable`:** Dispatch CC fallback plan review instead.

### CC Fallback Plan Review (when Codex unavailable)

When Codex is unavailable, dispatch a CC fallback reviewer to independently review the plan. The plan-document-reviewer (earlier step) already checked document structure and consistency — this reviewer focuses on what Codex would have caught: completeness, TDD adherence, and implementation risks.

```
Agent tool:
  subagent_type: "superpowers:code-reviewer"
  description: "CC fallback plan review (round {R})"
  run_in_background: false
  prompt: |
    You are an INDEPENDENT PLAN REVIEWER providing a second opinion on an implementation plan.
    A separate plan-document-reviewer has already checked structure and consistency.
    Your focus is different — completeness and implementation risk.

    ## Plan

    Read the plan at: {PLAN_PATH}
    Design doc: {DESIGN_DOC_PATH}

    ## Review Focus

    - **Completeness:** Does the plan cover ALL requirements from the design doc?
    - **Task ordering:** Are dependencies between tasks correctly sequenced?
    - **TDD adherence:** Does each task follow write-test → verify-fail → implement → verify-pass?
    - **Missing steps:** Are there implicit steps that should be explicit?
    - **Risk areas:** Which tasks are most likely to cause problems? Are they adequately planned?
    - **Testability:** Are the test cases meaningful, not just happy-path?

    Do NOT re-check document formatting or structure — that's already done.

    ## If Critical or Important Issues Found

    Fix them directly in the plan file. Commit with:
    `fix(plan): address CC plan review findings`

    ## Verdict

    Return exactly one of:
    - verdict: pass — No Critical or Important issues found. Plan is ready for execution.
    - verdict: fixed — Issues found and fixed. List what was found and what was changed.
```

**Handle CC fallback verdict:**
- `pass`: proceed to execution handoff
- `fixed`: dispatch a fresh CC fallback reviewer to verify fixes. Max 3 rounds — escalate to human if still finding issues. Then proceed to execution handoff.

## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review
