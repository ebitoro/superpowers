# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Superpowers is a multi-platform plugin that provides composable "skills" (process documentation) for AI coding agents. It enforces a structured development workflow: brainstorming -> design spec -> implementation plan -> subagent-driven execution with TDD and code review. Skills are markdown documents that agents load via the Skill tool and follow as mandatory process.

**Author:** Jesse Vincent (obra). **License:** MIT. **Current version:** defined in `package.json`, `.claude-plugin/plugin.json`, `.cursor-plugin/plugin.json`, `.claude-plugin/marketplace.json`, and `gemini-extension.json` (keep all five in sync).

## Architecture

### Plugin Loading

The plugin bootstraps via a **SessionStart hook** (`hooks/session-start`) that injects the `using-superpowers` skill into the agent's context. This is the entry point — it teaches the agent how to discover and invoke all other skills.

- **Claude Code**: `hooks/hooks.json` registers the SessionStart hook. The hook outputs JSON with `hookSpecificOutput.additionalContext`.
- **Cursor**: `hooks/hooks-cursor.json` registers the same hook. Output uses `additional_context` field instead.
- **Gemini CLI**: `gemini-extension.json` points to `GEMINI.md`, which `@`-includes the using-superpowers skill and tool mapping.
- **Codex/OpenCode**: Manual setup via fetch-and-follow instructions in `.codex/INSTALL.md` and `.opencode/INSTALL.md`.

`hooks/run-hook.cmd` is a polyglot bash/cmd wrapper for cross-platform hook execution (finds bash on Windows via Git for Windows, MSYS2, or PATH).

### Skill System

Skills live in `skills/<skill-name>/SKILL.md`. Each skill has YAML frontmatter with `name` and `description` fields (see [agentskills.io/specification](https://agentskills.io/specification)). The description controls when the agent activates the skill — it must describe triggering conditions only, never summarize the workflow (agents shortcut descriptions instead of reading the full skill).

**Core workflow chain:** `brainstorming` -> `writing-plans` -> `subagent-driven-development` (or `executing-plans`) -> `finishing-a-development-branch`

**Supporting skills:** `test-driven-development`, `systematic-debugging`, `verification-before-completion`, `requesting-code-review`, `receiving-code-review`, `using-git-worktrees`, `dispatching-parallel-agents`

**Meta skills:** `using-superpowers` (bootstrap, always loaded), `writing-skills` (creating new skills with TDD)

### Agents and Commands

- `agents/code-reviewer.md` — Agent definition for the code review subagent (used by subagent-driven-development's two-stage review).
- `commands/` — Deprecated slash commands (`brainstorm`, `write-plan`, `execute-plan`). These just tell users to use the skill instead.

### Brainstorm Visual Companion

`skills/brainstorming/scripts/` contains a zero-dependency Node.js WebSocket server (`server.cjs`) that serves HTML mockups during brainstorming sessions. Managed by `start-server.sh` / `stop-server.sh`. Content goes to `screen_dir`, user selections recorded in `state_dir/events`. Uses owner-PID lifecycle monitoring with platform-specific handling (disabled on Windows/MSYS2).

### Design and Plan Documents

Specs are saved to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`. Plans are saved to `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`. Older plans live in `docs/plans/`.

## Testing

Tests run real Claude Code sessions in headless mode and verify behavior through session transcript parsing (`.jsonl` files).

### Running Tests

```bash
# Integration test for subagent-driven-development
cd tests/claude-code
./test-subagent-driven-development-integration.sh

# Run all skill-triggering tests
cd tests/skill-triggering
./run-all.sh

# Run all explicit skill request tests
cd tests/explicit-skill-requests
./run-all.sh

# Brainstorm server unit tests
cd tests/brainstorm-server
npm test

# OpenCode integration tests
cd tests/opencode
./run-tests.sh

# Analyze token usage from a session
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<dir>/<session>.jsonl
```

### Test Requirements

- Must run from the **superpowers plugin directory** (skills only load from here)
- Claude Code installed and available as `claude` command
- Local dev marketplace enabled: `"superpowers@superpowers-dev": true` in `~/.claude/settings.json`
- Integration tests take 10-30 minutes (real agent sessions)
- Use `--permission-mode bypassPermissions` and `--add-dir` for headless tests

### Test Helpers

`tests/claude-code/test-helpers.sh` provides shared utilities (`create_test_project`, `cleanup_test_project`). New integration tests should source this file.

## Development Conventions

### Skill Authoring

Skills follow TDD: write pressure-test scenarios first (RED), write the skill (GREEN), close rationalization loopholes (REFACTOR). See `skills/writing-skills/SKILL.md` for the full methodology including testing with subagents.

Key constraints for SKILL.md frontmatter:
- `description` must start with "Use when..." and describe only triggering conditions
- Max 1024 characters total frontmatter, under 500 chars for description
- `name` uses only letters, numbers, hyphens

### Version Bumping

Version appears in five files that must stay in sync:
- `package.json`
- `.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`
- `.cursor-plugin/plugin.json`
- `gemini-extension.json`

### Cross-Platform Considerations

- Hook scripts use extensionless filenames (not `.sh`) to avoid Claude Code's Windows auto-detection
- `server.cjs` (not `.js`) because root `package.json` has `"type": "module"` which breaks `require()` on Node 22+
- Owner-PID monitoring is skipped on Windows/MSYS2 where PID namespace is invisible to Node.js
