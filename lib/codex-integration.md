# Codex Integration Reference

Shared patterns for skills that integrate with Codex as a reviewer/thought partner.

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

## Review Gate Pattern

All review gates follow the same logic:

1. Send content to Codex via `codex-reply`
2. If Codex passes: proceed
3. If Codex flags issues: fix and resubmit
4. Maximum **5 review rounds**. If still unresolved, proceed and clearly report unresolved flags to the user

**What counts as a pass:** Codex explicitly states the content is acceptable. Minor style suggestions that don't affect correctness can be noted but do not block a pass.

**What counts as a fail:** Bugs, logic errors, missing error handling, test gaps, violations of the design, security issues, or deviations from the plan.

## State Cleanup

The `finishing-a-development-branch` skill handles cleanup. When a branch is completed (merged, PR created, or discarded), remove the state directory:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
rm -rf "$MAIN_REPO/.codex-state"
```

For Option 3 (keep branch as-is), preserve state files so the user can resume later.
