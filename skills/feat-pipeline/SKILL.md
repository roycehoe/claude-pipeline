---
name: feat-pipeline
description: Runs the full feature development pipeline end-to-end: grill-with-docs → feat-plan → feat-execution → feat-audit (with one auto-retry on blockers) → feat-execute → feat-validate → type-check gate → simplify. Pauses at the Stage 3 and Stage 6 gates, for spec clarification, and on unresolvable errors. Drives a feature from description to committed code in a single command.
when_to_use: Invoke explicitly with /feat-pipeline to drive a feature from description to committed code through all pipeline stages without manual handoff.
allowed-tools: Read Write Bash(find *) Bash(grep *) Bash(ls *) Bash(git log *) Bash(git diff *) Glob Grep Agent Skill
---

# Feature Pipeline

Run all pipeline stages in sequence, pausing only for genuine human input: spec clarification in the early phases, feat-execution's constants, symbol and commit-order gates in Stage 3, the Stage 6 safety confirmation, and unresolvable ruff or pre-commit hook failures during execution. Do not pause between stages.

## When to enter this pipeline

Enter feat-pipeline when the feature trips ANY of feat-quick's size tripwires, the exact mirror of feat-quick's bail condition, so the two lanes partition cleanly:
- more than two stack layers, OR
- a state machine / lifecycle / status transition, OR
- a non-trivial migration (rename, non-nullable, backfill, or a column with existing readers/ordering, though an additive nullable column with no readers does not qualify), OR
- the build plus an honest review will not fit one context window.

If none of these hold and the spec only needs locking, use `/feat-quick` instead.

## Setup

**Re-entry check:** Before announcing, scan `plans/` for any `*.pipeline.json` file; if found, read it and map its `pipeline` keys to stages:

| pipeline.json key | Stage |
|---|---|
| `grill-with-docs` | Stage 1 |
| `feat-plan` | Stage 2 |
| `feat-execution` | Stage 3 |
| `feat-audit` | Stage 4 (status `passed` counts as completed) |
| `feat-execute` | Stage 6 |
| `feat-validate` | Stage 7 (status `passed` counts as completed) |

A `feat-tests` key from a pre-existing pipeline maps to nothing. Stage 5 is removed. Treat it as history and resume from the next real stage.

Find the last key whose status is `completed` or `passed`. If any stages are complete, announce: "Found an in-progress pipeline for **{feature-name}**. Last completed: {stage name}. Resume from Stage {N+1} ({next stage name}), or restart from Stage 1?" Wait for the response; if resuming, skip directly to that stage.

Announce to the user:

> Running full feature pipeline: **grill → plan → execution → audit → execute → validate → type-check → simplify**
> I will pause for spec clarification, Stage 3's constants and commit-order gates, the Stage 6 safety confirmation, and any errors I cannot resolve automatically.

## Cost discipline

The pipeline's dominant cost is not generation. It is the same inputs being re-ingested by each stage. Measured on one run: ~630k subagent tokens produced ~48k of artifacts, a 13:1 ratio. Four rules, in order of measured payoff:

1. **Never let a spawned worker inherit the session model.** Set `model` explicitly on every subagent: Haiku for mechanical work, Sonnet for judgment, Opus/Fable only for a stage that itself does correctness-critical reasoning. Opus-by-inheritance on fan-out is the single largest avoidable overspend. None of the feat- skills declare a model, so they all inherit whatever invoked them.
2. **Write a context pack once, then stop re-reading the source artifacts.** After Stage 2, condense the locked decisions, the verified anchor table, and the gate decisions into one short file. Point later stages at that plus the single artifact under review, not at the full brief + plan + explore + convention-doc family, which is tens of thousands of tokens re-read per cold agent.
3. **Tell every agent the line count and a chunking plan for any file over ~600 lines.** One run lost ~116k tokens to an agent that died reading a 1259-line plan in a single call and produced nothing. One sentence in the prompt prevents it.
4. **Reuse an agent when independence is not required.** Resume a prior agent with `SendMessage` rather than spawning a cold one that re-reads everything. The exception is real: an agent must never audit or verify its own output. Stage 4 and Stage 7 need fresh context. Stages that merely consume an artifact do not.

---

## Stage 1: grill-with-docs

Invoke the `grill-with-docs` skill with `$ARGUMENTS` as the feature description.

**Crisp-spec skip:** If passing a hand-written, already-crisp spec (you can state the one-line acceptance check without reading code) rather than a fuzzy description, skip the grill. Do not invoke grill-with-docs. Start at Stage 2, invoking feat-plan with `$ARGUMENTS` (the spec) directly; feat-plan's brief check handles a brief-less start and runs its terminology challenge only where needed. There will be no `-brief.md`, so any later stage that references the brief falls back to `plans/{feature-name}.md`.

This stage involves active interrogation. Answer clarification questions and scenario challenges as they arise; they are how the brief gets precise.

After the skill completes:
- Confirm `plans/{feature-name}-brief.md` was written.
- Read `plans/{feature-name}.pipeline.json` to record the **feature name**, used to construct every subsequent path.

Announce: "Stage 1 complete. Brief written. Starting feat-plan."

---

## Stage 2: feat-plan

Invoke the `feat-plan` skill with `plans/{feature-name}-brief.md` as the argument.

Because a brief exists, feat-plan skips its own terminology challenge but may still ask material questions where the brief left something unresolved. Answer these as they arise.

After the skill completes:
- Confirm `plans/{feature-name}.md` was written.

Announce: "Stage 2 complete. Feature plan written. Starting feat-execution."

---

## Stage 3: feat-execution

Invoke the `feat-execution` skill with `--auto-approve` as the argument.

In auto-approve mode, feat-execution skips only the pause after each drafted commit step, displaying steps as drafted so you can follow along; Stage 4's audit is the gate that catches problems in them. Three earlier gates still block and need your input: volatile-constants confirmation (Phase 3), any unresolved symbol blockers (Phase 4), and commit-order approval (Phase 5). Answer these as they arise.

After the skill completes:
- Confirm `plans/{feature-name}-execution.md` was written.

Announce: "Stage 3 complete. Execution plan written. Starting feat-audit."

---

## Stage 4: feat-audit (with one auto-retry)

Invoke the `feat-audit` skill.

**If the audit passes:**
Announce: "Audit passed. Proceeding to feat-execute."
Continue to Stage 5.

**If the audit fails:**
1. Read `plans/{feature-name}-audit.md`. Scan for any line matching `**Cause:** spec-ambiguity`.
2. **Spec-ambiguity route** (one or more blockers have `Cause: spec-ambiguity`):
   - Announce: "Audit found N blocker(s) including spec ambiguity. Re-running feat-plan to resolve the spec, then feat-execution."
   - Invoke the `feat-plan` skill with `plans/{feature-name}-brief.md` as argument (or `plans/{feature-name}.md` if no brief exists). Participate in any clarification questions that arise.
   - Invoke the `feat-execution` skill with `--auto-approve`.
   - Invoke the `feat-audit` skill again.
3. **Execution-gap route** (all blockers have `Cause: execution-gap`):
   - Announce: "Audit found N blocker(s). Auto-retrying feat-execution with the gap report."
   - Invoke the `feat-execution` skill with `--gap-report plans/{feature-name}-audit.md --auto-approve`.
   - Invoke the `feat-audit` skill again.
4. **If the second audit passes:** Announce "Audit passed on retry." Continue to Stage 6.
5. **If the second audit still fails:** Stop, present the remaining blockers to the user, and tell them: "The audit failed on both attempts. Resolve the blockers in `plans/{feature-name}-audit.md`, then re-run `/feat-execution` followed by `/feat-audit` before continuing with `/feat-execute`."

---

## Stage 5: feat-tests is removed

**Do not invoke `feat-tests` from this pipeline.** Go straight from a passing audit to Stage 6.

Measured across seven pipeline runs by the author, `feat-tests` produced 4,922 lines of Given/When/Then and **zero attributed catches**. The projects had no test set to run its output against, and one prior audit was found *deferring* a live assumption to it ("if /feat-tests exercises the ownership path it will confirm this at runtime"), a runtime confirmation that structurally never happens. It became a place findings were filed rather than resolved.

The two things it uniquely produced are re-homed, at no extra cost:
- **Landmine callouts** (a design decision that is a known trap, e.g. "branching on `not is_active` would mark a non-terminal run failed and lose a paid-for result") go in the audit's NOTES section.
- **Deferred runtime assumptions** go in the PR description, where a human reviewer sees them.

Invoke `/feat-tests` by hand if a test harness exists and you intend to run the cases. Do not run it to generate a document nothing consumes.

---

## Stage 6: feat-execute (pipeline mode)

**Safety confirmation:** Before invoking feat-execute, present the following block and wait for the user to type GO (case-insensitive); answer any questions, then re-present the block. Do not invoke feat-execute until GO is received.

```
## Before we begin

This execution session will:
- NOT push to any remote branch. All commits stay local
- NOT run data migrations. the migration command is run manually by a human after review
- NOT connect to a live database. Static analysis and file reads only

Note: `Bash(git *)` permission is active, which technically permits `git push`.
This gate exists because the tool permission does not block it. Only this confirmation does.

Type GO to proceed.
```

After GO is received, invoke the `feat-execute` skill with `--pipeline` as the argument.

In pipeline mode, feat-execute skips per-commit review pauses and commits each step straight through, but still stops for:
- Ruff failures that cannot be auto-corrected
- Pre-commit hook failures that cannot be resolved automatically

Participate in any errors that surface during this stage.

---

## Stage 6.5: Amendment cascade

After feat-execute completes, read `plans/{feature-name}.pipeline.json` and check whether `feat-execute.amendments` is greater than 0.

**If amendments were written:**
1. Read `plans/{feature-name}-execution.md` and display a formatted summary of the `## Amendments` section. For each amendment, show which execution plan step it deviated from and what changed.
2. Announce "Amendments found, carrying them into the PR description." Do **not** re-run `feat-tests` (Stage 5 is removed). Instead, record each amendment as a PR-description line so the reviewer sees where the shipped code deviated from the audited plan. Then proceed to Stage 7.

**If no amendments were written:** proceed directly to Stage 7.

---

## Stage 7: feat-validate

Invoke the `feat-validate` skill.

feat-validate compares the committed code against the feature plan requirements and reports any requirement with no corresponding code change.

**If validate passes:** Announce "Validation passed. Proceeding to simplify." Continue to Stage 8.

**If validate fails:** Stop, present the required actions from `plans/{feature-name}-validate.md`, and tell the user: "Validation found uncovered requirements. Add missing commits or amendments, then re-run `/feat-validate` before continuing with `/simplify`."

---

## Stage 7.5: Type-check gate [Blocker]

**Run this before simplify, on the committed code.** Nothing earlier in this pipeline type-checks anything: feat-audit reads prose, feat-execute runs `ruff` (which does not type-check), and feat-validate maps requirements to commits. That leaves an entire defect class uncaught to the end.

For every file the branch touched, collect type diagnostics, via the LSP if available, otherwise the project's checker. Then triage each one:

- **Real defect → fix and commit.** The highest-value case is a *widened producer with an unwidened consumer*: a plan step that adds a variant to a union, enum, or status map, where the schema or function that consumes it was never widened to match. This passes every prose-level check because both sides are individually correct; only the type relationship between them is wrong. It can be silent at runtime, since overlapping enum values may coerce and emit the right string today, then start raising the moment a non-overlapping member is added.
- **Faithful mirror of an existing pattern → leave it.** If the execution plan required copying a repo pattern verbatim, and the same diagnostic fires on the mirror source, it is pre-existing and must not be "fixed". Doing so breaks the mirror the plan mandated. Verify by checking the source it was copied from.
- **Known false positive → leave it, and say which class.** Monorepo path resolution, an unresolvable import for a module that genuinely does not exist yet (a contract-first dependency), and stale diagnostics from a mid-edit read are all expected.

**Diagnostics may not reach a subagent.** In some configurations they surface only in the main loop, so an executing subagent is blind to them. Run this gate in the main agent, not delegated.

---

## Stage 8: simplify

Invoke the `simplify` skill.

simplify reviews recently changed code for unnecessary complexity, duplication, and quality issues, and stages and commits any fixes it finds.

---

## Completion

After Stage 8 finishes:

1. Report: feature name, total commits made.
2. List anything that needs a human before merge: unresolved external dependencies, required env/settings rollout, migrations to apply, credentials to rotate, and every decision the PR description must state.
