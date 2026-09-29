---
name: feat-execution
description: Given a feature plan (the output of /feat-plan), produces a detailed plans/{feature-name}-execution.md with numbered atomic commits in strict dependency order. Each commit has a conventional commit message, file list, full implementation detail, and an expected-state checkpoint. Greps the codebase to verify all referenced methods exist and extracts volatile constants for user confirmation before drafting begins. Drafts one commit step at a time with pause-and-review after each.
when_to_use: Invoke explicitly with /feat-execution to translate a feature plan into atomic commits, after completing /feat-plan, or run when /feat-pipeline orchestrates it. Second step of the feat- pipeline.
allowed-tools: Read Write Bash(grep *) Bash(find *) Bash(ls *) Bash(git log *) Bash(git show *) Glob Grep WebSearch WebFetch
---

# Plan Execution

Translate a feature clarification doc into a detailed execution plan: an ordered sequence of atomic, independently-compilable commits that collectively implement the feature.

Do not write application code. Do not run migrations. Do not make commits. The output is a single document, `execution-plan.md`, that a future execution skill will implement step by step.

## Current project state

```!
git log --oneline -10
```

```!
ls plans/*.md 2>/dev/null
```

---

## Phase 1: Locate inputs

**Gap report (retry mode):** If `$ARGUMENTS` contains `--gap-report {path}`, read that file. It is the output of a failed `/feat-audit` run. Extract every blocker. Hold this list. Every blocker must be addressed during drafting. If an existing `plans/{feature-name}-execution.md` exists, read it to see what was previously drafted; revise only what the blockers require. Note in the output document's preamble that this is a revised plan addressing the audit gap report.

In gap-report mode, skip phases that are not implicated by the blockers:
- **Skip Phase 3** (volatile constants) unless a blocker explicitly names a constant value mismatch.
- **Skip Phase 5** (commit order) unless a blocker requires adding, removing, or reordering commits.
- **Phase 4** (symbol verification): only re-verify symbols that appear in the implementation detail of the steps being revised, not all symbols in the plan.

Proceed directly to Phase 6 (draft the affected steps only) after completing any phases that cannot be skipped.

**Clarification doc:** If a path was passed as `$ARGUMENTS` (and it is not the `--gap-report` path), read that file. Otherwise, scan the `plans/` directory for `.md` files that do not end in `-execution.md`, `-audit.md`, or `-tests.md`. If multiple candidates exist, list them and ask the user which to use. If none exist, stop. Tell the user to run `/feat-plan` first or provide the path explicitly.

**Pipeline state:** Derive the feature name from the clarification doc filename. Read `plans/{feature-name}.pipeline.json`. If the file does not exist, or if `feat-plan.status` is not `completed`, stop. Tell the user to run `/feat-plan` first to initialize the pipeline.

**Convention files:** Read the *nearest* `CLAUDE.md` and `CONVENTIONS.md` for every subsystem this plan touches, plus the project root's, and treat the most specific as authoritative (the file names are configurable, see the README). Extract the commit message format, implementation order, naming conventions, and any codebase-specific rules (e.g. required route decorators, bottom-up layer order). A root convention file that only delegates to subsystem files carries none of these. If you extracted nothing from it, say which subsystem files you read instead. These conventions must be followed in every drafted commit step, and they outrank this plan where they conflict.

Read the clarification doc in full. Extract and record:

1. **Feature name.** Used to derive the output filename and progress marker.
2. **All implementation steps** described in the doc, including any amendments appended after initial drafting.
3. **Volatile constants.** Every value that represents a business rule likely to change: retention periods, numeric thresholds, string literals in notification bodies, enum values that are not yet committed to code, timing constants. Flag anything that appears to be a decision rather than a derivation. Also flag any string literal or numeric literal used in application logic that is not a well-known framework value (HTTP status codes, `None`, `True`, `False`). These must be assigned a named constant at module level in the same commit step that introduces them.
4. **Named symbols.** Every method, function, class, and importable module referenced by name (e.g. `UserRepo.get_by_email`, `billing.invoices.close_invoice`, `notify_users`).

---

## Phase 2: Check for an existing progress marker

Look for `plans/feat-execution-{feature-name}.progress.json`.

If found, read it and report:
- How many steps have been drafted and approved
- How many remain
- Ask whether to resume from the last approved step or start fresh

If starting fresh, delete the existing progress marker before proceeding.

---

## Phase 3: Volatile-constant pre-flight

Present every volatile constant in a numbered list. For each entry, show:
- A short label describing what it controls
- The value as it appears in the clarification doc
- The specific plan step that uses it

Ask the user to confirm or correct each value explicitly. Do not proceed until every constant is confirmed. Record the confirmed values. Use them verbatim when writing any commit step that references them. A constant confirmed here must not silently change in a later step.

---

## Phase 4: Symbol-verification pre-flight

**Exploration artifact shortcut:** Before grepping the codebase, check if `plans/{feature-name}-explore.md` exists. If found, read it. Symbols listed in its "Symbols confirmed" table are already verified. Skip the grep for those. Only grep for symbols not covered by the artifact. The artifact was written at planning time; if it seems stale (e.g. the branch has diverged significantly), treat it as advisory, not authoritative.

For every named symbol extracted in Phase 1 that is not already confirmed by the exploration artifact, grep the codebase to verify it exists. Report results in two groups:

**Found:** symbol, file, line number.

**Not found:** symbol, and which plan step(s) reference it.

For each missing symbol, invoke the Blocker Resolution Protocol before asking the user to decide:

- **Codebase pattern.** Grep for the closest matching symbol name (partial name, camel-case variants, underscore variants). Quote any candidate found, its file, and a sample call site so the user can judge whether it is the intended method.
- **Best practice.** State the conventional substitute or naming convention for this symbol type (e.g. `get_by_x` → `get_by_x_or_none`, a CRUD method vs. a service method).
- **Canonical docs.** If the symbol belongs to a library or framework (SQLAlchemy, FastAPI, Pydantic, etc.), WebSearch the official docs for the nearest matching API and quote the correct usage.

After presenting proposed solutions and a recommendation, ask the user to resolve it:
- Is the symbol introduced by an earlier commit step (confirm it will exist by the time the referencing step runs)?
- Does one of the proposed solutions name the correct symbol, so adopt it?
- Does the referencing step need to be revised?

**Naming convention check:** For every repository (CRUD) method or service method named in the plan, look up the pattern in the naming table of the *nearest* convention file to the file that will contain it. For `app/repo/foo.py` that is `app/repo/CONVENTIONS.md`, not the project root. If the nearest file delegates upward, follow the chain until you find an actual table; if none exists anywhere on the chain, say so rather than treating the check as passed. Confirm the name matches the expected verb (or verb + noun) pattern before including it in a step. Flag any mismatch as a missing-symbol issue. Do not carry a non-conforming name into Phase 5 with a rationale attached. If you believe the documented table is wrong, that is a one-line question to the user, not a deviation you author.

Do not proceed to Phase 5 until every missing symbol is accounted for and every method name is confirmed against `CLAUDE.md` conventions.

---

## Phase 5: Establish commit order

Before drafting any step in detail, determine the complete ordered list of commits. Present it as a numbered list of commit titles only (no detail yet) and wait for user approval.

**Module placement gate:** For every new file this plan introduces, state which module description it maps to, citing the *nearest* convention file that describes that module (a file under `app/` is described by `app/CONVENTIONS.md`, not the project root). Apply the project's placement rules as written there; a typical one is that a `utils/` directory holds pure functions only, so a helper that makes network calls, does file I/O or calls an external service belongs in an integration layer instead. Record the mapping and the citation for each new file before proceeding.

Commits must follow bottom-up dependency order so each file compiles against already-committed code. Use the layer order defined in the project's convention file. If it defines none, use this default:

1. Enums and shared constants
2. Database models
3. Database migration: immediately after model changes; committed before any application code depends on the new columns
4. Exceptions and error codes
5. Integration / gateway functions (external calls)
6. Pure utility functions
7. Data-access (CRUD) methods
8. Schemas
9. Validators
10. Services
11. Routers / handlers
12. Application entry point: startup, background tasks, route registration

Deviate from this order only when the clarification doc's dependency structure requires it; explain why when you do.

If the user requests reordering, validate that no step would reference code from a later step before accepting the change.

**The same file appearing in multiple non-consecutive steps is correct and intentional.** Each step is atomic. It modifies a file for a single, distinct purpose. Do not consolidate steps to reduce file repetition. Example: `enums.py` appears in step 1 (add a new status value) and `services/orders.py` appears in steps 13, 14, and 15 (cancel, refund, nightly cleanup) because each step adds a distinct group of functions. Consolidating them would violate atomicity and make each commit harder to review and revert independently.

---

## Phase 6: Draft commit steps with pause-and-review

**Auto-approve mode:** If `$ARGUMENTS` contains `--auto-approve`, skip the pause after each step. Draft all steps in sequence without waiting for approval. Present each step as it is drafted so the user can follow along, without waiting for a response before continuing. In auto-approve mode, rejections are not possible. The user relies on `/feat-audit` to catch problems. Note at the start: "Running in auto-approve mode. All steps will be drafted without pauses. Run `/feat-audit` when complete."

Draft one commit step at a time. After presenting each step, wait for explicit approval before drafting the next. If the user rejects a step, revise it based on their feedback and re-present. Do not move on.

Each commit step must contain the following five sections:

### 1. Commit title
A conventional commit message that will be used verbatim as the git commit message:
- `feat:` for new functionality
- `fix:` for correcting wrong behaviour
- `chore:` for migrations, dependency changes, generated files
- `refactor:` for restructuring without behaviour change

**Renames must be their own commit.** If a step discovers an existing symbol (method, class, variable) needs renaming, do not embed the rename in the step that introduces or uses it. Create a dedicated `refactor:` commit step for the rename, placed immediately before the step that depends on the new name. This ensures the amendment protocol in the execution skill can correctly attribute the change and grep remaining steps for side effects.

### 2. Files
An explicit list of every file that changes in this commit. No file should appear in a later step if it could be completed here without creating a forward dependency.

### 3. Pattern reference
Before drafting the implementation detail, grep the codebase for the most analogous existing implementation in the same layer as the files in this step. Record the result as a per-file mapping:

- **Analogous file found:** record as `path/to/file.py:L{start}-L{end}` with a one-line description of the pattern it demonstrates (e.g. `routers/orders.py:L42-L58, auth dependency + response model pattern`).
- **No analogous file:** record as `none, introducing new pattern`.

This field is not optional. A step with no entry is incomplete. The execution skill reads this field before writing any code and will follow the named pattern structurally, or trigger the amendment protocol if it must diverge.

### 4. Implementation detail
Full technical detail, not a high-level description but exact instructions:
- Exact function signatures with all parameters, types, and return types
- Import statements for every symbol not already present in the file
- Code snippets for all non-obvious logic: join paths, bulk update patterns, enum comparisons, async wrapping, nullable FK patterns
- The verbatim confirmed values for every volatile constant this step uses (from Phase 3)
- For route additions: every per-route requirement the convention file sets (decorators, dependencies), citing the rule that requires it
- Do not write "use the same pattern as X". Look up X in the codebase (grep + read the relevant function) and write the concrete detail into this section. The pattern reference field names the anchor; this section describes what to do with it.
- Exception handlers must catch specific exception types, not broad `Exception` or bare `except`. Each clause must name the exception it catches.

### 5. Expected state after this commit
A checklist of verifiable conditions that must be true after this commit and before the next one begins. Write only conditions that can be verified statically, by reading files and grepping committed code, without running the application, executing migrations, or starting a server.

Acceptable items:
- Specific symbols are importable from specific modules (grep-verifiable)
- Specific functions, classes, or methods exist in specific files (grep-verifiable)
- `ruff check .` passes with no errors
- Specific guard conditions or exception raises exist in specific functions (grep-verifiable)

Do not write items that require a running server, a connected database, or a compiled migration:
- ❌ the dev server starts without errors
- ❌ Endpoint returns HTTP 204
- ❌ the migration command completes successfully
- ❌ Column exists in the live database

Those belong in a separate testing skill. The execution skill only verifies statically.

This block is what the future execution skill verifies at each commit boundary before proceeding.

---

After the user approves a step, update the progress marker:

```json
{
  "clarification_doc": "plans/{filename}",
  "output_file": "plans/{feature-name}-execution.md",
  "feature_name": "{feature-name}",
  "constants_confirmed": true,
  "symbols_verified": true,
  "commit_titles": ["{title-1}", "{title-2}", "..."],
  "steps_approved": [1, 2],
  "total_steps": 18,
  "last_updated": "{ISO-8601 timestamp}"
}
```

Write this to `plans/feat-execution-{feature-name}.progress.json`.

---

## Phase 7: Write the output file

Once all steps are approved, write `plans/{feature-name}-execution.md`. Create the `plans/` directory if it does not exist. If a file already exists at that path, ask before overwriting.

```markdown
# Execution Plan: {Feature Name}

## Amendments

_This section is left empty. When the execution skill discovers that implementation must deviate from a step, whether a method name is wrong, a join path differs, or a constant changes, it appends the amendment here rather than silently adapting. The plan remains the authoritative record._

---

## Completion Checklist

- [ ] Each step's "Expected state" block verified before proceeding to the next step
- [ ] `ruff check .` passes after every commit
- [ ] [Additional static checklist items derived from the clarification doc's resolved tradeoffs and gaps]

_Server-dependent checks (dev server start, migrations, endpoint responses) are out of scope for this skill. They are verified by a separate testing skill after execution is complete._

---

## 1. `{conventional commit message}`

**Files:** `path/to/file.py`, `path/to/other.py`

**Pattern reference:**
- `path/to/file.py` → `routers/analogous.py:L42-L58`, {description of pattern}
- `schemas/foo.py` → none, introducing new pattern

{Implementation detail: full technical instructions}

**Expected state after this commit:**
- [ ] {Verifiable condition}
- [ ] {Verifiable condition}
- [ ] `ruff check .` passes

---

## 2. `{conventional commit message}`

...
```

Confirm the written path to the user. Delete the progress marker once the file is successfully written. It has served its purpose.

Tell the user: "Run `/feat-audit` to verify this execution plan against the feature plan."

Update `plans/{feature-name}.pipeline.json`, merging the following entry into the `pipeline` object:

```json
"feat-execution": { "status": "completed", "output": "plans/{feature-name}-execution.md", "timestamp": "{ISO-8601}" }
```

Print a one-line pipeline state from the updated `plans/{feature-name}.pipeline.json`:

`Pipeline: {grill →} plan {icon} → execution {icon} → audit {icon} → execute {icon} → validate {icon}`

Use `✓` completed/passed · `✗` failed · `○` not started. Omit grill-with-docs if the key is absent.
