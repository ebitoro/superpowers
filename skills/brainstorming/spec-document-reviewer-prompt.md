# Spec Document Reviewer Prompt Template

Use this template when dispatching a spec document reviewer subagent.

**Purpose:** Verify the spec is complete, consistent, and ready for implementation planning.

**Dispatch after:** Spec document is written to docs/superpowers/specs/

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For | Action |
    |----------|------------------|--------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections, vague requirements | Fix them — fill in concrete details, don't just flag |
    | Consistency | Sections that contradict each other, architecture that doesn't match feature descriptions | Resolve the contradiction — pick one and align the rest |
    | Clarity | Requirements ambiguous enough to be interpreted two different ways | Pick the intended interpretation and make it explicit |
    | Scope | Covers multiple independent subsystems instead of a single implementation plan | Flag for decomposition — this needs to be split |
    | YAGNI | Unrequested features, over-engineering, speculative abstractions | Remove them |

    ## Calibration

    **Only flag issues that would cause real problems during implementation planning.**
    A missing section, a contradiction, or a requirement so ambiguous it could be
    interpreted two different ways — those are issues. Minor wording improvements,
    stylistic preferences, and "sections less detailed than others" are not.

    Approve unless there are serious gaps that would lead to a flawed plan.

    ## Output Format

    ## Spec Review

    **Status:** Approved | Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters for planning]

    **Recommendations (advisory, do not block approval):**
    - [suggestions for improvement]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
