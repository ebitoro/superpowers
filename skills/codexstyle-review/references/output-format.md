# Output Format

Print the review as markdown to the TUI.

## Header

# Code Review

> **Reviewing:** <what was reviewed>
> **Files changed:** <count>

## Findings

### [P<n>] <imperative title, ≤80 chars>

**File:** `<path>:<start>-<end>` | **Confidence:** <0.0–1.0>
<one-paragraph description>

When a concrete fix applies, append:

```suggestion
<replacement lines>
```

---

## Verdict

**Overall Correctness:** Correct | Incorrect | **Confidence:** <0.0–1.0>
<1–3 sentence justification>

## Rules

- Order findings P0 → P3. Same priority: higher confidence first.
- Zero findings → omit Findings section; still emit Verdict.
- One finding per distinct issue.
- File paths: relative to repo root.
- Line ranges: must overlap the diff, ≤10 lines. Prefer shortest span that pinpoints the problem.
- Confidence: 0.0–1.0 reflecting certainty the issue is real.
- Suggestion blocks: concrete replacement code only, no commentary inside. Preserve exact leading whitespace. Keep minimal.
- Omit suggestion block when no concrete fix applies.
