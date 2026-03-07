# team-driven-development Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (same session) or superpowers:executing-plans (separate session) to implement this plan task-by-task.

**Goal:** Create a new `team-driven-development` skill that uses Claude Code's AgentTeam feature to execute implementation plans with minimal main session context usage.

**Architecture:** Main session dispatches a Leader subagent via Task tool. Leader calls TeamCreate, orchestrates the entire plan by dispatching fresh agents per task, managing review cycles via agent-to-agent SendMessage, and reporting the final result back. Main session only sees the final verdict.

**Tech Stack:** Claude Code AgentTeam API (TeamCreate, SendMessage, TaskCreate/TaskUpdate/TaskList, TeamDelete), Codex MCP, Git

**Design doc:** `docs/plans/2026-02-21-team-driven-development-design.md`

---

### Task 1: Create SKILL.md entry point

The main skill file that the CC session reads when the user invokes `team-driven-development`. It instructs the main session to recover context, read prompt templates, dispatch the Leader, and wait.

**Files:**
- Create: `skills/team-driven-development/SKILL.md`

**Step 1: Create the skill file**

```markdown
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

> See `lib/codex-integration.md` for Codex patterns. All Codex interactions go through the codex-agent (`skills/codex-agent/SKILL.md`), dispatched as a team member by the Leader.

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
```

**Step 2: Verify the file has correct frontmatter**

Run: `head -4 skills/team-driven-development/SKILL.md`
Expected: YAML frontmatter with `name: team-driven-development` and `description`

**Step 3: Commit**

```bash
git add skills/team-driven-development/SKILL.md
git commit -m "feat(team-driven-dev): add SKILL.md entry point"
```

---

### Task 2: Create leader-prompt.md

The Leader agent template. The main session fills in placeholders and passes the result as the Task prompt. The Leader creates the team, orchestrates all tasks, manages review cycles, and reports back.

**Files:**
- Create: `skills/team-driven-development/leader-prompt.md`

**Step 1: Create the leader prompt template**

The file should contain the full Leader instructions as a markdown prompt template with these sections:

1. **Role statement:** "You are the Leader agent for team-driven development. You orchestrate plan execution by dispatching agents and managing review cycles."

2. **Inputs section** with placeholders: `{PLAN_TASKS}`, `{WORKTREE_PATH}`, `{DESIGN_DOC_PATH}`, `{CC_REVIEWER_PROMPT}`, `{IMPLEMENTER_PROMPT}`, `{FIX_AGENT_PROMPT}`

3. **Setup instructions:**
   - Call `TeamCreate` with team name `tdd-{timestamp}` (tdd = team-driven-dev)
   - Parse plan tasks from `{PLAN_TASKS}`, create a `TaskCreate` for each
   - Set internal flag `codex_available = true`
   - Determine `BASE_BRANCH` (the branch this worktree was created from, e.g., `main` or `codex-reviewer`). Use `git log --oneline --merges -1` or read from `.codex-state/` if available. Record for final review diff.

4. **Per-task workflow** (iterate through tasks in order):
   a. Mark task `in_progress` via `TaskUpdate`
   b. Dispatch Implementer via Task tool (NOT a teammate — regular subagent):
      - Use `{IMPLEMENTER_PROMPT}` template
      - Fill in task text, context, working directory
      - Wait for Task result
   c. Parse Implementer's summary (commit SHA, test results, concerns)
   d. If Implementer has concerns or questions, answer them and redispatch
   e. Dispatch reviewers based on `codex_available`:
      - If `true`: Dispatch CC Reviewer AND Codex Reviewer as teammates (`Task` with `team_name` and `name`)
        - CC Reviewer: name `cc-reviewer`, use `{CC_REVIEWER_PROMPT}` filled with commit SHAs, task spec, worktree path
        - Codex Reviewer: name `codex-reviewer`, subagent_type `superpowers:codex-agent`, prompt includes team instructions: "You are a Codex intermediary in a team. You will receive review orders from CC Reviewer via SendMessage. Parse the structured envelope (mode, thread_id, commit_range, task_summary, context, worktree_path) and execute as codex-agent. Send your findings back to CC Reviewer via SendMessage."
      - If `false`: Dispatch CC Reviewer only, include `codex_status: unavailable` in prompt
   f. Wait for CC Reviewer's verdict (delivered via SendMessage)
   g. If verdict contains `codex: unavailable`:
      - Set `codex_available = false` for all subsequent tasks
   h. **If pass:** Mark task `completed`, send `shutdown_request` to CC + Codex reviewers, move to next task
   i. **If fail — enter fix loop** (hard cap 10 outer rounds):
      - Dispatch Fix Agent as teammate (`Task` with `team_name`, name `fix-agent-{round}`)
      - Fix Agent receives: issue list, commit SHAs, worktree path, CC Reviewer's name
      - Fix Agent fixes issues, commits, contacts CC Reviewer directly via SendMessage
      - CC Reviewer re-reviews, sends new verdict to Leader via SendMessage
      - **Stagnation detection:** Track issue IDs per round. If same issue appears in 2+ consecutive rounds without resolution, escalate to user
      - On cap or stagnation: If Critical/Important unresolved, mark task `failed` and report to user. If only Minor, proceed with flags (append to `docs/unresolved-flags.md`, commit)
      - On pass: Mark task `completed`, shut down agents

5. **After all tasks:**
   - Dispatch fresh CC Reviewer + fresh Codex Reviewer for final branch review
   - Review full branch diff: `git diff {BASE_BRANCH}...HEAD`
   - Design doc reference from `{DESIGN_DOC_PATH}`
   - Same CC-as-orchestrator pattern
   - If final review passes: report success to main session
   - If final review fails: enter fix loop (same pattern)

6. **Cleanup:**
   - Send `shutdown_request` to all remaining teammates
   - Call `TeamDelete`

7. **Report format to main session:**
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

8. **Per-task audit record:**
   - After each task completes (pass or fail), write an audit record to `TaskUpdate` metadata:
     ```json
     {
       "task_id": "N",
       "rounds": [
         {"round": 1, "verdict": "fail", "issues": [{"id": "ISS-1", "severity": "important", "file": "path", "line": 42, "description": "...", "disposition": "open"}], "codex_thread_id": "sess_abc", "head_sha": "abc1234", "codex_status": "available"}
       ],
       "final_verdict": "pass",
       "total_rounds": 2
     }
     ```
   - Update the `rounds` array after each fix-loop iteration
   - Track issue IDs across rounds for stagnation detection (same ID in 2+ consecutive rounds = stagnation)

9. **Message protocol rules:**
   - Only accept structured verdicts from CC Reviewer (see CC Reviewer → Leader verdict format in design doc)
   - Never read full review content — only verdict + one-line issue summaries
   - Echo `**Active Codex thread_id:** <id>` when receiving Codex thread IDs (compaction safety)

**Step 2: Verify the file structure**

Run: `wc -l skills/team-driven-development/leader-prompt.md`
Expected: Under 300 lines

**Step 3: Commit**

```bash
git add skills/team-driven-development/leader-prompt.md
git commit -m "feat(team-driven-dev): add leader-prompt.md template"
```

---

### Task 3: Create cc-reviewer-prompt.md

The CC Reviewer template. Dispatched as a teammate by the Leader. Orchestrates the review phase: independent spec+quality review, Codex verification, consolidation. Communicates with Codex Reviewer and Fix Agent via SendMessage.

**Files:**
- Create: `skills/team-driven-development/cc-reviewer-prompt.md`

**Step 1: Create the CC Reviewer prompt template**

The file should contain the full CC Reviewer instructions with these sections:

1. **Role statement:** "You are the CC Reviewer in a team-driven development workflow. You review code for spec compliance and code quality, orchestrate Codex review, verify Codex findings, and report consolidated verdicts."

2. **Inputs** (filled by Leader at dispatch time):
   - `{TASK_SPEC}` — Full task requirements text
   - `{WHAT_WAS_IMPLEMENTED}` — Implementer's report summary
   - `{BASE_SHA}` — Base commit SHA
   - `{HEAD_SHA}` — Head commit SHA
   - `{WORKTREE_PATH}` — Worktree absolute path
   - `{CODEX_STATUS}` — "available" or "unavailable"
   - `{LEADER_NAME}` — Leader's teammate name (for SendMessage)
   - `{CODEX_REVIEWER_NAME}` — Codex Reviewer's teammate name (for SendMessage), empty if unavailable

3. **Phase 1 — Independent Review:**
   - Read the diff: `git diff {BASE_SHA}..{HEAD_SHA}`
   - Check spec compliance: Does implementation match `{TASK_SPEC}`? Missing requirements? Extra work?
   - Check code quality: Clean code, error handling, DRY, edge cases, test quality
   - **Finalize your own verdict BEFORE Phase 2. Do NOT revise your findings based on Codex output.**
   - Record own findings with severity (Critical/Important/Minor), file:line, description

4. **Phase 2 — Codex Verification** (skip if `{CODEX_STATUS}` is "unavailable"):
   - Send review order to `{CODEX_REVIEWER_NAME}` via SendMessage:
     ```
     ## Review Order
     mode: review-gate
     thread_id: new
     commit_range: {BASE_SHA}..{HEAD_SHA}
     task_summary: [one-line from TASK_SPEC]
     context: [what this task does]
     worktree_path: {WORKTREE_PATH}
     ```
   - Wait for Codex Reviewer's findings via SendMessage
   - For each Codex finding: read the actual code, verify it exists as described
     - **Verified:** Include in consolidated verdict
     - **False positive:** Dismiss, note count
     - **Downgraded:** Include with corrected severity
   - If questions about a Codex finding: SendMessage follow-up to Codex Reviewer (max 5 inner rounds total)
   - Save the `thread_id` from Codex Reviewer's response for re-reviews

5. **Phase 3 — Consolidation:**
   - Merge own findings + verified Codex findings (deduplicate)
   - Send verdict to `{LEADER_NAME}` via SendMessage using this format:
     ```
     verdict: [pass | fail]
     issues: [count]
       [For each: severity | one-line | file:line]
     codex: [N verified, M dismissed | "unavailable — CC-only review"]
     round: [current round number]
     ```

6. **Re-review handling** (when Fix Agent contacts you):
   - Receive fix report via SendMessage from Fix Agent
   - Parse: which issues were addressed, new commit SHAs
   - Re-read diff at new HEAD_SHA (using original BASE_SHA)
   - Repeat Phase 1 (re-review changed code)
   - If `{CODEX_STATUS}` is available: re-engage Codex Reviewer with saved thread_id
   - Send new verdict to Leader

7. **Rules:**
   - Never trust Implementer or Fix Agent reports blindly — read the actual code
   - Never revise own Phase 1 findings based on Codex (Phase 2) output
   - Only accept/reject/downgrade Codex findings — don't add new findings based on Codex suggestions
   - If Codex Reviewer reports unavailable mid-review: proceed CC-only, include `codex: unavailable` in verdict

**Step 2: Verify the file structure**

Run: `wc -l skills/team-driven-development/cc-reviewer-prompt.md`
Expected: Under 300 lines

**Step 3: Commit**

```bash
git add skills/team-driven-development/cc-reviewer-prompt.md
git commit -m "feat(team-driven-dev): add cc-reviewer-prompt.md template"
```

---

### Task 4: Create implementer-prompt.md

The Implementer template. Dispatched as a regular subagent (NOT a teammate) by the Leader. Implements one task following TDD, self-reviews, commits, and reports back via Task return.

**Files:**
- Create: `skills/team-driven-development/implementer-prompt.md`

**Step 1: Create the Implementer prompt template**

Adapt from `skills/subagent-driven-development/implementer-prompt.md`. The structure is identical — the only difference is that this implementer reports back via Task return (not SendMessage, since it's not a teammate).

The file should contain a dispatch template with:

1. **Header:** "Use this template when dispatching an implementer subagent via the Task tool (NOT as a teammate)."

2. **Template block** with placeholders:
   - `{TASK_NUMBER}` — Task number
   - `{TASK_NAME}` — Task name
   - `{TASK_TEXT}` — Full task text from plan
   - `{CONTEXT}` — Scene-setting: where this fits, dependencies, architectural context
   - `{WORKING_DIRECTORY}` — Worktree path

3. **Instructions for the implementer:**
   - Before beginning: ask questions if requirements are unclear
   - Implement exactly what the task specifies
   - Write tests (follow TDD if task says to)
   - Verify implementation works
   - Commit work with conventional commit format
   - Self-review: completeness, quality, YAGNI, testing

4. **Report format** (this goes back to Leader via Task return):
   ```
   Task {N} complete.
   - Implemented: [one-line summary]
   - Tests: [pass/fail count]
   - Commit: [SHA]
   - Concerns: [any, or "none"]
   ```

**Step 2: Verify the file structure**

Run: `wc -l skills/team-driven-development/implementer-prompt.md`
Expected: Under 100 lines (this is a simple template)

**Step 3: Commit**

```bash
git add skills/team-driven-development/implementer-prompt.md
git commit -m "feat(team-driven-dev): add implementer-prompt.md template"
```

---

### Task 5: Create fix-agent-prompt.md

The Fix Agent template. Dispatched as a teammate by the Leader. Fixes specific issues from the CC Reviewer's verdict, commits, and contacts the CC Reviewer directly for re-review via SendMessage.

**Files:**
- Create: `skills/team-driven-development/fix-agent-prompt.md`

**Step 1: Create the Fix Agent prompt template**

The file should contain:

1. **Header:** "Use this template when dispatching a Fix Agent as a teammate."

2. **Template block** with placeholders:
   - `{ISSUES}` — List of issues to fix (from CC Reviewer's verdict: severity, description, file:line)
   - `{WORKING_DIRECTORY}` — Worktree path
   - `{CC_REVIEWER_NAME}` — CC Reviewer's teammate name (for SendMessage)
   - `{ROUND}` — Current outer loop round number
   - `{BASE_SHA}` — Original task base commit (does not change)

3. **Instructions:**
   - Fix each issue listed in `{ISSUES}`
   - For each fix: read the code at the referenced file:line, understand the issue, implement the fix
   - Run tests after each fix to ensure no regressions
   - Commit all fixes with descriptive message: `fix: address review issues (round {ROUND})`
   - Do NOT add features, refactor unrelated code, or make improvements beyond the issue list

4. **After fixing, contact CC Reviewer directly via SendMessage:**
   ```
   SendMessage:
     type: message
     recipient: {CC_REVIEWER_NAME}
     summary: "Fixed {N} issues, requesting re-review"
     content: |
       ## Fix Report
       fixed_count: [N]
       commits: [SHA1, SHA2]
       addressed:
         1. [issue ID] — [what was done]
         2. [issue ID] — [what was done]
       tests: [pass/fail count]
   ```

5. **After sending the fix report:** Go idle. CC Reviewer will re-review and report the new verdict to the Leader. Your job is done.

6. **Rules:**
   - Fix ONLY the issues listed — nothing else
   - Always run tests before committing
   - Always contact CC Reviewer via SendMessage (not Leader)
   - If you cannot fix an issue, include it in the fix report with "unable to fix: [reason]"

**Step 2: Verify the file structure**

Run: `wc -l skills/team-driven-development/fix-agent-prompt.md`
Expected: Under 100 lines

**Step 3: Commit**

```bash
git add skills/team-driven-development/fix-agent-prompt.md
git commit -m "feat(team-driven-dev): add fix-agent-prompt.md template"
```

---

### Task 6: Update CLAUDE.md

Add `team-driven-development` to the development lifecycle and document its relationship to the existing skills.

**Files:**
- Modify: `CLAUDE.md:19-24` (Development Lifecycle section)

**Step 1: Update the pipeline diagram**

In `CLAUDE.md`, find the Development Lifecycle section (around line 19-24). Change the pipeline from:

```
brainstorming -> writing-plans -> [executing-plans | subagent-driven-development] -> finishing-a-development-branch
```

to:

```
brainstorming -> writing-plans -> [executing-plans | subagent-driven-development | team-driven-development] -> finishing-a-development-branch
```

**Step 2: Add the skill description**

After the `subagent-driven-development` bullet point (around line 29), add:

```markdown
- **team-driven-development**: Leader agent orchestrates via AgentTeam. Fresh implementer per task, CC Reviewer + Codex Reviewer communicate directly. Main session only sees final result. Preferred for plans with 5+ tasks to minimize context usage.
```

**Step 3: Verify the changes**

Run: `grep -n 'team-driven-development' CLAUDE.md`
Expected: At least 2 matches (pipeline diagram and description)

**Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add team-driven-development to development lifecycle"
```

---

### Task 7: Create skill trigger test

Add a natural-language prompt test that verifies the skill is triggered when a user asks to execute a plan with a team or minimal context usage.

**Files:**
- Create: `tests/skill-triggering/prompts/team-driven-development.txt`

**Step 1: Create the test prompt**

```
I have an implementation plan with 12 tasks and I want to execute it using a team of agents to minimize context window usage. The plan is at docs/plans/my-plan.md.
```

**Step 2: Verify the test file exists and matches format**

Run: `cat tests/skill-triggering/prompts/team-driven-development.txt`
Expected: Single natural-language prompt that implies team-based execution

**Step 3: Commit**

```bash
git add tests/skill-triggering/prompts/team-driven-development.txt
git commit -m "test: add skill trigger test for team-driven-development"
```

---

## Task Dependencies

```
Task 1 (SKILL.md) — no dependencies, sets up skill skeleton
Task 2 (leader-prompt.md) — depends on Task 1 (references SKILL.md structure)
Task 3 (cc-reviewer-prompt.md) — depends on Task 2 (Leader references this)
Task 4 (implementer-prompt.md) — depends on Task 2 (Leader references this)
Task 5 (fix-agent-prompt.md) — depends on Task 3 (Fix Agent contacts CC Reviewer)
Task 6 (CLAUDE.md update) — depends on Task 1 (skill must exist)
Task 7 (trigger test) — depends on Task 1 (skill must exist to be triggered)
```

Tasks 3, 4, and 7 can be done in parallel after Task 2 is complete.
Tasks 5 and 6 can be done in parallel after Task 3 is complete.
