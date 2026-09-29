---
name: feat-execute
description: Implements plans/{feature-name}-execution.md one commit at a time. Reads each step, explores the current codebase, writes the code, runs ruff check, cross-references the staged diff against the plan step, verifies the expected-state checklist, then commits. Pauses for user review after each commit. After the last commit, one independent subagent reviews the whole branch diff against the nearest convention files. Writes amendments to the plan when implementation must deviate. Resumes from a progress marker if interrupted.
when_to_use: Invoke explicitly with /feat-execute to implement the execution plan one commit at a time, after /feat-audit passes, or run when /feat-pipeline orchestrates it. Fifth step of the feat- pipeline.
allowed-tools: Read Edit Write Bash(git *) Bash(ruff *) Bash(grep *) Bash(find *) Bash(ls *) Glob Grep Agent WebSearch WebFetch
---

# Execute Plan

Implement `execution-plan.md` one atomic commit at a time. Each commit is independently compilable and leaves the repository in a working state. Do not implement multiple steps in one pass or skip the verification gates.

When implementation must deviate from the plan, whether a method name is different, a join path is wrong, or a constant must change, write the amendment to `## Amendments` in the plan file before committing. Never silently adapt.

## Blocker Resolution Protocol

Invoke this protocol whenever execution is blocked and the problem cannot be auto-resolved: a ruff error that survives `--fix`, a pre-commit hook failure requiring external action, a missing symbol with no clear substitute, or a material plan deviation requiring a decision. Run it *before* asking the user to choose.

**Step 1: Gather sources (in parallel where possible)**

- **Codebase pattern.** Grep the repo for files or functions that handle a similar problem. Quote the relevant snippet. If no analogous case exists, state that explicitly.
- **Best practice.** State the conventional fix based on engineering judgment. Identify the specific rule or invariant it satisfies.
- **Canonical docs.** WebSearch the official documentation for the tool, library, or error code involved. Quote the recommended approach and its source URL. If the docs present multiple meaningfully different approaches, list each as a separate option.

Omit a source only when it does not apply (e.g. the error is project-specific with no upstream docs, or no similar pattern exists in the codebase).

**Step 2: Present the comparison**

```
## Proposed Solutions

### A: Codebase pattern
{Description + snippet}
**Pros:** ...  **Cons:** ...

### B: Best practice
{Description}
**Pros:** ...  **Cons:** ...

### C: Canonical docs for {tool / library name}
{Description}
**Source:** {URL}
**Pros:** ...  **Cons:** ...
```

Add lettered options beyond C when canonical docs present more than one distinct approach.

**Step 3: Recommend**

> **Recommended:** {A / B / C}, {one-sentence reason based on the tradeoffs above}

**Step 4: Ask**

> Which solution would you like to implement? (A / B / C, or describe your own approach)

---

## Current project state

```!
git log --oneline -10
```

```!
git status --short
```

---

## Phase 1: Locate inputs

**Execution plan:** If a path was passed as `$ARGUMENTS`, read that file. Otherwise scan the `plans/` directory for files matching `*-execution.md`. If exactly one is found, use it. If multiple exist, list them and ask the user which to use. If none exist, stop. Tell the user to run `/feat-execution` first or provide the path explicitly.

**Pipeline gate (mandatory):** Derive the feature name from the execution plan filename. Read `plans/{feature-name}.pipeline.json`. If `feat-audit.status` is not `passed`, stop immediately. This gate cannot be bypassed:
- If status is `failed`: "The audit found `{N}` blocker(s) in `plans/{feature-name}-audit.md`. Execution cannot proceed on an unverified plan. Resolve the blockers, re-run `/feat-execution`, then re-run `/feat-audit`."
- If the entry is missing: "Run `/feat-audit` first. Executing without a passed audit risks implementing a plan with known gaps."

**Convention files:** Read `CLAUDE.md` (and `CONVENTIONS.md` if present) from the project root. Note the commit message format, linting requirements, any required decorators for new routes, and any conventions for new code. These apply to every commit this skill makes.

Read the execution plan in full:
- Note the `## Amendments` section (may be empty, which is normal)
- Note the `## Completion Checklist`
- Enumerate every numbered commit step and its title

---

## Phase 1b: Safety confirmation

**If `$ARGUMENTS` contains `--pipeline`:** Skip this phase entirely. The pipeline already presented the safety block and received GO at Stage 6 before invoking this skill.

**Normal mode:** Present the block below and wait for the user to type GO (case-insensitive). Answer any questions, then re-present the block. Do not proceed to Phase 2 until GO is received.

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

---

## Phase 2: Determine starting point

Look for `plans/feat-execute-{feature-name}.progress.json`.

**If found:** Read it. Cross-check `last_commit_hash` against the current `HEAD` via `git log -1 --format=%H`. Report to the user:
- Steps already completed
- Steps remaining
- Whether `HEAD` matches the recorded hash (if not, explain the discrepancy and ask how to proceed before continuing)

**If not found:** Check `git log --oneline` against the commit titles in the plan. If any plan commit messages already appear in git history, infer which steps are done and present this inference to the user for confirmation before proceeding.

**Unresolved uncommitted changes:** If `git status` shows staged or unstaged changes before the first step begins, stop. Ask the user to commit or stash them before continuing. The cross-reference gate cannot function cleanly against a dirty working tree.

---

## Phase 3: Execute commits one at a time

**Working directory:** Run all Bash commands without a `cd` prefix. The shell is already in the project root; `cd /path &&` prefixes are unnecessary and trigger a security prompt on every git and ruff call.

**Pipeline mode:** If `$ARGUMENTS` contains `--pipeline`, activate pipeline mode. In pipeline mode:
- Skip step 3h (review pause). After each commit, proceed straight to the next step.
- In the amendment protocol (Phase 4): write the amendment and check for side effects in remaining steps, but do not pause for confirmation. All amendments are reported in the Phase 6 completion summary.
- Still pause for: ruff failures that cannot be auto-corrected (step 3d), and pre-commit hook failures that cannot be resolved automatically (step 3g).

Repeat the following sequence for each remaining step. After each, pause and present a summary, then wait for explicit user approval before the next step. (In pipeline mode, skip the pause and proceed directly to the next step.)

### 3a. Read the step

Read the full commit step: title, files, pattern reference, implementation detail, and expected-state checklist.

**Pattern reference:** Read the step's **Pattern reference** field.

- If it names a file and line range (e.g. `routers/orders.py:L42-L58`): read those lines before writing any code. State explicitly: "Following pattern at `{path}:{lines}`." Write code that matches the referenced pattern's structure: decorator application, parameter ordering, naming conventions, import style.
- If it is "none, introducing new pattern": proceed without an anchor and note this in the review pause summary.

Treat any divergence from the named Pattern reference as a deviation requiring the amendment protocol (Phase 4). Do not silently adapt to a different structure.

### 3b. Explore current codebase state

Read every file listed in the step's **Files** section. Also read any file the implementation detail references by import or function call that you have not already read in this session. Understand the current state before making any change.

If a symbol named in the step does not exist in the codebase and was not expected to be introduced by a prior step in this plan, do not guess. Treat this as a deviation and follow the amendment protocol in Phase 4 before proceeding.

### 3c. Implement

Make all changes specified in the step, in file order as listed, with no partial implementations. Use the exact function signatures, parameter names, return types, import paths, and constant values written in the plan step. Use the confirmed volatile constants verbatim; do not substitute approximations.

When adding a new route, verify any required decorators or dependencies against the rules in the convention file before writing. If the plan step does not specify them, derive them from the convention file and note the derivation when presenting the summary.

When registering a route that could conflict with a parametric route, verify registration order. Literal-path routes must be registered before parametric routes.

### 3d. Ruff check gate

Run `ruff check .`.

- If it passes: continue.
- If it fails: run `ruff check . --fix` to auto-correct. Re-run `ruff check .`.
  - If it now passes: continue.
  - If it still fails: invoke the **Blocker Resolution Protocol** with the ruff error output as the blocked problem. Sources to gather: how similar violations are suppressed or fixed in this codebase (grep for `# noqa`, existing ruff config, or analogous code), the conventional fix per engineering best practice, and the official ruff docs for the specific rule code(s) failing. After presenting proposed solutions and a recommendation, offer the standard options:
    - **Fix:** implement the chosen solution, then re-run `ruff check .`.
    - **Rollback:** undo all changes made in this step (see rollback procedure below). The proposed solutions remain available if the user wants to retry the step with a specific approach.

**Rollback procedure:** Run `git restore --staged .` to unstage, then `git restore .` to revert all edited files to their committed state. Confirm to the user which files were restored and that the working tree is clean before offering to retry the step.

### 3e. Cross-reference gate

Stage the changed files (`git add` the files listed in the step). Run `git diff --cached` and compare the staged diff against the plan step's **Files** list and **Implementation detail**:

Check for:
- **Missing files:** A file listed in the plan step is not in the diff. Either the change was not made or it was made in a different file.
- **Extra files:** A file appears in the diff that is not listed in the plan step. Either scope crept or the change is in the wrong file.
- **Logic divergence:** A significant difference between what the plan specifies and what was implemented: different method signatures, missing guard conditions, wrong constant values, absent imports.

**Symbol resolution check:** For each function call and imported name introduced in the staged `.py` files, grep the committed codebase to verify the symbol exists. `ruff` checks syntax and style only. It cannot verify that a method called on a class instance (e.g. `OrderRepo(session).get_all_pending_cancel()`) actually exists on that class. If a called symbol is not found in committed code and is not defined elsewhere in the same staged changeset, flag it as a potential ordering error in the plan and follow the amendment protocol before committing.

Minor divergences (e.g. import ordering, a line break) do not require an amendment. Material divergences do. Follow the amendment protocol in Phase 4.

### 3f. Verify expected-state checklist

Work through every item in the step's **Expected state after this commit** block. All items should be statically verifiable. The plan-execution skill is designed to write only static checks. Verify each by grepping committed code or reading the relevant files directly:

- **Symbol importability:** grep the target module for the named symbol; confirm it is defined and at the correct scope.
- **Function or guard existence:** grep for the function name and inspect the relevant lines to confirm the guard condition or raise statement is present.
- **`ruff check .`:** already confirmed in 3d, so mark as verified.

Do not start a development server, run migrations, or execute any code to verify expected-state items. If an item in the plan requires a running server to verify, it was written incorrectly by the plan-execution skill. Flag it as a plan defect, skip that item, and note it in the review pause summary.

Report the checklist to the user during the review pause with each item marked verified or flagged.

### 3f2. Simplify gate

Before committing, review the staged diff for the issues `/simplify` would catch, scoped to what this step introduced:

- **Names:** does any new identifier fail to convey why it exists, what it does, or how it is used?
- **Single responsibility:** does any new function do two things that could be split?
- **Flag/output arguments:** new boolean parameters or arguments mutated instead of returned?
- **Magic literals:** implementation-level literals that should be named constants (distinct from the volatile-constant check, since this catches constants that are incidental to this step, not business-rule values already confirmed in Phase 3).
- **What-comments:** comments that describe what the code does rather than why.

Fix issues directly in the staged files. Re-run `ruff check .` after any edits. Fold all simplify fixes into this step's commit. Do not create a separate commit. If no issues are found, proceed silently.

### 3f3. Convention check has moved

**Do not spawn a per-commit convention subagent.** The check now runs once, over the whole branch diff, in **Phase 5.5**.

Measured on a 13-commit feature: 13 per-commit reviews returned 10 findings, of which **2 were kept** (a signature reflow and a variable rename, both cosmetic) and **8 were discarded**. Nearly every discard traced to the same cause: the subagent citing a convention doc that a later ruling had superseded, or a rule the audited plan had already carved out. One review per commit paid that noise thirteen times over and scaled linearly with commit count, while the two real findings would both have surfaced just as readily in a single pass at the end.

Proceed directly to 3g.

### 3g. Commit

Commit using the exact message from the plan step's **Commit title**. Do not paraphrase it. Do not amend it unless the user explicitly requests a change during the review pause.

```bash
git commit -m "{commit title from plan step}"
```

**If the commit fails due to a pre-commit hook:** Do not treat this as a rollback trigger. Read the hook output, identify the cause, fix the issue in the affected files, re-stage, and retry the commit. Repeat until the commit succeeds. If the hook failure cannot be resolved (e.g. it requires a secret or external service), invoke the **Blocker Resolution Protocol** with the hook failure as the blocked problem. Sources to gather: how this hook or a similar one is configured/bypassed in the repo, the conventional way to satisfy the hook's requirement, and the official docs for the hook tool involved. Present proposed solutions and a recommendation, then wait for the user's chosen solution before proceeding.

Record `git log -1 --format=%H` immediately after a successful commit. Update the progress marker.

### 3h. Review pause

Present a summary to the user:
- Commit hash and title
- Files changed (count and names)
- Any amendments written in this step
- Expected-state checklist results (verified / flagged)
- Next step title (so the user knows what is coming)

Approval is inferred from context. A clearly affirmative response ("ok", "looks good", "next", "continue") proceeds to the next step. If the user asks a question or expresses uncertainty, answer it and wait for a further response before proceeding. Do not interpret a question as approval.

**If the user rejects the step:** offer the rollback procedure (as described in 3d) to restore the working tree to its state before the step began. Log the rejection in the progress marker's `user_interventions` array before rolling back. After rollback, present the step again from 3a with the user's feedback in mind.

After each approved step, append a brief entry to `plans/feat-execute-{feature-name}.log.md` (creating it if it does not exist):

```markdown
## Step {N}: {commit title} ({commit hash})

- Amendments: {none / description}
- Expected-state: {all verified / N flagged}
- User corrections: {none / description}
```

This log survives context compaction and provides a human-readable record if the session is resumed later.

---

## Phase 4: Amendment protocol

Use this protocol whenever implementation must deviate from the plan: a method does not exist as named, a join path produces different results, a constant value must change, a file path is wrong, a pattern has been refactored since the plan was written.

1. **Stop before committing.** Do not commit a deviation without recording it.
2. **Write the amendment** to the `## Amendments` section of the execution plan file. Each amendment entry:
   ```
   ### Amendment for Step {N}: {one-line description}
   **Deviation:** {what the plan specified}
   **Actual:** {what was implemented and why}
   ```
3. **Check for side effects in remaining steps.** After writing the amendment, grep the text of all remaining (not yet executed) plan steps for the affected symbol, constant, or file path. Report any other steps that reference the same item. They may need updating before execution. Present this list to the user alongside the amendment.
4. **Present the amendment and the side-effect list to the user** before staging or committing. If the deviation is significant (different method, wrong table join, constant change), invoke the **Blocker Resolution Protocol** to propose how to handle it. Sources to gather: how the codebase already calls or substitutes the missing/changed symbol (grep for callers), the conventional substitute based on engineering judgment, and (if it is a library or framework method) what the official docs say about the correct API. After presenting proposed solutions and a recommendation, ask whether to adopt one of the proposed solutions or accept the amendment as-is.
5. **Proceed only after the user confirms** the amendment is acceptable. Log the amendment confirmation in the progress marker's `user_interventions` array.

The plan file is the authoritative record. An amendment keeps it honest.

---

## Phase 5: Progress marker

After each approved commit, write or update `plans/feat-execute-{feature-name}.progress.json`. Also update `plans/{feature-name}.pipeline.json`, merging the current in-progress state:

```json
"feat-execute": { "status": "in-progress", "steps_completed": {N}, "total_steps": {N}, "timestamp": "{ISO-8601}" }
```

The full progress marker:

```json
{
  "plan_file": "plans/{feature-name}-execution.md",
  "feature_name": "{feature-name}",
  "steps_completed": [1, 2, 3],
  "total_steps": 18,
  "last_commit_hash": "{HEAD hash after most recent commit}",
  "last_updated": "{ISO-8601 timestamp}",
  "user_interventions": [
    {
      "step": 3,
      "type": "amendment",
      "description": "get_by_email did not exist, used get_by_email_address"
    },
    {
      "step": 5,
      "type": "rejection",
      "description": "user rejected step; missing nullable FK pattern on deleted_by_id"
    }
  ]
}
```

Intervention types: `amendment` (deviation from plan accepted), `rejection` (step rolled back and retried), `correction` (user redirected implementation without a full rollback).

---

## Phase 5.5: Convention check (once per feature)

Runs **after the last commit lands**, before Completion. One pass over the whole branch diff, replacing the former per-commit check at 3f3.

The reviewer must be an agent that did not write the code. The author reads its own choices as intentional, and removing that bias is the entire point. Spawn **one** subagent, `model: sonnet`, read-only agent type. Give it the brief below and nothing else: not the plan, not the execution plan, not the brief. A reviewer that can see the spec reviews against the spec, which is `/feat-audit`'s job.

> Run `git diff {base}...HEAD` to see every change on this branch, where `{base}` is the commit the branch started from. For every file it touches, walk from that file's directory up to the project root, reading the nearest `CLAUDE.md` and `CONVENTIONS.md` you find on the way; treat the most specific as authoritative. A convention file that only delegates to subsystem files carries no rules of its own, so keep walking. If a rulings or decision-log file exists alongside them (e.g. `CONVENTIONS-RULINGS.md`), it outranks the others: read it first and let it govern on any conflict. Report every place the code breaches a documented convention: naming tables, module placement, layer responsibilities, return shapes, required decorators. For each finding give the offending `path:line`, the convention file and line it breaches, and the conforming form. Report nothing you cannot cite to a convention line. An opinion with no citation is not a finding. Before reporting, check whether the pattern you are flagging is the dominant shape in the surrounding code; if it is, the doc is likely stale and you should say so rather than file it as a breach. If you find no applicable rule anywhere on the chain, list the files you read and reply `no applicable conventions found`. If nothing breaches, reply `clean`.

Then act on what comes back:

- **`clean`:** proceed to Phase 6.
- **Cited findings:** fix them yourself, then commit them as **one** follow-up commit, `refactor: apply convention review findings`, or a `fix:` subject if any finding was a genuine defect. Re-run `ruff check .` first. Do not amend earlier commits.
- **`no applicable conventions found`:** report it, naming the files read. A convention chain that documents nothing is worth surfacing; it is not a pass.
- **A finding that contradicts the audited plan:** do not quietly follow either. The plan was audited against these same conventions, so a real conflict is a plan defect. Follow the amendment protocol.
- **A finding the reviewer flags as likely-stale-doc:** do not fix the code. Record it for a convention-doc update. The code follows the live convention; the document is what needs changing.

**Discard any finding with no convention citation**, however reasonable. Uncitable observations belong to 3f2 and `/simplify`; letting them in turns a conformance gate into a second-opinion generator.

Runs in `--pipeline` mode too, and adds no pause of its own.

**Note on sequencing:** this consolidation cuts the reviewer's cost roughly by the commit count, but the discard rate stays where it is until the convention docs stop contradicting each other. Updating the convention docs is what actually removes the noise. Until then, expect this single review to still spend most of its findings on superseded rules, and lean on the stale-doc branch above.

---

## Phase 6: Completion

After the final step is committed and approved:

1. Work through the **Completion Checklist** in `execution-plan.md`, verifying each static item (symbols, guards, ruff) by grep. If any checklist item requires a running server or live database, flag it as out of scope. Do not mark it as done or attempt to verify it.
2. If the `## Amendments` section is non-empty, present a consolidated summary of all amendments so the user has a single view of where implementation diverged from the plan.
3. If `user_interventions` in the progress marker is non-empty, present a summary: how many interventions, at which steps, of what type. This is the retrospective signal for how well the execution plan was written.
4. Delete the progress marker. Retain the log file (`plans/feat-execute-{feature-name}.log.md`). It is a useful record and will not be committed (the `plans/` directory is gitignored).
5. Update `plans/{feature-name}.pipeline.json`, merging the following entry into the `pipeline` object:

```json
"feat-execute": { "status": "completed", "steps_completed": {N}, "total_steps": {N}, "amendments": {count of entries in the ## Amendments section, 0 if empty}, "timestamp": "{ISO-8601}" }
```

6. If the `## Amendments` section is non-empty, present a prominent callout, visually distinct from the routine completion summary, since it requires user action:

   > **Amendments written, action required**
   > The execution plan deviated in {N} place(s) (see `## Amendments` above). Each one shipped differently from what the audit approved, so **every amendment must be stated in the PR description**. A reviewer comparing the plan to the diff will otherwise read the deviation as an unexplained change.

7. Print a one-line pipeline state from the updated `plans/{feature-name}.pipeline.json`:

   `Pipeline: {grill →} plan {icon} → execution {icon} → audit {icon} → execute {icon}`

   Use `✓` completed/passed · `✗` failed · `⋯` in-progress · `○` not started. Omit grill-with-docs if the key is absent.

8. Report done. Do not suggest follow-up tasks unless step 6 or the checklist reveals an unresolved item.
