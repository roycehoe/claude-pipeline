# portfolio-polish: Audit Validate

**Verdict:** REQUEST CHANGES
**Run:** 2026-09-29T15:00:00+08:00
**4 covered · 2 gap · 0 unknown**

This branch adds only tests, CI and docs, so the check reads the application code at branch head against each case. "Covered" means the application code already behaves as the case expects. "Gap" means it does not.

---

## Coverage

| ID | Title | Tier | Priority | Verdict | Notes |
|---|---|---|---|---|---|
| TC-001 | Each recipient's alert goes to that recipient's own contact | Regression | High | covered | Contact looked up per recipient, `backend/scripts.py:97-99`. No test proves it yet (D-001). |
| TC-002 | A failed send does not stop the nightly job for other recipients | Regression | High | gap | No exception handling around the send, `backend/scripts.py:94-103` |
| TC-003 | Suspended recipients are silent but their missed-day count still grows | Edge | Medium | covered | `backend/scripts.py:37-41`, `:59-65`, `:95-96` |
| TC-004 | The lint step fails on an undefined name | Edge | Medium | covered | `F82` is selected, `backend/pyproject.toml:44`; step runs at `.github/workflows/ci.yml:29-30` |
| TC-006 | Both alert paths send the destination number in one format | Edge | High | gap | Plus prefix at `backend/services/user.py:248`, none at `backend/scripts.py:85` and `:99` |
| TC-007 | Nightly job over a mixed set of recipients | Happy Path | Medium | covered | `backend/scripts.py:106-116`; also exercised by `test_alerts.py:98-135` |

---

## Blockers *(Tier 1 gaps only)*

### TC-002: A failed send does not stop the nightly job for other recipients
**What is absent:** exception handling in `_notify_admin_of_non_compliant_users` (`backend/scripts.py:91-103`) and `_run_end_of_day_cron_job` (`backend/scripts.py:106-116`).
**The test expects:** the job completes, later recipients are still messaged, and every counter is updated.
**The diff provides:** nothing. `send_whatsapp_message` raises on any non-OK reply (`backend/gateway.py:26-27`), the loop does not catch it, and the counter updates at `backend/scripts.py:109-110` run after the loop. One failed send leaves every counter unchanged for that night.

The blocker is in application code the branch does not touch. Two ways to clear it: catch and log per recipient in the loop, or keep the code and change `README.md:108` so it no longer implies failure handling. The tests on this branch are not the problem.

---

## Observations *(Tier 2 and 3 gaps and unknowns)*

### TC-006: Both alert paths send the destination number in one format
**Note:** The two formats are pinned by `test_alerts.py:68` and `:160`. Whether the provider accepts both is unknown from the code, so this is not a blocker. Normalise in one helper if the provider is strict.
