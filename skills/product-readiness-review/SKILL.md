---
name: product-readiness-review
description: Comprehensive product readiness review - edge cases, gaps, false positives, security, error handling. CC and Codex review independently then cross-verify findings. 
---

# Product Readiness Review

## Reasoning Effort

Use the highest reasoning effort (ultrathink) for ALL steps throughout this entire review process. Production readiness is the last line of defense — every missed edge case, security gap, or false positive in tests is a potential production incident. Deep analytical thinking across all phases is non-negotiable.

## Overview

Full-project audit for production readiness. CC and Codex review independently, merge findings into a document, then cross-verify every finding through discussion. The output is a committed review document.

**This skill is invoked via `/product-readiness-review` command only.** It does NOT trigger automatically.

## Codex Integration

> See `lib/codex-integration.md` for Codex patterns. All interactions go through the codex-agent (`skills/codex-agent/SKILL.md`).

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Start Codex review** — dispatch codex-agent with `mode: discuss` to send project context and request independent review. **Wait for result** — must have thread_id before proceeding.
2. **Parallel review** — CC explores codebase while Codex reviews independently (via codex-agent). **Wait for Codex result** before step 3.
3. **Create findings document** — merge CC findings + Codex findings
4. **Cross-verification** — dispatch codex-agent with `mode: cross-verify` for each finding. **Wait for each result** before updating the document.
5. **Update document** — mark each finding as confirmed, dismissed, or downgraded
6. **Write summary** — add summary section at bottom of document
7. **Commit** — commit the document to git
8. **Cleanup** — remove `.codex-state/` directory

## The Process

### Step 1: Start Codex Review

Dispatch codex-agent with `mode: discuss`, `thread_id: "new"`, and `profile: "xhigheffort"` (not `review-gate` — this is an open-ended review request, not a pass/fail gate). Save the returned `thread_id` for use in Step 4 cross-verification. Send the following message:

```
Product readiness review for this project.
Project root: <absolute-path>
Branch: <current-branch>
Recent commits: <last 10 commit onelines>

Your task: Review this entire project for production readiness.
Focus on: edge cases, gaps in error handling, false positives in tests,
security vulnerabilities, missing validation, race conditions,
unhandled failure modes, incomplete implementations, and any code
that could break in production.

Report your findings organized by severity:
- CRITICAL: Will break in production
- IMPORTANT: Should fix before shipping
- MINOR: Worth noting but not blocking

For each finding, include: file path, line reference, description,
and suggested fix.
```

The codex-agent handles thread creation/recovery automatically. It will also do an initial verification of Codex's findings before returning them.

### Step 2: Parallel Review

While waiting for Codex, CC performs its own independent review.

**CC reviews the project for:**

1. **Edge cases** — boundary conditions, empty inputs, overflow, off-by-one
2. **Error handling gaps** — uncaught exceptions, missing try/catch, silent failures, missing error propagation
3. **False positives in tests** — tests that pass for wrong reasons, assertions that don't test what they claim, mocked behavior that diverges from real behavior
4. **Security** — injection, unsanitized input, hardcoded secrets, exposed endpoints, missing auth checks
5. **Missing validation** — input validation, type checking, null/undefined guards
6. **Race conditions** — concurrent access, shared mutable state, async timing issues
7. **Incomplete implementations** — TODOs, placeholder code, stub functions, dead code
8. **Configuration** — hardcoded values that should be configurable, missing defaults, env var handling
9. **Dependency risks** — outdated deps, known vulnerabilities, unnecessary deps
10. **Observability** — missing logging at critical points, no error reporting, silent swallows

**How CC reviews:**
- Launch 2-3 explorer subagents in parallel targeting different areas of the codebase
- Read all key files identified by explorers
- Run tests and check output
- Check for common anti-patterns

### Step 3: Create Findings Document

After both CC and Codex complete their reviews, create `docs/product-readiness-review-YYYY-MM-DD.md`:

```markdown
# Product Readiness Review - YYYY-MM-DD

**Project:** <project-name>
**Branch:** <branch-name>
**Reviewed by:** Claude Code + Codex
**Status:** In Progress

---

## CC Findings

### Critical
- [ ] **[CC-1]** <file:line> — <description>
  - **Impact:** <what breaks>
  - **Fix:** <suggested fix>

### Important
- [ ] **[CC-2]** <file:line> — <description>
  - **Impact:** <what could go wrong>
  - **Fix:** <suggested fix>

### Minor
- [ ] **[CC-3]** <file:line> — <description>
  - **Fix:** <suggested fix>

---

## Codex Findings

### Critical
- [ ] **[CX-1]** <file:line> — <description>
  - **Impact:** <what breaks>
  - **Fix:** <suggested fix>

### Important
- [ ] **[CX-2]** <file:line> — <description>
  - **Impact:** <what could go wrong>
  - **Fix:** <suggested fix>

### Minor
- [ ] **[CX-3]** <file:line> — <description>
  - **Fix:** <suggested fix>

---

## Cross-Verification

| ID | Finding | Codex Verdict | CC Verdict | Final Status |
|----|---------|---------------|------------|--------------|
```

### Step 4: Cross-Verification

Go through every finding (both CC's and Codex's) one by one by dispatching codex-agent with `mode: cross-verify`. The agent handles the discussion with Codex and independent code verification, returning a clean verdict for each finding. This keeps the cross-verification work out of the main session's context.

For each finding, dispatch codex-agent with:
- `mode: cross-verify`
- `thread_id`: the thread ID saved from Step 1 (reuses the same thread for the duration of this review)
- `finding`: the finding ID, description, file, and line reference
- `message`: additional context (e.g., "This is a CC finding" or "This is a Codex finding")

The codex-agent will:
1. Send the finding to Codex for assessment
2. Independently verify against the actual code
3. Resolve any disagreements with Codex (max 3 rounds in the agent)
4. Return a verdict: **Confirmed**, **Dismissed**, **Downgraded**, **Escalated**, or **Disputed**

If the agent returns **Disputed**, record both CC reasoning and Codex reasoning in the document.

### Step 5: Update Document

After each finding is discussed, update the Cross-Verification table and the finding's status in the document. Mark checkbox as `[x]` for verified findings, strike through dismissed ones.

### Step 6: Write Summary

After all findings are cross-verified, add a summary at the bottom of the document:

```markdown
---

## Summary

**Review Date:** YYYY-MM-DD
**Total Findings:** <N>
**Confirmed:** <N> (Critical: <N>, Important: <N>, Minor: <N>)
**Dismissed:** <N>
**Downgraded:** <N>
**Escalated:** <N>
**Disputed:** <N>

### Production Readiness Verdict

**[READY / NOT READY / READY WITH CAVEATS]**

<1-3 sentence justification>

### Critical Items Requiring Action
1. <item>
2. <item>

### Recommended Before Ship
1. <item>
2. <item>

**Status:** Complete
```

Update the document status from "In Progress" to "Complete".

### Step 7: Commit

```bash
git add docs/product-readiness-review-YYYY-MM-DD.md
git commit -m "docs: product readiness review YYYY-MM-DD"
```

### Step 8: Cleanup

Remove the `.codex-state/` directory. The review is complete and threads are no longer needed.

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
rm -rf "$MAIN_REPO/.codex-state"
```

## Red Flags

**Never:**
- Skip the Codex review (both perspectives are required)
- Auto-dismiss findings without discussion
- Mark as "Confirmed" without both CC and Codex agreeing
- Skip cross-verification to save time
- Leave the document in "In Progress" status

**If Codex is unavailable:**
- Complete the CC review only
- Mark document as "CC Review Only — Codex Unavailable"
- Note in summary that cross-verification was not performed

## Integration

**Invoked by:** `/product-readiness-review` command only (not auto-triggered)

**Uses:**
- Explorer subagents for codebase analysis
- codex-agent (`skills/codex-agent/SKILL.md`) for Codex review and cross-verification
- `lib/codex-integration.md` for shared patterns
