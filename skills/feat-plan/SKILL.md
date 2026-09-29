---
name: feat-plan
description: Given a description of a feature or change, produces a structured high-level implementation plan grounded in the actual codebase. Explores relevant code, reads CLAUDE.md for conventions, asks clarifying questions one at a time with recommended answers, runs adversarial self-review with user-facing scenario stress-tests, surfaces unresolved tradeoffs, and writes the finalised plan to plans/{feature-name}.md.
when_to_use: Invoke explicitly with /feat-plan to produce a high-level implementation plan for a feature before coding, or run when /feat-pipeline orchestrates it. First step of the feat- pipeline.
allowed-tools: Read Write Bash(find *) Bash(grep *) Bash(ls *) Bash(git log *) Bash(git diff *) Bash(git show *) Glob Grep WebSearch WebFetch
---

# Plan Feature

Produce a structured, high-level implementation plan. Do not write code. The phases below are working scaffolding. They guide your process but must not appear as section headers in the output document.

Maintain a running **Decisions so far** scratch list throughout Phase 1. Each time a question resolves, whether by human answer, codebase evidence, or your own judgment, add it to the list immediately. Reference this list during Phase 2 and use it to populate the Clarifications section of the output.

## Preflight: `.gitignore` check

Before writing any output, verify that `plans/` will not be accidentally committed:

1. Run `find . -maxdepth 1 -name ".gitignore"` to check whether a `.gitignore` file exists at the project root.
2. If found, run `grep -n "^plans/" .gitignore` to check for a non-commented `plans/` entry.
3. **If `.gitignore` is absent, or `plans/` is not listed as an uncommented entry:**
   Warn the user: "`.gitignore` does not include `plans/`. Artifacts written to `plans/` should not be committed. Add `plans/` to `.gitignore`? [Y/n]"
   - If the user accepts and `.gitignore` exists: append `plans/` to `.gitignore`.
   - If the user accepts and no `.gitignore` exists: create `.gitignore` containing `plans/`.
   - If the user declines: proceed without changes.
4. **If `plans/` is already listed:** note silently and continue.

This check is non-blocking. Proceed to Phase 1 regardless of the outcome.

## Phase 1: Clarification round

**Update mode:** If `$ARGUMENTS` contains `--update`, the user is providing corrections to an existing plan. Read the existing `plans/{feature-name}.md` (derive the feature name from any other argument, or ask if ambiguous). Read the correction list from the conversation. Expect a numbered list of changes. Record each correction as a resolved decision in **Decisions so far**. Skip Steps A, B, and C entirely. All prior decisions are locked; only the corrections are new. Skip Phase 2 exploration unless a correction explicitly requires checking a new file. Proceed directly to Phase 3 (draft), applying the corrections to the relevant sections of the existing plan. After re-drafting, run Phase 4 (adversarial review) on changed sections only. Overwrite the existing plan file (ask first if the plan has a `feat-execution.status: completed` entry, since this invalidates downstream work). Then null out all downstream pipeline entries in `plans/{feature-name}.pipeline.json`:

```json
"feat-execution": null,
"feat-audit": null,
"feat-execute": null,
"feat-validate": null
```

Tell the user: "Plan updated. Downstream pipeline steps have been reset. Re-run `/feat-execution` to rebuild the execution plan."

**Brief check:** Before beginning, check whether `$ARGUMENTS` points to a `*-brief.md` file, or whether a `*-brief.md` file exists in `plans/`. If a brief is found, read it in full. It is the output of a prior `/grill-with-docs` session. Treat it as the authoritative feature description with locked terminology. Skip Step A entirely. Scan the brief for any material questions (Step B) it leaves unresolved. If none remain, proceed directly to Phase 2. Record all decisions in the brief as resolved in your **Decisions so far** list before proceeding.

If no brief is found, read the feature description in full, including any amendment or addendum sections. Then proceed through three steps in order:

### Step A: Terminology challenge

Scan the feature description for vague, overloaded, or undefined terms. For each one, propose a precise canonical name and confirm it with the user before asking anything else. Example: "You used 'account'. Do you mean the Organisation record or the User record? I'll use whichever you confirm as the canonical term throughout the plan."

Only flag terms where ambiguity would lead to a materially different plan. Skip general programming vocabulary.

### Step B: Identify material questions

After terminology resolves, identify anything where one answer leads to a fundamentally different plan. Ask only what's material. Don't front-load every conceivable question.

- If the feature changes how existing data is presented in responses (masking, redacting, filtering), always ask: does that change apply at all places the response is constructed, or only at specific call sites? The scope of the answer determines how many layers must be updated.

**Scan for contradictions.** Check the spec body against any amendments. If the spec body and an amendment say different things about the same behaviour, flag it as a question.

### Step C: Ask questions one at a time, with a recommendation

Before asking each question: (1) grep the codebase for an existing convention or pattern that answers it, and if the codebase answers it definitively, record as a resolved decision in **Decisions so far** and move to the next question without asking; (2) WebSearch or WebFetch the canonical documentation for the relevant technology (language spec, library docs, framework reference) and retrieve what it prescribes. Show what the codebase shows and what the canonical docs say, along with their tradeoffs. Then state your recommended answer.

Ask the most material remaining unresolved question. Wait for the user's response before continuing.

When the user answers, immediately record the resolved decision in your **Decisions so far** list.

Continue until every material question is resolved. Stop as soon as you have enough to proceed. Do not ask questions whose answers would not change the plan.

## Phase 2: Explore

**CLAUDE.md:** Read `CLAUDE.md` from the project root. Extract naming conventions, layer responsibilities, commit message format, and any codebase-specific rules. The plan must not propose an architecture that conflicts with these conventions. If no `CLAUDE.md` exists, proceed without it.

Search the codebase for everything the feature will touch:
- Layers, modules, and files that must change
- Existing patterns the plan must respect or replace
- Data models, schemas, and interfaces that will be affected
- Dependencies between components that constrain step ordering
- Anything that will break if the change is made naively

Use grep and file reads. Do not plan against assumptions. Plan against evidence.

**Pattern fidelity:** When the feature adds fields or methods to an existing entity, examine that entity's existing fields and methods first. Follow established patterns rather than inventing new ones.

**Reuse check:** When a new method could theoretically reuse an existing one, read the existing method's implementation, not just its signature, to determine whether reuse is valid, and if not, why.

**Response transformation scope:** When the feature changes how data is presented in API responses, grep for every place the affected response type is constructed across the codebase. Assuming a single serialization path is the most common source of data-exposure bugs in this class of feature.

**State machines:** If the feature introduces or modifies states (a lifecycle, a status field, a workflow), enumerate every state and every valid transition before drafting steps. Explicitly list which transitions are invalid and what guard enforces each one. A missing guard is the most common source of bugs in lifecycle features. Do not leave any transition implicit. This enumeration must appear in the output document.

**Test seams:** Identify the public interface(s) at which this feature's behaviour will be observed and tested, the seams. Prefer existing seams over new ones; use the highest seam that still isolates the behaviour; keep the count minimal (one is ideal). Put each proposed seam to the user using the Phase 1 Step C format, one at a time, with your recommendation, and record each confirmed seam in **Decisions so far**. Seams are recorded for the PR description and for manual verification. `/feat-tests` is no longer a pipeline stage, so do not treat "a test will cover this" as a resolution for anything.

**Follow-up questions:** If exploration surfaces new unknowns that would materially change the plan, or contradictions between the spec and the codebase, ask them one at a time using the same format as Phase 1 Step C: one question, your recommendation, wait. Record each resolved decision in **Decisions so far**.

**Exploration artifact:** As you read files during this phase, maintain a running record of: every file opened (path + one-line reason), every named symbol confirmed to exist (name, file, line number), and which layers or modules the feature touches. You will write this to `plans/{feature-name}-explore.md` in the Output step, so later skills skip redundant grepping.

## Phase 3: Draft

Produce a numbered list of steps. Each step:
- States what is being done in one or two sentences
- Is agnostic to the application codebase. Do not name specific frameworks, ORMs, or internal helpers; say "join tables and query" not "use [ORM] to join"
- Is aware of external dependencies. When a step involves a database query, an external API call, or a storage operation, describe the interaction clearly enough that the correct operation is unambiguous
- Includes an explicit "do not do X" constraint when omitting something is non-obvious (e.g. an internal field that must not be exposed in an API response)
- Includes external dependency specifics when Phase 2 reveals that the correct implementation is non-obvious or likely to be done wrong
- Names the specific files, modules, or layers involved where it aids clarity
- Is ordered so no step assumes work from a step that comes after it

## Phase 4: Adversarial review

### Internal critique

Critique the plan as a senior engineer would. Answer each question, then revise the steps in response:

- What are the weakest assumptions?
- What would a reviewer push back on?
- What does the plan not account for that the codebase evidence suggests it should?
- Are there unresolved tradeoffs, decisions the plan implicitly makes that the user may want to make explicitly?
- **State transitions:** Is every guard accounted for, including attempts to trigger a transition from an unexpected current state? List any unguarded transition explicitly.
- **Volatile constants:** Does the plan reference any value (time period, count, threshold, string literal) that is a business rule likely to change? Flag it as a configuration candidate.
- **Background and async failure paths:** If any step runs outside a normal request (CRON job, background task, event handler), what happens when a dependency returns unexpectedly or is missing? Does the plan account for this?
- **Notification decisions:** Does every step that changes state, whether via an endpoint or a background job, have an explicit decision about whether to notify users? State the decision even if it is "no notification."
- **User-facing text:** Does any step produce a notification, error message, or API string visible to end users? Does that text inadvertently expose internal implementation details: field names, retention SLAs, database states, or admin-only concepts?

Stay at the architectural and tradeoff level. Do not prescribe internal ordering within helper functions or other implementation details. Those belong in a subsequent coding conversation.

For unresolved tradeoffs: note them in the Resolved Tradeoffs section as pending, and include them in the follow-up question batch if a second clarification round is needed.

### Scenario stress-test

Invent 2-3 concrete user or system scenarios that probe the plan's edges: situations the internal critique cannot catch because they require a human to confirm intent. Present them to the user one at a time. For each:

- Describe the scenario concisely ("A user submits the form while a background job is mid-run on the same record")
- State whether the current plan handles it correctly, and how
- Ask the user to confirm or correct

Revise the steps in response to any answers that reveal a gap.

### ADR offer

After the stress-test, apply this three-condition test to every significant decision made during planning:

1. **Hard to reverse**: the cost of changing your mind later is meaningful
2. **Surprising without context**: a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off**: there were genuine alternatives and a specific one was chosen for specific reasons

If any decision passes all three, offer to write an ADR before finalising the plan. Keep ADRs to one paragraph: context, decision, and why. Place them at `docs/adr/NNNN-slug.md`, incrementing from the highest existing number. If the user declines, skip it.

## Phase 5: Gaps and breakage

Separately from the steps, state:
- Anything that would break if the plan were executed as written, that is not already addressed by a step
- Anything the codebase reveals is missing from the plan

If there is nothing to flag, omit this section entirely.

## Phase 6: Assumptions

List every assumption the plan rests on: facts treated as true but not verified from the codebase or the feature description. For each:
- State the assumption
- State what changes if it is wrong

## Output

The output document is a clean plan artifact. Use only the section headers listed below, no others. In particular, do not use phase labels (Phase 1, Phase 2, etc.) or methodology terms as section headers; those are working scaffolding, not document structure.

1. **Clarifications.** Decisions confirmed before or during planning, drawn from the **Decisions so far** list. For each, note whether it came from a human answer, codebase evidence, or planner judgment.
2. **State Machine.** If the feature touches a lifecycle or status field: a table of all states, valid transitions, invalid transitions, and the guard enforcing each. Omit if the feature has no state machine.
3. **Test Seams.** The public interface(s) at which the feature's behaviour will be observed, as confirmed during planning. For each: the seam, why it was chosen (existing vs new; why the highest available), and what it observes. These serve the PR description and manual verification; if a harness exists, `/feat-tests` can structure cases at them rather than re-deciding them. Omit only if the feature has no observable behaviour.
4. **Implementation Steps.** The numbered step list from Phase 3.
5. **Resolved Tradeoffs.** Design decisions made during planning. For each: what was decided and why. Captures both planner judgment calls and decisions resolved by the human. Mark any still-pending unresolved tradeoffs explicitly.
6. **Gaps and Breakage.** From Phase 5. Omit if empty.
7. **Assumptions.** From Phase 6.

Write to `plans/{feature-name}.md`. Create the `plans/` directory if it does not exist. Confirm the path when done.

Also write `plans/{feature-name}-explore.md` using the running record maintained during Phase 2:

```markdown
# Exploration: {Feature Name}

## Files read
- `path/to/file.py`: {one-line reason for reading}

## Symbols confirmed
| Symbol | File | Line |
|---|---|---|
| `ClassName.method_name` | `path/to/file.py` | 123 |

## Layers / modules affected
{Which top-level layers the feature touches and why}
```

Then update `plans/{feature-name}.pipeline.json`. If the file already exists (e.g. written by a prior `/grill-with-docs` run), read it first and merge the new entry. Do not overwrite the existing `pipeline` object. If it does not exist, create it:

```json
{
  "feature": "{feature-name}",
  "pipeline": {
    "grill-with-docs": { "status": "completed", ... },  ← preserve if present
    "feat-plan": { "status": "completed", "output": "plans/{feature-name}.md", "explore": "plans/{feature-name}-explore.md", "timestamp": "{ISO-8601}" }
  }
}
```

Tell the user: "Run `/feat-execution` to translate this plan into atomic commits."

Print a one-line pipeline state from the updated `plans/{feature-name}.pipeline.json`:

`Pipeline: {grill →} plan {icon} → execution {icon} → audit {icon} → execute {icon} → validate {icon}`

Use `✓` completed/passed · `○` not started. Omit grill-with-docs if the key is absent.
