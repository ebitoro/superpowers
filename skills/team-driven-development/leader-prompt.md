# Leader Instructions — Team-Driven Development

These are the orchestration instructions for team-driven development. The team has already been created by the SKILL.md entry point — you operate within it.

**Model note:** Leader orchestration is lightweight — sonnet is sufficient for this session.

## Inputs

- **Team name:** {TEAM_NAME} (already created — use for all teammate dispatches)
- **Plan tasks:** {PLAN_TASKS}
- **Worktree path:** {WORKTREE_PATH}
- **Design doc:** {DESIGN_DOC_PATH}

### Prompt Templates

Read these from `skills/team-driven-development/`:
- `implementer-prompt.md` — Implementer teammate dispatch prompt
- `codex-reviewer-prompt.md` — Codex Reviewer teammate dispatch prompt
- `agents/code-reviewer.md` — Self-review subagent prompt (used as `{SELF_REVIEW_PROMPT}`)

## Setup

1. Parse `{PLAN_TASKS}` — create a `TaskCreate` for each task (preserve order).
2. Set internal flag: `codex_available = true`.
3. Determine `BASE_BRANCH` (try in order, use first that succeeds):
   ```bash
   MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
   BASE_BRANCH="$(cat "$MAIN_REPO/.codex-state/base_branch" 2>/dev/null)"
   [ -z "$BASE_BRANCH" ] && BASE_BRANCH="$(git config "branch.$(git branch --show-current).merge" 2>/dev/null | sed 's|refs/heads/||')"
   [ -z "$BASE_BRANCH" ] && BASE_BRANCH="main"
   ```
   Record for final review diff.
4. **Dispatch persistent Codex Reviewer** as a teammate (if `codex_available`):
   - Task tool with `team_name: "{TEAM_NAME}"`, `name: "codex-reviewer"`, `subagent_type: "general-purpose"`, `model: "sonnet"`
   - Use `codex-reviewer-prompt.md` filled with: `{WORKTREE_PATH}`, `{TEAM_NAME}`
   - This reviewer persists for all tasks

## Per-Task Workflow

Iterate tasks **in order** (never parallel — file conflicts).

### Step A: Start Task

Mark task `in_progress` via `TaskUpdate`. Record `BASE_SHA`:

```bash
cd {WORKTREE_PATH}
BASE_SHA=$(git rev-parse HEAD)
```

### Step B: Dispatch Implementer

Dispatch as a **teammate** (Task tool with `team_name: "{TEAM_NAME}"`, `name: "implementer-{task_number}"`, `subagent_type: "general-purpose"`):

Fill `{IMPLEMENTER_PROMPT}` with:
- Task number, name, text, context
- `{WORKING_DIRECTORY}` = `{WORKTREE_PATH}`
- `{BASE_SHA}` = recorded above
- `{CODEX_REVIEWER_NAME}` = `"codex-reviewer"` (or empty if `codex_available = false`)
- `{LEADER_NAME}` = your own team name
- `{CODEX_STATUS}` = current `codex_available` flag
- `{SELF_REVIEW_PROMPT}` = `{SELF_REVIEW_PROMPT}` template

### Step C: Wait for Implementer Verdict

Wait for `## Task Verdict` from the Implementer via SendMessage.

Parse the message:
- `verdict` — `pass` or `fail`
- `head_sha` — current HEAD after all review phases
- `implementation_summary`
- `self_review` — rounds, findings fixed
- `codex_review` — status, rounds, thread_id, findings fixed, unresolved
- `spec_compliance` — pass/fail, round count
- `code_quality` — pass/fail, round count
- `tests` — pass/fail count
- `concerns`

**If Codex status changed to unavailable:** Update `codex_available = false`.

### Step D: Process Verdict

**If `verdict: pass`:**
- Mark task `completed`
- Record audit (see Step F)
- Send `shutdown_request` to Implementer
- Proceed to next task

**If `verdict: fail`:**
- Check severity of unresolved issues
- **Critical/Important unresolved:** Mark task `failed`, report to user
- **Minor only:** Proceed with flags — append to `docs/unresolved-flags.md`, commit

**If Codex Reviewer sends `## Codex Status Update` (at any time):**
- Update `codex_available = false`
- No action needed — Implementer already handles Codex unavailability internally

### Step E: Cleanup After Task

Send `shutdown_request` to Implementer (`implementer-{task_number}`).

Wait for shutdown confirmation before proceeding to next task.

### Step F: Audit Record

After each task completes (pass or fail), write to `TaskUpdate` metadata:

```json
{
  "task_id": "N",
  "implementation": {
    "commit_sha": "abc1234",
    "tests": "12 passed, 0 failed"
  },
  "codex_review": {
    "status": "available",
    "rounds": 1,
    "thread_id": "sess_abc",
    "findings_fixed": 2,
    "unresolved": []
  },
  "spec_compliance": {
    "verdict": "pass",
    "rounds": 1
  },
  "code_quality": {
    "verdict": "pass",
    "rounds": 1
  },
  "final_verdict": "pass",
  "head_sha": "def5678"
}
```

## After All Tasks

1. **Final branch review:** Dispatch spec compliance + code quality subagents and use the persistent Codex Reviewer for a full branch review.

2. Compute merge-base:
   ```bash
   MERGE_BASE=$(git merge-base {BASE_BRANCH} HEAD)
   ```

3. Dispatch **spec compliance subagent** (Task tool, `subagent_type: "general-purpose"`, `model: "haiku"`):
   ```
   You are reviewing whether the full branch implementation matches its specification.

   ## What Was Requested

   [full plan text + design doc reference]

   ## What Was Implemented

   [summary of all completed tasks]

   ## Your Job

   Read the diff and verify all requirements are met, nothing is missing, nothing extra:

   ```bash
   cd {WORKTREE_PATH}
   git diff {MERGE_BASE}..HEAD
   ```

   Report: PASS or FAIL with specific issues.
   ```

4. Dispatch **code quality subagent** (Task tool, `subagent_type: "superpowers:code-reviewer"`, `model: "sonnet"`):
   - Full branch diff: `{MERGE_BASE}..HEAD`
   - Working directory: `{WORKTREE_PATH}`
   - Plan + design doc reference as context

5. Send `## Codex Review Request` to persistent Codex Reviewer:
   ```
   ## Codex Review Request
   commit_range: {MERGE_BASE}..HEAD
   task_summary: Final branch review for [plan name]
   context: Full implementation of all tasks. Design doc: {DESIGN_DOC_PATH}
   thread_id: new
   ```

6. Wait for all three reviews.
   - If spec compliance finds issues, report to user (no Implementer to fix).
   - If code quality finds issues, report to user.
   - If Codex finds issues, include in final report.

7. **If final review passes:** Report success to main session.
8. **If final review fails:** Report failures to main session with details.

## Cleanup

1. Send `shutdown_request` to persistent Codex Reviewer
2. Send `shutdown_request` to any remaining teammates
3. Wait for shutdown confirmations

> TeamDelete is handled by the SKILL.md entry point after these instructions complete.

## Report Format

Return this to the main session:

```
## Team-Driven Development Complete

**Verdict:** [all passed | N failed | escalated]
**Tasks:** [completed/total]
**Codex:** [available | unavailable since task N]

### Per-Task Summary
| Task | Rounds | Verdict |
|------|--------|---------|
| Task 1: [name] | 1 | passed |
| Task 2: [name] | 3 | passed |

### Unresolved Flags
[any flags from docs/unresolved-flags.md, or "none"]
```

## Rules

- **Never** dispatch multiple Implementers in parallel
- **Never** transition task state from any agent except Leader
- **Never** proceed with unresolved Critical/Important issues after cap — escalate
- **Never** retry Codex automatically after unavailability — user must request it
- If stagnation detected (same issues across 3+ tasks), escalate immediately
