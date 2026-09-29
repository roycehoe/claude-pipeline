---
name: audit-grill
description: Takes the human discrepancy list and investigates each entry against the codebase and canonical docs. Classifies each as confirmed, misread, intentional, or unresolved, without asking the user questions. Reads the diff source from pipeline.json rather than re-deriving it. Writes plans/{pr-id}-discrepancies.md. Third step of the audit-pipeline, after the human gate.
when_to_use: Invoke explicitly with /audit-grill after the human gate in /audit-pipeline has provided a discrepancy list, or run as Stage 3 when /audit-pipeline orchestrates it.
allowed-tools: Read Write Bash(gh pr diff *) Bash(git diff *) Bash(find *) Bash(ls *) Bash(grep *) Glob Grep WebSearch WebFetch
---

# Audit Grill

Investigate the human discrepancy list against codebase evidence and canonical docs. Produce verdicts without asking the user any questions.

## Phase 1: Load inputs

**Locate the pipeline:** If `$ARGUMENTS` is provided, treat it as `pr-id` and read `plans/{pr-id}.pipeline.json`. Otherwise scan `plans/` for all `*.pipeline.json` files containing a `"pr"` key. If multiple exist, pick the one where `audit-prd.status` is `completed` and `audit-grill` is absent or incomplete. If ambiguous, list the options and ask.

Confirm `audit-prd.status` is `completed`. If not, stop. Tell the user to run `/audit-prd` first.

**Read the PRD:** Read `plans/{pr-id}-prd.md` in full.

**Read the discrepancy list:** Check for `plans/{pr-id}-discrepancies-raw.md`. If it exists, read it. If it does not, the discrepancies were provided inline in this conversation. Use those. If neither source is present, ask the user to provide their discrepancy list before continuing.

Number each discrepancy D-001, D-002, etc. in the order given.

**Get the diff from pipeline.json:** Read the `source` field from `plans/{pr-id}.pipeline.json`. If `source` is absent, stop:
> "`pipeline.json` has no `source` field. This means `audit-prd` was run with an older version of the skill. Re-run `/audit-prd` to regenerate the PRD and record the source, then continue."

Use the recorded source to fetch the diff:
- `source.type` is `"pr"`: run `gh pr diff {source.value}`
- `source.type` is `"branch"`: run `git diff {source.base}..HEAD`

Do not re-derive the diff independently. Do not fall back to origin/staging or origin/main.

---

## Phase 2: Investigate each discrepancy

For each discrepancy, run a three-step investigation. Do not ask the user anything. Work entirely from code and docs.

**Step 1: Codebase check**
Grep the diff and relevant source files for evidence that confirms or refutes the discrepancy. Read the file sections at the citation lines listed in the PRD for this area. Quote the single most relevant snippet.

**Step 2: Canonical docs check** *(when applicable)*
If the discrepancy involves the behavior of a library, framework, language feature, or external API: WebSearch or WebFetch the canonical documentation. Quote the relevant passage and its source URL. Skip this step if the discrepancy is entirely internal to application logic.

**Step 3: Assign verdict**

| Verdict | Assign when |
|---------|-------------|
| `confirmed` | Codebase evidence shows the behavior is present and is not what a reasonable implementation would do. Likely a bug, unintended side effect, or missing guard |
| `misread` | The human's concern was based on a misreading of the PRD or code. The discrepancy disappears under closer inspection |
| `intentional` | The behavior is surprising but defensible; consistent with a plausible design decision, even if undocumented |
| `unresolved` | Evidence is ambiguous; cannot determine from static analysis whether this is a defect or intentional |

Do not assign `confirmed` solely because the behavior is unusual. Require a concrete code path showing the problematic behavior.

---

## Phase 3: Write the verdicted discrepancy file

Write `plans/{pr-id}-discrepancies.md`. Create `plans/` if it does not exist.

```markdown
# {PR title or branch name}: Discrepancies

**{N} confirmed · {N} misread · {N} intentional · {N} unresolved**
**Investigated:** {ISO-8601}

---

## D-NNN: {short title}
**Verdict:** confirmed | misread | intentional | unresolved

**Discrepancy:** {original concern, verbatim or lightly paraphrased}

**Evidence:**
> `path/to/file.ext:NN`, {quoted snippet}

**Finding:** {one or two sentences: what the code actually does and why this verdict applies}

**Doc reference:** [{source title}]({URL}), {quoted passage}
*(omit this line if no canonical doc check was performed)*
```

Confirm the written path to the user.

Update `plans/{pr-id}.pipeline.json`, merging only the following into the `pipeline` object (the `human-gate` entry is written by the orchestrator at Stage 2, not here):

```json
"audit-grill": { "status": "completed", "output": "plans/{pr-id}-discrepancies.md", "timestamp": "{ISO-8601}" }
```

Print pipeline state:
`Pipeline: prd ✓ → gate ✓ → grill ✓ → tests ○ → validate ○`

**Recovery guidance:**

- **If zero discrepancies are `confirmed`:** Tell the user no confirmed defects were found. All discrepancies were either misreadings or intentional design choices. The audit pipeline ends here. Suggest a standard code review for a quality pass.

- **If `confirmed` discrepancies exist and no `unresolved` exist:** Tell the user to run `/audit-tests` to generate regression test cases.

- **If `unresolved` discrepancies exist:** Explicitly warn the user:
  > "**Note on `unresolved` items:** These will generate two-sided test cases in `/audit-tests`: one assuming the behavior is intentional, one assuming it is a defect. You will need to resolve which to keep before `/audit-validate` runs. Plan for a manual decision point after audit-tests completes."

  Then tell the user to run `/audit-tests`.
