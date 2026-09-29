---
name: feat-tests
description: Generates a structured plans/{feature-name}-tests.md from the feat-plan output (behavioral spec) and plans/{feature-name}-execution.md (implementation specifics). Extracts every testable behavior, groups by feature area, and writes complete Given/When/Then cases with prioritized summary table. NOT part of /feat-pipeline. Invoke by hand, and only when a test harness exists to run the cases against.
when_to_use: Invoke explicitly with /feat-tests when a test harness exists and you intend to run the generated cases, or when you specifically want a manual-QA checklist. Removed from /feat-pipeline. Do not run it to produce a document nothing consumes.
allowed-tools: Read Write Bash(find *) Bash(ls *) Glob Grep
---

# Generate Test Plan

Read a feature plan and produce a complete `plans/{feature-name}-tests.md`.

## Before you run this: check it will be consumed

**This skill was removed from `/feat-pipeline`.** Across seven prior pipeline runs it produced 4,922 lines of Given/When/Then and zero attributed catches, because there was no harness to run any of it against.

Run it only if one of these is true:
- A test harness exists and you are going to execute the cases.
- Someone has asked for a manual-QA or PR-review checklist and will actually walk it.

If neither holds, stop and say so rather than generating the document. If the project's conventions say there is no test set and not to write tests unless asked, that is a reason not to run, not a caveat to note in the output.

**Never defer a live assumption to this skill.** A note of the form "if /feat-tests exercises this path it will confirm it at runtime" describes a confirmation that will not happen. Resolve the assumption, or record it in the PR description where a human will see it.

## Phase 1: Read the sources

**Primary source, the behavioral specification:** Read the file given as the argument. If no argument is given, scan the `plans/` directory for a markdown file that is the plan-feature output, one with Clarifications, Implementation Steps, Resolved Tradeoffs, and Assumptions sections. If multiple candidates exist, list them and ask the user which to use. This is the source of truth for what the system should do. Use it for test structure, behavioral requirements, and expected outcomes. Derive the feature name from its filename.

**Supplementary source, the implementation specifics:** Look for `plans/{feature-name}-execution.md` and read it in full if found. Use it only to fill in concrete details for `Then` assertions: exact method names, function signatures, constant values, error codes, and field names. Do not use it as a source for test structure or behavioral requirements. Those come from the primary source only.

**Test seams:** If the plan has a **Test Seams** section, treat the listed seams as the agreed observation points. Structure each case's When/Then to exercise behaviour at those seams rather than proposing new ones. Do not re-decide seams the plan already fixed.

**Pipeline state:** Read `plans/{feature-name}.pipeline.json`. If `feat-audit.status` is not `passed`, stop:
- If status is `failed`: tell the user the audit found `{N}` blocker(s). Resolve them, re-run `/feat-execution` with the gap report, then re-run `/feat-audit` before proceeding.
- If the entry is missing: tell the user to run `/feat-audit` first.

Read both files in full before extracting testable behaviors.

## Phase 2: Extract testable behaviors

Work through every section of the plan and identify every testable behavior. Be exhaustive. A behavior is testable if verifiable by observing system state, output, return values, or side effects.

Go through each category in turn. Do not skip one because the plan seems not to cover it explicitly; it may imply coverage without stating it:

### 2a: Happy paths
Every operation the plan introduces has at least one happy-path test: preconditions met, caller is authorized, and the operation succeeds. If multiple caller types can perform the same operation with identical behavior, each gets its own test. If the operation produces a visible result (a new state, a returned value, a side effect), the test verifies it.

### 2b: Authorization and access control
For every operation, identify who is allowed and who is explicitly or implicitly denied. Each denied role, caller type, or ownership mismatch gets its own test. Do not combine "role A can" and "role B cannot" into one case. Generate one test per denied role per operation; do not assume a role's denial on one operation implies coverage for a different operation, even if both share the same underlying RBAC logic.

Also generate tests for these boundary conditions on every mutating operation:
- **Non-existent entity:** the target entity does not exist → verify the correct not-found result and no side effects.
- **Entity in wrong scope:** the target entity exists but belongs to a different scope than the one in the call (e.g. document belongs to project A, call is against project B) → verify rejection and unchanged state.
- **Caller not a member:** the caller is authenticated but has no membership or role in the target scope → verify rejection.

### 2c: State machine transitions
If the plan introduces or modifies any entity with named states:
- Map every valid transition → one test each (success case)
- Map every invalid transition the plan calls out → one test each (guard enforcement)
- Identify transitions a caller might plausibly attempt from a blocked state (e.g. repeating a one-way transition, acting on an already-terminated entity) → one test each

For each invalid transition test, verify the operation is rejected with the correct result and the entity state is unchanged afterward.

### 2d: Error conditions and result codes
For every named error code, error response value, or named failure result: generate one test per meaningful precondition, not one per error code. Ask two questions: (1) Could two different *inputs* that produce the same error be handled by different branches of the implementation? (2) Could two different *operations* (e.g. start vs. continue, create vs. update) reach the same validation logic via different code paths? If yes to either, each input and each operation is its own test case. Splitting matters: a bug could let one precondition or call site through while correctly blocking another.

Each test verifies the correct result code and error identifier are returned, and that system state is unchanged after the failure.

### 2e: Response and output schema
Every new field added to a response or output structure gets a test verifying it appears with the correct value or type. Every field the plan explicitly excludes from a response gets a test verifying it is absent. Every new response envelope or wrapper type gets a test verifying its shape.

### 2f: Background jobs and scheduled tasks
For each background job, scheduled task, or async process:
- **Happy path:** a qualifying item is processed and the expected state transition occurs
- **Skip condition:** an item that meets a guard condition (sub-state, in-progress flag, future timestamp) is skipped without modification
- **Timing boundary:** an item just past the trigger threshold is processed; an item before the threshold is not
- **Per-item failure isolation:** one item failing does not block remaining items from being processed
- **Side effects:** if the job sends notifications, triggers external calls, or produces other side effects, verify they occur, and that a side-effect failure does not block the main state transition unless the plan says it should

### 2g: Masking, redaction, and derived display values
If the plan describes any field whose representation in output differs from its stored value (redaction, masking, derived formatting), write tests for:
- The masked/redacted representation appears correctly when the condition is met
- The real value appears when the condition is not met
- The internal flag or source field does not leak into the output

### 2h: Concurrency and ordering constraints
If the plan names any route registration order, initialization order, or sequencing constraint, write a test or code-review check that would catch a violation of it.

### 2i: Legacy and regression paths
If the plan explicitly preserves an existing flow alongside the new one:
- Write a test that exercises the existing flow end-to-end and verifies it still works
- Write a test that verifies the two flows do not bleed into each other (new-flow operation rejected by old-flow handler, old-flow operation rejected by new-flow handler)

### 2j: Gaps, breakage, and assumptions
Re-read the "Gaps and Breakage" and "Assumptions" sections of the plan. For each item that represents a runtime risk (not merely a deployment coordination concern), write a test that would catch the failure if the assumption is wrong or the gap is not addressed.

### 2k: Cross-endpoint parameter contrasts
If the plan describes two related operations or endpoints that accept the same parameter or input but validate it differently (one accepts a value the other rejects, one returns a success where the other returns an error), generate a test that explicitly names the contrast. Do not rely on separate per-endpoint tests to imply the contrast. Write a test whose title and purpose is specifically "these two endpoints behave differently for the same input, and that is intentional." This catches a future change that unintentionally harmonises deliberately asymmetric behavior.

### 2l: Negative constraints
If the plan explicitly states that a new validator, filter, or check must NOT be applied in a specific context (e.g. "this guard applies only to writes, not reads"; "must not be called during a GET response builder"), generate a test that calls that context with an entity in a non-default state and verifies the operation completes successfully and returns full data. The test title should name both the context and the exclusion: "GET \<operation\> returns \<entity\> regardless of \<state\>" or similar.

The happy-path test for that context does not cover this: it uses a default-state entity and passes even if the guard is wrongly applied, as long as it happens to allow the default state through. The negative-constraint test must use an entity in a state that would be rejected if the guard were incorrectly applied.

These tests are the most likely to be silently broken by a future refactor that "helpfully" applies a new check everywhere.

## Phase 3: Group and prioritize

**Grouping:** Assign each test to a group for a cohesive feature area, typically a single operation, endpoint, background job, or cross-cutting concern (e.g. "Response Schema", "Authorization"). Groups map to the plan's major sections. Name groups after their operation or feature, not internal code concepts.

**Priority:** Assign each test exactly one priority:

**High** if any of the following apply:
- It is the primary happy path for a new or modified operation
- It guards against data loss, irreversible state change, or a security violation
- It verifies an unauthorized caller is rejected
- It covers a failure mode explicitly flagged in "Gaps and Breakage"
- It covers a state machine guard on a consequential transition

**Medium** if any of the following apply:
- It is a secondary happy path (additional authorized role, alternate valid input)
- It tests a state-conflict rejection on an operation that is plausibly attempted
- It verifies a schema field or response shape
- It tests a timing boundary condition
- It is a legacy regression test

**Low** if any of the following apply:
- It is a code-review or architecture check with no directly observable runtime outcome
- It tests a failure mode that requires unusual or synthetic setup
- It covers an assumption considered stable by the plan

## Phase 4: Write test cases

Number cases sequentially across all groups: TC-001, TC-002, … in group order, not priority order.

Write each case with this structure:

```
#### TC-NNN: <title>
`Priority: High | Medium | Low`

**Given** <preconditions: entity state, caller role or identity, relevant data values>
**When** <action: operation name, call signature, or input>
**Then** <expected outcome: result code, state fields, error identifier, side effects, be specific>
**Post** <post-condition independently verifiable after the operation, omit if identical to "Then">
```

Rules:
- "Then" must be specific and independently verifiable. State exact fields, exact values, exact result codes, exact error identifiers. Never use vague assertions like "succeeds" or "returns the correct data."
- "Post" captures a consequence that is distinct from the immediate return: a downstream state, a subsequent query result, a side effect. Omit it if it adds nothing.
- Titles are concise and describe the scenario, not the implementation. Use the actor and the outcome: "ADMIN restores a soft-deleted document", not "test restore endpoint."
- Each test is independent. It does not depend on the outcome of any other test case.
- Add a blockquote note below a test case only when it requires a non-obvious setup step, tests an implicit invariant the plan doesn't state explicitly, or calls out a code-review-only check.

## Phase 5: Write the file

Write to `plans/{feature-name}-tests.md`. Create the `plans/` directory if it does not exist. Do not prompt for confirmation before writing.

Structure the file exactly as follows:

```markdown
# <Feature Name>: Test Plan

## Scope

<One paragraph describing what is covered and what is explicitly out of scope. Call out any external systems or internals that are not covered.>

**Source:** These tests derive from the behavioral specification (`plans/{feature-name}.md`), not the implementation plan. Exact method signatures and constant values from the execution plan inform the assertions but do not drive test structure, ensuring tests remain valid after implementation changes.
**Test data:** Each case is independent. Arrange its preconditions fresh to avoid order dependency.
**Pass criteria:** <What constitutes a passing test: the observable outcome that must be verified.>

---

## Summary

| ID | Title | Group | Priority |
|---|---|---|---|
<one row per test, in TC-NNN order>

---

## Group 1: <Name> · <operation or context>

<test cases>

---

## Group 2: ...
```

After writing the file, report:
- The output path
- Total test case count
- Breakdown by priority: N High / N Medium / N Low

Tell the user: "Run `/feat-execute` to begin implementation."

Update `plans/{feature-name}.pipeline.json`, merging the following entry into the `pipeline` object:

```json
"feat-tests": { "status": "completed", "output": "plans/{feature-name}-tests.md", "timestamp": "{ISO-8601}" }
```

Print a one-line pipeline state from the updated `plans/{feature-name}.pipeline.json`:

`Pipeline: {grill →} plan {icon} → execution {icon} → audit {icon} → execute {icon}`

Use `✓` completed/passed · `✗` failed · `○` not started. Omit grill-with-docs if the key is absent.
