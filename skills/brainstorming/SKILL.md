---
name: brainstorming
description: "You MUST use this before any creative work - creating features, building components, adding functionality, or modifying behavior. Explores user intent, requirements and design before implementation."
---

# Brainstorming Ideas Into Designs

Help turn ideas into fully formed designs and specs through natural collaborative dialogue.

Start by understanding the current project context, then ask questions one at a time to refine the idea. Once you understand what you're building, present the design and get user approval.

<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it. This applies to EVERY project regardless of perceived simplicity.
</HARD-GATE>

## Anti-Pattern: "This Is Too Simple To Need A Design"

Every project goes through this process. A todo list, a single-function utility, a config change — all of them. "Simple" projects are where unexamined assumptions cause the most wasted work. The design can be short (a few sentences for truly simple projects), but you MUST present it and get approval.

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — dispatch Explore subagent to scan the project and return a concise summary (see Exploring the Project section below)
2. **Offer visual companion** (if topic will involve visual questions) — this is its own message, not combined with a clarifying question. See the Visual Companion section below.
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Propose 2-3 approaches** — with trade-offs and your recommendation
5. **Present design** — in sections scaled to their complexity, get user approval after each section
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
7. **Run spec review** — dispatch design-spec-review subagent to handle the full review loop (see After the Design section)
8. **Run Codex design review** — dispatch codex-design-review subagent to handle the full Codex review gate (see After the Design section)
9. **Transition to implementation** — invoke writing-plans skill to create implementation plan

## Process Flow

```dot
digraph brainstorming {
    "Explore subagent\n(returns project summary)" [shape=box style=filled fillcolor=lightyellow];
    "Visual questions ahead?" [shape=diamond];
    "Offer Visual Companion\n(own message, no other content)" [shape=box];
    "Ask clarifying questions" [shape=box];
    "Propose 2-3 approaches" [shape=box];
    "Present design sections" [shape=box];
    "User approves design?" [shape=diamond];
    "Write design doc" [shape=box];
    "design-spec-review subagent\n(review loop, returns verdict)" [shape=box style=filled fillcolor=lightyellow];
    "codex-design-review subagent\n(Codex gate, returns verdict)" [shape=box style=filled fillcolor=lightyellow];
    "Invoke writing-plans skill" [shape=doublecircle];

    "Explore subagent\n(returns project summary)" -> "Visual questions ahead?";
    "Visual questions ahead?" -> "Offer Visual Companion\n(own message, no other content)" [label="yes"];
    "Visual questions ahead?" -> "Ask clarifying questions" [label="no"];
    "Offer Visual Companion\n(own message, no other content)" -> "Ask clarifying questions";
    "Ask clarifying questions" -> "Propose 2-3 approaches";
    "Propose 2-3 approaches" -> "Present design sections";
    "Present design sections" -> "User approves design?";
    "User approves design?" -> "Present design sections" [label="no, revise"];
    "User approves design?" -> "Write design doc" [label="yes"];
    "Write design doc" -> "design-spec-review subagent\n(review loop, returns verdict)";
    "design-spec-review subagent\n(review loop, returns verdict)" -> "codex-design-review subagent\n(Codex gate, returns verdict)";
    "codex-design-review subagent\n(Codex gate, returns verdict)" -> "Invoke writing-plans skill";
}
```

**Yellow nodes run as subagents** — their work stays out of the main session's context. The main session only receives the final summary/verdict.

**The terminal state is invoking writing-plans.** Do NOT invoke frontend-design, mcp-builder, or any other implementation skill. The ONLY skill you invoke after brainstorming is writing-plans.

## Exploring the Project

Dispatch an Explore subagent to scan the project and return a concise summary. This keeps raw file contents out of the main session's context.

```
Agent tool:
  subagent_type: "Explore"
  description: "Explore project context"
  prompt: |
    Explore this project thoroughly and return a concise summary covering:
    - **Tech stack**: languages, frameworks, key dependencies
    - **Architecture**: project structure, main components, how they connect
    - **Patterns**: coding conventions, naming, file organization
    - **Relevant context for <user's topic>**: files, modules, or patterns directly related to what we're building
    - **Recent activity**: last 5-10 commits, any in-progress work
    - **Existing tests**: test framework, test locations, coverage approach

    Keep the summary under 500 words. Focus on what someone designing a new feature would need to know.
    Do NOT include raw file contents — summarize.
```

Use the returned summary as your project understanding for the rest of brainstorming. If you need specific details during design questions, you can read individual files directly — but avoid bulk exploration in the main session.

## The Process

**Understanding the idea:**

- Use the project summary from the Explore subagent as your starting context
- Before asking detailed questions, assess scope: if the request describes multiple independent subsystems (e.g., "build a platform with chat, file storage, billing, and analytics"), flag this immediately. Don't spend questions refining details of a project that needs to be decomposed first.
- If the project is too large for a single spec, help the user decompose into sub-projects: what are the independent pieces, how do they relate, what order should they be built? Then brainstorm the first sub-project through the normal design flow. Each sub-project gets its own spec → plan → implementation cycle.
- For appropriately-scoped projects, ask questions one at a time to refine the idea
- Prefer multiple choice questions when possible, but open-ended is fine too
- Only one question per message - if a topic needs more exploration, break it into multiple questions
- Focus on understanding: purpose, constraints, success criteria

**Exploring approaches:**

- Propose 2-3 different approaches with trade-offs
- Present options conversationally with your recommendation and reasoning
- Lead with your recommended option and explain why

**Presenting the design:**

- Once you believe you understand what you're building, present the design
- Scale each section to its complexity: a few sentences if straightforward, up to 200-300 words if nuanced
- Ask after each section whether it looks right so far
- Cover: architecture, components, data flow, error handling, testing
- Be ready to go back and clarify if something doesn't make sense

**Design for isolation and clarity:**

- Break the system into smaller units that each have one clear purpose, communicate through well-defined interfaces, and can be understood and tested independently
- For each unit, you should be able to answer: what does it do, how do you use it, and what does it depend on?
- Can someone understand what a unit does without reading its internals? Can you change the internals without breaking consumers? If not, the boundaries need work.
- Smaller, well-bounded units are also easier for you to work with - you reason better about code you can hold in context at once, and your edits are more reliable when files are focused. When a file grows large, that's often a signal that it's doing too much.

**Working in existing codebases:**

- Explore the current structure before proposing changes. Follow existing patterns.
- Where existing code has problems that affect the work (e.g., a file that's grown too large, unclear boundaries, tangled responsibilities), include targeted improvements as part of the design - the way a good developer improves code they're working in.
- Don't propose unrelated refactoring. Stay focused on what serves the current goal.

## After the Design

**Documentation:**

- Write the validated design (spec) to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
  - (User preferences for spec location override this default)
- Use elements-of-style:writing-clearly-and-concisely skill if available
- Commit the design document to git

**Spec Review (subagent):**
After writing the spec document, dispatch the design-spec-review subagent to handle the entire review loop. This keeps all review round-trips out of the main session's context.

```
Agent tool:
  subagent_type: "superpowers:design-spec-review"
  description: "Spec review for design doc"
  prompt: |
    spec_path: <absolute-path-to-design-doc>
    reviewer_prompt_path: <absolute-path-to-spec-document-reviewer-prompt.md>
```

**Handle result:**
- `approved`: proceed to Codex design review
- `issues_remaining`: surface to user for guidance (the subagent exhausted 5 rounds)

**Codex Design Review (subagent):**
After spec review passes, create a Codex thread and dispatch the codex-design-review subagent. See `lib/codex-integration.md` for the underlying protocol.

1. **Create Codex thread** in the main session:
   ```
   Agent tool:
     subagent_type: "superpowers:codex-agent"
     description: "Init Codex thread for design review"
     prompt: |
       mode: init
       profile: xhigheffort
   ```
   Save the returned `thread_id`. If `status: unavailable`, skip Codex review and proceed (inform user).

2. **Dispatch codex-design-review** with the thread_id:
   ```
   Agent tool:
     subagent_type: "superpowers:codex-design-review"
     description: "Codex design review"
     prompt: |
       spec_path: <absolute-path-to-design-doc>
       summary: <1-2 sentence summary of what is being designed>
       thread_id: <thread_id from step 1>
       worktree_path: <absolute-path-to-worktree if applicable>
   ```

**Handle result:**
- `pass` or `pass-with-flags`: proceed to implementation handoff
- `fail` with remaining issues: surface to user for guidance (the subagent exhausted 5 rounds)
- Codex `unavailable`: the subagent returns `pass` with a note — proceed (inform user)

**Implementation:**

- Invoke the writing-plans skill to create a detailed implementation plan
- Do NOT invoke any other skill. writing-plans is the next step.

## Key Principles

- **One question at a time** - Don't overwhelm with multiple questions
- **Multiple choice preferred** - Easier to answer than open-ended when possible
- **YAGNI ruthlessly** - Remove unnecessary features from all designs
- **Explore alternatives** - Always propose 2-3 approaches before settling
- **Incremental validation** - Present design, get approval before moving on
- **Be flexible** - Go back and clarify when something doesn't make sense

## Visual Companion

A browser-based companion for showing mockups, diagrams, and visual options during brainstorming. Available as a tool — not a mode. Accepting the companion means it's available for questions that benefit from visual treatment; it does NOT mean every question goes through the browser.

**Offering the companion:** When you anticipate that upcoming questions will involve visual content (mockups, layouts, diagrams), offer it once for consent:
> "Some of what we're working on might be easier to explain if I can show it to you in a web browser. I can put together mockups, diagrams, comparisons, and other visuals as we go. This feature is still new and can be token-intensive. Want to try it? (Requires opening a local URL)"

**This offer MUST be its own message.** Do not combine it with clarifying questions, context summaries, or any other content. The message should contain ONLY the offer above and nothing else. Wait for the user's response before continuing. If they decline, proceed with text-only brainstorming.

**Per-question decision:** Even after the user accepts, decide FOR EACH QUESTION whether to use the browser or the terminal. The test: **would the user understand this better by seeing it than reading it?**

- **Use the browser** for content that IS visual — mockups, wireframes, layout comparisons, architecture diagrams, side-by-side visual designs
- **Use the terminal** for content that is text — requirements questions, conceptual choices, tradeoff lists, A/B/C/D text options, scope decisions

A question about a UI topic is not automatically a visual question. "What does personality mean in this context?" is a conceptual question — use the terminal. "Which wizard layout works better?" is a visual question — use the browser.

If they agree to the companion, read the detailed guide before proceeding:
`skills/brainstorming/visual-companion.md`
