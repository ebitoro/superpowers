# Subagent Orchestration Fixes Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix three orchestration issues in the subagent-driven-development skill: transient LSP false positives, redundant commits, and orphaned agent resume.

**Architecture:** All changes are prompt-level text edits to a single file (`skills/subagent-driven-development/SKILL.md`). No code, no hooks, no new files.

**Tech Stack:** Markdown (skill prompt files)

**Design doc:** `docs/superpowers/specs/2026-03-21-subagent-orchestration-fixes-design.md`

---

### Task 1: Add build validation step (Step 1b)

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md:281`

- [ ] **Step 1: Replace the implementer verdict handling line**

Replace line 281:
```
Handle the implementer verdict (see Handling Implementer Verdicts below). If `pass`, write the review state file and proceed to Step 2.
```

With:
```markdown
Handle the implementer verdict (see Handling Implementer Verdicts below). If `pass`, proceed to Step 1b.

### Step 1b: Build Validation

After the implementer returns `pass`, independently verify the build from the main session **before writing the review state file**:

\`\`\`bash
cd {WORKING_DIRECTORY}
# Use the project's build command (dotnet build, npm run build, cargo build, etc.)
\`\`\`

- **Build succeeds:** Write the review state file and proceed to Step 2. Ignore any stale LSP/IDE annotations — the build is ground truth.
- **Build fails with compile errors:** Re-dispatch the implementer with the build error output as additional context. Do not write the review state file.
- **Build fails for environment/tooling reasons** (missing SDK, network issues, etc.): Diagnose and fix the environment issue in the main session. Do not re-dispatch the implementer — their code is not at fault.
```

- [ ] **Step 2: Verify the edit**

Read back lines 281-298 of the modified file to confirm the new Step 1b is correctly inserted between Step 1 and Step 2, with proper markdown formatting.

- [ ] **Step 3: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat(skill): add post-implementer build validation step (Step 1b)"
```

---

### Task 2: Add commit discipline guidance

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md:269` (Per-Task Flow preamble)
- Modify: `skills/subagent-driven-development/SKILL.md:731` (Red Flags — line number will have shifted after Task 1)

- [ ] **Step 1: Add commit discipline paragraph to Per-Task Flow preamble**

After line 269 (`For each task:`), insert:

```markdown

**Commit discipline:** The main session NEVER commits implementation code. All commits come from subagents (implementer, review-and-fix). The main session only writes/updates review state files and `.dev-state/` breadcrumbs. Before any commit attempt in the main session, verify uncommitted changes exist with `git status --short` — if clean, skip the commit.
```

- [ ] **Step 2: Add Red Flag entry for redundant commits**

In the Red Flags "Never" list (after the line `- Add Codex batch review checkpoints to the task list when Codex is unavailable`), add:

```markdown
- Commit code that a subagent already committed (always check `git status` first)
```

- [ ] **Step 3: Verify both edits**

Read the Per-Task Flow preamble (around line 269-272) and the Red Flags Never list to confirm both insertions are correct and consistent.

- [ ] **Step 4: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat(skill): add commit discipline guidance to prevent redundant commits"
```

---

### Task 3: Add orphaned agent resume Red Flag

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md` (Red Flags section — line numbers shifted after Tasks 1-2)

- [ ] **Step 1: Add Red Flag entry for orphaned agent resume**

In the Red Flags "Never" list (after the entry added in Task 2), add:

```markdown
- Use SendMessage to continue a subagent that has already returned a verdict — it wastes tokens reconstructing stale context. For follow-up work, dispatch a fresh agent. The only valid SendMessage is answering an implementer's pre-verdict questions.
```

- [ ] **Step 2: Verify the edit**

Read the full Red Flags "Never" list to confirm the new entry is present and the list reads coherently with both new entries from Tasks 2 and 3.

- [ ] **Step 3: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat(skill): add SendMessage guidance to prevent orphaned agent resume"
```
