---
name: audit-tests
description: Generates a tiered test plan from the verdicted discrepancy list and the PRD. Regression cases cover confirmed defects; edge cases cover intentional and unresolved entries; happy-path cases cover behavioral areas that share code paths with confirmed discrepancies. Writes plans/{pr-id}-tests.md. Fourth step of the audit-pipeline.
when_to_use: Invoke explicitly with /audit-tests after /audit-grill completes, or run as Stage 4 when /audit-pipeline orchestrates it.
allowed-tools: Read Write Bash(find *) Bash(ls *) Glob Grep
---

# Audit Tests

Generate a tiered test plan from the discrepancy verdicts and the PRD.

## Phase 1: Read the sources

**Locate the pipeline:** If `$ARGUMENTS` is provided, treat it as `pr-id`. Otherwise scan `plans/` for `*.pipeline.json` files with a `"pr"` key. Pick the one where `audit-grill.status` is `completed` and `audit-tests` is absent or incomplete.

Confirm `audit-grill.status` is `completed`. If not, stop. Tell the user to run `/audit-grill` first.

Read in full:
- `plans/{pr-id}-discrepancies.md`: the verdicted discrepancy list
- `plans/{pr-id}-prd.md`: the behavioral description

---

## Phase 2: Extract test cases by tier

Work through each tier in order.

### Tier 1: Regression cases
Source: all `confirmed` discrepancies.

For each confirmed discrepancy, write one or more test cases that would catch the defect if it were introduced into a clean codebase. Ask: what input, state, or sequence would trigger the problematic behavior? What would a correct implementation return instead?

Each regression test must:
- Reference the discrepancy ID (D-NNN) in a blockquote note
- Verify the corrected behavior, not merely that the defect path is blocked
- Cover each distinct trigger condition as a separate test case if the discrepancy has more than one

### Tier 2: Edge cases
Source: `intentional` and `unresolved` discrepancies.

For `intentional` entries: write one test that documents and verifies the surprising behavior is working as designed. This acts as a regression guard against someone "fixing" a deliberate choice.

For `unresolved` entries: write exactly two tests that probe the ambiguous case from both sides: one assuming the behavior is intentional (and should be preserved), one assuming it is a defect (and should be corrected). Mark both with a blockquote that names the discrepancy ID and makes the required decision explicit. These pairs must be resolved by a human before `/audit-validate` runs.

### Tier 3: Happy path
Source: PRD behavioral areas.

Generate Tier 3 tests only for behavioral areas that share code paths with at least one `confirmed` discrepancy. These are the areas where something was found wrong, so verifying the surrounding happy path is meaningful. Areas with no confirmed discrepancies are implicitly validated by their absence from the discrepancy list; do not generate Tier 3 tests for them.

For each qualifying area, write one test covering the primary success path. Do not duplicate coverage already in Tier 1 or Tier 2.

---

## Phase 3: Prioritize

Assign each test case exactly one priority:

**High** if any of the following apply:
- It covers a `confirmed` discrepancy (all Tier 1 cases)
- It covers one side of an `unresolved` pair (both sides of a pair are High, since the pair carries real ambiguity)
- It guards against data loss, irreversible state change, or a security violation

**Medium** if any of the following apply:
- It documents an `intentional` behavior as a regression guard
- It covers the happy path for a behavioral area adjacent to a confirmed defect (Tier 3)

**Low** if any of the following apply:
- It tests a minor edge case within an otherwise covered flow

---

## Phase 4: Write test cases

Number sequentially across all tiers: TC-001, TC-002, … in tier order (Tier 1 first, then 2, then 3), then by group within each tier.

```
#### TC-NNN: <title>
`Priority: High | Medium | Low`  `Tier: Regression | Edge | Happy Path`

**Given** <preconditions: entity state, caller role or identity, relevant data values>
**When** <action: operation name, call signature, or input>
**Then** <expected outcome: specific fields, values, result codes, side effects, never "succeeds" or "returns the correct data">
**Post** <independently verifiable post-condition, omit if identical to "Then">
```

Rules:
- "Then" is specific and independently verifiable: exact field names, exact values, exact result codes.
- Titles describe the scenario, not the implementation. "Session is invalidated after logout" not "test logout endpoint."
- Each test is independent of every other.
- Regression tests include a blockquote: `> Covers D-NNN: {discrepancy title}`
- Each side of an unresolved pair includes:
  ```
  > D-NNN is unresolved. This test assumes the behavior IS [intentional/a defect].
  > Before running /audit-validate, delete the test that does not match the intended behavior.
  ```

---

## Phase 5: Write the file

Write `plans/{pr-id}-tests.md`. Create `plans/` if it does not exist.

```markdown
# {PR title or branch name}: Test Plan

## Scope

<One paragraph: what is covered (confirmed defects, edge cases, happy paths for affected areas) and what is not (behavioral areas with no confirmed discrepancies, external dependencies not in the diff).>

**Source:** Regression and edge cases derive from `plans/{pr-id}-discrepancies.md`. Happy-path cases derive from `plans/{pr-id}-prd.md` for areas adjacent to confirmed defects only.
**Test data:** Each case is independent. Arrange preconditions fresh.
**Unresolved pairs:** Each `unresolved` discrepancy produces two test cases. One must be deleted before running `/audit-validate`. See blockquotes for which decision is required.

---

## Summary

| ID | Title | Tier | Priority |
|---|---|---|---|
<one row per test, in TC-NNN order>

---

## Tier 1: Regression Cases

<test cases>

---

## Tier 2: Edge Cases

<test cases>

---

## Tier 3: Happy Path

<test cases>
```

If Tier 3 is empty (no behavioral areas share code paths with confirmed discrepancies), omit that section.

Confirm the written path and report:
- Total test count
- Breakdown by priority (N High / N Medium / N Low)
- Breakdown by tier
- Number of unresolved pairs requiring a human decision before `/audit-validate`

Update `plans/{pr-id}.pipeline.json`, merging:

```json
"audit-tests": { "status": "completed", "output": "plans/{pr-id}-tests.md", "timestamp": "{ISO-8601}" }
```

Print pipeline state:
`Pipeline: prd ✓ → gate ✓ → grill ✓ → tests ✓ → validate ○`

**If unresolved pairs exist:** Tell the user:
> "**Action required before `/audit-validate`:** {N} unresolved discrepancy pair(s) need a human decision. For each pair in Tier 2, read both sides and delete the test that does not match the intended behavior. Then run `/audit-validate`."

**Otherwise:** Tell the user: "Run `/audit-validate` to check which test cases the implementation covers."
