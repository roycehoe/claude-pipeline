---
name: feat-quick
description: The lean, one-shot lane for shipping a small, well-scoped feature in a single command: grill the spec, propose the exact code changes for approval, then commit them atomically. Reuses grill-with-docs for the interrogation, then collapses planning and execution into one approval gate, skipping the audit/tests/validate stages of the full /feat-pipeline.
when_to_use: Invoke explicitly with /feat-quick for small, well-scoped features where the full feat- pipeline is overkill. A single command to take a fuzzy description through interrogation, a code-level change proposal, and atomic commits, with a single approval gate and no audit/test/validate stages. If the feature turns out to touch many layers, introduce a state machine, or need a database migration, the skill bails out and hands off to /feat-pipeline.
allowed-tools: Read Write Edit Bash(git add *) Bash(git commit *) Bash(git status*) Bash(git diff*) Bash(git log*) Bash(grep *) Bash(find *) Bash(ls *) Glob Grep Skill
---

# Feat Quick

The fast lane for small features. The full `/feat-pipeline` runs eight stages with audit, tests, and validation, the right call for anything substantial, but heavy when all you want is to ship a small, well-understood change. `feat-quick` keeps the part that actually de-risks a feature (a relentless interrogation that locks the spec) and the part that protects your repo (an explicit approval gate before any code is written), and drops the rest.

The shape: **one command, one approval gate.** You answer the grilling until the spec is locked, then you approve the exact code changes. That single approval is also what authorizes the commits, and the skill commits them. Nothing is hands-off. The interactivity is the point, because it's what keeps a "simple" feature honestly simple.

This skill reuses `grill-with-docs` outright for the interrogation rather than reimplementing it. But it deliberately does **not** call `feat-plan` or `feat-execution`. Those are heavy by design (volatile-constant confirmation, symbol verification, per-step pauses, their own output files), and pulling them in would defeat the whole purpose. Instead it produces one lightweight, code-level proposal inline. That trade, reusing the grilling and purpose-building the proposal, is the reason this lane exists.

---

## Setup: resume check

Before anything else, **glob `plans/*-brief.md`** to discover any existing brief. Do not reconstruct the filename from `$ARGUMENTS`. `grill-with-docs` chooses the feature name when it writes the brief, and a name you derive independently can diverge from the one on disk, causing a resume to miss the brief and needlessly re-grill. Scan for the file; don't assume its name.

- **No brief found.** Start at Phase 0.
- **Exactly one brief found.** A prior run already locked the spec. Grilling is the expensive, human-intensive part. Never redo it. Announce: "Found an existing brief for **{feature-name}**. Skipping the interrogation and resuming from the change proposal." Read the brief in full and skip directly to **Phase 2 (complexity gate)**. (The proposal and commits are cheap to regenerate, so there is no finer-grained resume state to track. The brief's existence is the only signal needed.)
- **More than one brief found.** Match each against `$ARGUMENTS`; if exactly one clearly corresponds to this feature, resume from it as above. Otherwise ask the user which feature this is, rather than guessing.

---

## Phase 0: Should you even grill?

The grill in Phase 1 is mandatory, but it is pure latency when the spec is already crisp. Before invoking it, ask one question: **can you write the one-line acceptance check for this feature without reading any code?**

- **Yes, and the change is single-layer**, or it is a behaviour-preserving refactor with a green test set as its oracle. The spec is already locked and feat-quick adds only ceremony. Decline and route the user to Plan Mode:
  > The spec is already crisp and the change is small. Plan mode is the right lane, no grill needed.

  Then end.
- **No** (the spec is fuzzy, borrowed, or leans on overloaded terms). Grilling is exactly what de-risks this. Continue to Phase 1.

(This only runs on the no-brief path. If the resume check found an existing brief, the grill already happened. Skip straight to Phase 2.)

---

## Phase 1: Grill

Invoke the `grill-with-docs` skill with `$ARGUMENTS` as the feature description.

This is an active interrogation. Answer its clarification questions and scenario challenges as they come. They are how the spec gets precise enough to one-shot. When `grill-with-docs` finishes, it will have written `plans/{feature-name}-brief.md`. That brief is the persisted spec: if this session is interrupted, the resume check above means you never have to be grilled again.

Read the brief once `grill-with-docs` confirms its path, and record the feature name. Every path below derives from it.

---

## Phase 2: Complexity gate

`feat-quick` skips the audit, test-plan, and validation stages that catch mistakes in larger features. That trade is only safe if the feature is small AND low-risk. Run these checks **mechanically** against the brief and the files it names. Do not eyeball them. This gate runs in the same context that will write the proposal and shares its blind spots, so it must lean on tool output, not judgment.

**Size tripwires. If ANY fires, STOP and bail to the pipeline:**

1. **Many layers.** Count the distinct stack layers the brief's Decisions touch (database/models, migration, CRUD, services, routers, schemas, background jobs, entrypoint). More than two → bail.
2. **A state machine.** `grep -niE 'status|state|phase|stage|lifecycle'` the touched model files, and scan the brief for lifecycle verbs (archive, publish, expire, approve, retry, cancel, soft-delete). Any new or modified status field or transition → bail.
3. **A non-trivial migration.** `ls` the migrations dir and check the brief for column/table/index changes. It counts as non-trivial, and bails, if it renames, adds a non-nullable column, backfills, or alters a column with existing readers or ordering. An **additive nullable column with no readers does NOT count**. It is safe for the lean lane.

If any size tripwire fires, stop. Do not propose changes. Tell the user:

> This feature isn't a good fit for the lean lane. It {names the tripwire(s), e.g. "spans models, a migration, and services" / "introduces a deletion lifecycle"}. Skipping audit and tests on a change this size is risky. The spec is already locked in `plans/{feature-name}-brief.md`, so run **`/feat-pipeline`**. It will pick up the brief with no rework and run the full stages.

Then end. The brief on disk means the handoff costs the user nothing.

**Blast-radius tripwire. Does NOT bail, but mandates a risk pass:**

4. **High blast radius.** Does the change touch authentication/authorization, money/billing arithmetic, data deletion, or redaction/masking? A small auth fix is still small. Do NOT inflate it to eight stages. Instead set a `RISK_PASS_REQUIRED` flag and keep going; Phase 4.5 will append a fresh-context review before you finish. Risk and size are orthogonal: match the lane to size, the check to risk.

If no size tripwire fires, continue to Phase 3.

---

## Phase 3: Change proposal

Produce a single, concrete, code-level proposal, and present it for approval before writing anything. This is the collapsed plan-and-execution step: enough detail that the user is approving the *actual change*, not a vague intention.

First, ground the proposal in the codebase:
- Read `CLAUDE.md` from the project root for naming conventions, layer responsibilities, and commit format. The proposal must not conflict with these.
- Grep and read every file the change will touch. Confirm that any symbol you plan to call actually exists. Do not propose against assumptions.
- For each file you'll modify, find the most analogous existing code and follow its pattern rather than inventing a new one.

Then present the proposal in this exact structure:

```
## Proposed change: {feature-name}

### Commits
A numbered list of atomic commits in dependency order. Each line: the conventional-commit
subject (under 72 chars) and the file(s) it touches. Commits are atomic by purpose, one
distinct change each, ordered so no commit depends on a later one. Do not group unrelated
changes together to reduce commit count.

### Changes
For each commit, the actual code to be written: real diffs or full code blocks for the
files involved, not prose descriptions. Show enough that the user can read the change and
approve it as written. Match the surrounding code's style.

### Notes
Anything the user should weigh in on: a constant whose value is a judgment call, an edge
case the brief left implicit, a place the change could ripple wider than expected. Keep it
short. If there's nothing material, omit this section.
```

Then present this approval-and-safety block and wait for the user to type `GO` (case-insensitive) before writing anything. Approving the proposal and authorizing the commits are one decision. This is the skill's single gate, and it exists because reviewing a diff on paper is far cheaper than reviewing it after it's committed.

```
## Approve this change

Reply GO to build the proposal exactly as written above. This will:
- commit each step locally, using the conventional-commit subjects shown
- NOT push to any remote
- NOT run a database migration or connect to a live database

Or tell me what to change.
```

If the user requests changes, revise the proposal and re-present it. Do not start writing code until you receive `GO`. Answer any questions, re-present the block, and do not proceed until `GO` is received.

---

## Phase 4: Execute

After `GO`, implement the approved proposal commit by commit, in the order presented:
- Make only the changes in the approved proposal. If implementing reveals that the proposal was wrong, whether a symbol doesn't exist or a signature differs, stop and tell the user what you found and how the proposal needs to change, rather than silently adapting. A surprise during execution is a sign the proposal wasn't ready.
- Commit each step atomically using its conventional-commit subject verbatim. Follow `CLAUDE.md`'s commit rules: Conventional Commits format, subject under 72 characters, **no attribution or co-author trailers**.
- Stage only the files belonging to that commit. Do not bundle multiple commits' files together.

---

## Phase 4.5: Risk pass (only if RISK_PASS_REQUIRED was set in Phase 2)

If the blast-radius tripwire fired, do not finish on the author's own judgment. Invoke a fresh-context review against the commits you just made:
- auth/authz, data deletion, or redaction/masking → a security-review skill or a fresh-context security pass
- money/billing arithmetic → `/feat-tests`

The point is the fresh frame: "looks right" from the author who wrote the diff is not "is right". Address anything it surfaces before writing the summary.

---

## Phase 5: Summary

After the last commit, report concisely:
- The feature name
- Every file changed
- Each commit made, by subject line, in order

Then end. No further stages run. If the user later wants tests or a deeper review, point them at `/feat-tests` or `/feat-pipeline`.
