# Implementer Subagent Prompt Template

Use this template when dispatching an implementer subagent via the Task tool (NOT as a teammate).
The Leader fills in all placeholders before dispatch.

```
Task tool (general-purpose):
  description: "Implement Task {TASK_NUMBER}: {TASK_NAME}"
  prompt: |
    You are implementing Task {TASK_NUMBER}: {TASK_NAME}

    ## Task Description

    {TASK_TEXT}

    ## Context

    {CONTEXT}

    ## Before You Begin

    If you have questions about:
    - The requirements or acceptance criteria
    - The approach or implementation strategy
    - Dependencies or assumptions
    - Anything unclear in the task description

    **Ask them now.** Raise any concerns before starting work.

    ## Your Job

    Once you're clear on requirements:
    1. Implement exactly what the task specifies
    2. Write tests (following TDD if task says to)
    3. Verify implementation works
    4. Commit your work with conventional commit format
    5. Self-review (see below)
    6. Report back using the exact format below

    Work from: {WORKING_DIRECTORY}

    **While you work:** If you encounter something unexpected or unclear, **ask questions**.
    Don't guess or make assumptions — pause and clarify.

    ## Before Reporting Back: Self-Review

    Review your work with fresh eyes:

    **Completeness:** Did I fully implement everything? Missing requirements? Edge cases?
    **Quality:** Is this my best work? Clear names? Clean, maintainable code?
    **Discipline:** YAGNI — only what was requested? Followed existing patterns?
    **Testing:** Tests verify behavior (not just mock it)? TDD if required? Comprehensive?

    If you find issues during self-review, fix them before reporting.

    ## Report Format

    When done, report **exactly** this (Leader parses it):

    ```
    Task {TASK_NUMBER} complete.
    - Implemented: [one-line summary of what you built]
    - Tests: [pass/fail count, e.g. "4 passed, 0 failed"]
    - Commit: [full SHA of your commit]
    - Concerns: [any issues or risks, or "none"]
    ```

    If you could not complete the task, report:

    ```
    Task {TASK_NUMBER} blocked.
    - Reason: [why you could not complete it]
    - Partial commit: [SHA if any work was committed, or "none"]
    - Questions: [what you need answered to proceed]
    ```
```
