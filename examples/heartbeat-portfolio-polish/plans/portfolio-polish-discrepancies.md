# portfolio-polish: Discrepancies

**2 confirmed · 1 misread · 2 intentional · 1 unresolved**
**Investigated:** 2026-09-29T14:20:00+08:00

The investigation read the application code at branch head, because the diff adds only tests, CI and docs. Application paths cited here are unchanged from `master`.

---

## D-001: README overstates what the alert-routing test proves
**Verdict:** confirmed

**Discrepancy:** README says the tests cover routing of alerts to each recipient's own contact. The test seems to send one message.

**Evidence:**
> `README.md:95`, the test list ends with routing of each recipient's alerts to that recipient's own contact
> `backend/tests/test_alerts.py:156`, `make_recipient(name="Theirs", user_id=other_admin.id, can_record_mood=False)`
> `backend/tests/test_alerts.py:160`, `assert [message.to for message in sent_messages] == ["..."]`

**Finding:** The second recipient has already checked in (`can_record_mood=False`), so the nightly job never alerts for them. Only one message is produced, so the test cannot fail if alerts were sent to the wrong contact. The code routes per recipient (`backend/scripts.py:97`), but no test proves it.

---

## D-002: A failed send aborts the whole nightly job
**Verdict:** confirmed

**Discrepancy:** What happens to the nightly job if a message send fails?

**Evidence:**
> `backend/gateway.py:26-27`, `if not response.ok: raise WhatsappMessageException(response.json())`
> `backend/scripts.py:94-103`, the notify loop has no exception handling
> `backend/scripts.py:107-110`, notifications run before the counters are updated

**Finding:** One failed send raises out of the loop, so later recipients get no message and no counter is updated that night. The tests replace the send function (`backend/tests/conftest.py:66-67`), so this path is never exercised. README line 108 says the alert flows are covered by tests, which is true for success and silent about failure.

---

## D-003: The two alert paths format the destination number differently
**Verdict:** unresolved

**Discrepancy:** The two alert paths do not seem to format the phone number the same way.

**Evidence:**
> `backend/services/user.py:248`, `f"+65{admin.contact_number}"`
> `backend/scripts.py:85`, `f"65{admin.contact_number}"`
> `backend/tests/test_alerts.py:68` and `:160`, each test asserts its own format

**Finding:** The sad-mood path sends a plus-prefixed number and the nightly path does not. The gateway posts the value unchanged (`backend/gateway.py:20-27`), so whether both formats are accepted depends on the messaging provider. Static analysis cannot settle it. Both tests pin the current behaviour rather than question it.

**Doc reference:** none checked. The provider documentation for accepted number formats was not consulted in this offline example.

---

## D-004: Suspended recipients keep accruing missed days
**Verdict:** intentional

**Discrepancy:** Suspended users still get their counter bumped every night?

**Evidence:**
> `backend/scripts.py:37-41`, the non-compliant list filters on `can_record_mood` only, not `is_suspended`
> `backend/scripts.py:31`, the errant list does filter `is_suspended: False`
> `backend/services/user.py:206-210`, a check-in sets the missed-day count to 0

**Finding:** The count grows while suspended, but suspension is re-checked against the unsuspended list, so no second suspension message goes out, and the next check-in resets the count. The test at `test_alerts.py:138-146` asserts silence and stays suspended, and does not look at the counter.

---

## D-005: CI lint checks far less than "ruff" suggests
**Verdict:** intentional

**Discrepancy:** CI lint says ruff but the config is tiny.

**Evidence:**
> `backend/pyproject.toml:42-44`, "Stricter rules are not enabled yet because the existing modules carry unused imports." `select = ["E401", "E7", "E9", "F63", "F7", "F82"]`
> `README.md:104`, "The lint step checks for syntax errors and undefined names only."

**Finding:** The narrow rule set is a deliberate, documented choice in both the config and the README.

---

## D-006: CI on a plain branch push
**Verdict:** misread

**Discrepancy:** Will CI run when I push the branch?

**Evidence:**
> `.github/workflows/ci.yml:3-6`, `push: branches: [master]` and `pull_request:`

**Finding:** A push to `portfolio-polish` alone does not run CI. Opening a pull request does, through the `pull_request` trigger. README line 104 says exactly this.
