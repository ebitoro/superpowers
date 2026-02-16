---
name: code-review
description: Use when Codex needs to review code changes — per-task review, batch review, or final review. Do NOT use for design documents or implementation plans.
---

When you receive a message tagged `[SKILL: code-review]`, follow this process exactly:

## 1. Inspect the Changes

Using the commit SHAs provided:
- Read the changed files in your sandbox
- Understand what was implemented and why (from the summary provided)
- Check the test files for coverage

## 2. Evaluate

Check for:
- **Correctness**: Logic errors, off-by-one, wrong conditions, missing returns
- **Security**: Injection, XSS, path traversal, hardcoded secrets, unsafe deserialization
- **Error handling**: Unhandled exceptions, missing null checks, silent failures
- **Test gaps**: Untested paths, missing edge case tests, tests that always pass
- **Design alignment**: Does the code match the stated task/design? Over/under-built?
- **Race conditions**: Shared state, async issues, file system races

## 3. Respond in This Exact Format

```
VERDICT: PASS | FAIL

FINDINGS:
- [CRITICAL] <file>:<line>: <one-line description>
- [IMPORTANT] <file>:<line>: <one-line description>
- [MINOR] <file>:<line>: <one-line description>

NOTES:
- <non-blocking suggestion, one line each>
```

## Rules

- VERDICT is PASS if zero CRITICAL and zero IMPORTANT findings
- VERDICT is FAIL if any CRITICAL or IMPORTANT finding exists
- Each finding is ONE line with `file:line` reference. No paragraphs.
- If a finding spans multiple lines, use the starting line
- FINDINGS section is empty if PASS with no issues
- NOTES section is for style preferences, minor refactors, or nice-to-haves that do NOT block the verdict
- Do NOT paste code snippets. The reviewer can read the file.
- Do NOT repeat the diff back. Do NOT summarize what you read.
- Do NOT add preamble or explanation outside the format above.
