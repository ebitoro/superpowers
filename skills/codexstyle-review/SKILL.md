---
name: codexstyle-review
description: Use when asked to review a diff, branch, PR, uncommitted changes, or specific commits for bugs and issues
---

# Code Review

Produce a structured, prioritized code review with actionable findings.

## Arguments

Free-text describing what to review:
- `changes on feature-branch against main`
- `uncommitted changes`
- `last 3 commits`
- `PR #42`

## Workflow

1. **Gather the diff.** Parse the argument to determine the appropriate git command(s). Run them. If ambiguous, ask for clarification.
2. **Read context.** For each changed file, read surrounding code and related files (tests, callers, types) to understand intent and detect regressions.
3. **Review.** Evaluate every hunk against criteria in `references/review-criteria.md`. List ALL qualifying findings — do not stop at the first.
4. **Output.** Format per `references/output-format.md` and print to the TUI.

## Core rules

- Role: reviewer for code written by another engineer.
- Flag only issues the author would fix if made aware.
- Zero findings is valid — do not invent issues.
- Review only — do not generate fixes, open PRs, or push code.