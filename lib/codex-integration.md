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
profile: <optional — Codex config profile to use for this thread>
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
thread_id: <optional — "new" for fresh thread, or a specific ID for retries>
message: <review request — see "What to Include" below>
context: <design doc reference, plan task, etc.>
worktree_path: <optional worktree absolute path>
profile: <optional — Codex config profile for new threads (ignored when reusing existing thread)>
```

**`cross-verify`** — Cross-verify a specific finding with Codex:
```
mode: cross-verify
thread_id: <optional — specific ID to continue on same thread>
finding: <the finding — ID, description, file, line>
message: <additional context>
worktree_path: <optional worktree absolute path>
```

### Thread Ownership

The **caller** owns thread lifecycle, not the codex-agent. The agent is a stateless proxy — it uses whatever thread the caller provides.

- **No `thread_id`**: Agent falls back to persistent `codex_thread_id` file (design/plan phases — brainstorming, writing-plans)
- **`thread_id: "new"`**: Agent creates a fresh thread and returns the ID. Does NOT save to disk — caller manages persistence.
- **`thread_id: <specific-id>`**: Agent uses that thread (for retries within a review gate)

**Compaction safety:** Callers MUST echo the returned `thread_id` as `**Active Codex thread_id:** <id>` into the conversation. This ensures compaction preserves it (see CLAUDE.md compaction rule).

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
- **thread_id**: The Codex thread ID used (save it for retries)
- **thread_status**: Whether the thread was reused, recovered, or newly created
- **codex_notes**: Non-blocking suggestions worth passing along

### Review Gate Loop

1. Dispatch codex-agent with `mode: review-gate` and `thread_id: "new"` (or a saved ID for retries)
2. Echo the returned `thread_id` as `**Active Codex thread_id:** <id>` (compaction rule)
3. If verdict is `pass`: done
4. If verdict is `fail` with verified issues: fix the issues, dispatch agent again with the saved `thread_id`
5. Maximum **5 rounds**. If still unresolved, proceed and track flags.

False positives are filtered by the agent — only real issues come back.

## Codex Profiles

Codex CLI supports configuration profiles (defined in `~/.codex/config.toml`). Profile is set **per-thread** at creation time — `codex-reply` does not accept a profile parameter.

| Profile | Used For | Rationale |
|---|---|---|
| `higheffort` | Per-task implementation reviews | Faster, sufficient for incremental task reviews |
| `xhigheffort` | Design review, plan review, final code review, standalone code review, product readiness review | Higher quality for high-stakes gate reviews |

**Rules:**
- Pass `profile` in `create-thread` and `review-gate` (with `thread_id: "new"`) dispatches only
- Never pass `profile` when reusing an existing thread (the thread already has its profile)
- If no `profile` is specified, Codex uses its default configuration

## State Directory

All Codex state files live in `.codex-state/` at the **main repo root** (not per-worktree).

```bash
# Resolves to main repo root even from inside a worktree
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.codex-state"
mkdir -p "$STATE_DIR"
```

**Files:**
- `$STATE_DIR/codex_thread_id` — Codex thread ID for design/plan phases (managed by codex-agent for brainstorming/writing-plans)
- `$STATE_DIR/current_design_doc` — path to the approved design doc (relative to repo root)

Implementation review thread IDs are caller-managed. They survive compaction via the echo rule (see CLAUDE.md), not via disk files.

**Ensure `.codex-state/` is gitignored:**
```bash
grep -q '.codex-state/' "$MAIN_REPO/.gitignore" 2>/dev/null || echo '.codex-state/' >> "$MAIN_REPO/.gitignore"
```

## Codex Availability

If the codex-agent reports `status: unavailable` (MCP not connected, usage limit, or any error), skip all Codex steps and proceed without Codex review. Inform the user that Codex review was skipped and why.

## Dispatch Pattern

```
Agent tool:
  subagent_type: "superpowers:codex-agent"
  model: "sonnet"
  description: "Codex review for Task N"
  prompt: |
    mode: review-gate
    ...
```

If codex-agent reports `status: unavailable`, mark Codex unavailable for remaining tasks and proceed without Codex review.

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
