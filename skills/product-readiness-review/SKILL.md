---
name: product-readiness-review
description: Comprehensive product readiness review - edge cases, gaps, false positives, security, error handling. CC and Codex review independently then cross-verify findings. Invoked via /product-readiness-review command only.
---

# Product Readiness Review

Full-project audit for production readiness. CC and Codex review independently, merge findings into a document, then cross-verify every finding through discussion. The output is a committed review document.

**This skill is invoked via `/product-readiness-review` command only.** It does NOT trigger automatically.

## Codex Integration

> **Reference:** See `lib/codex-integration.md` for shared patterns (state directory, thread management, model selection, availability).

**Thread setup** — on startup, locate the state directory and check for an existing Codex thread:

```bash
MAIN_REPO="$(cd "$(git rev-parse --git-common-dir)/.." && pwd)"
cat "$MAIN_REPO/.codex-state/codex_thread_id" 2>/dev/null
```

- If a thread ID exists, validate it with a `codex-reply` ping. Reuse if valid, create new if expired (see `lib/codex-integration.md` for recovery steps).
- If no thread ID exists, create a new thread via `codex` MCP tool, save to `$MAIN_REPO/.codex-state/codex_thread_id`.

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Set up Codex thread** — recover or create thread, send project context
2. **Parallel review** — CC explores codebase while Codex reviews independently
3. **Create findings document** — merge CC findings + Codex findings
4. **Cross-verification** — CC and Codex discuss each finding via `codex-reply`
5. **Update document** — mark each finding as confirmed, dismissed, or downgraded
6. **Write summary** — add summary section at bottom of document
7. **Commit** — commit the document to git
8. **Cleanup** — remove `.codex-state/` directory

## The Process

### Step 1: Set Up Codex Thread

Recover or create a Codex thread (see Codex Integration above). Then send a context message:

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

Go through every finding (both CC's and Codex's) one by one via `codex-reply`. For each finding:

1. **CC findings** — send each CC finding to Codex: "I found [finding]. Do you agree this is a real issue? Can you verify by checking the code at [file:line]?"
2. **Codex findings** — CC independently verifies each Codex finding by reading the code and checking if the issue is real.

For each finding, determine:
- **Confirmed** — both agree it's a real issue
- **Dismissed** — both agree it's a false positive (explain why)
- **Downgraded** — real but less severe than initially rated (explain why)
- **Escalated** — more severe than initially rated (explain why)

**Discussion protocol:**
- Send one finding at a time to Codex
- Wait for Codex's assessment
- If both agree, mark the finding and move to the next one
- If CC and Codex disagree, continue exchanging reasoning until both agree (max 5 attempts per finding)
- If still disagreeing after 5 attempts, mark as "Disputed" and update the document with both perspectives including each side's reasoning

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

Remove the `.codex-state/` directory. The review is complete and the thread is no longer needed.

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
- Codex via `codex-reply` for independent review and cross-verification
- `lib/codex-integration.md` for thread management patterns
