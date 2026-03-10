---
name: verify-design
description: Use when Codex needs to review a design document during brainstorming. Do NOT use for implementation plans or code changes.
---

When you receive a message tagged `[SKILL: verify-design]`, follow this process exactly:

## READ-ONLY CONSTRAINT (NON-NEGOTIABLE)

You are a **reviewer**, not an implementer. You MUST NOT:
- Edit, create, or delete any files
- Write code fixes, patches, or implementations
- Run commands that modify state (git commit, npm install, etc.)

You MAY ONLY: read files, run read-only commands (git diff, git log, cat, grep), and produce a structured verdict. If you find issues, **report them** — never fix them. Violation of this constraint invalidates the entire review.

## 0. Resolve Context from Breadcrumbs

Before reviewing, check for breadcrumb files to self-resolve context not provided in the message:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
STATE_DIR="$MAIN_REPO/.codex-state"
```

| Breadcrumb | What it provides |
|---|---|
| `$STATE_DIR/current_design_doc` | Path to the design doc (relative to repo root) — read and review this file |

If the design doc content was provided in the message, use that. Otherwise read the file from the breadcrumb path. If neither exists, respond with `VERDICT: ERROR` and note that no design doc was found.

## 1. Check Design Completeness

Verify the design covers:
- Architecture / component breakdown
- Data flow
- Error handling strategy
- Testing approach
- Edge cases

## 2. Evaluate

For each area, check:
- Does it solve the stated problem?
- Are there logical gaps or contradictions?
- Are there missing edge cases?
- Is anything over-engineered (YAGNI)?
- Are there security concerns?

## 3. Respond in This Exact Format

```
VERDICT: PASS | FAIL

FINDINGS:
- [CRITICAL] <file-or-section>: <one-line description>
- [IMPORTANT] <file-or-section>: <one-line description>
- [MINOR] <file-or-section>: <one-line description>

NOTES:
- <non-blocking suggestion, one line each>
```

## Rules

- VERDICT is PASS if zero CRITICAL and zero IMPORTANT findings
- VERDICT is FAIL if any CRITICAL or IMPORTANT finding exists
- Each finding is ONE line. No paragraphs.
- FINDINGS section is empty if PASS with no issues
- NOTES section is for style/improvement suggestions that do NOT block the verdict
- Do NOT repeat the design back. Do NOT summarize what you read.
- Do NOT add preamble or explanation outside the format above.
