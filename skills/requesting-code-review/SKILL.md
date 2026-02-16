---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# Requesting Code Review

Two-stage review: dispatch superpowers:code-reviewer subagent, then Codex review gate. Both must pass before proceeding.

**Core principle:** Review early, review often. Two reviewers catch what one misses.

## When to Request Review

**Mandatory:**
- After each task in subagent-driven development
- After each batch in executing-plans
- After completing major feature
- Before merge to main

**Optional but valuable:**
- When stuck (fresh perspective)
- Before refactoring (baseline check)
- After fixing complex bug

## How to Request

### Step 1: Commit and get git SHAs

All changes MUST be committed before code review. Codex reviews by commit SHA, not raw diff text. If there are uncommitted changes, commit them first.

```bash
# Commit any uncommitted work
git add -A && git commit -m "feat: <description>"

BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

### Step 2: Dispatch code-reviewer subagent

Use Task tool with superpowers:code-reviewer type, fill template at `code-reviewer.md`

**Placeholders:**
- `{WHAT_WAS_IMPLEMENTED}` - What you just built
- `{PLAN_OR_REQUIREMENTS}` - What it should do
- `{BASE_SHA}` - Starting commit
- `{HEAD_SHA}` - Ending commit
- `{DESCRIPTION}` - Brief summary

### Step 3: Act on subagent feedback

- Fix Critical issues immediately
- Fix Important issues before proceeding
- Note Minor issues for later
- Push back if reviewer is wrong (with reasoning)
- Re-dispatch subagent if fixes were needed, until it passes

### Step 4: Codex review gate

> **Reference:** See `lib/codex-integration.md` for shared patterns (state directory, availability, review gate logic).
>
> **All Codex interactions go through the `codex-agent` subagent** (`agents/codex-agent.md`). See `lib/codex-integration.md` "Codex Agent (Preferred Pattern)" for dispatch format.

After the code-reviewer subagent passes, dispatch codex-agent with `mode: review-gate` for a second opinion. The agent handles thread recovery, Codex communication, and response verification internally.

**What to include in the review-gate message** (see `lib/codex-integration.md` "Efficient Codex Communication"):
- The commit SHAs (`{BASE_SHA}..{HEAD_SHA}`) — NOT the raw diff text
- A short summary of what was implemented and why
- Test results summary (pass/fail counts)
- Any context the caller provides (design doc, plan reference, etc.)

Also provide `worktree_path` if working in a worktree.

The codex-agent verifies every Codex finding against the actual code, filters out false positives and out-of-scope items, and returns only verified issues. This saves the main session from having to do verification work.

**Review gate loop:** If the codex-agent returns `fail` with verified issues, fix them and redispatch (max 5 rounds). False positives are already filtered by the agent.

**If codex-agent reports `status: unavailable`:** Skip this step and proceed with the subagent's result only. Inform the caller that Codex review was skipped.

### Step 5: Track unresolved flags

If the verdict is **pass with flags**, append the unresolved items to `docs/unresolved-flags.md` following the format in `lib/codex-integration.md`, then commit the change. This file is version-controlled so flags survive across sessions and merges.

### Step 6: Report result

Report the combined result to the caller:
- Code-reviewer subagent assessment
- Codex agent report (verdict, verified issues count, dismissed count, or skipped)
- Overall verdict: **pass** (both passed), **pass with flags** (subagent passed, codex-agent had unresolved minor items), or **needs fixes**
- If flags were tracked, mention: "Unresolved flags committed to `docs/unresolved-flags.md`"

## Example

```
[Just completed Task 2: Add verification function]

You: Let me request code review before proceeding.

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[Dispatch superpowers:code-reviewer subagent]
  WHAT_WAS_IMPLEMENTED: Verification and repair functions for conversation index
  PLAN_OR_REQUIREMENTS: Task 2 from docs/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types

[Subagent returns]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
  Assessment: With fixes

[Fix progress indicators, re-dispatch subagent]
[Subagent returns]: All clear. Ready to proceed.

[Dispatch codex-agent with mode: review-gate]
  message: "Review a7981ec..3df7661. Implemented verifyIndex() and repairIndex().
   Tests: 4 passing."
  worktree_path: /path/.worktrees/deploy
Codex Agent Report: verdict=pass, 0 verified issues, 0 dismissed,
  codex_notes: "consider adding JSDoc to public API"

Result: Pass (subagent approved, Codex agent approved with minor note)

[Continue to Task 3]
```

## Integration with Workflows

**Subagent-Driven Development:**
- Review after EACH task (per-task diff)
- Final review after all tasks (full branch diff)
- Catch issues before they compound

**Executing Plans:**
- Review after each batch (batch diff)
- Get feedback, apply, continue

**Ad-Hoc Development:**
- Review before merge
- Review when stuck

## Red Flags

**Never:**
- Skip review because "it's simple"
- Ignore Critical issues
- Proceed with unfixed Important issues
- Argue with valid technical feedback
- Skip Codex review to save time (it catches cross-cutting issues the subagent misses)

**If reviewer wrong:**
- Push back with technical reasoning
- Show code/tests that prove it works
- Request clarification

See template at: requesting-code-review/code-reviewer.md
