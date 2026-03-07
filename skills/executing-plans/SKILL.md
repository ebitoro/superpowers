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

> See `lib/codex-integration.md` for Codex patterns. All interactions go through the codex-agent (`skills/codex-agent/SKILL.md`).

Codex review for each batch is handled by `superpowers:requesting-code-review`, which dispatches the codex-agent internally.

## The Process

### Step 1: Verify Worktree

Check if inside a git worktree (`git worktree list`). If NOT in a worktree, dispatch the `worktree-setup` agent (see `agents/worktree-setup.md`) with the branch name. The agent runs on Sonnet and handles the full setup.

### Step 2: Load and Review Plan
1. Read plan file
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed

### Step 3: Execute Batch
**Default: First 3 tasks**

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

### Step 4: Code Review

When batch complete, request code review before reporting to user.

**REQUIRED SUB-SKILL:** Use superpowers:requesting-code-review with the batch diff. The code review skill handles both the subagent review and the Codex review gate.

### Step 5: Report
When batch complete and code review passed:
- Show what was implemented
- Show verification output
- Show code review result (subagent + Codex verdicts)
- Say: "Ready for feedback."

### Step 6: Continue
Based on feedback:
- Apply changes if needed
- Execute next batch
- Repeat Step 3 -> Step 4 -> Step 5 for each batch

### Step 7: Update Documentation

After all tasks complete and verified, dispatch a doc update subagent (opt-in — only runs if project CLAUDE.md has `## Post-Implementation Docs`):

```
Agent tool:
  subagent_type: "general-purpose"
  model: "opus"
  description: "Update post-implementation documentation"
  prompt: |
    You are a documentation updater. Use the superpowers:update-docs-after-implementation skill.

    Working directory: [worktree absolute path]
    Base SHA: [commit before first task]

    Read all commits since BASE_SHA, find the Post-Implementation Docs list
    in the project CLAUDE.md, update each document, and commit.

    If no Post-Implementation Docs section exists in CLAUDE.md, report
    "No post-implementation docs configured — skipping" and exit.
```

If status is `error`, report but continue. Doc update failure does not block completion.

### Step 8: Complete Development

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

**Return to Review (Step 2) when:**
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
- **worktree-setup agent** - REQUIRED: Set up isolated workspace before starting. Dispatch `agents/worktree-setup.md` (runs on Sonnet, keeps setup out of context window).
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:requesting-code-review** - Reviews each batch (subagent + Codex)
- **superpowers:update-docs-after-implementation** - Update project docs after final review (opt-in via project CLAUDE.md)
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
