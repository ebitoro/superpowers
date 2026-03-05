# CLAUDE.md - Superpowers Plugin

Custom fork of [superpowers](https://github.com/obra/superpowers) by obra. Adds Codex MCP integration as a secondary reviewer and thought partner throughout the development lifecycle.

## Project Structure

```
.claude-plugin/     # Plugin metadata (plugin.json, marketplace.json)
skills/             # Skill definitions (SKILL.md + supporting files per skill)
lib/                # Shared references (codex-integration.md, skills-core.js)
agents/             # Agent prompt templates (code-reviewer.md, codex-agent.md)
codex/skills/       # Codex skill prompts (verify-design, verify-plan, code-review)
commands/           # Slash command definitions (brainstorm, write-plan, execute-plan)
hooks/              # Session hooks (session-start.sh, hooks.json)
docs/               # Documentation and plan storage
tests/              # Skill triggering and behavior tests
```

## Development Lifecycle

The skills form a pipeline. Each stage is a separate session:

```
brainstorming -> writing-plans -> [executing-plans | subagent-driven-development | team-driven-development] -> update-docs-after-implementation (opt-in) -> finishing-a-development-branch
```

- **brainstorming**: Explores idea, consults Codex, produces design doc. Terminal state = committed design doc (context window is typically full after brainstorming; fresh session avoids compaction loss).
- **writing-plans**: Reads design doc from `.codex-state/current_design_doc`, creates bite-sized TDD implementation plan. Enforces worktree before starting. Writes breadcrumbs (`current_plan`, `current_worktree`) to enable `/clear` before execution.
- **executing-plans**: Batch execution (3 tasks/batch) with code review checkpoints between batches.
- **subagent-driven-development**: Fresh subagent per task + three-stage review (spec compliance, code quality, Codex per-task). Recovers plan path and worktree from breadcrumbs if context was cleared.
- **team-driven-development**: Leader agent orchestrates via AgentTeam. Fresh implementer per task handles full review pipeline (self-review, Codex, spec compliance, code quality). Persistent Codex Reviewer as shared service. Main session only sees final verdicts. Preferred for plans with 5+ tasks to minimize context usage.
- **requesting-code-review**: Two-stage gate: code-reviewer subagent first, then Codex review gate. Both must pass.
- **update-docs-after-implementation**: Opt-in. Reads all commits, updates documents listed in project CLAUDE.md under `## Post-Implementation Docs`. Runs after final review, before finishing.
- **finishing-a-development-branch**: Verify tests, present 4 options (merge/PR/keep/discard), cleanup worktree and state.

## Codex Integration

All Codex patterns are documented in `lib/codex-integration.md` (single source of truth).

**Core principle:** Codex is a reference, not authority. CC must independently verify every Codex claim against the actual code before accepting it. When Codex and the code contradict, the code is ground truth.

### Codex Agent (Primary Pattern)

**All Codex interactions go through the `codex-agent` subagent** (`agents/codex-agent.md`). This offloads thread management, Codex communication, and response verification to a dedicated agent — preserving the main session's context window.

The agent supports four modes:
- `create-thread` — Start a new Codex conversation (brainstorming only)
- `discuss` — Send a message, get a verified response
- `review-gate` — Send content for review, get a filtered verdict (false positives removed)
- `cross-verify` — Cross-verify a specific finding with Codex

Skills dispatch the codex-agent via the Task tool. The agent handles thread recovery, skill selection, and response verification internally. See `lib/codex-integration.md` for dispatch format.

### State Directory

State lives at `.codex-state/` in the **main repo root** (not per-worktree). Resolved via:
```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.codex-state"
```

Uses `--git-common-dir` (not `--show-toplevel`) so worktrees resolve to the main repo.

### Files
- `.codex-state/codex_thread_id` - Codex thread ID for design/plan phases (managed by codex-agent for brainstorming/writing-plans)
- `.codex-state/current_design_doc` - Path to approved design doc (relative to repo root)
- `.codex-state/current_plan` - Path to implementation plan (relative to repo root). Written by `writing-plans`, read by `subagent-driven-development` and `team-driven-development`. Cleaned up by `finishing-a-development-branch`.
- `.codex-state/current_worktree` - Absolute path to worktree. Written by `writing-plans`, read by `subagent-driven-development` and `team-driven-development`. Cleaned up by `finishing-a-development-branch`.

Implementation review thread IDs are caller-managed — each skill saves/retrieves its own `thread_id` as needed (see `lib/codex-integration.md` for the pattern).

### Review Gate Pattern

Max 5 rounds. Dispatch codex-agent, fix verified issues, redispatch until pass. The agent filters out false positives so only real issues come back. If still unresolved after 5 rounds, proceed and report unresolved flags to user.

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
- Two-stage review: code-reviewer subagent + Codex review gate (via codex-agent)
- Never skip either stage
- Fix all Critical and Important issues before proceeding
- Codex review (through codex-agent) catches cross-cutting issues the subagent misses

## Versioning

Both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` must have matching version numbers. Bump both when making changes.

## Gitignored Paths
- `.worktrees/` - Git worktree directories
- `.private-journal/` - Private notes
- `.claude/` - Claude Code session data
- `.codex-state/` - Must be added to `.gitignore` at runtime if missing
