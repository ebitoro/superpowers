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
   DESIGN_DOC="$MAIN_REPO/$(cat "$MAIN_REPO/.codex-state/current_design_doc" 2>/dev/null)"
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

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

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

## Remember
- Exact file paths always
- Complete code in plan (not "add validation")
- Exact commands with expected output
- Reference relevant skills with @ syntax
- DRY, YAGNI, TDD, frequent commits

## Plan Review Loop

After completing each chunk of the plan:

1. Dispatch plan-document-reviewer subagent (see plan-document-reviewer-prompt.md) for the current chunk
   - Provide: chunk content, path to spec document
2. If ❌ Issues Found:
   - Fix the issues in the chunk
   - Re-dispatch reviewer for that chunk
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to next chunk (or execution handoff if last chunk)

**Chunk boundaries:** Use `## Chunk N: <name>` headings to delimit chunks. Each chunk should be ≤1000 lines and logically self-contained.

**Review loop guidance:**
- Same agent that wrote the plan fixes it (preserves context)
- If loop exceeds 5 iterations, surface to human for guidance
- Reviewers are advisory - explain disagreements if you believe feedback is incorrect

## Codex Plan Review Gate

After the plan review loop passes (all chunks approved by the plan-document-reviewer), run a Codex review gate on the complete plan. This uses the Tiered Review Gate (3+3 pattern) from `lib/codex-integration.md` to preserve context.

**Save breadcrumbs before review:**
```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.codex-state"
mkdir -p "$STATE_DIR"
echo "<relative-path-to-plan>" > "$STATE_DIR/current_plan"
echo "$(pwd)" > "$STATE_DIR/current_worktree"
```

**Create Codex thread (foreground)** before dispatching the subagent:
```
Agent tool:
  subagent_type: "superpowers:codex-agent"
  description: "Init Codex thread for plan review"
  prompt: |
    mode: init
    profile: xhigheffort
```
Save the returned `thread_id`. If `status: unavailable`, skip Codex plan review and proceed to execution handoff (inform user).

**Tier 1 — Dispatch plan-review-gate subagent (foreground, 3 rounds max):**

**IMPORTANT: Dispatch in foreground and wait for the result.** Do NOT background this agent. The review gate must complete before proceeding to execution handoff.

```
Agent tool:
  subagent_type: "superpowers:plan-review-gate"
  description: "Codex plan review gate"
  run_in_background: false
  prompt: |
    plan_path: <absolute-path-to-plan>
    design_doc_path: <absolute-path-to-design-doc>
    worktree_path: <absolute-path-to-worktree>
    thread_id: <thread_id from init step>
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

**If codex-agent reports `status: unavailable`:** Skip Codex plan review and proceed to execution handoff. Inform user.

## Execution Handoff

After saving the plan:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Ready to execute?"**

**Execution path depends on harness capabilities:**

**If harness has subagents (Claude Code, etc.):**
- **REQUIRED:** Use superpowers:subagent-driven-development
- Do NOT offer a choice - subagent-driven is the standard approach
- Fresh subagent per task + two-stage review

**If harness does NOT have subagents:**
- Execute plan in current session using superpowers:executing-plans
- Batch execution with checkpoints for review
