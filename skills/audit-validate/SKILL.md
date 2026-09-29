---
name: audit-validate
description: For each test case in the audit test plan, checks whether the implementation handles it via static analysis of the diff. Reads the diff source from pipeline.json. Tier 1 gaps are blockers; Tier 1 unknowns require manual verification and produce APPROVE WITH CAVEATS. Appends each run as a timestamped section to preserve history. Fifth step of the audit-pipeline.
when_to_use: Invoke explicitly with /audit-validate after /audit-tests completes and any unresolved pairs have been resolved, or run as Stage 5 when /audit-pipeline orchestrates it.
allowed-tools: Read Write Bash(gh pr diff *) Bash(git diff *) Bash(find *) Bash(ls *) Bash(grep *) Glob Grep
---

# Audit Validate

Check each test case against the implementation via static diff analysis.

## Phase 1: Load inputs

**Locate the pipeline:** If `$ARGUMENTS` is provided, treat it as `pr-id`. Otherwise scan `plans/` for `*.pipeline.json` files with a `"pr"` key. Pick the one where `audit-tests.status` is `completed` and `audit-validate` is absent or incomplete.

Confirm `audit-tests.status` is `completed`. If not, stop. Tell the user to run `/audit-tests` first.

Read in full:
- `plans/{pr-id}-tests.md`
- `plans/{pr-id}-discrepancies.md`

**Unresolved pair check:** Scan `plans/{pr-id}-tests.md` for any test case blockquote containing "D-NNN is unresolved." If any are found, stop:
> "The test plan contains {N} unresolved pair(s) that have not been resolved. Delete the unwanted side of each pair in `plans/{pr-id}-tests.md`, then re-run `/audit-validate`."

---

## Phase 2: Get the diff

Read the `source` field from `plans/{pr-id}.pipeline.json`. If `source` is absent, stop:
> "`pipeline.json` has no `source` field. Re-run `/audit-prd` to regenerate the PRD and record the source."

Fetch the diff using the recorded source:
- `source.type` is `"pr"`: run `gh pr diff {source.value}`
- `source.type` is `"branch"`: run `git diff {source.base}..HEAD`

Do not re-derive the diff. Do not fall back to origin/staging or origin/main.

---

## Phase 3: Coverage check

For each test case in the test plan, determine whether the implementation handles it. Check:
1. Does the diff contain code that would execute on the described `When` action?
2. Does that code path produce the outcome stated in `Then`?

Assign one verdict per test:

| Verdict | Meaning |
|---------|---------|
| `covered` | A code path in the diff handles this case and produces the specified outcome |
| `gap` | No code path in the diff handles this case, or the outcome would differ from what `Then` specifies |
| `unknown` | Cannot determine from static analysis. The behavior depends on runtime state or external systems not visible in the diff |

For every `gap` or `unknown`, identify specifically what is absent or ambiguous. Name the file and function where the missing behavior should appear.

**Tier 1 `unknown` handling:** `unknown` on a Tier 1 (regression) case cannot be treated as covered. The defect may still exist but simply cannot be verified statically. These cases do not block the verdict but require explicit manual verification before the APPROVE stands. Track them separately in the report.

---

## Phase 4: Write the validate report

**Determine the run timestamp:** Use the current ISO-8601 timestamp to name this run: `{YYYYMMDD-HHmm}`.

**Write to:** `plans/{pr-id}-validate-{timestamp}.md`. Each re-run produces a new file with its own timestamp. Do not overwrite prior runs.

**Overall verdict:**
- `APPROVE`: no Tier 1 gaps and no Tier 1 unknowns
- `APPROVE WITH CAVEATS`: no Tier 1 gaps, but one or more Tier 1 unknowns exist (manual verification required)
- `REQUEST CHANGES`: one or more Tier 1 gaps exist

```markdown
# {PR title or branch name}: Audit Validate

**Verdict:** APPROVE | APPROVE WITH CAVEATS | REQUEST CHANGES
**Run:** {ISO-8601 timestamp}
**{N} covered · {N} gap · {N} unknown**

---

## Coverage

| ID | Title | Tier | Priority | Verdict | Notes |
|---|---|---|---|---|---|
| TC-001 | ... | Regression | High | covered | - |
| TC-002 | ... | Regression | High | gap | Missing null-check in `auth/session.py:validate()` |
| TC-003 | ... | Regression | High | unknown | Behavior depends on runtime session state not visible in diff |
| TC-004 | ... | Edge | Medium | covered | - |

---

## Blockers *(Tier 1 gaps only)*

### TC-NNN: {title}
**What is absent:** {specific file, function, or condition}
**The test expects:** {the Then clause}
**The diff provides:** {what the code actually does instead, or nothing}

---

## Manual Verification Required *(Tier 1 unknowns only)*

These regression cases cannot be verified by static analysis. The APPROVE WITH CAVEATS verdict stands only after each of these has been manually confirmed.

### TC-NNN: {title}
**Why unknown:** {what runtime state or external system makes this unverifiable statically}
**How to verify manually:** {suggested approach: run the test, inspect logs, check DB state}

---

## Observations *(Tier 2 and 3 gaps and unknowns)*

### TC-NNN: {title}
**Note:** {what is missing or ambiguous and why it is not a blocker}
```

Omit "Blockers" if verdict is not REQUEST CHANGES. Omit "Manual Verification Required" if verdict is APPROVE. Omit "Observations" if no Tier 2/3 gaps or unknowns exist.

Confirm the written path to the user.

Update `plans/{pr-id}.pipeline.json`, merging:

```json
"audit-validate": {
  "status": "passed|passed-with-caveats|failed",
  "verdict": "APPROVE|APPROVE WITH CAVEATS|REQUEST CHANGES",
  "covered": {N},
  "gap": {N},
  "unknown_tier1": {N},
  "output": "plans/{pr-id}-validate-{timestamp}.md",
  "timestamp": "{ISO-8601}"
}
```

Use `passed` for APPROVE, `passed-with-caveats` for APPROVE WITH CAVEATS, `failed` for REQUEST CHANGES.

Print pipeline state:
`Pipeline: prd ✓ → gate ✓ → grill ✓ → tests ✓ → validate ✓|⚠|✗`

Use `✓` for APPROVE, `⚠` for APPROVE WITH CAVEATS, `✗` for REQUEST CHANGES.

---

## Phase 5: Recovery guidance

**If APPROVE:** Tell the user all regression cases are covered. Suggest `/simplify` for a cleanup pass, or stop if the branch is not checked out locally.

**If APPROVE WITH CAVEATS:** Present the manual verification section. Tell the user:
> "The implementation has no static gaps, but {N} Tier 1 case(s) require manual verification before this audit can close. Verify each case above, then consider this audit complete or re-run `/audit-validate` after any code changes."

**If REQUEST CHANGES:** Present each blocker. Tell the user:
> "The implementation has {N} uncovered regression case(s). The PR needs code changes before this audit can close. Once changes are made, re-run `/audit-validate`. A new timestamped report will be written alongside the previous one."
