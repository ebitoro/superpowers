# Skill: verify-plan

Sent to Codex when reviewing an implementation plan during writing-plans.

## Instructions for Codex

When you receive a message tagged `[SKILL: verify-plan]`, follow this process exactly:

### 1. Check Plan Against Design

Verify:
- Every design requirement has at least one task covering it
- No tasks exceed the design scope (YAGNI)
- Task ordering respects dependencies

### 2. Check Task Quality

For each task, verify:
- Has a clear, testable deliverable
- Is bite-sized (2-5 minutes of work)
- Follows TDD: task specifies writing a failing test BEFORE the implementation
- Includes test criteria (what test to write, what it asserts)
- Does not bundle unrelated changes
- Has enough context for an implementer who hasn't read the full plan

### 3. Check TDD Compliance

Verify the plan enforces test-driven development:
- Each implementation task has a corresponding test task that comes BEFORE it (or is part of the same task with test-first ordering)
- Test tasks specify: what to test, expected behavior, and assertion
- No implementation task lacks a test
- Integration / end-to-end tests are included where tasks interact

### 4. Check Coverage Gaps

Look for:
- Missing error handling tasks
- Missing edge case tasks
- Missing cleanup / teardown tasks
- Tasks that assume prior state without a setup task
- Missing test tasks for any implementation task (test gaps)
- Missing negative test cases (invalid input, error paths, boundary conditions)
- Missing integration tests where multiple tasks connect

### 5. Respond in This Exact Format

```
VERDICT: PASS | FAIL

FINDINGS:
- [CRITICAL] Task <N>: <one-line description>
- [IMPORTANT] Task <N>: <one-line description>
- [TDD] Task <N>: <one-line description of TDD violation>
- [TEST-GAP] <one-line description of missing test>
- [MISSING] <one-line description of missing task>
- [MINOR] Task <N>: <one-line description>

NOTES:
- <non-blocking suggestion, one line each>
```

### Rules

- VERDICT is PASS if zero CRITICAL, zero IMPORTANT, zero TDD, zero TEST-GAP, and zero MISSING findings
- VERDICT is FAIL if any CRITICAL, IMPORTANT, TDD, TEST-GAP, or MISSING finding exists
- Each finding is ONE line. No paragraphs.
- Use `Task <N>` to reference specific tasks by their number in the plan
- Use `[TDD]` for tasks that violate test-first ordering (implementation before test, or no test specified)
- Use `[TEST-GAP]` for missing tests (no negative tests, no edge case tests, no integration tests)
- Use `[MISSING]` for non-test tasks that should exist but don't
- FINDINGS section is empty if PASS with no issues
- NOTES section is for ordering suggestions, wording tweaks, etc. that do NOT block the verdict
- Do NOT repeat the plan back. Do NOT summarize what you read.
- Do NOT add preamble or explanation outside the format above.
