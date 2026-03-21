# Subagent-Driven Development — Orchestration Fixes

**Date:** 2026-03-21
**Scope:** Three targeted fixes for process errors observed in subagent-driven-development sessions

## Problem

Three recurring issues were observed in subagent-driven-development logs:

1. **Transient LSP false positives** — After an implementer subagent returns, the IDE's language server reports errors on files that were modified during multi-edit sequences (e.g., `CS0103: TryResolveSkill does not exist`). The method exists in the final file — the LSP cache hadn't re-indexed. The build and tests pass, but the stale annotations create noise and confusion.

2. **Redundant commit attempts** — The implementer subagent commits its own work (per implementer-prompt.md). The main session then also attempts to commit the same files, getting "no changes added to commit" (exit code 1). This is harmless but wasteful and confusing in logs.

3. **Orphaned agent resume** — The main session sends a `SendMessage` to a completed agent (one that already returned a verdict). The agent is reconstructed from transcript in the background, wastes tokens, and produces no useful output.

## Fix 1: Post-Implementer Build Validation

### What changes

Add a build validation step to the Per-Task Flow in `SKILL.md`, between handling the implementer verdict and dispatching the spec compliance review.

### Where

`skills/subagent-driven-development/SKILL.md` — Per-Task Flow section, after Step 1 (Dispatch Implementer), before Step 2 (Spec Compliance Review).

### Design

After the implementer returns `pass`:

1. Run the project's build command from the main session (e.g., `dotnet build`, `npm run build`, `cargo build`)
2. If build succeeds: proceed to Step 2. Any stale LSP annotations are transient — ignore them.
3. If build fails with real compile errors: re-dispatch the implementer with the build error output as additional context.

**Rationale:** The implementer already runs build verification (Phase 3 — Final Verification in implementer-prompt.md), so this is a redundant safety net. But the main session's independent verification serves as ground truth and explicitly addresses the LSP false-positive scenario. Build commands are authoritative; LSP is advisory.

### Text to add

New subsection after the implementer verdict handling (line ~281), before Step 2:

```markdown
### Step 1b: Build Validation

After the implementer returns `pass`, independently verify the build from the main session:

\`\`\`bash
cd {WORKING_DIRECTORY}
# Use the project's build command (dotnet build, npm run build, cargo build, etc.)
\`\`\`

- **Build succeeds:** Proceed to Step 2. Ignore any stale LSP/IDE annotations — the build is ground truth.
- **Build fails:** Re-dispatch the implementer with the build error output as additional context.
```

## Fix 2: No Main-Session Commits for Subagent Work

### What changes

Add explicit guidance that the main session never commits implementation code.

### Where

`skills/subagent-driven-development/SKILL.md` — Per-Task Flow section preamble (before Step 1), and Red Flags section.

### Design

The implementer-prompt.md already instructs subagents to commit their own work (Phase 1 Step 2: "Commit with conventional commit format", Phase 2: "Commit fixes"). Review-and-fix subagents also commit their fixes. The main session should never duplicate these commits.

### Text to add

1. Add to the Per-Task Flow preamble (after line 268, before Step 1):

```markdown
**Commit discipline:** The main session NEVER commits implementation code. All commits come from subagents (implementer, review-and-fix). The main session only writes/updates review state files and `.dev-state/` breadcrumbs. Before any commit attempt in the main session, verify uncommitted changes exist with `git status --short` — if clean, skip the commit.
```

2. Add to Red Flags "Never" list:

```markdown
- Commit code that a subagent already committed (always check `git status` first)
```

## Fix 3: No SendMessage to Completed Agents

### What changes

Add guidance prohibiting `SendMessage` to agents that have already returned a verdict.

### Where

`skills/subagent-driven-development/SKILL.md` — Red Flags section.

### Design

Once a subagent returns a verdict (`pass`, `fixed`, `fail`, `blocked`, `needs_context`, `all-false-positives`), that agent is done. For any follow-up work (re-review after fixes, additional context), dispatch a fresh agent. Sending `SendMessage` to a completed agent reconstructs it from transcript in the background, wasting tokens and producing no useful output.

The only valid use of `SendMessage` is answering an implementer's questions before it has returned a verdict — this is already documented in the "If subagent asks questions" section.

### Text to add

Add to Red Flags "Never" list:

```markdown
- Use SendMessage to continue a subagent that has already returned a verdict — it wastes tokens reconstructing stale context. For follow-up work, dispatch a fresh agent. The only valid SendMessage is answering an implementer's pre-verdict questions.
```

## Files Changed

| File | Change |
|------|--------|
| `skills/subagent-driven-development/SKILL.md` | Add Step 1b (build validation), commit discipline guidance, two Red Flag entries |

## Testing

These are prompt-level changes (guidance text), not code. Verification:

- Review the modified SKILL.md for consistency with existing flow
- Confirm the new step numbering doesn't break references
- Confirm Red Flags section remains well-organized
