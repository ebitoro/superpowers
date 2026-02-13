---
name: brainstorming
description: "You MUST use this before any creative work - creating features, building components, adding functionality, or modifying behavior. Explores user intent, requirements and design before implementation."
---

# Brainstorming Ideas Into Designs

## Overview

Help turn ideas into fully formed designs and specs through natural collaborative dialogue.

Start by understanding the current project context, then ask questions one at a time to refine the idea. Once you understand what you're building, present the design and get user approval.

<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it. This applies to EVERY project regardless of perceived simplicity.
</HARD-GATE>

## Anti-Pattern: "This Is Too Simple To Need A Design"

Every project goes through this process. A todo list, a single-function utility, a config change — all of them. "Simple" projects are where unexamined assumptions cause the most wasted work. The design can be short (a few sentences for truly simple projects), but you MUST present it and get approval.

## Codex Integration

You use Codex as a reviewer and thought partner via MCP tools throughout the brainstorming process.

**Thread management:**
- At the start of brainstorming, call `codex` MCP tool to create a new thread. Save the returned thread ID as `CODEX_THREAD_ID`.
- All subsequent Codex interactions use `codex-reply` with `CODEX_THREAD_ID` to maintain conversation continuity.
- Send a single "context + mission" message when starting the thread so Codex has full context of what is being designed.

<CRITICAL>
**Context window protection — `CODEX_THREAD_ID`:**
`CODEX_THREAD_ID` must survive compact. After obtaining the thread ID, immediately write it to a temporary file:
```
echo "CODEX_THREAD_ID=<id>" > /tmp/codex_thread_id
```
Before any `codex-reply` call, if `CODEX_THREAD_ID` is not in your working memory, read it back:
```
cat /tmp/codex_thread_id
```
</CRITICAL>

**Codex availability:**
If Codex is unavailable (MCP not connected, usage limit hit, or any error from `codex` or `codex-reply`), skip all Codex steps and proceed without Codex review. Inform the user that Codex review was skipped and why.

**Codex is consulted at three points:**
1. **After idea exploration** — You discuss the refined idea with Codex to validate understanding and surface blind spots.
2. **After exploring approaches** — You share proposed approaches with Codex. If you and Codex recommend different approaches, present both recommendations to the user with clear attribution (e.g., "I recommend A because X. Codex recommends B because Y.").
3. **Before presenting design to user** — Codex must review and pass the design. See "Codex Review Gate" below.

### Codex Review Gate

Before presenting the design to the user, you send the full design to Codex for review via `codex-reply`.

- If Codex passes: proceed to present design to user.
- If Codex flags issues: you fix the design and resubmit for review.
- Maximum 5 review rounds. If the design still has unresolved issues after 5 rounds, present the design to the user and clearly state what Codex still flagged as unresolved.

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — launch 2-3 subagent code-explorers in parallel (see "Codebase Exploration")
2. **Read key files** — read all files identified by the explorer agents to build deep understanding
3. **Start Codex thread** — call `codex` MCP tool, save `CODEX_THREAD_ID` to `/tmp/codex_thread_id`, send context + mission
4. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
5. **Discuss refined idea with Codex** — via `codex-reply`, validate understanding and surface blind spots
6. **Propose 2-3 approaches** — with trade-offs and your recommendation
7. **Discuss approaches with Codex** — via `codex-reply`, if recommendations differ, note both for user
8. **Codex review gate** — send design to Codex via `codex-reply`, iterate up to 5 times until pass
9. **Present final design to user** — include any unresolved Codex flags if review gate did not fully pass
10. **Write design doc** — save to `docs/plans/YYYY-MM-DD-<topic>-design.md`, commit, and write breadcrumb file to `/tmp/current_design_doc`

## Process Flow

```dot
digraph brainstorming {
    "Explore context (subagents)" [shape=box];
    "Read key files" [shape=box];
    "Start Codex thread" [shape=box];
    "Codex available?" [shape=diamond];
    "Ask clarifying questions" [shape=box];
    "Discuss idea with Codex" [shape=box];
    "Propose 2-3 approaches" [shape=box];
    "Discuss approaches with Codex" [shape=box];
    "Draft design sections" [shape=box];
    "Codex review gate" [shape=diamond];
    "Fixes < 5?" [shape=diamond];
    "Present design to user" [shape=box];
    "User approves?" [shape=diamond];
    "Write design doc" [shape=doublecircle];

    "Explore context (subagents)" -> "Read key files";
    "Read key files" -> "Start Codex thread";
    "Start Codex thread" -> "Codex available?";
    "Codex available?" -> "Ask clarifying questions" [label="no, skip all Codex steps"];
    "Codex available?" -> "Ask clarifying questions" [label="yes"];
    "Ask clarifying questions" -> "Discuss idea with Codex";
    "Ask clarifying questions" -> "Propose 2-3 approaches" [label="no Codex"];
    "Discuss idea with Codex" -> "Propose 2-3 approaches";
    "Propose 2-3 approaches" -> "Discuss approaches with Codex";
    "Propose 2-3 approaches" -> "Draft design sections" [label="no Codex"];
    "Discuss approaches with Codex" -> "Draft design sections";
    "Draft design sections" -> "Codex review gate";
    "Draft design sections" -> "Present design to user" [label="no Codex"];
    "Codex review gate" -> "Present design to user" [label="pass"];
    "Codex review gate" -> "Fixes < 5?" [label="fail"];
    "Fixes < 5?" -> "Draft design sections" [label="yes, fix and resubmit"];
    "Fixes < 5?" -> "Present design to user" [label="no, present with unresolved flags"];
    "Present design to user" -> "User approves?";
    "User approves?" -> "Draft design sections" [label="no, revise"];
    "User approves?" -> "Write design doc" [label="yes"];
}
```

**The terminal state is the committed design doc.** Do NOT invoke any implementation skill. The user manually proceeds from here.

## The Process

### Codebase Exploration

**Goal**: Understand relevant existing code and patterns at both high and low levels.

**Actions**:
1. Launch 2-3 code-explorer `subagent`s in parallel. Each agent should:
   - Trace through the code comprehensively and focus on getting a comprehensive understanding of abstractions, architecture and flow of control
   - Target a different aspect of the codebase (e.g., similar features, high level understanding, architectural understanding, user experience, etc.)
   - Include a list of 5-10 key files to read

   **Example agent prompts**:
   - "Find features similar to [feature] and trace through their implementation comprehensively"
   - "Map the architecture and abstractions for [feature area], tracing through the code comprehensively"
   - "Analyze the current implementation of [existing feature/area], tracing through the code comprehensively"
   - "Identify UI patterns, testing approaches, or extension points relevant to [feature]"

2. Once the agents return, read all files identified by agents to build deep understanding
3. Present comprehensive summary of findings and patterns discovered

### Starting the Codex Thread

- Call `codex` MCP tool to start a new thread
- Save the returned thread ID as `CODEX_THREAD_ID`
- Immediately persist to file: `echo "CODEX_THREAD_ID=<id>" > /tmp/codex_thread_id`
- Send a single message via `codex-reply` containing:
  - Summary of the project context gathered by subagents
  - The user's idea/request
  - Codex's role: reviewer and thought partner for this brainstorming session
- If `codex` fails, inform the user and proceed without Codex for the rest of the session

### Understanding the Idea

- Ask questions one at a time to refine the idea
- Prefer multiple choice questions when possible, but open-ended is fine too
- Only one question per message — if a topic needs more exploration, break it into multiple questions
- Focus on understanding: purpose, constraints, success criteria
- After questions are resolved, discuss the refined idea with Codex via `codex-reply` to validate understanding and surface blind spots

### Exploring Approaches

- Propose 2-3 different approaches with trade-offs
- Present options conversationally with your recommendation and reasoning
- Lead with your recommended option and explain why
- Discuss approaches with Codex via `codex-reply`
- If you and Codex recommend different approaches, tell the user: "I recommend [approach] because [reason]. Codex recommends [approach] because [reason]."

### Presenting the Design

- Draft the design internally first
- Submit to Codex review gate via `codex-reply` before showing to user
- Iterate with Codex up to 5 rounds if issues are flagged
- Once Codex passes (or 5 rounds exhausted), present to user
- If presenting with unresolved Codex flags, clearly list what remains unresolved
- Scale each section to its complexity: a few sentences if straightforward, up to 200-300 words if nuanced
- Ask after each section whether it looks right so far
- Cover: architecture, components, data flow, error handling, testing
- Be ready to go back and clarify if something doesn't make sense

## After the Design

**Documentation:**
- Write the validated design to `docs/plans/YYYY-MM-DD-<topic>-design.md`
- Use elements-of-style:writing-clearly-and-concisely skill if available
- Commit the design document to git
- Write breadcrumb file: `echo "docs/plans/YYYY-MM-DD-<topic>-design.md" > /tmp/current_design_doc`

**Brainstorming ends here.** The user manually proceeds with worktree creation and implementation planning.

## Key Principles

- **One question at a time** — Don't overwhelm with multiple questions
- **Multiple choice preferred** — Easier to answer than open-ended when possible
- **YAGNI ruthlessly** — Remove unnecessary features from all designs
- **Explore alternatives** — Always propose 2-3 approaches before settling
- **Incremental validation** — Present design, get approval before moving on
- **Be flexible** — Go back and clarify when something doesn't make sense
- **Codex as second opinion** — Use Codex to catch blind spots, not as a blocker
- **Transparency** — Always tell the user when you and Codex disagree
- **Graceful degradation** — If Codex is unavailable, proceed without it
