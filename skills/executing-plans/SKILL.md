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

**Context recovery** — on startup, read `CODEX_THREAD_ID` from `.codex-state/codex_thread_id`. If valid (file exists, non-empty, test `codex-reply` succeeds), reuse the thread. If missing or unavailable, proceed without Codex and inform the user.

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

### Step 3: Codex Code Review Gate

When batch complete, submit the batch for Codex code review before reporting to user. Follow the review gate pattern from `lib/codex-integration.md`.

**What to send to Codex via `codex-reply`:**
- List of files changed in this batch (use `git diff --name-only` against the branch point)
- The actual diff (`git diff` for the batch's commits)
- Verification/test output
- Which tasks from the plan were implemented
- The worktree path note

### Step 4: Report
When batch complete and Codex review passed (or 5 rounds exhausted):
- Show what was implemented
- Show verification output
- Show Codex review result (pass, or unresolved flags)
- Say: "Ready for feedback."

### Step 5: Continue
Based on feedback:
- Apply changes if needed
- Execute next batch
- Repeat Step 2 → Step 3 → Step 4 for each batch

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
- Every batch goes through Codex code review before reporting to user
- Reference skills when plan says to
- Between batches: just report and wait
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
