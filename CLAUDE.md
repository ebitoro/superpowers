# CLAUDE.md - Superpowers Plugin

Custom fork of [superpowers](https://github.com/obra/superpowers) by obra. Adds Codex MCP integration as a secondary reviewer and thought partner throughout the development lifecycle.

## Project Structure

```
.claude-plugin/     # Plugin metadata (plugin.json, marketplace.json)
skills/             # Skill definitions (SKILL.md + supporting files per skill)
lib/                # Shared references (codex-integration.md, skills-core.js)
agents/             # Agent prompt templates (code-reviewer.md, codex-agent.md, plan-review-gate.md)
codex/skills/       # Codex skill prompts (verify-design, verify-plan, code-review)
commands/           # Slash command definitions (deprecated — use skills)
hooks/              # Session hooks (session-start.sh, hooks.json)
docs/               # Documentation and plan storage
tests/              # Skill triggering and behavior tests
```

## Development Lifecycle

The skills form a pipeline. Each stage is a separate session:

```
brainstorming -> writing-plans -> [subagent-driven-development | executing-plans] -> finishing-a-development-branch
```

- **brainstorming**: Explores idea, consults Codex (design review gate), produces design doc
- **writing-plans**: Creates TDD implementation plan, runs Codex plan review gate (tiered 3+3)
- **subagent-driven-development**: Fresh subagent per task + three-stage per-task review (self-review, spec, quality) + Codex batch review checkpoints between task groups + CC/Codex final review gates
- **executing-plans**: Sequential execution with Codex final review gate
- **requesting-code-review**: Two-stage gate: code-reviewer subagent first, then Codex review gate

## Codex Integration

All Codex patterns are documented in `lib/codex-integration.md` (single source of truth).

**Core principle:** Codex is a reference, not authority. CC must independently verify every Codex claim against the actual code before accepting it. When Codex and the code contradict, the code is ground truth.

### Codex Agent (Main Session Pattern)

Main session Codex interactions go through the `codex-agent` skill (`skills/codex-agent/SKILL.md`). This offloads thread management, Codex communication, and response verification to a dedicated agent.

The agent supports five modes: `init`, `create-thread`, `discuss`, `review-gate`, `cross-verify`.

### Direct Codex Calls (Subagent Pattern)

Subagents call `codex`/`codex-reply` MCP directly because they don't have the Agent tool. They inline the verification protocol (read code at cited locations, filter false positives).

### State Directory

State lives at `.dev-state/` in the **main repo root** (not per-worktree). Resolved via:
```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.dev-state"
```

### Files
- `.dev-state/codex_thread_id` - Codex thread ID for design/plan phases
- `.dev-state/current_design_doc` - Path to approved design doc
- `.dev-state/current_plan` - Path to implementation plan
- `.dev-state/current_worktree` - Absolute path to worktree
- `.dev-state/last_codex_batch_sha` - Last Codex batch review boundary (subagent-driven-development)

### Review Gate Pattern

Max 5 rounds. Main session dispatches codex-agent (has Agent tool). Subagents call `codex`/`codex-reply` MCP directly (no Agent tool). Both verify findings against actual code, filter false positives, fix verified issues, and retry until pass.

## Editing Skills

### SKILL.md Files
- Each skill lives in `skills/<name>/SKILL.md` with optional supporting files
- YAML frontmatter: `name` and `description` (description appears in skill list)
- Skills are invoked via the `Skill` tool, never by reading the file directly

### Shared References
- `lib/codex-integration.md` is referenced by all Codex-aware skills. Changes here propagate to all skills.

## Conventions

### Git
- Worktrees for all feature work
- Worktree directory: `.worktrees/` (gitignored)
- Conventional commit format
- SSH remote: `git@github.com:ebitoro/superpowers.git`

### Code Principles
- DRY, YAGNI, KISS
- TDD: write failing test, make it pass, commit
- Bite-sized tasks (2-5 minutes each)

### Review Requirements
- Two-stage review: code-reviewer subagent + Codex review gate
- Never skip either stage
- Fix all Critical and Important issues before proceeding

## Versioning

Both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` must have matching version numbers.

## Gitignored Paths
- `.worktrees/` - Git worktree directories
- `.private-journal/` - Private notes
- `.claude/` - Claude Code session data
- `.dev-state/` - Development workflow state files
