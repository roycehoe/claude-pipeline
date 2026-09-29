# portfolio-polish: Test Plan

## Scope

Covers the two confirmed discrepancies (D-001 routing claim, D-002 failed send), the two intentional behaviours worth guarding (D-004 suspended counter, D-005 narrow lint), the unresolved number-format question (D-003, one side kept), and one happy path for the nightly job, the only behavioral area that shares code with a confirmed defect. Not covered: check-in behaviour, sad-mood alerts and CI triggering, which have no confirmed discrepancy. The sign-in path and the live message gateway are outside the diff.

**Source:** Regression and edge cases derive from `plans/portfolio-polish-discrepancies.md`. The happy-path case derives from `plans/portfolio-polish-prd.md`.
**Test data:** Each case is independent. Arrange preconditions fresh.
**Unresolved pairs:** D-003 produced a pair, TC-005 and TC-006. TC-005 has been deleted; see the resolution log at the end.

---

## Summary

| ID | Title | Tier | Priority |
|---|---|---|---|
| TC-001 | Each recipient's alert goes to that recipient's own contact | Regression | High |
| TC-002 | A failed send does not stop the nightly job for other recipients | Regression | High |
| TC-003 | Suspended recipients are silent but their missed-day count still grows | Edge | Medium |
| TC-004 | The lint step fails on an undefined name | Edge | Medium |
| TC-006 | Both alert paths send the destination number in one format | Edge | High |
| TC-007 | Nightly job over a mixed set of recipients | Happy Path | Medium |

TC-005 was deleted at the resolution gate (see the log at the end).

---

## Tier 1: Regression Cases

#### TC-001: Each recipient's alert goes to that recipient's own contact
`Priority: High`  `Tier: Regression`

> Covers D-001: README overstates what the alert-routing test proves

**Given** two contacts, A and B, each linked to one recipient, and neither recipient has checked in today
**When** the nightly job runs
**Then** two `non_compliant_user` messages are sent; the one naming A's recipient goes to A's number and the one naming B's recipient goes to B's number
**Post** both recipients have a missed-day count of 1

#### TC-002: A failed send does not stop the nightly job for other recipients
`Priority: High`  `Tier: Regression`

> Covers D-002: A failed send aborts the whole nightly job

**Given** three recipients who have not checked in, and a send function that raises `WhatsappMessageException` for the first message only
**When** the nightly job runs
**Then** the job completes without raising; messages for the second and third recipients are attempted; the failure is logged
**Post** all three recipients have a missed-day count of 1 and check-ins are re-opened

---

## Tier 2: Edge Cases

#### TC-003: Suspended recipients are silent but their missed-day count still grows
`Priority: Medium`  `Tier: Edge`

> Covers D-004: Suspended recipients keep accruing missed days. Guards a deliberate choice.

**Given** a suspended recipient with a missed-day count of 3 who has not checked in
**When** the nightly job runs
**Then** no message is sent, the recipient is still suspended, and the missed-day count is 4

#### TC-004: The lint step fails on an undefined name
`Priority: Medium`  `Tier: Edge`

> Covers D-005: CI lint checks far less than "ruff" suggests. Guards the documented rule set.

**Given** a backend module containing a reference to a name that is never defined
**When** `poetry run ruff check .` runs in `backend/`
**Then** it exits non-zero and reports rule `F821`

#### TC-006: Both alert paths send the destination number in one format
`Priority: High`  `Tier: Edge`

> Covers D-003: number format differs between alert paths. Resolved as a defect at the resolution gate; the "unresolved" marker was removed so audit-validate can run.

**Given** a sad-mood alert and a missed-check-in alert for the same contact
**When** both are sent
**Then** both destinations are identical strings

---

## Tier 3: Happy Path

#### TC-007: Nightly job over a mixed set of recipients
`Priority: Medium`  `Tier: Happy Path`

**Given** one recipient who checked in, one who missed a first day, and one who missed a second day
**When** the nightly job runs
**Then** exactly two `non_compliant_user` messages are sent, no `suspend_user` message is sent, and the missed-day counts are 0, 1 and 2 respectively

---

## Resolution log

D-003 was resolved by the reviewer before validation: TC-005 deleted, TC-006 kept. Reason: static analysis cannot show that the provider accepts both formats, so one format is the safer side.
