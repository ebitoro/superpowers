# team-driven-development Design

**Date:** 2026-02-21
**Status:** Approved
**Codex Thread:** 019c7f4b-c269-7231-9d83-52e7a949c08c

## Problem

The current `subagent-driven-development` skill orchestrates everything from the main CC session. Every implementer report, spec review, code quality review, and Codex verdict flows back into main session context. On plans with 10+ tasks, this causes repeated compaction and context loss.

## Solution

A new skill using Claude Code's AgentTeam feature. A **Leader agent** (not the main session) orchestrates the plan. The main session only sees the final result. Review is handled by agent-to-agent communication, keeping both the main session and the leader's context minimal.

## Architecture

### Team Structure

One team per plan execution. `TeamCreate` once, `TeamDelete` at the end.

| Agent | Lifecycle | Type | Role |
|-------|-----------|------|------|
| **Leader** | Persistent (entire plan) | Inherited model | Orchestrator. Dispatches agents, reads verdicts, manages task state transitions. |
| **Implementer** | Fresh per task | general-purpose | Implements one task. TDD, self-review, commit. Reports summary to Leader. |
| **CC Reviewer** | Fresh per task | code-reviewer | Review orchestrator. Checks spec+quality. Commands Codex. Verifies findings. Consolidates verdict. |
| **Codex Reviewer** | Fresh per task | codex-agent | Codex intermediary. Receives structured review orders from CC. Sends verified findings back. Answers follow-ups. |
| **Fix Agent** | Fresh per fix round | general-purpose | Fixes specific issues. Contacts CC Reviewer directly for re-review. |

### Per-Task Flow

```
Leader (codex_available?)
  │
  ├─1→ Dispatch Implementer (Task tool, general-purpose)
  │      Implement, test, commit, self-review
  │      Implementer → Leader: summary (what, tests, commit SHA, concerns)
  │
  ├─2→ Dispatch reviewers:
  │      ├─ codex_available=true  → CC Reviewer + Codex Reviewer (as teammates)
  │      └─ codex_available=false → CC Reviewer only (with codex_status: unavailable)
  │      CC Reviewer orchestrates (see Review Phase below)
  │      CC → Leader: verdict (pass/fail + one-line per issue + severity)
  │
  ├─3→ [If fail] Dispatch Fix Agent (Task tool, general-purpose)
  │      Fix, commit → Fix Agent contacts CC Reviewer directly
  │      CC re-reviews → CC → Leader: new verdict
  │      (Outer loop: hard cap 10 rounds + stagnation detection)
  │
  └─4→ [If pass] Mark task complete, shut down CC + Codex reviewers, next task
```

### Task State Machine

Each task transitions through states. **Leader has exclusive transition authority.**

```
pending → implementing → reviewing → fixing → passed
                                  ↗        ↘
                            (re-review)   escalated → failed
```

## Review Phase Detail

CC Reviewer is the **review orchestrator**. It performs three phases sequentially.

### Phase 1: Independent Review

CC does its own spec compliance + code quality check. Finalizes its own verdict BEFORE seeing Codex findings. CC must NOT revise its own findings based on Codex output.

### Phase 2: Codex Verification

CC receives Codex findings, verifies each against actual code (accept/reject). May ask Codex follow-up questions (max 5 inner rounds). Skipped entirely when `codex_status: unavailable`.

### Phase 3: Consolidation

CC merges own findings + verified Codex findings → sends consolidated verdict to Leader.

## Message Protocols

### Implementer → Leader

```
Task N complete.
- Implemented: [one-line summary]
- Tests: [pass/fail count]
- Commit: [SHA]
- Concerns: [any, or "none"]
```

### CC Reviewer → Codex Reviewer (Review Order)

Structured envelope that maps to codex-agent fields:

```
## Review Order
mode: review-gate
thread_id: [new | saved-thread-id]
commit_range: [BASE_SHA]..[HEAD_SHA]
task_summary: [one-line]
context: [what this task does in the broader feature]
worktree_path: [path]
```

Codex Reviewer internally translates this into `codex` MCP calls. Returns standard codex-agent report format.

**Thread lifecycle within a task:**
- First review: `thread_id: new` — Codex creates fresh thread
- Fix-loop re-reviews: reuse the same thread_id (passed via CC → Codex messages)
- Thread_id stored in task metadata for auditability

### Codex Reviewer → CC Reviewer (Findings)

Standard codex-agent report format per `skills/codex-agent/SKILL.md`.

### CC Reviewer → Leader (Verdict)

```
verdict: [pass | fail]
issues: [count]
  [For each: severity | one-line | file:line]
codex: [N verified, M dismissed | "unavailable — CC-only review"]
round: [current round number]
```

### Fix Agent → CC Reviewer (Re-review Request)

```
## Fix Report
fixed_count: [N]
commits: [SHA1, SHA2]
addressed:
  1. [issue ID] — [what was done]
  2. [issue ID] — [what was done]
tests: [pass/fail count]
```

## Fix Loop

### Outer Loop (Leader → Fix Agent → CC Reviewer → Leader)

- Hard cap at 10 rounds
- **Stagnation detection:** Same finding appearing in 2+ consecutive rounds without resolution = escalate to user via Leader
- On escalation: Leader reports stuck issues to main session, user decides (fix manually, skip, or abandon)

### Inner Loop (CC ↔ Codex Q&A)

- Max 5 rounds per review cycle
- After 5 rounds, CC proceeds with whatever information it has

### Severity-Based Exit

- **Critical/Important unresolved after cap:** Task fails, escalate to user
- **Minor unresolved:** Proceed with flags (append to `docs/unresolved-flags.md`, commit)

### SHA Management for Re-reviews

- `BASE_SHA` = original task's base commit (never changes)
- `HEAD_SHA` = latest fix commit (recomputed each round)
- Full task diff always reviewed, prevents regression blindness

## Codex Unavailability

### Detection

If Codex Reviewer reports `status: unavailable` during any task's review:

1. CC Reviewer proceeds with CC-only verdict for that task
2. CC Reviewer marks verdict as `degraded`
3. CC Reviewer reports to Leader: `codex: unavailable — [reason]`

### Leader Remembers

Leader sets an internal flag `codex_available: false`. For all subsequent tasks:

- Leader dispatches CC Reviewer **only** (no Codex Reviewer)
- Leader includes in CC Reviewer's dispatch: `codex_status: unavailable`
- CC Reviewer skips all Codex-related steps entirely

**Result:** One failed Codex dispatch per plan (max), not one per task.

### Recovery

No automatic retry. If the user wants to re-check Codex availability mid-plan, they can instruct the Leader to retry once. If that dispatch succeeds, Leader clears the flag and resumes normal CC+Codex flow.

## After All Tasks

1. Leader dispatches **fresh** CC Reviewer + **fresh** Codex Reviewer for final branch review (respects `codex_available` flag)
2. Full branch diff (`main...HEAD`)
3. Design doc reference from `.codex-state/current_design_doc`
4. Same CC-as-orchestrator pattern
5. Leader reports final result to main session
6. Then: `superpowers:finishing-a-development-branch`

## Per-Task Audit Record

Stored in task metadata (`TaskUpdate` metadata field):

```json
{
  "task_id": "N",
  "rounds": [
    {
      "round": 1,
      "verdict": "fail",
      "issues": [
        {
          "id": "ISS-1",
          "severity": "important",
          "file": "auth.js",
          "line": 42,
          "description": "missing validation",
          "disposition": "open"
        }
      ],
      "codex_thread_id": "sess_abc",
      "head_sha": "abc1234",
      "codex_status": "available"
    }
  ],
  "final_verdict": "pass",
  "total_rounds": 2
}
```

## State Management

Same breadcrumb pattern as `subagent-driven-development`, resolved via `git-common-dir`:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.codex-state"
mkdir -p "$STATE_DIR"
grep -q '.codex-state/' "$MAIN_REPO/.gitignore" 2>/dev/null || echo '.codex-state/' >> "$MAIN_REPO/.gitignore"
```

**Files:**
- `current_plan` — path to plan file
- `current_worktree` — worktree path
- `current_design_doc` — design doc path

## Context Budget Summary

| Agent | Receives | Context Load |
|-------|----------|-------------|
| **Main session** | Final plan result only | Minimal |
| **Leader** | Implementer summaries + CC verdicts (structured) | Low (grows slowly) |
| **Implementer** | Full task text + context | Medium (dies after task) |
| **CC Reviewer** | Code diffs + Codex findings | Medium (dies after task) |
| **Codex Reviewer** | Review orders + follow-ups | Medium (dies after task) |
| **Fix Agent** | Issue list + file context | Low (dies after fixing) |

## Comparison: subagent-driven-development vs. team-driven-development

| Aspect | subagent-driven-dev | team-driven-dev |
|--------|--------------------|--------------------|
| Main session context | High (all results flow back) | Minimal (final result only) |
| Leader context | N/A (main session IS leader) | Low (structured verdicts only) |
| Review stages | 3 sequential (spec, quality, Codex) | 2 parallel (merged CC + Codex) |
| Review Q&A | None (one-shot Codex) | Up to 5 rounds CC↔Codex |
| Fix communication | Through main session | Fix Agent → CC Reviewer direct |
| Codex unavailability | Check every task | Detect once, skip rest |
| Compaction risk | High on 10+ tasks | Low (fresh agents, lean leader) |
| Outer fix loop cap | Inherited from Codex (5) | 10 + stagnation detection |

## Integration

- **Reads plan from** `.codex-state/current_plan` (same as subagent-driven-development)
- **Verifies worktree** before starting (same pattern, dispatches `worktree-setup` agent if needed)
- **Final review:** Team-internal (CC-as-orchestrator pattern with full branch diff)
- **Finishing:** `superpowers:finishing-a-development-branch`
- **Implementer agents** should follow `superpowers:test-driven-development`

## Red Flags

**Never:**
- Dispatch multiple implementers in parallel (file conflicts)
- Let CC Reviewer revise its own findings based on Codex output
- Skip the independent CC review phase (Phase 1 must complete before Phase 2)
- Allow any agent except Leader to transition task state
- Proceed with unresolved Critical/Important issues after cap

**If Codex Reviewer reports unavailable:**
- Leader flags it, stops dispatching Codex for remaining tasks
- CC Reviewer proceeds CC-only for all subsequent tasks

**If stagnation detected:**
- Escalate to user immediately, do not continue looping
