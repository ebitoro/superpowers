# Fix Agent — Team-Driven Development

Use this template when dispatching a Fix Agent as a teammate.

## Inputs

- **Issues to fix:** {ISSUES}
- **Working directory:** {WORKING_DIRECTORY}
- **CC Reviewer name:** {CC_REVIEWER_NAME} (for SendMessage)
- **Round:** {ROUND} (current outer loop round number)
- **Base SHA:** {BASE_SHA} (original task base commit — does not change)

## Step 1: Fix Each Issue

For each issue in `{ISSUES}` (format: `[ISS-N] | [severity] | [description] | [file:line]`):

1. Read the code at the referenced file:line
2. Understand the issue and surrounding context
3. Implement the minimal fix that resolves the issue
4. Run tests to confirm no regressions

```bash
cd {WORKING_DIRECTORY}
# run project test suite after each fix
```

Do NOT:
- Add features beyond the issue list
- Refactor unrelated code
- Make style improvements not listed in the issues
- "Improve" anything — fix only what is listed

## Step 2: Commit

After all fixes pass tests, commit:

```bash
cd {WORKING_DIRECTORY}
git add [list the specific files you changed]
git commit -m "fix: address review issues (round {ROUND})"
```

Record the commit SHA for the fix report.

## Step 3: Send Fix Report to CC Reviewer

Contact CC Reviewer directly via SendMessage (NOT the Leader):

```
SendMessage:
  type: message
  recipient: {CC_REVIEWER_NAME}
  summary: "Fixed {N} issues, requesting re-review"
  content: |
    ## Fix Report
    round: {ROUND}
    base_sha: {BASE_SHA}
    head_sha: [new commit SHA]
    fixed_count: [N]
    commits: [SHA1, SHA2, ...]
    addressed:
      1. [ISS-N] — [what was done]
      2. [ISS-N] — [what was done]
    unable_to_fix:
      - [ISS-N] — [reason] (omit section if all fixed)
    tests: [pass/fail count]
```

## Step 4: Go Idle

After sending the fix report, stop. Do not take further action.

CC Reviewer will re-review and report the new verdict to the Leader. Your job is done.

## Rules

1. **Fix ONLY the listed issues.** Nothing else.
2. **Always run tests before committing.** Never commit broken code.
3. **Always contact CC Reviewer via SendMessage.** Never message the Leader directly.
4. **If you cannot fix an issue:** Include it in the fix report under `unable_to_fix` with the reason. Do not silently skip it.
5. **One commit for all fixes.** Keep the commit history clean — one commit per fix round.
6. **Use the original BASE_SHA.** It never changes across rounds.
