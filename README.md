# agent-pipeline

Multi-agent plan, audit, execute and validate pipelines for coding agents. They run on [Claude Code](https://claude.com/claude-code), written as skills.

Two pipelines share one idea: put human judgment where mistakes are cheap to fix, and let automation do the rest once intent is locked.

- **feat-pipeline** takes a feature description to committed code. Human input is heaviest at the start (interrogation, plan) and lightest at the end (execution).
- **audit-pipeline** takes a large, unfamiliar diff and reconstructs what it actually does, then hunts for gaps between that and your mental model. Automation runs first, human judgment enters once there is a concrete document to react to.

Author: Royce Hoe, https://fancybinary.sg/, https://github.com/roycehoe

## Stage diagrams

Feature pipeline (`/feat-pipeline {description}`):

```
grill-with-docs -> feat-plan -> feat-execution -> feat-audit -> feat-execute -> feat-validate -> type-check -> simplify
   (optional)                                       gate           GO gate         gate
                                                     |
                                    FAIL: one auto-retry via feat-execution --gap-report
```

Audit pipeline (`/audit-pipeline {PR URL | branch}`):

```
audit-prd -> [HUMAN GATE] -> audit-grill -> audit-tests -> [RESOLUTION GATE] -> audit-validate -> simplify
 cited PRD    your list of    verdicts:      tiered test     resolve any         APPROVE /
 file:line    surprises       confirmed /    plan            two-sided pairs     CAVEATS /
                              misread /                                          REQUEST CHANGES
                              intentional /
                              unresolved
```

Lean lane (`/feat-quick {description}`): grill, one approval gate showing the exact code changes, then atomic commits. No audit, no validate. It hands off to `feat-pipeline` if the feature turns out to span many layers, add a state machine or need a migration.

## Which one to use

| Tool | Use when |
|---|---|
| `/feat-pipeline` | Building a new feature from a description to committed code, especially across several layers |
| `/feat-quick` | A small, well-scoped, single-layer feature, or a behaviour-preserving refactor with tests |
| `/audit-pipeline` | Deep investigation of a large, unfamiliar PR or branch with no spec or ticket context |
| `/feat-status` | See which pipeline steps are done and which to run next |
| `/feat-tests` | By hand only, when a test harness exists to run the generated cases against. Not part of `feat-pipeline` |

A code review tells you whether the code is good against what it claims to do. `audit-pipeline` first reconstructs what the code does, then compares. Use a review when you trust the intent and want a second opinion on execution. Use the audit when the change is big, you have no context, and you want to understand it before judging it.

## Skills

| Skill | Role |
|---|---|
| `feat-plan` | High-level plan grounded in the codebase, with adversarial self-review |
| `feat-execution` | Turns the plan into ordered atomic commits, verifying every named symbol exists |
| `feat-audit` | Gate: cross-checks the execution plan against the plan (coverage, fidelity, constants, guards) |
| `feat-execute` | Implements one commit at a time, then a fresh-context convention review of the branch |
| `feat-validate` | Gate: checks the committed diff covers every plan requirement |
| `feat-pipeline` | Orchestrates the stages above with resume support |
| `feat-quick` | One-shot lane for small features |
| `feat-status` | Status board from `plans/*.pipeline.json` |
| `feat-tests` | Optional manual-QA test plan |
| `grill-with-docs` | Pre-planning interrogation that locks terminology and decisions into a brief |
| `simplify` | Structure and complexity pass over changed code |
| `audit-prd` | Cited behavioural description of a diff, every claim anchored to `file:line` |
| `audit-grill` | Investigates each discrepancy and assigns a verdict, without asking you questions |
| `audit-tests` | Tiered test plan: regression, edge, happy path |
| `audit-validate` | Static check of each test case against the code, with a verdict |
| `audit-pipeline` | Orchestrates the audit stages with the two human gates |

## Install

Copy the `skills/` directories into a skills folder Claude Code reads.

```sh
# for every project
cp -R skills/* ~/.claude/skills/

# or for one project
mkdir -p .claude/skills && cp -R skills/* .claude/skills/
```

Restart Claude Code, then invoke a skill by name, for example `/feat-pipeline add a CSV export to the reports page`.

The pipelines call each other (`feat-pipeline` invokes `grill-with-docs`, `feat-plan`, `feat-execution`, `feat-audit`, `feat-execute`, `feat-validate` and `simplify`), so install the whole set.

## Configuration

Nothing needs configuring to start. These are the places the skills read from your repo, so you can point them at your own files.

| What | Default | Notes |
|---|---|---|
| Convention files | `CLAUDE.md`, and `CONVENTIONS.md` beside it | The skills walk up from each touched file to the project root and treat the nearest file as authoritative. A file that only delegates upward carries no rules and the walk continues. Rename by editing the paragraphs that read `CLAUDE.md` in `feat-plan`, `feat-execution`, `feat-execute` and `feat-quick`. |
| Rulings file | `CONVENTIONS-RULINGS.md`, optional | If present it outranks the other convention files in the branch-wide review in `feat-execute`. |
| Layer order | Enums, models, migration, exceptions, gateway, utils, data access, schemas, validators, services, routers, entry point | Used by `feat-execution` to order commits. Define your own order in your convention file and it takes precedence. |
| Lint command | `ruff check .` | Used by `feat-execute` after each commit. Change the command in `skills/feat-execute/SKILL.md` and its `allowed-tools` line for another linter. |
| Type-check gate | LSP diagnostics, else your project's checker | Stage 7.5 of `feat-pipeline`. Runs in the main agent. |
| Migrations | Never run by the pipeline | A human runs the migration command after review. |
| Plans directory | `plans/` | All stage artifacts and `*.pipeline.json` state live here. Add `plans/` to your project's `.gitignore`. |
| Code review and security review | Not included | `audit-pipeline` option C and `feat-quick`'s risk tripwire refer to a review skill. Point them at your own. |
| Writing cleanup | Not included | `audit-prd` and `grill-with-docs` mention an optional concision pass. Any writing-cleanup skill works. |

Requirements: Claude Code, `git`, and `gh` for auditing a GitHub PR by URL.

## Worked example

[`examples/heartbeat-portfolio-polish/`](examples/heartbeat-portfolio-polish/) is a run of the audit path against a local, unpushed branch of an open-source app, so its citations point at a snapshot. It holds the cited PRD, the reviewer's gate notes, verdicted discrepancies, the tiered test plan and the validate report.

## Design notes

- Every PRD claim carries a `file:line` citation, and claims that cannot be cited are flagged `[uncertain]` rather than dropped. The reviewer scans citations instead of re-reading the diff.
- The human gate sits before grilling. Grilling a whole PRD is expensive and unfocused, grilling a short list of surprises is cheap and sharp.
- `feat-audit` cross-checks two plan documents. `feat-validate` checks committed code. The first cannot see code, so both exist.
- Spawned workers get an explicit model tier so fan-out does not silently inherit the most expensive one.

## License

MIT, see [LICENSE](LICENSE).
