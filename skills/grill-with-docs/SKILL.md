---
name: grill-with-docs
description: Pre-planning interrogation that sharpens a vague feature description into a precise, terminology-locked brief. Challenges the spec against the codebase, stress-tests with concrete scenarios, and writes plans/{feature-name}-brief.md for /feat-plan to consume. Built for features that are fuzzy, span unfamiliar concepts, or arrive as someone else's spec.
when_to_use: Invoke explicitly with /grill-with-docs as optional step 0 of the feat- pipeline, before /feat-plan.
allowed-tools: Read Write Bash(find *) Bash(grep *) Bash(ls *) Bash(git log *) Bash(git diff *) Glob Grep WebSearch WebFetch
---

# Grill With Docs

Interrogate a feature description until the scope, terminology, and boundaries are unambiguous. Produce a tight brief that /feat-plan can consume without re-running its own clarification round.

## Phase 1: Setup

Ask the user for two things if not already provided:
1. The feature description: what they want to build
2. A short kebab-case feature name: used to derive the output filename

If `$ARGUMENTS` is provided, treat it as the feature description and derive the feature name from it.

---

## Phase 2: Explore the codebase

Before interrogating, understand the terrain. Search for:
- Existing code that the feature will touch or depend on
- Entities, models, and concepts the description mentions. Check whether they already exist, and under what names
- Patterns the feature would logically extend or break
- Any existing behaviour that the user's description seems to assume or contradict

Use grep and file reads; take notes to challenge the user's assumptions during interrogation.

---

## Phase 3: Interrogation

Interview the user relentlessly on every aspect until scope, terminology, and boundaries are unambiguous. Walk each branch of the design tree, resolving decision dependencies one by one.

Rules:
- Ask one question at a time. Wait for the answer before continuing.
- For each question, before recommending: (1) grep the codebase for an existing convention or pattern that answers it, and if found definitively, record as resolved and skip asking; (2) WebSearch or WebFetch the canonical documentation for the relevant technology (language spec, library docs, framework reference) for what it prescribes. Show what the codebase shows and what the canonical docs say, with their tradeoffs, then state your recommended answer. The user decides.
- Record each resolved decision immediately in a running **Decisions so far** list.

Apply these lenses in order:

### Sharpen fuzzy language

When the user uses a vague or overloaded term, propose a precise canonical name before asking anything else. "You said 'account'. Do you mean the Organisation record or the User record? I'll use whichever you confirm as the canonical term."

Only challenge terms where ambiguity would lead to a materially different design. Skip general programming vocabulary.

### Challenge against the code

When the user states how something works or what something is, check whether the code agrees; if not, surface the contradiction immediately: "You said partial cancellation is possible, but the code cancels entire Orders. Which is right?"

Do not ask questions that the codebase already answers definitively. Grep first; only ask if the code is ambiguous or silent.

### Identify material questions

After terminology is locked, identify anything where one answer leads to a fundamentally different design. Ask only what is material.

If the feature changes how existing data is presented in responses (masking, redacting, filtering), always ask: does that change apply at all places the response is constructed, or only at specific call sites?

If the feature introduces a shared client, connection pool, or resource needing startup availability (e.g. an HTTP client, cache connection, background scheduler), ask explicitly how it should be initialized. Read `CLAUDE.md` for the project's established startup pattern (e.g. a lifespan context manager, a settings model validator, a module-level singleton) and present those options by name. Lock the answer as a decision in the brief. It must flow through to the execution plan's step for the application entry point.

Scan for contradictions. If the description says two inconsistent things, flag it as a question before moving on.

### Stress-test with concrete scenarios

Invent 2-3 scenarios that probe the feature's edges: situations forcing precision about boundaries between concepts. For each:
- Describe the scenario concisely
- State whether the feature as described handles it, and how
- Ask the user to confirm or correct

Revise your understanding in response to any answer that reveals a gap.

Continue until every material question is resolved. Stop once you have enough for feat-plan to act on without re-asking the same questions.

---

## Phase 4: ADR offer

After the interrogation, apply this three-condition test to every significant decision made:

1. **Hard to reverse**: reversing later would cost meaningfully
2. **Surprising without context**: a future reader would wonder "why did they do it this way?"
3. **The result of a real trade-off**: genuine alternatives existed, and one was chosen for specific reasons

If a decision passes all three, offer to write an ADR before finalising the brief. Keep ADRs to one paragraph: context, decision, why. Place them at `docs/adr/NNNN-slug.md`, incrementing from the highest existing number. If the user declines, skip it.

---

## Phase 5: Write the brief

Write `plans/{feature-name}-brief.md`. Create the `plans/` directory if it does not exist.

After writing, edit the brief for concision (an optional writing-cleanup skill such as `/unslop` works if you have one). This document is the locked spec `/feat-plan` reads, so vague or padded phrasing propagates into the plan and then into the code.

```markdown
# {Feature Name}: Brief

## Feature description

{One paragraph. What is being built, in precise terms, using only the canonical vocabulary resolved during interrogation.}

## Resolved terminology

| Term | Canonical meaning |
|---|---|
| {term} | {precise definition as agreed} |

## Decisions

{Numbered list of every material decision resolved during the session. For each: what was decided and why.}

## Scenarios confirmed

{The 2-3 stress-test scenarios and how the feature handles each, as confirmed by the user.}

## Out of scope

{Anything the interrogation explicitly ruled out. A future reader should know what was considered and rejected.}
```

Confirm the written path to the user.

Write or update `plans/{feature-name}.pipeline.json`:

```json
{
  "feature": "{feature-name}",
  "pipeline": {
    "grill-with-docs": { "status": "completed", "output": "plans/{feature-name}-brief.md", "timestamp": "{ISO-8601}" }
  }
}
```

Tell the user: "Run `/feat-plan plans/{feature-name}-brief.md` to continue."
