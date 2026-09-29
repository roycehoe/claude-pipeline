---
name: feat-status
description: Shows the current pipeline state for one or all in-flight features. Reads plans/*.pipeline.json and prints a status board showing which steps are completed, passed, failed, or pending.
when_to_use: Invoke explicitly with /feat-status to see which pipeline steps are done and which feat- step to run next.
allowed-tools: Read Bash(find *) Bash(ls *) Glob
---

# Feature Pipeline Status

Read pipeline state files and print a status board for each in-flight feature.

## Phase 1: Locate pipeline files

If a feature name was passed as `$ARGUMENTS`, read `plans/{feature-name}.pipeline.json` directly. If the file does not exist, report that no pipeline has been started for that feature and stop.

If no argument was given, scan `plans/` for all `*.pipeline.json` files. If none exist, report that no feature pipelines have been started in this project and stop. If multiple exist, show the status board for all of them.

---

## Phase 2: Render the status board

For each pipeline file, walk the `pipeline` object's keys **in insertion order**. Render one row per key. Do not hardcode which keys to expect. Render whatever is present.

```
Feature: {feature-name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  {N}. {key}   {icon} {status}  {annotation}
  ...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Next step: {next-step}
```

Number rows from 0 in insertion order. Left-align key names for readability.

**Status icons:**
- `✓`: `completed` or `passed`
- `✗`: `failed`
- `⋯`: `in-progress`
- `○`: key is present but status is `null`, or key is absent from the object

**Annotations.** Append to the row when the entry contains these fields:

- Entry has `"output"` and status is `failed`: append `(see {output})`
- Entry has `"steps_completed"` and `"total_steps"` and status is `in-progress`: append `({steps_completed}/{total_steps} commits done)`
- Entry has `"blockers"` and status is `failed`: append `({blockers} blocker(s))`

An entry may match more than one annotation rule. Append all that apply.

---

## Phase 3: Determine next step

Walk the `pipeline` keys in insertion order. Find the first key whose status is not `completed` and not `passed`:

- **Status is `null` or key is absent from the pipeline object** → `Run /{key}`
- **Status is `failed`** → `Resolve issues in {entry.output} then re-run /{key}` (if `output` is present); otherwise `Re-run /{key}`
- **Status is `in-progress`** → `Continue /{key} (step {steps_completed + 1} of {total_steps})` (if progress fields present); otherwise `Continue /{key}`

If every key in the pipeline object has status `completed` or `passed` → `Pipeline complete`
