---
name: update-docs-after-implementation
description: Use after all implementation tasks and final review pass, before finishing-a-development-branch. Reads all commits from the worktree branch, identifies documents to update from the project CLAUDE.md, and updates them with a summary of changes. Triggers automatically when subagent-driven-development, team-driven-development, or executing-plans complete their final review.
---

# Update Docs After Implementation

Read all changes made during implementation and update project documentation listed in the project's CLAUDE.md.

**Core principle:** One subagent reads all commits, updates all listed docs, commits once. Runs only if the project opts in via CLAUDE.md.

## CLAUDE.md Convention

Projects opt in by adding a `## Post-Implementation Docs` section to their **project-level** CLAUDE.md (not global `~/.claude/CLAUDE.md`):

```markdown
## Post-Implementation Docs
- docs/ARCHITECTURE.md
- docs/API.md
```

Each entry is a path relative to the repo root. If the section is missing or empty, skip this skill silently — the project has not opted in.

## When This Runs

Dispatched by execution skills (subagent-driven-development, team-driven-development, executing-plans) after the final review passes and before invoking `finishing-a-development-branch`. The main session dispatches a single subagent that handles everything.

## The Process

### Step 1: Find the Doc List

Read the project's CLAUDE.md (the one in the repo root, not `~/.claude/CLAUDE.md`). Parse the `## Post-Implementation Docs` section to extract file paths.

```bash
# Find project CLAUDE.md
REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
PROJECT_CLAUDE="$REPO_ROOT/CLAUDE.md"
```

If no `## Post-Implementation Docs` section exists, report "No post-implementation docs configured" and exit.

### Step 2: Determine the Change Scope

Get the base branch and collect all changes:

```bash
# Determine base branch
BASE_BRANCH=$(git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null)

# All commits in this branch
git log --oneline $BASE_BRANCH..HEAD

# Summary of files changed
git diff --stat $BASE_BRANCH..HEAD

# Detailed diff for understanding what changed
git diff $BASE_BRANCH..HEAD --no-color
```

### Step 3: Read and Update Each Document

For each document listed in `## Post-Implementation Docs`:

1. **Check if it exists.** If not, create it with a basic structure appropriate to its name.
2. **Read the current content.**
3. **Understand what changed** from the commits and diff.
4. **Update only the sections affected by the implementation.** Don't rewrite the entire document — surgical updates to relevant sections.
5. **Preserve the document's existing style, structure, and voice.** Match the formatting conventions already in use.

**Update guidelines:**
- Add new sections for new features/components introduced
- Update existing sections where behavior or API changed
- Remove references to deleted features/APIs
- Update code examples if the API surface changed
- Don't add implementation details that belong in code comments
- Keep it factual — describe what the system does, not the story of how it was built

### Step 4: Commit

```bash
# Stage only the updated docs
git add <each updated doc path>

# Single commit for all doc updates
git commit -m "docs: update post-implementation documentation

Updated: <list of files>
Based on changes from: <branch-name> (<N> commits)"
```

## Dispatch Format

The main session dispatches this as a subagent:

```
Agent tool:
  subagent_type: "general-purpose"
  model: "opus"
  description: "Update post-implementation documentation"
  prompt: |
    You are a documentation updater. Use the superpowers:update-docs-after-implementation skill.

    Working directory: {WORKING_DIRECTORY}
    Base SHA: {BASE_SHA} (commit before first implementation task)

    Read all commits since BASE_SHA, find the Post-Implementation Docs list
    in the project CLAUDE.md, update each document, and commit.

    If no Post-Implementation Docs section exists in CLAUDE.md, report
    "No post-implementation docs configured — skipping" and exit.
```

## Output

Report back to main session:

```
## Doc Update Result
status: [updated | skipped | error]
documents_updated:
  - <path>: <one-line summary of what changed>
commit_sha: <sha of the docs commit, or "none">
reason: <if skipped or error, explain why>
```

## Red Flags

**Never:**
- Rewrite documents from scratch (surgical updates only)
- Add documents not listed in CLAUDE.md
- Modify source code or tests
- Skip committing the changes
- Update global `~/.claude/CLAUDE.md` — only read the project-level one

**Always:**
- Exit silently if no `## Post-Implementation Docs` section found
- Preserve existing document style and structure
- Commit doc updates separately from implementation commits
