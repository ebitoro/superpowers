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

**All Codex interactions go through the `codex-agent` skill** (`skills/codex-agent/SKILL.md`).

### How to Dispatch

Use the Agent tool to dispatch the codex-agent in **foreground** — do NOT use `run_in_background`. The caller always needs the result before proceeding. Four modes:

**`init`** — Availability check + thread creation (no message sent):
```
mode: init
profile: <optional — Codex config profile to use for this thread>
```

**`create-thread`** — Start a new Codex conversation and persist the thread ID. No message sent — thread starts empty. Only used when the caller explicitly requests this mode:
```
mode: create-thread
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

**Compaction safety:** Callers MUST echo the returned `thread_id` as `**Active Codex thread_id:** <id>` into the conversation. This ensures compaction preserves it.

### What to Include in review-gate Messages

The codex-agent selects the right Codex review skill automatically based on context. CC just needs to provide enough information:

- **Commit SHAs** covering the changes (e.g., `{BASE_SHA}..{HEAD_SHA}`)
- **Short summary** of what was implemented and why (1-2 sentences)
- **Test results** summary (pass/fail counts)
- **Context** (design doc reference, plan task number, etc.)

<HARD-GATE>
**NEVER send raw diffs, full code, or file contents to Codex.** Codex has sandbox access — it runs `git diff` and reads files itself. Send ONLY commit SHAs and a short summary. Sending code wastes tokens and slows reviews.
</HARD-GATE>

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
4. If verdict is `fail` with verified issues:
   a. **Independently verify each finding** — read the actual code/content at the cited location and confirm the issue exists. The codex-agent filters obvious false positives, but the caller MUST still verify before acting. Never fix an issue you haven't confirmed yourself.
   b. Fix confirmed issues. Dismiss any remaining false positives.
   c. Dispatch codex-agent again with the saved `thread_id`
5. Maximum **5 rounds**. If still unresolved, proceed and track flags.

## Direct Codex Calls (Subagent Context)

Subagents spawned via the Agent tool do NOT have access to the Agent tool themselves. This means they cannot dispatch `codex-agent`. Instead, subagents call `codex` and `codex-reply` MCP tools directly, with the verification protocol inlined.

**When this applies:** Any subagent that needs Codex review — implementer subagents, codex-design-review, plan-review-gate, final-review subagents.

**When codex-agent is still used:** The main session (which HAS the Agent tool) uses codex-agent for init, thread creation, and any Codex interactions it does directly (writing-plans Tier 2 escalation, requesting-code-review, subagent-driven-development final review).

### Direct Call Protocol

1. **Thread creation** (when needed): Call `codex` MCP — never pass `model`, always pass `sandbox: "read-only"`, pass `profile` if starting a new thread
2. **Review messages**: Call `codex-reply` MCP — never pass `model`, always prepend read-only reminder
3. **Message format**: `[SKILL: code-review]` tag + read-only reminder + worktree note + review content (commit SHAs only)
4. **Response verification**: Read actual code at every cited location, classify as verified/false-positive/downgraded
5. **Ground truth rule**: When Codex and code contradict, code wins

### Read-Only Reminder (prepend to ALL messages)

```
IMPORTANT: You are in a READ-ONLY sandbox. Do NOT edit files, write fixes, run tests, or take any action. All tests have already been run and passed by the caller. Report findings only.
```

### Worktree Note (include when in a worktree)

```
NOTE: Implementation is in worktree at <path>.
All file paths are relative to the worktree root.
```

## Tiered Review Gate (3+3 Pattern)

For context-heavy skills (e.g., `writing-plans`) where the main session has already consumed significant context, the review gate loop can be offloaded to a subagent. This splits the work into two tiers:

**Tier 1 — Subagent (3 rounds max):**
1. Dispatch a dedicated review-gate subagent (e.g., `agents/plan-review-gate.md`) in **foreground**.
2. The subagent calls `codex`/`codex-reply` MCP directly (subagents don't have the Agent tool), **independently verifies** each finding against the actual code/content, fixes verified issues, and sends follow-up reviews via `codex-reply`. Up to 3 rounds.
3. The subagent returns:
   - **verdict**: `pass`, `fail`, or `pass-with-flags`
   - **severity**: `can_proceed` (minor issues) or `must_fix` (blocking issues) — only meaningful when verdict is `fail`
   - **unresolved_issues**: Remaining issues with verification notes
   - **thread_id**: For the caller to continue on the same Codex thread

**Tier 2 — Main session escalation (3 rounds max):**
Only triggered when Tier 1 returns `fail` + `must_fix`.
1. Main session takes over with the returned `thread_id`
2. Dispatches codex-agent directly. **Independently verifies each finding** — reads the actual code/content at the cited location, confirms the issue exists, dismisses false positives, fixes confirmed issues. Up to 3 more rounds.
3. If still `fail` + `must_fix` after 3 rounds → escalate to user
4. If `can_proceed` → append flags to `docs/unresolved-flags.md` and proceed

**Severity assessment criteria:**
- `can_proceed`: Style, naming, optional enhancements, suggestions conflicting with design decisions
- `must_fix`: Missing error handling in critical paths, incorrect tests, architectural problems, dependency ordering errors, security issues

**Independent verification is required at every tier.** The codex-agent filters obvious false positives. The Tier 1 subagent verifies findings before fixing. The Tier 2 main session verifies again during escalation. At no point should any actor fix an issue it hasn't confirmed by reading the actual code/content.

## Codex Profiles

Codex CLI supports configuration profiles (defined in `~/.codex/config.toml`). Profile is set **per-thread** at creation time — `codex-reply` does not accept a profile parameter.

| Profile | Used For | Rationale |
|---|---|---|
| `higheffort` | Per-task implementation reviews | Faster, sufficient for incremental task reviews |
| `xhigheffort` | Design review, plan review, final code review, standalone code review | Higher quality for high-stakes gate reviews |

**Rules:**
- Pass `profile` in `create-thread` and `review-gate` (with `thread_id: "new"`) dispatches only
- Never pass `profile` when reusing an existing thread (the thread already has its profile)
- If no `profile` is specified, Codex uses its default configuration

## State Directory

All Codex state files live in `.dev-state/` at the **main repo root** (not per-worktree).

```bash
# Resolves to main repo root even from inside a worktree
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.dev-state"
mkdir -p "$STATE_DIR"
```

**Files:**
- `$STATE_DIR/codex_thread_id` — Codex thread ID for design/plan phases (managed by codex-agent for brainstorming/writing-plans)
- `$STATE_DIR/current_design_doc` — path to the approved design doc (relative to repo root)
- `$STATE_DIR/current_plan` — path to the implementation plan (relative to repo root)
- `$STATE_DIR/current_worktree` — absolute path to worktree
- `$STATE_DIR/last_codex_batch_sha` — commit SHA marking the end of the last Codex batch review (used by subagent-driven-development for incremental batch ranges)

Implementation review thread IDs are caller-managed. They survive compaction via the echo rule, not via disk files.

**Ensure `.dev-state/` is gitignored:**
```bash
grep -q '.dev-state/' "$MAIN_REPO/.gitignore" 2>/dev/null || echo '.dev-state/' >> "$MAIN_REPO/.gitignore"
```

## Codex Availability & CC Fallback Review

If the codex-agent reports `status: unavailable` (MCP not connected, usage limit, or any error), **dispatch a CC fallback review** instead of skipping the review entirely. This ensures every review gate has at least one independent reviewer — either Codex or a fresh CC agent.

### CC Fallback Review Pattern

When Codex is unavailable at any review gate:

1. **Inform the user:** "Codex unavailable — running CC fallback review instead."
2. **Dispatch a `superpowers:code-reviewer` subagent** in foreground with a review prompt tailored to what Codex would have reviewed (design, plan, batch, or final review). Each skill defines its own CC fallback prompt.
3. **Handle verdict:**
   - `pass`: proceed to the next step
   - `fixed`: dispatch a fresh reviewer to verify the fixes. Max **3 rounds** — escalate to human if still finding issues.
4. **Track unresolved flags** the same way as Codex reviews: append to `docs/unresolved-flags.md`.

The CC fallback is a **blocking gate** — the review must pass before proceeding, just like the Codex gate it replaces. The difference: fewer rounds (3 vs 5) since CC reviews iterate faster.

**Why not just skip?** Self-review has confirmation bias — the implementer reviewing its own work. A fresh CC agent with no implementation context provides a genuinely different perspective, even if it's the same model family as the implementer. The cost is low and the catch rate is meaningful.

### Dispatch Pattern

```
Agent tool:
  subagent_type: "superpowers:codex-agent"
  description: "Codex review for Task N"
  prompt: |
    mode: review-gate
    ...
```

If codex-agent reports `status: unavailable`, dispatch the CC fallback review for that gate (see skill-specific prompts). Mark Codex unavailable for remaining tasks — all subsequent gates use CC fallback.

## Tracking Unresolved Flags

When the review gate passes with unresolved flags (round limit hit, or pass-with-flags), append them to `docs/unresolved-flags.md` and **commit the change**. This file is version-controlled so flags survive across sessions.

**Location:** `docs/unresolved-flags.md` (relative to repo/worktree root)

**Format:**
```markdown
## [Task/Batch name] - [YYYY-MM-DD HH:MM]
- **Source:** [which review — brainstorming design, Codex batch review, Codex final review]
- **Flags:**
  - [flag 1 description]
  - [flag 2 description]
```

**Rules:**
- Append-only during implementation. Never delete or edit existing entries.
- Every pass-with-flags or round-limit exhaustion MUST append and commit.
- `finishing-a-development-branch` reads and reports all accumulated flags before presenting options.
- When flags are resolved later, remove the entries and commit with `fix: resolve Codex flag — <description>`.
