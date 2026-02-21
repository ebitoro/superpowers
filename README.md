# Superpowers (Ebitoro Fork)

A custom fork of [superpowers](https://github.com/obra/superpowers) by [obra](https://github.com/obra). Adds **Codex MCP integration** as a secondary reviewer and thought partner throughout the entire development lifecycle.

## What Changed from Upstream

- **Codex as thought partner** -- brainstorming consults Codex to validate ideas and surface blind spots
- **Three-stage per-task review** -- spec compliance, code quality, then Codex review (subagent-driven-development)
- **Two-stage code review gate** -- code-reviewer subagent + Codex review gate (requesting-code-review)
- **Persistent Codex threads** -- thread ID saved to `.codex-state/` with recovery when threads expire
- **Worktree-safe state** -- uses `git rev-parse --git-common-dir` so state resolves correctly from worktrees
- **Session-boundary design** -- brainstorming ends at committed design doc; fresh session for planning avoids compaction loss

## How It Works

Your coding agent doesn't jump into writing code. It steps back, asks what you're building, and runs through a structured pipeline:

1. **Brainstorm** the idea with Codex as a second opinion
2. **Write a plan** with bite-sized TDD tasks
3. **Execute** via subagents with three-stage review per task
4. **Final code review** through both a subagent and Codex
5. **Finish** by merging, creating a PR, or keeping the branch

Skills trigger automatically. You don't invoke them manually.

## Installation

**Note:** Installation differs by platform. Claude Code or Cursor have built-in plugin marketplaces. Codex and OpenCode require manual setup.

### Claude Code (via Plugin Marketplace)

Register the marketplace:

```bash
/plugin marketplace add ebitoro/superpowers
```

Install the plugin:

```bash
/plugin install superpowers@ebipowers-dev
```

### Codex MCP Setup

Add Codex as an MCP server (model is configured in `~/.codex/config.toml`, do NOT pass `--model` here):

```bash
claude mcp add --transport stdio --scope user codex -- codex -c model_reasoning_effort=xhigh --sandbox read-only --ask-for-approval never mcp-server
```

### Cursor (via Plugin Marketplace)

In Cursor Agent chat, install from marketplace:

```text
/plugin-add superpowers
```

## The Workflow

Each stage runs in a separate session to avoid context window compaction:

```
brainstorming -> writing-plans -> [executing-plans | subagent-driven-development] -> finishing-a-development-branch
```

1. **brainstorming** -- Refines ideas through questions, consults Codex at three points (idea validation, approach comparison, design review gate). Terminal state = committed design doc.

2. **writing-plans** -- Reads design doc from `.codex-state/current_design_doc`. Enforces worktree before starting. Creates bite-sized TDD tasks (2-5 min each).

3. **subagent-driven-development** or **executing-plans** -- Fresh subagent per task with three-stage review (spec compliance, code quality, Codex), or batch execution with code review checkpoints.

4. **requesting-code-review** -- Two-stage gate: code-reviewer subagent first, then Codex review gate. Both must pass.

5. **finishing-a-development-branch** -- Verifies tests, presents 4 options (merge/PR/keep/discard), cleans up worktree and `.codex-state/`.

### Verify Installation

Start a new session in your chosen platform and ask for something that should trigger a skill (for example, "help me plan this feature" or "let's debug this issue"). The agent should automatically invoke the relevant superpowers skill.

## Codex Integration

All Codex patterns live in `lib/codex-integration.md` (single source of truth).

- **State directory**: `.codex-state/` at main repo root (not per-worktree)
- **Thread recovery**: test thread on startup, recreate if expired (~20min TTL), re-send context from design doc
- **Model selection**: never pass `model` parameter to MCP tools; use `~/.codex/config.toml`
- **Compaction rule**: when a codex-agent dispatch returns a `thread_id`, the caller must echo `**Active Codex thread_id:** <id>` into the conversation so compaction preserves it
- **Review gate**: max 5 rounds, fix and resubmit until pass
- **Graceful degradation**: if Codex unavailable, skip Codex steps and proceed

## What's Inside

### Skills Library

**Core Workflow**
- **brainstorming** -- Socratic design refinement with Codex consultation
- **writing-plans** -- Detailed implementation plans with worktree enforcement
- **executing-plans** -- Batch execution with code review checkpoints (includes Codex)
- **subagent-driven-development** -- Fresh subagent per task with three-stage review
- **requesting-code-review** -- Two-stage gate: subagent + Codex
- **receiving-code-review** -- Responding to feedback with technical rigor
- **finishing-a-development-branch** -- Merge/PR decision workflow with state cleanup

**Development**
- **test-driven-development** -- RED-GREEN-REFACTOR cycle
- **systematic-debugging** -- 4-phase root cause process
- **verification-before-completion** -- Evidence before assertions

**Infrastructure**
- **using-git-worktrees** -- Isolated workspaces for feature work
- **dispatching-parallel-agents** -- Concurrent subagent workflows

**Meta**
- **writing-skills** -- Create new skills following best practices
- **using-superpowers** -- Introduction to the skills system

## Project Structure

```
.claude-plugin/     # Plugin metadata (plugin.json, marketplace.json)
skills/             # Skill definitions (SKILL.md + supporting files)
lib/                # Shared references (codex-integration.md, skills-core.js)
agents/             # Agent prompt templates (code-reviewer.md)
commands/           # Slash command definitions
hooks/              # Session hooks
docs/               # Documentation and plan storage
tests/              # Skill triggering and behavior tests
```

## Philosophy

- **Test-Driven Development** -- Write tests first, always
- **Systematic over ad-hoc** -- Process over guessing
- **Complexity reduction** -- Simplicity as primary goal
- **Evidence over claims** -- Verify before declaring success
- **Two reviewers catch what one misses** -- Subagent + Codex

## Upstream

This fork tracks [obra/superpowers](https://github.com/obra/superpowers). To pull upstream changes:

```bash
git fetch upstream
git merge upstream/main
```

## License

MIT License - see LICENSE file for details.

## Credits

Original [superpowers](https://github.com/obra/superpowers) by [Jesse Vincent](https://github.com/obra). Codex integration by [Ebitoro](https://github.com/ebitoro).
