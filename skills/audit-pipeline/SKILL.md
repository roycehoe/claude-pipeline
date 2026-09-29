---
name: audit-pipeline
description: Runs the full PR audit pipeline end-to-end: audit-prd → human gate → audit-grill → audit-tests → unresolved resolution gate → audit-validate → simplify. Pauses at the human gate, the unresolved resolution gate, and unresolvable errors. Re-reads pipeline.json at the start of each stage to confirm pr-id. Built for reviewing a large, unfamiliar PR with no spec or ticket context.
when_to_use: Invoke explicitly with /audit-pipeline to run the full PR audit pipeline from start to finish.
allowed-tools: Read Write Bash(find *) Bash(ls *) Bash(grep *) Bash(git branch *) Bash(gh pr view *) Glob Grep Skill
---

# PR Audit Pipeline

Run all audit stages in sequence. Pause only at the human gate, the unresolved resolution gate, and unresolvable errors. Re-read `pipeline.json` at the start of every stage to confirm `pr-id`. Never rely on a value held in working memory from a previous stage.

## Setup

**Re-entry check:** Scan `plans/` for any `*.pipeline.json` file containing a `"pr"` key. If one is found, read it and map its `pipeline` keys to stages:

| pipeline.json key | Stage |
|---|---|
| `audit-prd` | Stage 1 |
| `human-gate` | Stage 2 |
| `audit-grill` | Stage 3 |
| `audit-tests` | Stage 4 |
| `audit-validate` | Stage 5 (`passed` or `passed-with-caveats` counts as completed) |
| `simplify` | Stage 6 |

Find the last key whose status is `completed`, `passed`, or `passed-with-caveats`. If any stages are complete, announce:
> "Found an in-progress audit for **{pr-id}**. Last completed: {stage name}. Resume from Stage {N+1} ({next stage name}), or restart from Stage 1?"

Wait for the user's response. If resuming, skip directly to the appropriate stage.

Announce to the user:
> Running full audit pipeline: **prd → gate → grill → tests → [resolution gate] → validate → simplify**
> I will pause at the discrepancy gate, at the unresolved test resolution gate, and for errors I cannot resolve automatically.

---

## Stage 1: audit-prd

Invoke the `audit-prd` skill with `$ARGUMENTS` as the input (PR URL, branch name, or empty for current branch).

After the skill completes:
- Read `plans/*.pipeline.json` (the file just written) to extract and lock `pr-id`. Use this value for every subsequent stage. Do not interpolate or guess.
- Confirm `plans/{pr-id}-prd.md` was written.

Announce: "Stage 1 complete. PRD written."

---

## Stage 2: Human gate

Re-read `plans/{pr-id}.pipeline.json` to confirm `pr-id` is still the correct value before proceeding.

Present the PRD to the user:

> "The PRD is ready at `plans/{pr-id}-prd.md`. Read through it and list anything that:
> - Contradicts your mental model of what this PR should do
> - Looks unintentional or surprising
> - You want to interrogate further
>
> A short phrase per item is enough. When you have your list, share it here."

Wait for the user's response. Do not proceed until the user provides input.

**If the user provides discrepancies:**
- Write the list verbatim to `plans/{pr-id}-discrepancies-raw.md`.
- Write the human-gate entry to `plans/{pr-id}.pipeline.json`, merging into the `pipeline` object:
  ```json
  "human-gate": { "status": "completed", "output": "plans/{pr-id}-discrepancies-raw.md", "timestamp": "{ISO-8601}" }
  ```
- Announce: "Stage 2 complete. Discrepancy list captured. Starting audit-grill."

**If the user says they have no discrepancies:**
Present three options:
> "No discrepancies noted. Without confirmed defects, Tiers 1 and 2 would be empty. Only Tier 3 happy-path tests could be generated, which is a different workflow from a full audit.
>
> **A.** Stop here. The PRD is your output.
> **B.** Continue to `/audit-tests` for Tier 3 happy-path coverage only (note: Tiers 1 and 2 will be empty).
> **C.** Run a standard code review (your own review skill or command) instead.
>
> Which would you prefer?"

Wait for the user's choice. If **A**, stop. If **C**, hand the PR URL from `source.value` in pipeline.json to your code-review skill or command of choice, then stop. If **B**, write an empty `plans/{pr-id}-discrepancies-raw.md`, write the human-gate and empty audit-grill entries to pipeline.json, and skip directly to Stage 4.

---

## Stage 3: audit-grill

Re-read `plans/{pr-id}.pipeline.json` to confirm `pr-id` before invoking.

Invoke the `audit-grill` skill with `{pr-id}` as the argument.

After the skill completes:
- Confirm `plans/{pr-id}-discrepancies.md` was written.
- Re-read the file to check verdict counts.

**If zero discrepancies are `confirmed`:**
> "Grill found no confirmed defects. All discrepancies were misreadings or intentional design choices. The audit pipeline ends here. Consider a standard code review for a quality pass."
Stop the pipeline.

Announce: "Stage 3 complete. Discrepancies verdicted. Starting audit-tests."

---

## Stage 4: audit-tests

Re-read `plans/{pr-id}.pipeline.json` to confirm `pr-id` before invoking.

Invoke the `audit-tests` skill with `{pr-id}` as the argument.

After the skill completes:
- Confirm `plans/{pr-id}-tests.md` was written.
- Re-read the pipeline.json to check if unresolved pairs were generated.

Announce: "Stage 4 complete. Test plan written."

---

## Stage 4.5: Unresolved resolution gate

Re-read `plans/{pr-id}-tests.md` and count test cases containing the phrase "D-NNN is unresolved" in their blockquotes (substituting actual IDs).

**If no unresolved pairs exist:** proceed directly to Stage 5.

**If unresolved pairs exist:** pause and present:
> "{N} unresolved discrepancy pair(s) require a human decision before validation can run.
>
> For each pair in `plans/{pr-id}-tests.md`, read both sides and delete the test that does not match the intended behavior. The remaining test will be evaluated by `/audit-validate`.
>
> Unresolved pairs:
> {list each D-NNN with its title}
>
> When you have resolved all pairs, reply 'done' to continue."

Wait for the user to confirm. Do not invoke `audit-validate` until confirmation is received.

Announce: "Stage 4.5 complete. Unresolved pairs resolved. Starting audit-validate."

---

## Stage 5: audit-validate

Re-read `plans/{pr-id}.pipeline.json` to confirm `pr-id` before invoking.

Invoke the `audit-validate` skill with `{pr-id}` as the argument.

After the skill completes, re-read pipeline.json to check the recorded verdict:

**If `passed` (APPROVE):**
Announce: "Stage 5 complete. All regression cases covered. Proceeding to simplify."
Continue to Stage 6.

**If `passed-with-caveats` (APPROVE WITH CAVEATS):**
Present the manual verification section from the validate report. Tell the user:
> "All static regression cases are covered, but {N} Tier 1 case(s) require manual verification. Review the cases above, verify them manually, then run Stage 6 (`/simplify`) when ready, or stop here if you prefer to close the audit after manual verification."
Wait for the user to confirm before continuing.

**If `failed` (REQUEST CHANGES):**
Stop. Present the blockers from the validate report. Tell the user:
> "The implementation has {N} uncovered regression case(s). The PR needs code changes before this audit can close. Once changes are made, re-run `/audit-validate` to confirm coverage."

---

## Stage 6: simplify

Re-read `plans/{pr-id}.pipeline.json` to confirm `pr-id` before proceeding.

**Applicability check:** If the source type in pipeline.json is `"pr"` and the PR branch is not checked out locally, skip this stage:
> "Simplify is not applicable. The PR branch is not checked out locally."

Otherwise invoke the `simplify` skill.

After the skill completes, write the simplify entry to `plans/{pr-id}.pipeline.json`:
```json
"simplify": { "status": "completed", "timestamp": "{ISO-8601}" }
```

---

## Completion

Re-read `plans/{pr-id}-discrepancies.md` to get the confirmed defect count.
Re-read `plans/{pr-id}-tests.md` to get the total test case count.

Report:
- PR audited: `{pr-id}`
- Verdict: `{verdict from pipeline.json}`
- Confirmed defects found: `{N}`
- Test cases written: `{N}`

"Audit complete. Artifacts are in `plans/{pr-id}-*.md`."
