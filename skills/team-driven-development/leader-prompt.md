# Leader Instructions — Team-Driven Development

These are the orchestration instructions for team-driven development. The team has already been created by the SKILL.md entry point — you operate within it.

**Model note:** This prompt works with any model, including Sonnet.

## Inputs

- **Team name:** {TEAM_NAME} (already created — use for all teammate dispatches)
- **Plan tasks:** {PLAN_TASKS}
- **Worktree path:** {WORKTREE_PATH}
- **Design doc:** {DESIGN_DOC_PATH}
- **Work model:** {WORK_MODEL} (model for implementers, review subagents, fixer — default "opus")

### Prompt Templates

Read these from `skills/team-driven-development/`:
- `implementer-prompt.md` — Implementer teammate dispatch prompt
- `codex-reviewer-prompt.md` — Codex Reviewer teammate dispatch prompt
- `final-reviewer-prompt.md` — Final Reviewer teammate dispatch prompt

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

Dispatch as a **teammate** (Task tool with `team_name: "{TEAM_NAME}"`, `name: "implementer-{task_number}"`, `subagent_type: "general-purpose"`, `model: "{WORK_MODEL}"`):

Fill `{IMPLEMENTER_PROMPT}` with:
- Task number, name, text, context
- `{WORKING_DIRECTORY}` = `{WORKTREE_PATH}`
- `{BASE_SHA}` = recorded above
- `{CODEX_REVIEWER_NAME}` = `"codex-reviewer"` (or empty if `codex_available = false`)
- `{LEADER_NAME}` = your own team name
- `{CODEX_STATUS}` = current `codex_available` flag
- `{WORK_MODEL}` = `"{WORK_MODEL}"`

### Step C: Wait for Implementer Verdict

Wait for `## Task Verdict` from the Implementer via SendMessage.

Parse the message:
- `verdict` — `pass` or `fail`
- `head_sha` — current HEAD after all review phases
- `implementation_summary`
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
- **Critical/Important unresolved:** Mark task `failed`
  - If the failed task does NOT block subsequent tasks: continue with the next task
  - If the failed task blocks downstream tasks: pause and escalate to the user
- **Minor only:** Proceed with flags — append to `docs/unresolved-flags.md`, commit

**If Codex Reviewer sends `## Codex Status Update` (at any time):**
- Update `codex_available = false`
- No action needed — Implementer already handles Codex unavailability internally

### Step E: Cleanup After Task

Send `shutdown_request` to Implementer (`implementer-{task_number}`).

Wait for shutdown confirmation before proceeding to next task.

### Step E.1: Implementer Timeout Recovery

If an Implementer goes idle without sending a `## Task Verdict`:

1. Check for commits since `BASE_SHA`:
   ```bash
   cd {WORKTREE_PATH}
   git log --oneline {BASE_SHA}..HEAD
   ```

2. **Commits exist:** Send `shutdown_request` to the idle Implementer. Dispatch a new Implementer with the same task and add to the prompt: "Review existing commits since {BASE_SHA} before continuing implementation."

3. **No commits:** Send `shutdown_request` to the idle Implementer. Dispatch a fresh retry with the original task prompt.

4. **Max 1 retry per task.** If the retry also fails, mark the task as failed and escalate to the user.

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

### Step 1: Compute Merge Base

```bash
cd {WORKTREE_PATH}
MERGE_BASE=$(git merge-base {BASE_BRANCH} HEAD)
HEAD_SHA=$(git rev-parse HEAD)
```

### Step 2: Dispatch Final Reviewer

Dispatch a persistent `final-reviewer` teammate (Task tool with `team_name: "{TEAM_NAME}"`, `name: "final-reviewer"`, `subagent_type: "general-purpose"`, `model: "opus"`).

Fill `final-reviewer-prompt.md` with:
- `{WORKTREE_PATH}`, `{TEAM_NAME}`, `{BASE_BRANCH}`, `{DESIGN_DOC_PATH}`
- `{PLAN_TASKS}` — full plan text
- `{COMPLETED_TASKS_SUMMARY}` — summary of all completed tasks and their verdicts
- `{CODEX_REVIEWER_NAME}` = `"codex-reviewer"` (or empty if `codex_available = false`)
- `{LEADER_NAME}` = your own team name
- `{WORK_MODEL}` = `"{WORK_MODEL}"`

### Step 3: Send Initial Review Request

Send to `final-reviewer` via SendMessage:

```
## Review Request
type: initial
merge_base: {MERGE_BASE}
head_sha: {HEAD_SHA}
```

### Step 4: Process Final Review Verdict

Wait for `## Final Review Verdict` from the Final Reviewer.

**If `overall_verdict: pass`:** Proceed to Cleanup.

**If `overall_verdict: fail`:**

1. Dispatch a `fixer` teammate (Task tool with `team_name: "{TEAM_NAME}"`, `name: "fixer"`, `subagent_type: "general-purpose"`, `model: "{WORK_MODEL}"`). Reuse the same fixer across rounds.
2. Send the specific issues to the fixer with instructions to fix, test, and commit.
3. Wait for the fixer to report `FIXED: [head_sha]`.
4. Send a re-review request to `final-reviewer`:
   ```
   ## Review Request
   type: re-review
   merge_base: {MERGE_BASE}
   head_sha: [updated HEAD from fixer]
   ```
5. Wait for the updated `## Final Review Verdict`.
6. Repeat until pass or **max 5 rounds** — then proceed with flags and report unresolved issues to the main session.

### Step 5: Report to Main Session

**If final review passes:** Report success.
**If final review fails after 5 rounds:** Report failures with details of unresolved issues.

## Cleanup

1. Send `shutdown_request` to `final-reviewer`
2. Send `shutdown_request` to `fixer` (if dispatched)
3. Send `shutdown_request` to persistent Codex Reviewer
4. Send `shutdown_request` to any remaining teammates
5. Wait for shutdown confirmations

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
- If 3+ consecutive tasks produce the same Codex finding category (e.g., repeated naming issues, repeated missing error handling), escalate to user as a systemic issue rather than fixing per-task
