# Codex Integration Reference

Shared patterns for skills that integrate with Codex as a reviewer/thought partner.

## Codex Agent (Preferred Pattern)

**All Codex interactions should go through the `codex-agent` subagent** (`agents/codex-agent.md`). This preserves the main session's context window by offloading thread management, Codex communication, and response verification to a dedicated agent.

### Why Use the Agent

Talking to Codex directly from the main session has two costs:
1. **Codex responses** consume context (often verbose, may include false positives)
2. **Verification** of Codex responses (reading code, checking claims) consumes even more context

The codex-agent handles both in a subagent. The main session receives only a clean, verified summary.

### How to Dispatch

Use the Task tool to dispatch the codex-agent. The agent supports four modes:

**`create-thread`** — Start a new Codex conversation (brainstorming only):
```
Dispatch codex-agent with:
  mode: create-thread
  context: <summary of project and mission for Codex>
```

**`discuss`** — Send a discussion message and get a verified response:
```
Dispatch codex-agent with:
  mode: discuss
  message: <your message to Codex>
  context: <optional additional context>
  worktree_path: <optional worktree absolute path>
```

**`review-gate`** — Send content for review, get a verified verdict:
```
Dispatch codex-agent with:
  mode: review-gate
  message: <review request with commit SHAs, summary, test results>
  context: <design doc reference, plan task, etc.>
  worktree_path: <optional worktree absolute path>
```

**`cross-verify`** — Cross-verify a specific finding with Codex:
```
Dispatch codex-agent with:
  mode: cross-verify
  finding: <the finding to verify — ID, description, file, line>
  message: <additional context about the finding>
  worktree_path: <optional worktree absolute path>
```

### What the Agent Returns

The agent returns a structured report with:
- **verdict** (for review-gate): `pass`, `fail`, or `pass-with-flags`
- **verified_issues**: Only issues confirmed by reading the actual code
- **dismissed_count**: How many false positives were filtered out
- **thread_status**: Whether the thread was reused, recovered, or newly created
- **codex_notes**: Non-blocking suggestions worth passing along

### Review Gate Loop with the Agent

The review gate loop still happens in the calling skill, but each round dispatches the agent:

1. Dispatch codex-agent with `mode: review-gate`
2. If verdict is `pass`: proceed
3. If verdict is `fail` with verified issues: fix the issues, then dispatch agent again
4. Maximum **5 rounds**. If still unresolved, proceed and track flags.

The key benefit: false positives are filtered out by the agent, so the main session only spends context on real issues that need fixing.

### Fallback: Direct Codex Calls

If the codex-agent cannot be dispatched (e.g., Task tool unavailable), skills may fall back to calling `codex` and `codex-reply` MCP tools directly using the patterns documented below. The verification rules in "Core Principle" still apply.

---

## Core Principle: Codex as Reference, Not Authority

Codex is a **reference**, not a source of truth. CC must **independently verify** every Codex claim against the actual code before accepting it. This applies to all interactions — reviews, cross-verification, brainstorming feedback.

**Rules:**
- Never accept a Codex finding without reading the code yourself
- If Codex says "line 42 has a bug" — read line 42 and confirm
- If Codex says "this function is missing validation" — check if the validation actually exists
- If Codex dismisses a finding — verify the dismissal reasoning against the code
- Codex has read-only sandbox access and may have stale or incomplete context
- When Codex and the code contradict each other, the code is the ground truth

## State Directory

All Codex state files live in `.codex-state/` at the **main repo root** (not `/tmp/`, not per-worktree).

**Why the main repo root?** Brainstorming runs in the main repo. Writing-plans and execution may run in a worktree. Using the main repo root ensures all skills find the same state regardless of where they run.

```bash
# Resolves to main repo root even from inside a worktree
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.codex-state"
mkdir -p "$STATE_DIR"
```

**Files:**
- `$STATE_DIR/codex_thread_id` — Codex thread ID for conversation continuity
- `$STATE_DIR/current_design_doc` — path to the approved design doc (relative to repo root)

> **Note:** Unresolved flags are tracked in `docs/unresolved-flags.md` (version-controlled, committed to git), NOT in `.codex-state/`. See "Tracking Unresolved Flags" below.

**Ensure `.codex-state/` is gitignored (run from main repo):**
```bash
grep -q '.codex-state/' "$MAIN_REPO/.gitignore" 2>/dev/null || echo '.codex-state/' >> "$MAIN_REPO/.gitignore"
```

## Thread Management

**Creating a thread (brainstorming only):**
```bash
# After calling `codex` MCP tool and getting thread ID:
echo "<thread-id>" > "$STATE_DIR/codex_thread_id"
```

**Recovering a thread (all other skills):**
```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.codex-state"
CODEX_THREAD_ID=""
if [ -f "$STATE_DIR/codex_thread_id" ]; then
    CODEX_THREAD_ID=$(cat "$STATE_DIR/codex_thread_id")
fi
```

**Validation and recovery:**
1. Read the thread ID from the file
2. Test it with a `codex-reply` call (e.g. a short "ping" message)
3. If it succeeds: reuse the thread
4. If it fails with "Session not found" or similar: the thread has expired. **Create a new thread** by calling `codex` MCP tool, save the new thread ID to the state file, and send a context message summarizing what has happened so far (read the design doc to rebuild context). Inform the user that the previous Codex thread expired and a new one was created.
5. If it fails for other reasons (MCP not connected, usage limit): treat Codex as unavailable

## Model Selection

When calling the `codex` or `codex-reply` MCP tools, **do NOT pass the `model` parameter**. Let Codex use the model configured in `~/.codex/config.toml` or the MCP server startup flags. Passing a model override can cause account compatibility errors.

## Codex Availability

If Codex is unavailable (MCP not connected, usage limit hit, or any error from `codex` or `codex-reply`), skip all Codex steps and proceed without Codex review. Inform the user that Codex review was skipped and why.

## Working Directory Awareness

Codex's MCP server uses the CWD from when it was started (the main repo), NOT the worktree. Since Codex is a reviewer and does not touch files, this is fine — but all messages to Codex via `codex-reply` MUST include:

```
NOTE: Implementation is in worktree at <worktree-absolute-path>.
All file paths are relative to the worktree root.
```

## Efficient Codex Communication

Codex has its own filesystem access (read-only sandbox). Do NOT paste full diffs or file contents into `codex-reply` messages — this wastes tokens and can hit message limits.

**For code reviews, send:**
1. The commit SHA(s) covering the changes (CC must commit before sending to Codex)
2. A short summary of what was implemented and why
3. The test results summary (pass/fail counts, not full output)
4. The worktree path note (see Working Directory Awareness)
5. Any relevant context (design doc reference, plan task number)

**Example message to Codex:**
```
Review commit abc1234..def5678 on branch feature/auth.
NOTE: Implementation is in worktree at /path/to/.worktrees/auth.

Implemented: JWT token validation middleware (Task 3 from docs/plans/2025-01-15-auth-design.md)
Tests: 12 passing, 0 failing
Files changed: src/middleware/auth.ts, tests/middleware/auth.test.ts

Please review for correctness, security issues, and alignment with the design.
```

Codex can then inspect the actual files and git log in its sandbox. This is far more efficient than pasting hundreds of lines of diff.

## Review Gate Pattern

All review gates follow the same logic:

1. Send content to Codex via `codex-reply`
2. If Codex passes: proceed
3. If Codex flags issues: fix and resubmit
4. Maximum **5 review rounds**. If still unresolved, proceed and clearly report unresolved flags to the user
5. **Track unresolved flags** — append any unresolved items to `docs/unresolved-flags.md` and commit (see below)

**What counts as a pass:** Codex explicitly states the content is acceptable. Minor style suggestions that don't affect correctness can be noted but do not block a pass.

**What counts as a fail:** Bugs, logic errors, missing error handling, test gaps, violations of the design, security issues, or deviations from the plan.

## Tracking Unresolved Flags

When the review gate passes with unresolved flags (5-round limit hit, or pass with minor notes worth revisiting), append them to `docs/unresolved-flags.md` and **commit the change**.

This file is version-controlled so flags survive across sessions, worktree changes, and branch merges. They represent real technical debt that needs to be addressed.

**Location:** `docs/unresolved-flags.md` (relative to repo/worktree root)

**Format:**
```markdown
## [Task/Batch name] - [YYYY-MM-DD HH:MM]
- **Source:** [which review — brainstorming design, per-task Codex, batch review, final review]
- **Flags:**
  - [flag 1 description]
  - [flag 2 description]
```

**Rules:**
- Append-only during implementation. Never delete or edit existing entries.
- Every "pass with flags" or 5-round exhaustion MUST append to this file and commit.
- `finishing-a-development-branch` reads and reports all accumulated flags before presenting options.
- When flags are resolved later, remove the entries and commit with `fix: resolve Codex flag — <description>`.

## State Cleanup

The `finishing-a-development-branch` skill handles cleanup. When a branch is completed (merged, PR created, or discarded), remove the state directory:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
rm -rf "$MAIN_REPO/.codex-state"
```

For Option 3 (keep branch as-is), preserve state files so the user can resume later.
