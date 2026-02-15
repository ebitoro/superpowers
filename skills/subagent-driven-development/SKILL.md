---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute plan by dispatching fresh subagent per task, with three-stage review after each: spec compliance, code quality, then Codex review.

**Core principle:** Fresh subagent per task + three-stage review (spec, quality, Codex) = high quality, fast iteration

## When to Use

```dot
digraph when_to_use {
    "Have implementation plan?" [shape=diamond];
    "Tasks mostly independent?" [shape=diamond];
    "Stay in this session?" [shape=diamond];
    "subagent-driven-development" [shape=box];
    "executing-plans" [shape=box];
    "Manual execution or brainstorm first" [shape=box];

    "Have implementation plan?" -> "Tasks mostly independent?" [label="yes"];
    "Have implementation plan?" -> "Manual execution or brainstorm first" [label="no"];
    "Tasks mostly independent?" -> "Stay in this session?" [label="yes"];
    "Tasks mostly independent?" -> "Manual execution or brainstorm first" [label="no - tightly coupled"];
    "Stay in this session?" -> "subagent-driven-development" [label="yes"];
    "Stay in this session?" -> "executing-plans" [label="no - parallel session"];
}
```

**vs. Executing Plans (parallel session):**
- Same session (no context switch)
- Fresh subagent per task (no context pollution)
- Three-stage review after each task: spec compliance, code quality, then Codex
- Faster iteration (no human-in-loop between tasks)

## Codex Integration

> **Reference:** See `lib/codex-integration.md` for shared patterns (state directory, availability, review gate logic, working directory awareness, cleanup).

**Context recovery** — on startup, locate the state directory and read the thread ID. The state directory is at the **main repo root**, not the worktree root. You MUST run this exact command to find it:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
cat "$MAIN_REPO/.codex-state/codex_thread_id"
```

Then validate the thread ID with a `codex-reply` ping:
- If it succeeds: reuse the thread
- If "Session not found": create new thread via `codex`, save ID to `$MAIN_REPO/.codex-state/codex_thread_id`, re-send context from design doc
- If other error: skip Codex, inform user

See `lib/codex-integration.md` for full recovery details.

## The Process

```dot
digraph process {
    rankdir=TB;

    subgraph cluster_per_task {
        label="Per Task";
        "Dispatch implementer subagent (./implementer-prompt.md)" [shape=box];
        "Implementer subagent asks questions?" [shape=diamond];
        "Answer questions, provide context" [shape=box];
        "Implementer subagent implements, tests, commits, self-reviews" [shape=box];
        "Dispatch spec reviewer subagent (./spec-reviewer-prompt.md)" [shape=box];
        "Spec reviewer subagent confirms code matches spec?" [shape=diamond];
        "Implementer subagent fixes spec gaps" [shape=box];
        "Dispatch code quality reviewer subagent (./code-quality-reviewer-prompt.md)" [shape=box];
        "Code quality reviewer subagent approves?" [shape=diamond];
        "Implementer subagent fixes quality issues" [shape=box];
        "Mark task complete in TodoWrite" [shape=box];
    }

    "Read plan, extract all tasks with full text, note context, create TodoWrite" [shape=box];
    "More tasks remain?" [shape=diamond];
    "Final code review (requesting-code-review)" [shape=box];
    "Use superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];

    "Read plan, extract all tasks with full text, note context, create TodoWrite" -> "Dispatch implementer subagent (./implementer-prompt.md)";
    "Dispatch implementer subagent (./implementer-prompt.md)" -> "Implementer subagent asks questions?";
    "Implementer subagent asks questions?" -> "Answer questions, provide context" [label="yes"];
    "Answer questions, provide context" -> "Dispatch implementer subagent (./implementer-prompt.md)";
    "Implementer subagent asks questions?" -> "Implementer subagent implements, tests, commits, self-reviews" [label="no"];
    "Implementer subagent implements, tests, commits, self-reviews" -> "Dispatch spec reviewer subagent (./spec-reviewer-prompt.md)";
    "Dispatch spec reviewer subagent (./spec-reviewer-prompt.md)" -> "Spec reviewer subagent confirms code matches spec?";
    "Spec reviewer subagent confirms code matches spec?" -> "Implementer subagent fixes spec gaps" [label="no"];
    "Implementer subagent fixes spec gaps" -> "Dispatch spec reviewer subagent (./spec-reviewer-prompt.md)" [label="re-review"];
    "Spec reviewer subagent confirms code matches spec?" -> "Dispatch code quality reviewer subagent (./code-quality-reviewer-prompt.md)" [label="yes"];
    "Dispatch code quality reviewer subagent (./code-quality-reviewer-prompt.md)" -> "Code quality reviewer subagent approves?";
    "Code quality reviewer subagent approves?" -> "Implementer subagent fixes quality issues" [label="no"];
    "Implementer subagent fixes quality issues" -> "Dispatch code quality reviewer subagent (./code-quality-reviewer-prompt.md)" [label="re-review"];
    "Code quality reviewer subagent approves?" -> "Codex per-task review (codex-reply)" [label="yes"];
    "Codex per-task review (codex-reply)" [shape=box style=dashed];
    "Codex flags issues?" [shape=diamond style=dashed];
    "Implementer subagent fixes Codex issues" [shape=box style=dashed];
    "Codex per-task review (codex-reply)" -> "Codex flags issues?";
    "Codex flags issues?" -> "Mark task complete in TodoWrite" [label="no (pass)"];
    "Codex flags issues?" -> "Implementer subagent fixes Codex issues" [label="yes (max 5 rounds)"];
    "Implementer subagent fixes Codex issues" -> "Codex per-task review (codex-reply)" [label="resubmit"];
    "Mark task complete in TodoWrite" -> "More tasks remain?";
    "More tasks remain?" -> "Dispatch implementer subagent (./implementer-prompt.md)" [label="yes"];
    "More tasks remain?" -> "Final code review (requesting-code-review)" [label="no"];
    "Final code review (requesting-code-review)" -> "Use superpowers:finishing-a-development-branch";
}
```

## Per-Task Codex Review

> **Reference:** See `lib/codex-integration.md` for shared patterns (review gate logic, availability).

After the code quality reviewer approves, ensure changes are committed, then send the commit SHAs to Codex via `codex-reply` for a third opinion. This catches cross-cutting issues that subagent reviewers miss because they lack project-wide context.

**Verify before acting:** Codex is a reference, not authority. When Codex flags an issue, CC must read the relevant code and independently confirm the issue exists before fixing it. If Codex's claim doesn't match the code, push back with evidence. See `lib/codex-integration.md` "Core Principle" for details.

**Important:** The implementer subagent should have already committed as part of its workflow. Verify with `git status` — if uncommitted changes remain, commit them before sending to Codex.

**What to send** (see `lib/codex-integration.md` "Efficient Codex Communication"):
- The commit SHA(s) covering the task — NOT the raw diff text
- A short summary of what was implemented and the task spec
- Test results summary (pass/fail counts)
- The worktree path note (see `lib/codex-integration.md`)

**Review gate:** Follow the standard review gate pattern from `lib/codex-integration.md` (max 5 rounds, fix and resubmit until pass). If passing with unresolved flags, append them to `docs/unresolved-flags.md` and commit (see `lib/codex-integration.md` for format).

**If Codex is unavailable:** Skip this step and proceed with the subagent results only. Inform the user that Codex per-task review was skipped.

> Dashed nodes in the process diagram are skipped when Codex is unavailable.

## Final Code Review

After all tasks complete, request a final code review of the entire implementation.

**REQUIRED SUB-SKILL:** Use superpowers:requesting-code-review with:
- Full branch diff (`git diff main...HEAD` or equivalent)
- The design doc (read from `.codex-state/current_design_doc`)
- The implementation plan filename
- Test results summary

The code review skill handles both the subagent review and the Codex review gate. Only proceed to `finishing-a-development-branch` after the review passes.

## Prompt Templates

- `./implementer-prompt.md` - Dispatch implementer subagent
- `./spec-reviewer-prompt.md` - Dispatch spec compliance reviewer subagent
- `./code-quality-reviewer-prompt.md` - Dispatch code quality reviewer subagent

## Example Workflow

```
You: I'm using Subagent-Driven Development to execute this plan.

[Read plan file once: docs/plans/feature-plan.md]
[Extract all 5 tasks with full text and context]
[Create TodoWrite with all tasks]

Task 1: Hook installation script

[Get Task 1 text and context (already extracted)]
[Dispatch implementation subagent with full task text + context]

Implementer: "Before I begin - should the hook be installed at user or system level?"

You: "User level (~/.config/superpowers/hooks/)"

Implementer: "Got it. Implementing now..."
[Later] Implementer:
  - Implemented install-hook command
  - Added tests, 5/5 passing
  - Self-review: Found I missed --force flag, added it
  - Committed

[Dispatch spec compliance reviewer]
Spec reviewer: Spec compliant - all requirements met, nothing extra

[Get git SHAs, dispatch code quality reviewer]
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Approved.

[Send commit SHAs + summary to Codex via codex-reply]
  "Review abc1234..def5678. Implemented install-hook command.
   Tests: 5 passing. Worktree at /path/.worktrees/hooks."
Codex: Pass. Minor note: consider adding --dry-run flag for safety.

[Mark Task 1 complete]

Task 2: Recovery modes
...

[After all tasks]
[Request final code review via requesting-code-review skill]
  Code-reviewer subagent: All requirements met
  Codex: Missing input validation on recovery mode parameter
  [Fix: add validation]
  [Re-review]
  Code-reviewer subagent: Approved
  Codex: Pass

Done!
```

## Advantages

**vs. Manual execution:**
- Subagents follow TDD naturally
- Fresh context per task (no confusion)
- Parallel-safe (subagents don't interfere)
- Subagent can ask questions (before AND during work)

**vs. Executing Plans:**
- Same session (no handoff)
- Continuous progress (no waiting)
- Review checkpoints automatic

**Efficiency gains:**
- No file reading overhead (controller provides full text)
- Controller curates exactly what context is needed
- Subagent gets complete information upfront
- Questions surfaced before work begins (not after)

**Quality gates:**
- Self-review catches issues before handoff
- Three-stage review: spec compliance, code quality, then Codex
- Codex per-task review catches cross-cutting issues subagents miss
- Final code review (requesting-code-review) catches cross-task issues
- Review loops ensure fixes actually work
- Spec compliance prevents over/under-building
- Code quality ensures implementation is well-built

**Cost:**
- More subagent invocations (implementer + 2 reviewers per task + Codex per task)
- Controller does more prep work (extracting all tasks upfront)
- Review loops add iterations
- Per-task Codex review + final Codex review adds token cost
- But catches issues early (cheaper than debugging later)

## Red Flags

**Never:**
- Start implementation on main/master branch without explicit user consent
- Skip reviews (spec compliance OR code quality OR Codex per-task OR final review)
- Proceed with unfixed issues
- Dispatch multiple implementation subagents in parallel (conflicts)
- Make subagent read plan file (provide full text instead)
- Skip scene-setting context (subagent needs to understand where task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance (spec reviewer found issues = not done)
- Skip review loops (reviewer found issues = implementer fixes = review again)
- Let implementer self-review replace actual review (both are needed)
- **Start code quality review before spec compliance is approved** (wrong order)
- **Start Codex review before code quality is approved** (wrong order)
- **Start final code review before all tasks are complete** (wrong order)
- Move to next task while any review has open issues

**If subagent asks questions:**
- Answer clearly and completely
- Provide additional context if needed
- Don't rush them into implementation

**If reviewer finds issues:**
- Implementer (same subagent) fixes them
- Reviewer reviews again
- Repeat until approved
- Don't skip the re-review

**If subagent fails task:**
- Dispatch fix subagent with specific instructions
- Don't try to fix manually (context pollution)

## Integration

**Required workflow skills:**
- **worktree-setup agent** - REQUIRED: Set up isolated workspace before starting. Dispatch `agents/worktree-setup.md` (runs on Sonnet, keeps setup out of context window).
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:requesting-code-review** - Final review (subagent + Codex) for entire implementation
- **superpowers:finishing-a-development-branch** - Complete development after all tasks

**Subagents should use:**
- **superpowers:test-driven-development** - Subagents follow TDD for each task

**Alternative workflow:**
- **superpowers:executing-plans** - Use for parallel session instead of same-session execution
