---
name: team-driven-development
description: Use when executing implementation plans with independent tasks using AgentTeam for minimal context usage
---

# Team-Driven Development

Execute plan by acting as team Leader via AgentTeam, dispatching Implementer and Codex Reviewer teammates for each task. Per-task review: in-session self-review + Codex (parallel) → spec compliance → code quality, all inside the Implementer. Branch-level review: Final Reviewer teammate runs spec + quality + Codex in parallel.

**Core principle:** Leader is lightweight orchestration — dispatches, waits for verdicts, moves to next task. Implementer owns per-task review (in-session self-review + Codex parallel → spec → quality). Final Reviewer owns branch-level review (spec + quality + Codex in parallel). Codex Reviewer persists across tasks as a shared service.

## When to Use

Use instead of `subagent-driven-development` when:
- Plan has 5+ tasks (context savings matter)
- You want to minimize main session compaction risk
- Tasks are mostly independent

Use `subagent-driven-development` instead when:
- Plan has < 5 tasks (team overhead not worth it)
- You need tight human-in-loop control per task

## Model Configuration

All subagents inherit the session's model. Agent types with a `model` field in their SKILL.md frontmatter (e.g., `superpowers:codex-agent`) use their configured model instead.

## Codex Integration

> See `lib/codex-integration.md` for Codex patterns. All Codex interactions go through a persistent Codex Reviewer teammate that dispatches the codex-agent (`skills/codex-agent/SKILL.md`) internally. The codex-agent selects the right Codex skill automatically based on message content.

## The Process

### Recover Context (Before Starting)

Check for session breadcrumbs left by `writing-plans`:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
PLAN_BREADCRUMB="$MAIN_REPO/.codex-state/current_plan"
WORKTREE_BREADCRUMB="$MAIN_REPO/.codex-state/current_worktree"
```

<HARD-GATE>
**If breadcrumbs exist**, follow this EXACT order — do NOT read the plan before cd'ing into the worktree:

1. Read the worktree path from `$WORKTREE_BREADCRUMB`
2. **`cd` into the worktree FIRST** — the plan file path is relative and only exists in the worktree branch
3. Read the plan file path from `$PLAN_BREADCRUMB`
4. Construct the absolute path: `$WORKTREE_PATH/$PLAN_PATH`
5. Load and parse the plan file using the absolute path
</HARD-GATE>

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
- `implementer-prompt.md` — used when dispatching Implementer teammates
- `codex-reviewer-prompt.md` — used when dispatching persistent Codex Reviewer

### After All Tasks Complete

When the Leader instructions finish (all tasks processed, final review done):

1. Send `shutdown_request` to any remaining teammates
2. Call `TeamDelete` to clean up the team

If all tasks passed:
- **Dispatch update-docs subagent** (opt-in — only if project CLAUDE.md has `## Post-Implementation Docs`):

```
Agent tool:
  subagent_type: "general-purpose"
  description: "Update post-implementation documentation"
  prompt: |
    You are a documentation updater. Use the superpowers:update-docs-after-implementation skill.

    Working directory: {WORKTREE_PATH}
    Base SHA: [commit before first task]

    Read all commits since BASE_SHA, find the Post-Implementation Docs list
    in the project CLAUDE.md, update each document, and commit.

    If no Post-Implementation Docs section exists in CLAUDE.md, report
    "No post-implementation docs configured — skipping" and exit.
```

- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch

If tasks failed or were escalated:
- Report failures to user with issue details
- User decides: fix manually, re-run failed tasks, or abandon

## Prompt Templates

- `./leader-prompt.md` — Leader orchestration reference (followed by this session)
- `./implementer-prompt.md` — Implementer teammate dispatch prompt
- `./codex-reviewer-prompt.md` — Persistent Codex Reviewer teammate dispatch prompt
- `./final-reviewer-prompt.md` — Persistent Final Reviewer teammate dispatch prompt

## Red Flags

**Never:**
- Start implementation on main/master branch without explicit user consent
- Dispatch multiple implementation agents in parallel (file conflicts)
- Dispatch the Leader as a regular subagent (it won't have team tool access)
- Skip any review phase (in-session self-review + Codex → spec compliance → code quality)

## Integration

**Required workflow skills:**
- **worktree-setup agent** — Set up isolated workspace before starting
- **superpowers:writing-plans** — Creates the plan this skill executes
- **superpowers:update-docs-after-implementation** — Update project docs after final review (opt-in via project CLAUDE.md)
- **superpowers:finishing-a-development-branch** — Complete development after all tasks

**Implementer agents should use:**
- **superpowers:test-driven-development** — Follow TDD for each task
