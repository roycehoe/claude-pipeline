---
name: feat-audit
description: Cross-checks plans/{feature-name}-execution.md against the feat-plan output to verify coverage, fidelity, constant drift, state machine guard completeness, resolved tradeoffs, and each step's self-consistency. Runs a file existence preflight before semantic checks. Flags stale paths from codebase drift as blockers. Produces a structured gap report at plans/{feature-name}-audit.md. Blockers must be resolved before feat-execute runs. On failure, the gap report is formatted for feat-execution to consume on re-run.
when_to_use: Invoke explicitly with /feat-audit after /feat-execution and before /feat-execute, or run as the pipeline gate when /feat-pipeline orchestrates it. Third step of the feat- pipeline.
allowed-tools: Read Bash(find *) Bash(ls *) Glob Grep
---

# Plan Audit

Cross-check `plans/{feature-name}-execution.md` against the feat-plan output and produce a structured gap report. A single blocker is a FAIL. Do not proceed to /feat-execute until all blockers are resolved.

## Cost and model routing: read before running

This skill has no Agent tool: every phase runs inline in whatever agent invokes it. It therefore inherits the caller's model, and on an Opus/Fable session that is the single largest avoidable cost in the pipeline. Route deliberately:

| Phases | Work | Model |
|---|---|---|
| 1, 1b, 3, 3b, 6, 11b | Mechanical: path existence, literal extraction, string comparison | **Haiku** |
| 4, 5, 7, 8, 9, 10, 11 | Semantic judgment against the plan's meaning | **Sonnet** |
| Re-audit after a FAIL | Confirming fixes landed without regression | **Sonnet**; escalate only if a finding is genuinely subtle |

If an orchestrator wraps this skill and fans phases out to subagents, **every worker's model must be set explicitly.** Never let a worker inherit the session model.

**Phases 4-9 are the reason this stage exists. Do not skip or downgrade them below Sonnet.** They are the only place the plan's state machine and its cross-step ordering claims are read against the execution steps. `feat-execute` receives neither as input, so nothing downstream can recover a guard the plan enumerated and a step silently dropped.

## Reading large inputs

Execution plans routinely exceed 1000 lines. Before reading one, get its length (`wc -l`), then read it in explicit `offset`/`limit` chunks of ~450 lines. Reading a large plan in one call risks truncation, and a truncated read that silently drops the later commit steps produces a clean-looking audit of half a document. Note the actual line count in the gap report so a reader can tell the whole file was covered.

## Phase 1: Locate inputs

**Plan-feature output:** If a path was passed as `$ARGUMENTS`, read that file. Otherwise, scan `plans/` for `.md` files not ending in `-execution.md`, `-audit.md`, or `-tests.md`. Look for one with a Clarifications section, an Implementation Steps section, and a Resolved Tradeoffs section. These are the markers of plan-feature output. If multiple candidates exist, list them and ask the user which to use. If none exist, stop. Tell the user to run `/feat-plan` first. Derive the feature name from its filename.

**Execution plan:** Read `plans/{feature-name}-execution.md`. If it does not exist, stop. Tell the user to run `/feat-execution` first.

**Pipeline state:** Read `plans/{feature-name}.pipeline.json`. If the file does not exist, or if `feat-execution.status` is not `completed`, stop. Tell the user to run `/feat-execution` first.

**Brief:** Check if `plans/{feature-name}-brief.md` exists. If found, read it. Extract every locked decision: terminology confirmed during interrogation, explicit constraints, and decisions recorded as resolved. Store these as `BR-NNN` items for Phase 9.

Read all located files in full before proceeding.

---

## Phase 1b: File existence preflight [Blocker]

Scan every commit step in the execution plan for file paths. A file path is any string that:
- Appears after `**Files:**` in a commit step
- Ends in a recognised source extension (`.py`, `.ts`, `.js`, `.tsx`, `.jsx`, `.go`, `.rb`, `.java`, `.sql`, etc.)
- Is listed explicitly as a path segment (e.g. `app/models/user.py`, `src/components/Foo.tsx`)

Skip paths being **created** in that commit step. A new file won't exist yet, and that's expected. A file is "new" if its implementation detail says it introduces the file (phrases like "create", "add new file", "does not yet exist").

For every **existing** (non-new) file path: use `find` or `ls` to verify it exists in the current working tree.

Produce two lists:

**Found:** file path, commit step(s) that reference it.

**Not found:** file path, commit step(s) that reference it.

Flag each missing existing file as a blocker in this format (for the Phase 12 gap report):

```
B-NNN. File existence: `{path}` referenced in Step N does not exist in the codebase
Requirement: Step N, {commit title}
Gap: The execution plan modifies a file that is not present at the expected path.
Required fix: Confirm the correct path (grep or find) and update Step N, or confirm the file is introduced by an earlier step in this plan.
```

If all referenced existing files are found, note: "File existence preflight: all N referenced files found." and proceed.

If any files are missing, record them as blockers and continue through all remaining phases. Collect every issue before writing the gap report.

---

## Phase 2: Extract from plan-feature output

Record the following:

1. **Requirements.** Every distinct implementation requirement in the Implementation Steps section. Number them R-001, R-002, etc.
2. **State machine.** If present: every valid transition, every invalid transition, and the guard enforcing each. If the State Machine section is absent, note that no state machine was specified.
3. **Volatile constants.** Every value flagged during planning as a business rule likely to change: retention periods, numeric thresholds, string literals, enum values. Include confirmed values.
4. **Resolved tradeoffs.** Each decision and the chosen option.
5. **Assumptions.** All listed assumptions.
6. **Gaps and breakage.** All listed gaps and breakage items.

---

## Phase 3: Extract from execution-plan.md

Record:

1. **Commit steps.** Every numbered commit step with its title and implementation detail.
2. **Constant values.** Every specific value used in commit steps that corresponds to a volatile constant from Phase 2.
3. **Guards and conditions.** Every guard condition, validation check, or exception raise described in commit steps.

---

## Phase 3b: Assign cause to each blocker

Before writing the gap report, assign a `Cause` to every blocker identified in Phases 4-9:

- **`spec-ambiguity`**: the plan (`plans/{feature-name}.md`) is contradictory, incomplete, or ambiguous such that no correct execution of it as written could satisfy the requirement. The failure lies in the source spec, not the translation.
- **`execution-gap`**: the plan clearly specified the requirement, but the execution plan missed, misread, or implemented it incorrectly. The failure lies in the translation.

**When genuinely ambiguous:** classify as `execution-gap`. Its retry path (re-run feat-execution) is less disruptive than spec-ambiguity's (re-run feat-plan). Default to the less disruptive route when cause is unclear.

Record the assigned cause alongside each blocker; use it in the blocker template (Phase 12) and recovery guidance (Phase 13).

---

## Phase 4: Coverage check [Blocker]

For each requirement R-NNN extracted in Phase 2: identify which commit step(s) in execution-plan.md implement it.

Flag as a blocker when:
- No commit step implements the requirement
- A commit step partially implements it but omits a material component

For each gap, record which requirement is uncovered and what a correct commit step must address to close it.

---

## Phase 5: Fidelity check [Blocker]

For each requirement R-NNN and its mapped commit step(s): does the implementation correctly interpret the specification?

Flag as a blocker when:
- The commit step implements a materially different design than the spec requires (wrong layer, wrong approach, wrong entity)
- The commit step inverts a condition (guards when it should allow, allows when it should guard)
- The commit step omits a constraint explicitly stated in the requirement

Do not flag stylistic differences or implementation choices consistent with the spec's intent.

---

## Phase 6: Constant drift check [Blocker]

For each volatile constant extracted in Phase 2: find where execution-plan.md uses it and compare values.

Flag as a blocker when:
- execution-plan.md uses a different value than what plan-feature confirmed
- execution-plan.md does not use the constant at all in any step that requires it

---

## Phase 7: State machine guard check [Blocker]

If no state machine was specified in Phase 2, skip this phase.

For each invalid transition listed in the State Machine section: identify the commit step in execution-plan.md that implements the guard enforcing it.

Flag as a blocker when:
- No commit step implements the guard for an invalid transition
- A commit step implements the state change but omits the guard condition entirely

---

## Phase 8: Resolved tradeoffs check [Blocker]

For each resolved tradeoff in plan-feature: does execution-plan.md implement the chosen option?

Flag as a blocker when:
- A commit step implements the rejected alternative
- A commit step implements a hybrid that was not the chosen option

---

## Phase 9: Brief fidelity check [Blocker]

Skip this phase if no brief was found in Phase 1.

For each locked decision `BR-NNN` extracted from the brief: does the plan-feature output (`plans/{feature-name}.md`) honour it?

Flag as a blocker when:
- The plan contradicts a locked decision (uses a different term, proposes a design that was explicitly ruled out, or changes a confirmed constraint)
- The plan silently omits a decision that would materially affect implementation (e.g. a confirmed scope boundary that the plan's steps implicitly cross)

Do not flag where the plan is silent on a brief decision that has no bearing on implementation steps.

---

## Phase 10: Scope creep check [Warning]

For each commit step in execution-plan.md: does it trace back to a requirement in plan-feature?

Flag as a warning when:
- A commit step has no corresponding requirement in plan-feature
- A commit step introduces changes to files or systems not mentioned in plan-feature

Do not flag steps implementing a clearly necessary dependency (e.g. a migration preceding an application-layer change) even if plan-feature didn't enumerate it explicitly.

---

## Phase 11: Assumption check [Warning]

For each assumption listed in plan-feature: does any commit step in execution-plan.md contradict it?

Flag as a warning when:
- A commit step explicitly implements something inconsistent with a listed assumption
- A commit step modifies a system the assumption treats as stable

---

## Phase 11b: Step self-consistency check [Blocker]

**Mechanical. No convention docs, no judgment.** For each commit step, compare the step's `**Files:**` list against its own Implementation detail prose and flag any disagreement:

1. **File in the list, forbidden by the prose.** The prose says a listed file is "not touched by this commit", "unchanged", "no diff", or names it only to forbid an edit. An executor resolving the contradiction in favour of the Files list writes exactly the change the prose forbids.
2. **File named in the prose as edited, absent from the list.** The prose describes writing to a path the Files list omits.

Use this blocker format:

```
B-NNN. Step self-consistency: Step N's Files list disagrees with its own implementation detail
Requirement: Step N, {commit title}
Gap: Files lists `{path}`, but the implementation detail says {quote the contradicting phrase}.
Required fix: Remove `{path}` from the Files list, or correct the prose to describe the edit, whichever matches intent.
Cause: execution-gap
```

**Why this phase is mechanical and why it is the one kept.** Nothing downstream can catch this class. `feat-execute`'s cross-reference gate compares the *staged diff* against the plan's Files list and implementation detail, so an executor that resolves a self-contradictory step by following the Files list produces a diff that matches the Files list exactly, and the gate passes clean on a step that just did what its own prose forbade. Only a check that reads the plan against itself, before any diff exists, sees it.

**Convention compliance is deliberately NOT checked here** (this phase previously did so). It is dropped for two reasons, both measured:
- `feat-execute`'s convention check (Phase 5.5, once per feature over the whole branch diff) does the same job against better evidence, namely real committed code judged by a fresh subagent with the spec withheld, rather than against plan prose.
- Doing it here re-derives, per feature and forever, catches whose real cause is a stale convention doc. When a rule is settled but the checked-in doc still states the superseded version, the fix is to update the convention docs once, not to pay an audit to rediscover the ruling on every feature. If this phase starts flagging convention drift again, that is a signal the convention docs need syncing, not that the phase should come back.

---

## Phase 12: Write gap report

Write `plans/{feature-name}-audit.md`. Create the `plans/` directory if it does not exist.

Write each blocker to describe what is missing or incorrect in the execution plan: "the execution plan does not cover X" or "step N implements a different approach than the spec requires." Do not attribute fault to the author; the goal is a gap report feat-execution can act on directly.

```markdown
# Plan Audit: {Feature Name}

**Verdict:** PASS | FAIL
**{N} blocker(s) · {N} warning(s)**
**Audited:** {ISO-8601 timestamp}

---

## Blockers

### B-NNN. {Check type}: {one-line description}
**Requirement:** {R-NNN, exact text from plan-feature, or transition/constant/tradeoff reference}
**Gap:** {what is wrong or missing in execution-plan.md}
**Required fix:** {what a correct execution step must address, specific enough for plan-execution to act on}
**Cause:** spec-ambiguity | execution-gap

---

## Warnings

### W-NNN. {Check type}: {one-line description}
**Reference:** {what in execution-plan.md triggered this warning}
**Issue:** {what the concern is}
**Recommendation:** {suggested resolution}

---

## Coverage Map

| Requirement | Description | Covered by |
|---|---|---|
| R-001 | {description} | Step N, {commit title} |
| R-002 | {description} | ❌ Not covered |
```

Confirm the written path to the user.

Update `plans/{feature-name}.pipeline.json`, merging the following entry into the `pipeline` object. Use `passed` if verdict is PASS, `failed` if FAIL:

```json
"feat-audit": { "status": "passed|failed", "blockers": {N}, "output": "plans/{feature-name}-audit.md", "timestamp": "{ISO-8601}" }
```

Print a one-line pipeline state from the updated `plans/{feature-name}.pipeline.json`:

`Pipeline: {grill →} plan {icon} → execution {icon} → audit {icon} → execute {icon}`

Use `✓` completed/passed · `✗` failed · `○` not started. Omit grill-with-docs if the key is absent from the JSON.

---

## Phase 13: Recovery guidance

**If verdict is PASS:** Tell the user the execution plan is clear to proceed. Next step: `/feat-execute`.

**If verdict is FAIL:** Present the blockers. Scan the blockers for their `Cause:` field:

- **If any blocker has `Cause: spec-ambiguity`:** Instruct the user to re-run `/feat-plan` (the spec needs revision), then `/feat-execution`, then re-audit. The gap report's `Required fix` entries describe what the plan must clarify.
- **If all blockers have `Cause: execution-gap`:** Instruct the user to run `/feat-execution --gap-report plans/{feature-name}-audit.md`, targeted repair mode, which skips phases unaffected by the blockers and revises only the failing steps. Then re-audit.

Do not proceed to /feat-execute. The gap report gates next steps.
