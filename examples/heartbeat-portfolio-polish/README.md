# Worked example: auditing a branch with the audit pipeline

This is a real run of the audit path (`audit-prd`, human gate, `audit-grill`, `audit-tests`, resolution gate, `audit-validate`) against a small branch of [Heartbeat](https://github.com/roycehoe/heartbeat), an open-source daily check-in app. The branch, `portfolio-polish`, adds tests, a CI workflow and a README to a FastAPI backend. It changes no application code.

The stages were run by hand, following each skill's instructions and output format, so each file below is what the skill writes to `plans/`. No personal data, credentials or environment values are quoted.

| Stage | File |
|---|---|
| audit-prd | [portfolio-polish-prd.md](plans/portfolio-polish-prd.md) |
| human gate | [portfolio-polish-discrepancies-raw.md](plans/portfolio-polish-discrepancies-raw.md) |
| audit-grill | [portfolio-polish-discrepancies.md](plans/portfolio-polish-discrepancies.md) |
| audit-tests (with resolution gate) | [portfolio-polish-tests.md](plans/portfolio-polish-tests.md) |
| audit-validate | [portfolio-polish-validate-20260929-1500.md](plans/portfolio-polish-validate-20260929-1500.md) |

What the run found, in one paragraph: the README claimed a test proved per-recipient alert routing, but the test only produced one message (confirmed); a failed message send aborts the whole nightly job and no test reaches that path (confirmed, and the validate stage returns REQUEST CHANGES on it); the two alert paths format the destination number differently (unresolved, resolved by the reviewer); and three items were intentional or misread.

Things to notice: every PRD claim carries a `file:line`; the reviewer's gate notes are informal phrases, and grill turns them into verdicts with evidence; audit-tests only writes happy-path cases for areas that share code with a confirmed defect; and validate reads the code at branch head because this branch has no application diff.
