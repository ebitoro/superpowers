---
name: team-driven-development
description: Use when executing implementation plans with independent tasks using AgentTeam for minimal context usage
---

# Team-Driven Development

Execute plan by acting as team Leader via AgentTeam, dispatching Implementer and Reviewer teammates for each task. Heavy work (implementation, code review, Codex review) runs in teammate contexts — main session only handles structured orchestration messages.

**Core principle:** This session creates the team and orchestrates via AgentTeam tools. Implementer handles its own review loop (self-review subagent + persistent Codex Reviewer, in parallel). CC Reviewer focuses on spec+quality only, communicating directly with Implementer for fixes. Agent-to-agent communication offloads the heavy context.

## When to Use

Use instead of `subagent-driven-development` when:
- Plan has 5+ tasks (context savings matter)
- You want to minimize main session compaction risk
- Tasks are mostly independent

Use `subagent-driven-development` instead when:
- Plan has < 5 tasks (team overhead not worth it)
- You need tight human-in-loop control per task

## Codex Integration

> See `lib/codex-integration.md` for Codex patterns. All Codex interactions go through a persistent Codex Reviewer teammate that dispatches the codex-agent (`agents/codex-agent.md`) internally.

## The Process

### Recover Context (Before Starting)

Check for session breadcrumbs left by `writing-plans`:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
PLAN_BREADCRUMB="$MAIN_REPO/.codex-state/current_plan"
WORKTREE_BREADCRUMB="$MAIN_REPO/.codex-state/current_worktree"
```

**If breadcrumbs exist:** Read plan path and worktree path, `cd` into worktree, load plan.
**If not:** Ask user for plan file path.

> Breadcrumbs persist in `.codex-state/` — other skills (Codex agent, requesting-code-review) reference them downstream. Cleanup happens in `finishing-a-development-branch`.

### Verify Worktree

Check if inside a git worktree (`git worktree list`). If NOT in a worktree, dispatch the `worktree-setup` agent (see `agents/worktree-setup.md`).

### Create Team

Create the team for this plan execution:

```
TeamCreate with team_name: "tdd-{timestamp}"
```

Record the team name — it is passed to the Leader instructions as `{TEAM_NAME}`.

### Act as Leader

You ARE the Leader. The orchestration runs in this session, not in a subagent. Subagents spawned without `team_name` do not have access to AgentTeam tools (TeamCreate, SendMessage, TaskCreate, etc.), so the Leader must operate within the team context.

Read `leader-prompt.md` from `skills/team-driven-development/` and follow its instructions. Supply these values from context recovery:
- `{TEAM_NAME}` — the team name created above
- `{PLAN_TASKS}` — full text of all tasks from the plan file
- `{WORKTREE_PATH}` — absolute worktree path
- `{DESIGN_DOC_PATH}` — path to design doc (from `.codex-state/current_design_doc`)

Read the prompt templates as the Leader instructions direct:
- `cc-reviewer-prompt.md` — used when dispatching CC Reviewer teammates
- `implementer-prompt.md` — used when dispatching Implementer teammates
- `codex-reviewer-prompt.md` — used when dispatching persistent Codex Reviewer
- `agents/code-reviewer.md` — used as `{SELF_REVIEW_PROMPT}` for Implementer's self-review subagent

### After All Tasks Complete

When the Leader instructions finish (all tasks processed, final review done):

1. Send `shutdown_request` to any remaining teammates
2. Call `TeamDelete` to clean up the team

If all tasks passed:
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch

If tasks failed or were escalated:
- Report failures to user with issue details
- User decides: fix manually, re-run failed tasks, or abandon

## Prompt Templates

- `./leader-prompt.md` — Leader orchestration reference (followed by this session)
- `./cc-reviewer-prompt.md` — CC Reviewer teammate dispatch prompt
- `./implementer-prompt.md` — Implementer teammate dispatch prompt
- `./codex-reviewer-prompt.md` — Persistent Codex Reviewer teammate dispatch prompt

## Red Flags

**Never:**
- Start implementation on main/master branch without explicit user consent
- Skip review (CC Reviewer is mandatory for every task)
- Dispatch multiple implementation agents in parallel (file conflicts)
- Dispatch CC Reviewer before Implementer signals readiness
- Dispatch the Leader as a regular subagent (it won't have team tool access)

## Integration

**Required workflow skills:**
- **worktree-setup agent** — Set up isolated workspace before starting
- **superpowers:writing-plans** — Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** — Complete development after all tasks

**Implementer agents should use:**
- **superpowers:test-driven-development** — Follow TDD for each task
