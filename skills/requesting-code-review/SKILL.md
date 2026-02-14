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

### Step 1: Get git SHAs

```bash
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

After the code-reviewer subagent passes, send the diff to Codex via `codex-reply` for a second opinion. If you don't have `CODEX_THREAD_ID` in working memory, read it from the state directory:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
cat "$MAIN_REPO/.codex-state/codex_thread_id"
```

**What to send to Codex:**
- The diff (`git diff {BASE_SHA}..{HEAD_SHA}`)
- What was implemented and why
- Test results summary
- Any context the caller provides (design doc, plan reference, etc.)
- The worktree path note (see `lib/codex-integration.md`)

**Review gate:** Follow the standard review gate pattern from `lib/codex-integration.md` (max 5 rounds, fix and resubmit until pass).

**If Codex is unavailable:** Skip this step and proceed with the subagent's result only. Inform the caller that Codex review was skipped.

### Step 5: Report result

Report the combined result to the caller:
- Code-reviewer subagent assessment
- Codex review result (pass, or unresolved flags, or skipped)
- Overall verdict: **pass** (both passed), **pass with flags** (subagent passed, Codex had unresolved minor items), or **needs fixes**

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

[Send diff to Codex via codex-reply]
Codex: Pass. Minor suggestion: consider adding JSDoc to public API.

Result: Pass (subagent approved, Codex approved with minor note)

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
