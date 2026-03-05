---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute plan by dispatching fresh subagent per task. Each subagent handles implementation + full review pipeline (self-review, Codex, spec compliance, code quality) and reports a structured verdict.

**Core principle:** Fresh subagent per task with self-contained review pipeline = high quality, minimal main session context

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
- Implementer handles full review pipeline internally (minimal main session context)
- Faster iteration (no human-in-loop between tasks)

## Model Configuration

| Role | Default | Overridable | Rationale |
|------|---------|-------------|-----------|
| **Implementer** | **Inherits from session** | **Yes** | User controls cost/quality tradeoff |
| Spec compliance reviewer | Opus | No | Deep analysis needs strongest model |
| Code quality reviewer | Opus | No | Deep analysis needs strongest model |
| Codex reviewer | Sonnet | No | Relay to Codex MCP, lightweight |

**Model override:** Include a model preference in your prompt when invoking this skill. Examples:
- "execute this plan using sonnet for implementers"
- "run with sonnet model"

The override applies to the implementer only. Review subagents always use their fixed models.

## Codex Integration

> See `lib/codex-integration.md` for Codex patterns. The implementer subagent dispatches codex-agent directly via the Task tool — no persistent teammate needed.

## The Process

### Recover Context (Before Starting)

Check for session breadcrumbs left by `writing-plans`. This enables `/clear` between planning and execution without losing state:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
PLAN_BREADCRUMB="$MAIN_REPO/.codex-state/current_plan"
WORKTREE_BREADCRUMB="$MAIN_REPO/.codex-state/current_worktree"
```

**If breadcrumbs exist** (post-`/clear` or new session):
1. Read the worktree path from `$WORKTREE_BREADCRUMB` and `cd` into it
2. Read the plan file path from `$PLAN_BREADCRUMB`
3. Load and parse the plan file

> Breadcrumbs persist in `.codex-state/` — other skills (Codex agent, requesting-code-review) reference them downstream. Cleanup happens in `finishing-a-development-branch`.

**If breadcrumbs do not exist** (same-session handoff from `writing-plans`):
- The plan is already in context from the current session
- Ask the user for the plan file path if it is not in context

### Parse Model Override

Check the user's prompt for a model preference. Look for patterns like "use sonnet", "using sonnet for implement", "with sonnet", etc.

- **If found:** Set `work_model` to the specified model (e.g., "sonnet", "haiku")
- **If not found:** Leave `work_model` unset (implementer inherits from session)

This only affects the implementer. Review subagents (spec, quality, codex) use fixed models regardless.

### Verify Worktree

Check if inside a git worktree (`git worktree list`). If NOT in a worktree, dispatch the `worktree-setup` agent (see `agents/worktree-setup.md`) with the branch name. The agent runs on Sonnet and handles the full setup.

### Initialize Codex Thread (Before Any Tasks)

Create the Codex review thread upfront so every implementer receives a concrete thread_id (never "new"). This prevents implementers from calling `codex` MCP directly to create threads.

Dispatch codex-agent:

```
Agent tool:
  subagent_type: "superpowers:codex-agent"
  model: "sonnet"
  description: "Initialize Codex review thread"
  prompt: |
    mode: discuss
    thread_id: "new"
    profile: "higheffort"
    message: |
      Starting implementation review session.
      Plan: [plan name or one-line summary]
      We will review individual task diffs as they are implemented.
    worktree_path: [worktree absolute path]
```

**Do NOT start any tasks until this completes.** You must have a concrete `codex_thread_id` or a definitive "unavailable" status before proceeding.

**If result received:** Extract `thread_id` from the response. Set `codex_thread_id` to this value and `codex_status = "available"`.

**If Codex is unavailable** (timeout, MCP error, etc.): Set `codex_status = "unavailable"` and `codex_thread_id = "none"`. All tasks will skip Codex review.

<HARD-GATE>
Do NOT dispatch any implementer subagent until `codex_thread_id` and `codex_status` are resolved. Starting tasks without these values causes tasks to silently skip Codex review.
</HARD-GATE>

### Per-Task Workflow

```dot
digraph process {
    rankdir=TB;

    subgraph cluster_per_task {
        label="Per Task";
        "Record BASE_SHA" [shape=box];
        "Dispatch implementer subagent (./implementer-prompt.md)" [shape=box];
        "Implementer asks questions?" [shape=diamond];
        "Answer questions, re-dispatch implementer" [shape=box];
        "Implementer runs full review pipeline internally" [shape=box];
        "Parse ## Task Verdict from response" [shape=box];
        "Verdict is pass?" [shape=diamond];
        "Dispatch fix subagent or escalate to user" [shape=box];
        "Mark task complete in TodoWrite" [shape=box];
    }

    "Read plan, extract all tasks with full text, note context, create TodoWrite" [shape=box];
    "More tasks remain?" [shape=diamond];
    "Dispatch final-review subagent (./final-review-prompt.md)" [shape=box];
    "Parse ## Final Review Verdict" [shape=box];
    "Final verdict pass?" [shape=diamond];
    "Use superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];

    "Read plan, extract all tasks with full text, note context, create TodoWrite" -> "Record BASE_SHA";
    "Record BASE_SHA" -> "Dispatch implementer subagent (./implementer-prompt.md)";
    "Dispatch implementer subagent (./implementer-prompt.md)" -> "Implementer asks questions?";
    "Implementer asks questions?" -> "Answer questions, re-dispatch implementer" [label="yes"];
    "Answer questions, re-dispatch implementer" -> "Dispatch implementer subagent (./implementer-prompt.md)";
    "Implementer asks questions?" -> "Implementer runs full review pipeline internally" [label="no"];
    "Implementer runs full review pipeline internally" -> "Parse ## Task Verdict from response";
    "Parse ## Task Verdict from response" -> "Verdict is pass?";
    "Verdict is pass?" -> "Mark task complete in TodoWrite" [label="yes"];
    "Verdict is pass?" -> "Dispatch fix subagent or escalate to user" [label="no"];
    "Dispatch fix subagent or escalate to user" -> "Dispatch implementer subagent (./implementer-prompt.md)" [label="retry"];
    "Mark task complete in TodoWrite" -> "More tasks remain?";
    "More tasks remain?" -> "Record BASE_SHA" [label="yes"];
    "More tasks remain?" -> "Dispatch final-review subagent (./final-review-prompt.md)" [label="no"];
    "Dispatch final-review subagent (./final-review-prompt.md)" -> "Parse ## Final Review Verdict";
    "Parse ## Final Review Verdict" -> "Final verdict pass?";
    "Final verdict pass?" -> "Dispatch update-docs subagent (if configured)" [label="yes"];
    "Dispatch update-docs subagent (if configured)" [shape=box];
    "Dispatch update-docs subagent (if configured)" -> "Use superpowers:finishing-a-development-branch";
    "Final verdict pass?" -> "Escalate to user" [label="no"];
}
```

## Per-Task Dispatch

For each task, the main session:

### Step 1: Record BASE_SHA

```bash
BASE_SHA=$(git rev-parse HEAD)
```

### Step 2: Determine CODEX_STATUS and CODEX_THREAD_ID

Use the values established in "Initialize Codex Thread" (before the task loop). Both are set once and propagated:

- `codex_thread_id` — the concrete thread ID from initialization (never "new" — the main session already created the thread)
- `codex_status` — "available" or "unavailable" from initialization

**Update on status change:** If a task's verdict reports `codex_review.status: unavailable`, set `codex_status = "unavailable"` for all subsequent tasks.

### Step 3: Dispatch Implementer

Fill the template from `./implementer-prompt.md` with these 8 inputs:

| Variable | Source |
|----------|--------|
| `{TASK_NUMBER}` | Task index from plan |
| `{TASK_NAME}` | Task title from plan |
| `{TASK_TEXT}` | Full task text from plan (paste, don't reference file) |
| `{CONTEXT}` | Scene-setting: where this fits, dependencies, architecture |
| `{WORKING_DIRECTORY}` | Worktree absolute path |
| `{BASE_SHA}` | From Step 1 |
| `{CODEX_STATUS}` | From Step 2 |
| `{CODEX_THREAD_ID}` | From Step 2 (always a concrete ID from pre-initialized thread, never "new") |

Dispatch via Task tool (`subagent_type: "general-purpose"`). If `work_model` is set, add `model: "{work_model}"` to the Task tool call.

The implementer handles the full review pipeline internally (self-review → Codex → spec compliance → code quality) and returns a `## Task Verdict`. Review subagents dispatched by the implementer use fixed models (opus for spec/quality, sonnet for codex) regardless of the implementer's model.

## Parsing the Task Verdict

Scan the implementer's response for `## Task Verdict`. Extract:

- **`verdict`**: `pass` or `fail`
- **`head_sha`**: The commit after all implementation and fixes
- **`codex_review.status`**: `available` or `unavailable` — propagate to subsequent tasks
- **`concerns`**: Non-blocking risks worth noting

**If `verdict: pass`:**
- Mark task complete in TodoWrite
- Proceed to next task

**If `verdict: fail`:**
- Check `unresolved` fields for specifics
- Dispatch a fix subagent with the specific unresolved issues, or escalate to user
- Don't try to fix manually (context pollution)

**If `concerns` is non-empty:**
- Report concerns to user but continue

**If `codex_review.status: unavailable`:**
- Set `codex_status = "unavailable"` for remaining tasks
- Inform user that Codex review was skipped

## Final Code Review

After all tasks complete, dispatch a final-review subagent to review the entire implementation. The subagent handles the full review cycle (code-reviewer + Codex) and fixes any issues it finds — the main session only sees the final verdict.

### Dispatch Final Review Subagent

Determine the base SHA for the full implementation scope (the commit before the first task started — typically recorded before Task 1, or use `origin/main`).

Fill the template from `./final-review-prompt.md` with these inputs:

| Variable | Source |
|----------|--------|
| `{BASE_SHA}` | Commit before first task (full implementation scope) |
| `{WORKING_DIRECTORY}` | Worktree absolute path |
| `{DESIGN_DOC_PATH}` | From `.codex-state/current_design_doc` (empty string if missing) |
| `{PLAN_FILE_PATH}` | Plan file path |
| `{CODEX_STATUS}` | Current codex_status value |

Dispatch via Task tool (`subagent_type: "general-purpose"`, `model: "opus"`). The final review creates its own fresh Codex thread with `profile: "xhigheffort"` — a clean context for the high-stakes final review, separate from the per-task `higheffort` thread.

### Parse the Final Review Verdict

Scan the subagent's response for `## Final Review Verdict`. Extract:

- **`verdict`**: `pass` or `fail`
- **`head_sha`**: The commit after all fixes
- **`codex_review.status`**: Whether Codex was available
- **`unresolved_flags`**: Minor items tracked in `docs/unresolved-flags.md`
- **`concerns`**: Non-blocking risks

**If `verdict: pass`:**
- Report result to user
- Proceed to `finishing-a-development-branch`

**If `verdict: fail`:**
- Report the unresolved issues to user
- Let user decide: re-dispatch final review, fix manually, or proceed anyway

## Update Post-Implementation Docs

After the final review passes and before invoking `finishing-a-development-branch`, dispatch a subagent to update project documentation. This step is **automatic but opt-in** — it only runs if the project's CLAUDE.md has a `## Post-Implementation Docs` section.

```
Agent tool:
  subagent_type: "general-purpose"
  model: "opus"
  description: "Update post-implementation documentation"
  prompt: |
    You are a documentation updater. Use the superpowers:update-docs-after-implementation skill.

    Working directory: [worktree absolute path]
    Base SHA: [commit before first task — the original BASE_SHA from Task 1]

    Read all commits since BASE_SHA, find the Post-Implementation Docs list
    in the project CLAUDE.md, update each document, and commit.

    If no Post-Implementation Docs section exists in CLAUDE.md, report
    "No post-implementation docs configured — skipping" and exit.
```

**If status is `updated`:** Report updated documents to user, then proceed to `finishing-a-development-branch`.

**If status is `skipped`:** Proceed to `finishing-a-development-branch` silently.

**If status is `error`:** Report error to user but still proceed to `finishing-a-development-branch` (doc update failure should not block completion).

## Prompt Templates

- `./implementer-prompt.md` — Dispatch implementer subagent (handles full review pipeline internally)
- `./final-review-prompt.md` — Dispatch final review subagent (handles review + fixes, reports verdict)
- `./spec-reviewer-prompt.md` — Spec compliance reviewer (dispatched by implementer, not main session)
- `./code-quality-reviewer-prompt.md` — Code quality reviewer (dispatched by implementer, not main session)

## Example Workflow

```
You: I'm using Subagent-Driven Development to execute this plan.

[Read plan file once: docs/plans/feature-plan.md]
[Extract all 5 tasks with full text and context]
[Create TodoWrite with all tasks]

[Initialize Codex thread: dispatch codex-agent with mode=discuss, thread_id="new"]
[Result: codex_thread_id = thread_abc123, codex_status = "available"]

Task 1: Hook installation script
BASE_SHA=$(git rev-parse HEAD)

[Dispatch implementer subagent with full task text + context + BASE_SHA + codex_status + codex_thread_id=thread_abc123]

Implementer: "Before I begin - should the hook be installed at user or system level?"

You: "User level (~/.config/superpowers/hooks/)"

[Re-dispatch implementer with answer]

[Implementer runs full pipeline internally:
  - Implements, tests, commits
  - Self-review: finds missed --force flag, fixes
  - Codex review: pass (1 false positive dismissed) — uses thread_abc123 via codex-agent
  - Spec compliance: pass
  - Code quality: pass]

## Task Verdict
task: 1
verdict: pass
base_sha: abc1234
head_sha: def5678
implementation_summary: Hook installation script with --force flag
codex_review:
  status: available
  rounds: 1
  thread_id: thread_abc123
  findings_fixed: 0
spec_compliance: pass (round 1)
code_quality: pass (round 1)
tests: 5/5 passing
concerns: none

[Mark Task 1 complete]

Task 2: Recovery modes
BASE_SHA=$(git rev-parse HEAD)
[Dispatch implementer with codex_thread_id = thread_abc123 (same pre-initialized thread)]
...

[After all tasks]
[Dispatch final-review subagent with BASE_SHA from before Task 1, worktree path, design doc, plan file, codex_status]

[Final-review subagent runs internally:
  - Dispatches code-reviewer subagent (opus): 1 Important finding
  - Fixes the finding, re-dispatches: pass
  - Dispatches codex-agent: pass (1 false positive dismissed)
  - Reports ## Final Review Verdict: pass]

[Parse verdict: pass, proceed to finishing-a-development-branch]

Done!
```

## Red Flags

**Never:**
- Start any task before Codex thread initialization completes (wait for `codex_thread_id` or "unavailable" — see HARD-GATE above)
- Start implementation on main/master branch without explicit user consent
- Dispatch multiple implementation subagents in parallel (conflicts)
- Make subagent read plan file (provide full text instead)
- Skip scene-setting context (subagent needs to understand where task fits)
- Ignore subagent questions (answer before letting them proceed)
- Dispatch spec/quality reviewers from the main session (implementer handles them internally)
- Skip parsing `## Task Verdict` (always parse and act on the verdict)
- Ignore `codex_review.status` changes in the verdict (propagate to subsequent tasks)
- Start final code review before all tasks are complete
- Run final code review in the main session (dispatch the final-review subagent instead)
- Move to next task while verdict is `fail`

**If subagent asks questions:**
- Answer clearly and completely
- Provide additional context if needed
- Re-dispatch with the answer

**If verdict is `fail`:**
- Dispatch fix subagent with specific unresolved issues
- Don't try to fix manually (context pollution)

## Integration

**Required workflow skills:**
- **worktree-setup agent** - REQUIRED: Set up isolated workspace before starting. Dispatch `agents/worktree-setup.md` (runs on Sonnet, keeps setup out of context window).
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:update-docs-after-implementation** - Update project docs after final review (opt-in via project CLAUDE.md)
- **superpowers:finishing-a-development-branch** - Complete development after all tasks

**Subagents should use:**
- **superpowers:test-driven-development** - Subagents follow TDD for each task

**Alternative workflow:**
- **superpowers:executing-plans** - Use for parallel session instead of same-session execution
