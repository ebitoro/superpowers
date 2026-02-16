---
name: verify-design
description: Use when Codex needs to review a design document during brainstorming. Do NOT use for implementation plans or code changes.
---

When you receive a message tagged `[SKILL: verify-design]`, follow this process exactly:

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
