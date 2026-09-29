---
name: audit-prd
description: Reads a PR diff and produces a cited behavioral description of what the code does, not what it claims to do. Every claim is anchored to a file:line citation. Claims that cannot be cited are flagged [uncertain]. Records the diff source in pipeline.json so downstream skills do not re-derive it. Writes plans/{pr-id}-prd.md. First step of the audit-pipeline.
when_to_use: Invoke explicitly with /audit-prd to generate a cited PRD from a PR diff, or run as Stage 1 when /audit-pipeline orchestrates it.
allowed-tools: Read Write Bash(gh pr view *) Bash(gh pr diff *) Bash(git diff *) Bash(git log *) Bash(git branch *) Bash(git merge-base *) Bash(find *) Bash(ls *) Bash(grep *) Glob Grep
---

# Audit PRD

Produce a cited behavioral description of what a PR or diff observably does, not what it claims to do.

## Phase 1: Identify the source

**If `$ARGUMENTS` is a GitHub PR URL:**
- Extract the PR number. Set `pr-id` to `pr-{N}`.
- Fetch metadata: `gh pr view $ARGUMENTS --json number,title,headRefName,baseRefName`
- Fetch the diff: `gh pr diff $ARGUMENTS`
- Build source record: `{ "type": "pr", "value": "{URL}", "base": "{baseRefName}", "head": "{headRefName}" }`

**If `$ARGUMENTS` is a branch name:**
- Set `pr-id` to the branch name with `/` replaced by `-`, lowercased.
- Run `git merge-base origin/staging HEAD 2>/dev/null || git merge-base origin/main HEAD` to find the base commit. Note which remote succeeded.
- Run `git diff {base}..HEAD`.
- Build source record: `{ "type": "branch", "value": "{branch}", "base": "{base commit}", "remote": "origin/staging|origin/main" }`

**If no argument is given:**
- Run `git branch --show-current` and derive `pr-id` (replace `/` with `-`, lowercase).
- Same base detection and diff as the branch case.
- Build source record the same way.

If the diff is empty, stop and report that no changes were found.

**Size check:** Count the lines in the diff. If it exceeds 500 lines, warn the user:
> "This diff is large ({N} lines). The PRD will group behaviors by area. Claims spanning broad patterns may be marked `[uncertain]` where a specific citation would require reading hundreds of lines. Consider re-running `/audit-prd` on individual files if more precision is needed."
Continue regardless.

---

## Phase 2: Survey the changed files

List all changed files from the diff. For each file determine:
- Which layer it belongs to (model, service, router/handler, background job, config, test, migration, etc.)
- Which domain concepts it touches based on path and filename

Group changed files into behavioral areas: cohesive concerns rather than file groupings. Aim for 3-8 areas. Examples: "Authentication Flow", "Order State Machine", "Background Sync Job", "API Response Schema". Do not read file contents in this phase; group by name and path only.

---

## Phase 3: Extract behavioral claims

For each behavioral area, read the relevant diff sections carefully. For each observable behavior introduced or modified, write one claim.

**Rules:**
- Claims describe what the system does, not what the code does. "The session is invalidated when X" not "the `logout()` method calls `session.delete()`."
- Each claim is followed by one or more `file:line` citations, the specific diff lines that establish it.
- Claims that cannot be directly cited are appended with `[uncertain: inferred from context, not explicit in diff]`. Do not omit inferred behaviors; flag them.
- Cite the line that enacts the behavior, not scaffolding or imports.
- One behavior per claim. Do not bundle.

Also note any behavior that existed before and is explicitly removed or replaced by the diff.

---

## Phase 4: Identify cross-cutting signals

After per-area extraction, scan the full diff for:

- **Authorization changes**: lines that add, remove, or modify permission checks, guards, or middleware
- **Schema changes**: new, removed, or renamed fields in any response, model, or serializer
- **Error handling changes**: new error codes, changed error conditions, removed validations
- **Dependency changes**: new external calls, changed timeouts, removed error boundaries

If any are found, add a "Cross-cutting Changes" area. If none, skip it.

---

## Phase 5: Write the PRD

Write `plans/{pr-id}-prd.md`. Create `plans/` if it does not exist.

After writing, edit the document for concision as a final pass (an optional writing-cleanup skill such as `/unslop` works if you have one). A human reads this to decide what to investigate, so padding and hedging cost them time.

```markdown
# {PR title or branch name}: Behavioral Description

> This document describes what the code observably does, derived from the diff.
> It does not reflect what the PR description claims. Where the two differ, trust this document.

**Source:** {PR URL | `{branch}` vs `{base}`}
**Generated:** {ISO-8601}

---

## {Area 1 Name}

- {Behavioral claim} → `path/to/file.ext:NN`
- {Behavioral claim} → `path/to/file.ext:NN`, `path/to/other.ext:NN`
- {Behavioral claim} [uncertain: inferred from pattern, not stated explicitly]

---

## {Area 2 Name}

...

---

## Cross-cutting Changes

...

---

## Removed Behaviors

- {Description of what was removed} → `path/to/file.ext:NN`
```

Omit "Removed Behaviors" and "Cross-cutting Changes" sections if empty.

Confirm the written path to the user.

Write `plans/{pr-id}.pipeline.json`:

```json
{
  "pr": "{pr-id}",
  "source": {
    "type": "pr|branch",
    "value": "{URL or branch name}",
    "base": "{base ref or commit}",
    "remote": "{origin/staging or origin/main, omit for PR type}"
  },
  "pipeline": {
    "audit-prd": { "status": "completed", "output": "plans/{pr-id}-prd.md", "timestamp": "{ISO-8601}" }
  }
}
```

Print pipeline state:
`Pipeline: prd ✓ → gate ○ → grill ○ → tests ○ → validate ○`

Tell the user:
> "PRD written. Read through it and note anything that contradicts your mental model, looks unintentional, or is surprising. A short phrase per item is enough. No need for full sentences. When you have your list, share it and run `/audit-grill`."
