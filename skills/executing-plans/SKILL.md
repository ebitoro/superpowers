---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

## Overview

Load plan, review critically, execute tasks in batches, report for review between batches.

**Core principle:** Batch execution with checkpoints for architect review.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

## Codex Integration

> **Reference:** See `lib/codex-integration.md` for shared patterns (state directory, availability, review gate logic, working directory awareness, cleanup).

**Context recovery** — on startup, read `CODEX_THREAD_ID` from `.codex-state/codex_thread_id` and validate it. Follow the validation and recovery steps in `lib/codex-integration.md` (reuse if valid, create new thread if expired, skip if Codex unavailable).

## The Process

### Step 1: Load and Review Plan
1. Read plan file
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed

### Step 2: Execute Batch
**Default: First 3 tasks**

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

### Step 3: Code Review

When batch complete, request code review before reporting to user.

**REQUIRED SUB-SKILL:** Use superpowers:requesting-code-review with the batch diff. The code review skill handles both the subagent review and the Codex review gate.

### Step 4: Report
When batch complete and code review passed:
- Show what was implemented
- Show verification output
- Show code review result (subagent + Codex verdicts)
- Say: "Ready for feedback."

### Step 5: Continue
Based on feedback:
- Apply changes if needed
- Execute next batch
- Repeat Step 2 -> Step 3 -> Step 4 for each batch

### Step 6: Complete Development

After all tasks complete and verified:
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
- Follow that skill to verify tests, present options, execute choice

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker mid-batch (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly

**Ask for clarification rather than guessing.**

## When to Revisit Earlier Steps

**Return to Review (Step 1) when:**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking

**Don't force through blockers** - stop and ask.

## Remember
- Review plan critically first
- Follow plan steps exactly
- Don't skip verifications
- Every batch goes through code review (subagent + Codex) before reporting to user
- Reference skills when plan says to
- Between batches: just report and wait
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:requesting-code-review** - Reviews each batch (subagent + Codex)
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
