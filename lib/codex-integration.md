# Codex Integration Reference

Shared patterns for skills that integrate with Codex as a reviewer/thought partner.

## Core Principle: Codex as Reference, Not Authority

Codex is a **reference**, not a source of truth. CC must **independently verify** every Codex claim against the actual code before accepting it.

**Rules:**
- Never accept a Codex finding without reading the code yourself
- If Codex says "line 42 has a bug" — read line 42 and confirm
- If Codex says "this function is missing validation" — check if the validation actually exists
- If Codex dismisses a finding — verify the dismissal reasoning against the code
- When Codex and the code contradict each other, the code is the ground truth
- Codex has read-only sandbox access and may have stale or incomplete context

## Codex Agent

**All Codex interactions go through the `codex-agent` subagent** (`agents/codex-agent.md`).

### How to Dispatch

Use the Task tool to dispatch the codex-agent. Four modes:

**`create-thread`** — Start a new Codex conversation (brainstorming only):
```
mode: create-thread
context: <summary of project and mission for Codex>
```

**`discuss`** — Send a discussion message and get a verified response:
```
mode: discuss
message: <your message to Codex>
context: <optional additional context>
worktree_path: <optional worktree absolute path>
```

**`review-gate`** — Send content for review, get a verified verdict:
```
mode: review-gate
message: <review request — see "What to Include" below>
context: <design doc reference, plan task, etc.>
worktree_path: <optional worktree absolute path>
```

**`cross-verify`** — Cross-verify a specific finding with Codex:
```
mode: cross-verify
finding: <the finding — ID, description, file, line>
message: <additional context>
worktree_path: <optional worktree absolute path>
```

### What to Include in review-gate Messages

The codex-agent selects the right Codex review skill automatically based on context. CC just needs to provide enough information:

- **Commit SHAs** covering the changes — NOT raw diffs (Codex can read files in its sandbox)
- **Short summary** of what was implemented and why
- **Test results** summary (pass/fail counts)
- **Context** (design doc reference, plan task number, etc.)

Codex inspects actual files and git log in its sandbox. Sending SHAs instead of diffs is far more token-efficient.

### What the Agent Returns

- **verdict** (for review-gate): `pass`, `fail`, or `pass-with-flags`
- **verified_issues**: Only issues confirmed by reading the actual code
- **dismissed_count**: How many false positives were filtered out
- **thread_status**: Whether the thread was reused, recovered, or newly created
- **codex_notes**: Non-blocking suggestions worth passing along

### Review Gate Loop

1. Dispatch codex-agent with `mode: review-gate`
2. If verdict is `pass`: proceed
3. If verdict is `fail` with verified issues: fix the issues, then dispatch agent again
4. Maximum **5 rounds**. If still unresolved, proceed and track flags.

False positives are filtered by the agent — only real issues come back.

## State Directory

All Codex state files live in `.codex-state/` at the **main repo root** (not per-worktree).

```bash
# Resolves to main repo root even from inside a worktree
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.codex-state"
mkdir -p "$STATE_DIR"
```

**Files:**
- `$STATE_DIR/codex_thread_id` — Codex thread ID (managed by codex-agent)
- `$STATE_DIR/current_design_doc` — path to the approved design doc (relative to repo root)

**Ensure `.codex-state/` is gitignored:**
```bash
grep -q '.codex-state/' "$MAIN_REPO/.gitignore" 2>/dev/null || echo '.codex-state/' >> "$MAIN_REPO/.gitignore"
```

## Codex Availability

If the codex-agent reports `status: unavailable` (MCP not connected, usage limit, or any error), skip all Codex steps and proceed without Codex review. Inform the user that Codex review was skipped and why.

## Tracking Unresolved Flags

When the review gate passes with unresolved flags (5-round limit hit, or pass-with-flags), append them to `docs/unresolved-flags.md` and **commit the change**. This file is version-controlled so flags survive across sessions.

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
- Every pass-with-flags or 5-round exhaustion MUST append and commit.
- `finishing-a-development-branch` reads and reports all accumulated flags before presenting options.
- When flags are resolved later, remove the entries and commit with `fix: resolve Codex flag — <description>`.
