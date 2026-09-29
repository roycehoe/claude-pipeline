# portfolio-polish: Behavioral Description

> This document describes what the code observably does, derived from the diff.
> It does not reflect what the PR description claims. Where the two differ, trust this document.

**Source:** `portfolio-polish` vs `master` (merge-base `24787a7`, head `34ac555`), 8 files, 3 commits. Line numbers are in the branch head.
**Generated:** 2026-09-29T14:00:00+08:00

The diff adds no application code. Everything below is a test, a CI job, project config or documentation. "Behavior" here means what the tests assert about the existing application and what the README and CI now promise.

---

## Test harness

- Tests run against in-memory SQLite, not the production database → `backend/tests/conftest.py:41-45`
- Only three tables are created (users, recipients, moods), because the other models use Postgres-only array columns → `backend/tests/conftest.py:46-55`
- Application settings are read from environment variables at import time, so the fixtures set dummy values before any application import → `backend/tests/conftest.py:9-22`
- No test can send a real message: the send function is replaced by a list recorder in both modules that import it → `backend/tests/conftest.py:63-68`
- Pytest resolves imports from `backend/` and collects only `backend/tests/` → `backend/pyproject.toml:37-39`
- Ruff is limited to syntax errors and undefined names; the config comment says stricter rules are off because existing modules carry unused imports → `backend/pyproject.toml:41-44`

## Check-in behaviour (as asserted)

- A first check-in stores the mood, raises the check-in streak by one, zeroes the missed-day count, blocks further check-ins that day and sends no message → `backend/tests/test_checkin.py:18-30`
- A second check-in on the same day is rejected with HTTP 429 → `backend/tests/test_checkin.py:33-42`
- A check-in lifts a suspension and zeroes the missed-day count → `backend/tests/test_checkin.py:45-52`

## Sad-mood alert (as asserted)

- One sad mood does not trigger an alert; two in a row do, and the threshold constant is 2 → `backend/tests/test_alerts.py:36-45`
- A non-sad mood between two sad ones prevents the alert → `backend/tests/test_alerts.py:48-54`
- The second sad check-in sends one message from the `consecutive_sad_moods` template; the destination has a leading plus sign → `backend/tests/test_alerts.py:57-68`
- The first sad check-in sends nothing → `backend/tests/test_alerts.py:71-78`

## Nightly missed-check-in job (as asserted)

- Suspension fires on every third missed day (a modulo test, so also the sixth) → `backend/tests/test_alerts.py:84-95`
- A recipient who did not check in gets one `non_compliant_user` message, the streak resets to 0, the missed-day count becomes 1, and check-ins re-open → `backend/tests/test_alerts.py:98-110`
- A recipient who did check in is left alone, and check-ins re-open for the next day → `backend/tests/test_alerts.py:113-122`
- On the third missed day the recipient is suspended and two messages go out: `non_compliant_user`, then `suspend_user` → `backend/tests/test_alerts.py:125-135`
- A suspended recipient triggers no message and stays suspended → `backend/tests/test_alerts.py:138-146`
- With two recipients under different contacts, one message is sent and its destination has no plus sign → `backend/tests/test_alerts.py:149-160`

## CI

- Lint and test run on push to `master` and on every pull request, not on pushes to other branches → `.github/workflows/ci.yml:3-6`
- One backend job on Python 3.11 installs the Postgres client headers, then dependencies with Poetry, then runs `ruff check .` and `pytest` → `.github/workflows/ci.yml:17-33`
- The frontend is not built, linted or tested in CI [uncertain: inferred from the absence of a frontend job, not stated in the diff]

## Documentation claims

- The README states the nightly job runs at midnight Singapore time and messages the contact for every recipient who did not check in → `README.md:27`
- It states suspension happens after every third missed day and that the contact is told once → `README.md:29`
- It states the tests cover routing of each recipient's alerts to that recipient's own contact → `README.md:95`
- It states CI lint checks syntax errors and undefined names only → `README.md:104`
- It states the check-in and alert flows are implemented and covered by tests, and lists the sign-in path, the live message gateway and the frontend as not covered → `README.md:108`

---

## Cross-cutting Changes

- Dependency: `pytest` added as a dev dependency and the lock file regenerated with a newer Poetry header → `backend/pyproject.toml:31`, `backend/poetry.lock:1`
- No authorization, schema or error-handling code changed.
