---
name: feat-validate
description: Post-execution gate. Compares the commits written by feat-execute against the feature plan requirements and reports any requirement with no corresponding code change. Mirrors feat-audit but runs against committed code rather than the execution plan. Writes plans/{feature-name}-validate.md.
when_to_use: Invoke explicitly with /feat-validate after /feat-execute completes, or run as Stage 7 when /feat-pipeline orchestrates it.
allowed-tools: Read Write Bash(grep *) Bash(find *) Bash(git diff *) Bash(git log *) Bash(git show *) Bash(ls *) Glob Grep
---

# Validate Execution

Post-execution coverage check: verify committed code satisfies every requirement in the feature plan. Not a re-run of `feat-audit`. That skill checks plan documents against each other; this skill checks committed code against requirements.

---

## Phase 1: Locate inputs

**Feature plan:** If a path was passed as `$ARGUMENTS`, read that file. Otherwise scan `plans/` for `.md` files that do not end in `-execution.md`, `-audit.md`, `-tests.md`, or `-validate.md`. If multiple candidates exist, list them and ask which to use. Derive the feature name from the filename.

**Pipeline state:** Read `plans/{feature-name}.pipeline.json`. If `feat-execute.status` is not `completed`, stop. Tell the user to run `/feat-execute` first.

**Execution log:** Read `plans/feat-execute-{feature-name}.log.md`. If it does not exist, stop. The log is required to identify which commits belong to this feature; tell the user it was expected at that path.

**Execution plan:** Read `plans/{feature-name}-execution.md`. Used to cross-reference requirements against commit steps.

Read all located files in full before proceeding.

---

## Phase 2: Identify the commit range

Parse `plans/feat-execute-{feature-name}.log.md` to extract every commit hash recorded during execution. Commit hashes appear in section headers of the form:

```
## Step {N}: {commit title} ({commit hash})
```

Collect these hashes in step order. The **base** is the parent of the first commit: `{first_hash}^`.

Run:

```bash
git log --oneline {first_hash}^..HEAD
```

Confirm the listed commits match the hashes in the log. If not (e.g. commits were made outside feat-execute), report the discrepancy and ask whether to proceed with the full range or only the logged hashes.

---

## Phase 3: Get the full diff

Run:

```bash
git diff {first_hash}^..HEAD
```

This is the complete set of changes introduced by this feature. Hold it for Phase 4.

---

## Phase 4: Extract requirements

From the feature plan, extract every distinct implementation requirement. Number them R-001, R-002, etc., exactly as feat-audit does. The same numbering is used if both reports exist for the same feature, allowing cross-referencing.

For each requirement, record:
- Its number and a short description
- The layer(s) it touches (router, service, CRUD, model, etc.)
- Any named symbols, fields, or constants it specifies

---

## Phase 5: Coverage check

For each requirement R-NNN, scan the diff for changes that address it. Assign one of three verdicts:

**Covered.** The diff contains changes that clearly implement the requirement: the named symbol exists, the specified field is present, the described logic is visible in the diff.

**Partial.** The diff contains some changes related to the requirement but not all of it. For example: the model field is added but the service function that uses it is absent; the route is registered but the schema field is missing.

**Missing.** The diff contains no changes that address the requirement.

When Partial or Missing, name what's absent: the file, function, or field that should exist but doesn't. This is what the user needs to add a commit or write an amendment.

---

## Phase 6: Write the validate report

Write `plans/{feature-name}-validate.md`. Create the `plans/` directory if it does not exist.

**Verdict:** PASS if every requirement is Covered. FAIL if any requirement is Partial or Missing.

```markdown
# Execution Validate: {Feature Name}

**Verdict:** PASS | FAIL
**Commit range:** {first_hash}^..{HEAD hash}
**{N} covered · {N} partial · {N} missing**
**Validated:** {ISO-8601 timestamp}

---

## Coverage

| Requirement | Description | Verdict | Notes |
|---|---|---|---|
| R-001 | {description} | Covered | - |
| R-002 | {description} | Partial | Service function missing: `services/foo.py::bar()` |
| R-003 | {description} | Missing | No changes found in any relevant file |

---

## Required actions

### R-NNN: {description} · Partial / Missing

**What is absent:** {specific file, function, or field}
**Suggested fix:** Add a new commit implementing the missing piece, or write an amendment to `plans/{feature-name}-execution.md` if the requirement is intentionally deferred.
```

Omit the "Required actions" section entirely if verdict is PASS.

Confirm the written path to the user.

Update `plans/{feature-name}.pipeline.json`, merging the following entry into the `pipeline` object:

```json
"feat-validate": { "status": "passed|failed", "covered": {N}, "partial": {N}, "missing": {N}, "output": "plans/{feature-name}-validate.md", "timestamp": "{ISO-8601}" }
```

---

## Phase 7: Recovery guidance

**If verdict is PASS:** Tell the user the implementation covers all requirements. Next step: `/simplify`.

**If verdict is FAIL:** Present the required actions section. For each Partial or Missing requirement:
- If a small amount of code is missing: suggest adding a new commit, then re-running `/feat-validate`.
- If the gap is large or represents a design deviation: suggest an amendment to the execution plan and re-running `/feat-execute` for the affected steps, then re-running `/feat-validate`.

Do not proceed to `/simplify` until the verdict is PASS.

Print a one-line pipeline state from the updated `plans/{feature-name}.pipeline.json`:

`Pipeline: {grill →} plan {icon} → execution {icon} → audit {icon} → execute {icon} → validate {icon}`

Use `✓` completed/passed · `✗` failed · `○` not started. Omit grill-with-docs if the key is absent.
