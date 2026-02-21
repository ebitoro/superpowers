---
name: team-driven-development
description: Use when executing implementation plans with independent tasks using AgentTeam for minimal context usage
---

# Team-Driven Development

Execute plan by dispatching a Leader agent that manages a team of implementers and reviewers. Main session context stays minimal — only the final result flows back.

**Core principle:** Leader orchestrates via AgentTeam. Fresh agents per task. CC Reviewer commands Codex Reviewer directly. Agent-to-agent communication keeps main session lean.

## When to Use

Use instead of `subagent-driven-development` when:
- Plan has 5+ tasks (context savings matter)
- You want to minimize main session compaction risk
- Tasks are mostly independent

Use `subagent-driven-development` instead when:
- Plan has < 5 tasks (team overhead not worth it)
- You need tight human-in-loop control per task

## Codex Integration

> See `lib/codex-integration.md` for Codex patterns. All Codex interactions go through the codex-agent (`agents/codex-agent.md`), dispatched as a team member by the Leader.

## The Process

### Recover Context (Before Starting)

Check for session breadcrumbs left by `writing-plans`:

MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
PLAN_BREADCRUMB="$MAIN_REPO/.codex-state/current_plan"
WORKTREE_BREADCRUMB="$MAIN_REPO/.codex-state/current_worktree"

**If breadcrumbs exist:** Read plan path and worktree path, `cd` into worktree, load plan.
**If not:** Ask user for plan file path.

### Verify Worktree

Check if inside a git worktree (`git worktree list`). If NOT in a worktree, dispatch the `worktree-setup` agent.

### Read Prompt Templates

Read these files from `skills/team-driven-development/`:
- `leader-prompt.md`
- `cc-reviewer-prompt.md`
- `implementer-prompt.md`
- `fix-agent-prompt.md`

### Dispatch Leader

Use the Task tool to dispatch the Leader as a `general-purpose` subagent:

- **description:** "Team-driven development: execute [plan name]"
- **prompt:** Constructed from `leader-prompt.md` template, with these placeholders filled:
  - `{PLAN_TASKS}` — Full text of all tasks from the plan file
  - `{WORKTREE_PATH}` — Absolute worktree path
  - `{DESIGN_DOC_PATH}` — Path to design doc (from `.codex-state/current_design_doc`)
  - `{CC_REVIEWER_PROMPT}` — Full content of `cc-reviewer-prompt.md`
  - `{IMPLEMENTER_PROMPT}` — Full content of `implementer-prompt.md`
  - `{FIX_AGENT_PROMPT}` — Full content of `fix-agent-prompt.md`

The Leader handles team creation, task execution, review cycles, and cleanup internally.

### Wait for Result

The Leader reports back with:
- Final verdict (all tasks passed / some failed / escalated)
- Per-task summary (task name, rounds, verdict)
- Unresolved flags (if any)
- Codex availability status

### After Leader Returns

If all tasks passed:
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch

If tasks failed or were escalated:
- Report failures to user with issue details
- User decides: fix manually, re-run failed tasks, or abandon

## Prompt Templates

- `./leader-prompt.md` — Leader orchestration instructions
- `./cc-reviewer-prompt.md` — CC Reviewer dispatch template
- `./implementer-prompt.md` — Implementer dispatch template
- `./fix-agent-prompt.md` — Fix Agent dispatch template

## Red Flags

**Never:**
- Start implementation on main/master branch without explicit user consent
- Skip review (CC Reviewer is mandatory for every task)
- Dispatch multiple implementation agents in parallel (file conflicts)
- Manually orchestrate tasks from main session (defeats the purpose)

## Integration

**Required workflow skills:**
- **worktree-setup agent** — Set up isolated workspace before starting
- **superpowers:writing-plans** — Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** — Complete development after all tasks

**Implementer agents should use:**
- **superpowers:test-driven-development** — Follow TDD for each task
