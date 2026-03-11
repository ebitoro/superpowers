---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

## Overview

Load plan, review critically, execute all tasks, report when complete.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

**Note:** Tell your human partner that Superpowers works much better with access to subagents. The quality of its work will be significantly higher if run on a platform with subagent support (such as Claude Code or Codex). If subagents are available, use superpowers:subagent-driven-development instead of this skill.

## The Process

### Step 1: Verify Worktree

Never implement on main/master. Check if you're in a worktree:
```bash
git rev-parse --git-common-dir 2>/dev/null
```
If the result equals `.git` (not a worktree), invoke `superpowers:using-git-worktrees` to create one before continuing. If already in a worktree, proceed.

### Step 2: Load and Review Plan
1. Find and read plan file — the user may have cleared the session, so discover the plan:
   - **Check breadcrumb first:**
     ```bash
     MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
     PLAN_PATH="$MAIN_REPO/$(cat "$MAIN_REPO/.dev-state/current_plan" 2>/dev/null)"
     ```
   - **If breadcrumb missing or file not found:** scan `docs/superpowers/plans/` for the most recent plan file (by filename date prefix or modification time)
   - **If multiple candidates:** ask the user which one
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed

### Step 3: Execute Tasks

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

### Step 4: Codex Review Gate

After all tasks complete and verified, run a Codex review gate on the full implementation. See `lib/codex-integration.md` for full protocol.

1. Get commit SHAs covering all implementation:
   ```bash
   BASE_SHA=$(git merge-base HEAD origin/main)  # or the branch base
   HEAD_SHA=$(git rev-parse HEAD)
   ```

2. Dispatch codex-agent (foreground):
   ```
   Agent tool:
     subagent_type: "superpowers:codex-agent"
     description: "Codex final review"
     prompt: |
       mode: review-gate
       thread_id: "new"
       message: |
         Final review of complete implementation.
         Commits: <BASE_SHA>..<HEAD_SHA>
         Summary: <what the plan implemented>
         Tests: <pass/fail summary>
       context: Implementation of <plan-file-path>
       worktree_path: <worktree-path>
       profile: xhigheffort
   ```

3. Echo `**Active Codex thread_id:** <id>`
4. If `pass`: proceed to Step 5
5. If `fail`: **independently verify each finding** — read the actual code at the cited location and confirm the issue exists. Dismiss false positives. Fix confirmed issues, then redispatch with saved `thread_id`. Max 5 rounds.
6. Track any unresolved flags in `docs/unresolved-flags.md`
7. If `unavailable`: skip Codex review and proceed (inform user).

### Step 5: Complete Development

After all tasks complete and Codex review passes (or is skipped):
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
- Follow that skill to verify tests, present options, execute choice

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
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
- Reference skills when plan says to
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
