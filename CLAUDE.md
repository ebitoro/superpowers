# CLAUDE.md - Superpowers Plugin

Custom fork of [superpowers](https://github.com/obra/superpowers) by obra. Adds Codex MCP integration as a secondary reviewer and thought partner throughout the development lifecycle.

## Project Structure

```
.claude-plugin/     # Plugin metadata (plugin.json, marketplace.json)
skills/             # Skill definitions (SKILL.md + supporting files per skill)
lib/                # Shared references (codex-integration.md, skills-core.js)
agents/             # Agent prompt templates (code-reviewer.md)
commands/           # Slash command definitions (brainstorm, write-plan, execute-plan)
hooks/              # Session hooks (session-start.sh, hooks.json)
docs/               # Documentation and plan storage
tests/              # Skill triggering and behavior tests
```

## Development Lifecycle

The skills form a pipeline. Each stage is a separate session:

```
brainstorming -> writing-plans -> [executing-plans | subagent-driven-development] -> finishing-a-development-branch
```

- **brainstorming**: Explores idea, consults Codex, produces design doc. Terminal state = committed design doc (context window is typically full after brainstorming; fresh session avoids compaction loss).
- **writing-plans**: Reads design doc from `.codex-state/current_design_doc`, creates bite-sized TDD implementation plan. Enforces worktree before starting.
- **executing-plans**: Batch execution (3 tasks/batch) with code review checkpoints between batches.
- **subagent-driven-development**: Fresh subagent per task + three-stage review (spec compliance, code quality, Codex per-task) in the same session.
- **requesting-code-review**: Two-stage gate: code-reviewer subagent first, then Codex review gate. Both must pass.
- **finishing-a-development-branch**: Verify tests, present 4 options (merge/PR/keep/discard), cleanup worktree and state.

## Codex Integration

All Codex patterns are documented in `lib/codex-integration.md` (single source of truth).

**Core principle:** Codex is a reference, not authority. CC must independently verify every Codex claim against the actual code before accepting it. When Codex and the code contradict, the code is ground truth.

Key points:

### State Directory

State lives at `.codex-state/` in the **main repo root** (not per-worktree). Resolved via:
```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.codex-state"
```

Uses `--git-common-dir` (not `--show-toplevel`) so worktrees resolve to the main repo.

### Files
- `.codex-state/codex_thread_id` - Codex thread ID for conversation continuity
- `.codex-state/current_design_doc` - Path to approved design doc (relative to repo root)

### Thread Recovery

Codex threads can expire (~20min TTL observed). Skills must:
1. Read thread ID from `.codex-state/codex_thread_id`
2. Test with a `codex-reply` ping
3. If valid: reuse
4. If "Session not found": create new thread, save ID, re-send context from design doc
5. If other error: skip Codex, inform user

### Model Selection

Never pass the `model` parameter to `codex` or `codex-reply` MCP tools. Let the model be set by `~/.codex/config.toml` or MCP server startup flags. Passing a model override causes account compatibility errors.

### Review Gate Pattern

Max 5 rounds. Fix and resubmit until pass. If still unresolved after 5 rounds, proceed and report unresolved flags to user.

## Editing Skills

### SKILL.md Files
- Each skill lives in `skills/<name>/SKILL.md` with optional supporting files
- YAML frontmatter: `name` and `description` (description appears in skill list)
- Skills are invoked via the `Skill` tool, never by reading the file directly
- Rigid skills (TDD, debugging): follow exactly. Flexible skills (patterns): adapt to context.

### Shared References
- `lib/codex-integration.md` is referenced by all Codex-aware skills. Changes here propagate to all skills.
- When updating Codex patterns, update `lib/codex-integration.md` first, then verify each skill's reference is still accurate.

### Checklist Pattern
- Skills with checklists use numbered steps that must be completed in order
- Each step should be one clear action
- TodoWrite is used to track checklist progress

## Conventions

### Git
- Worktrees for all feature work (enforced by `using-git-worktrees` skill)
- Worktree directory: `.worktrees/` (gitignored)
- Frequent small commits following conventional commit format
- Never start implementation on main/master without explicit user consent
- SSH remote: `git@github.com:ebitoro/superpowers.git`

### Code Principles
- DRY, YAGNI, KISS
- TDD: write failing test, make it pass, commit
- Bite-sized tasks (2-5 minutes each)
- Self-documenting names

### Review Requirements
- Two-stage review: code-reviewer subagent + Codex review gate
- Never skip either stage
- Fix all Critical and Important issues before proceeding
- Codex review catches cross-cutting issues the subagent misses

## Versioning

Both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` must have matching version numbers. Bump both when making changes.

## Gitignored Paths
- `.worktrees/` - Git worktree directories
- `.private-journal/` - Private notes
- `.claude/` - Claude Code session data
- `.codex-state/` - Must be added to `.gitignore` at runtime if missing
