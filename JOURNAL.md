## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/146

**Issue title:** PII scrubber fails to redact parenthesized US phone numbers

**Tier:** Tier 1

**Problem summary:**
The phone-number pattern used in pii_scrubber.py only matches dashed formats like 324-901-1234. It does not match the parenthesized format (324) 901-1234, which is one of the most common ways US phone numbers are written.
As a result,
scrub() leaves numbers in this format completely unredacted in the output.
detect() incorrectly reports that no PII is present when the text contains a parenthesized phone number.

Expected behavior:
Both detect() and scrub() should recognize (324) 901-1234 as a phone number, alongside already-supported formats like 324-901-1234.

**Issue Selection Criteria**
- I understood the issue and the expected behavior clearly.
- I've located the relevant files and confirmed they exist in the codebase
- I've understood the surrounding code well
- I can describe clearly what the user sees before the fix and what they see after.
- I've selected Tier 1, as the scope is realistic for this first open source contribution. 
- It is self contained and there are no blockers and dependencies.
- I've checked the issue comments and the ledger's Claims count, and I'm fine with how many others are on this issue.
- I've found the test file and read the related unit test cases.

**Branch name:** fix/146-PII-US-phone-no-format-parenthesis

**Setup confirmation:** App runs locally at localhost:5173

**Cohort ledger:** Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** 
- https://github.com/Thenmani/pathreview/commit/8c1280f, 
- https://github.com/Thenmani/pathreview/commit/6204c9f

**Reproduction summary:**
Wrote few failing test cases to confirm that both `detect()`
and `scrub()` fail to recognize parenthesized US phone numbers like `(324) 901-1234`.
`detect()` returns an empty list and `scrub()` leaves the number unredacted, confirming
the root cause is the `phone_us` regex in `safety/pii_scrubber.py`.

**Reproduction steps:**
1. Located the relevant file: `safety/pii_scrubber.py`, and the test file: `tests/unit/test_pii_scrubber.py`.
2. Added test cases, `test_paren_phone_reproduces_bug`, `test_paren_phone_detect_reproduces_bug`, `test_paren_phone_scrub_reproduces_bug` that calls `detect()` and `scrub()` on the input `"Call me at (324) 901-1234"`.
3. Ran the test.
4. Test failed, confirming the bug.

**Additional test cases written, to reproduce the bugs:**
Beyond the single required reproduction test, I wrote several more to explore
the shape of the bug more thoroughly. All of these currently fail, since the fix hasn't been applied yet.
- `test_phone_paren_no_space_after_close` — `(555)123-4567`
- `test_phone_paren_dash_after_close` — `(555)-123-4567`
- `test_phone_paren_dot_after_close` — `(555).123.4567`
- `test_detect_paren_phone_no_space` — confirms `detect()` also misses the no-space case
- `test_paren_phone_only_pii_detected` — confirms the false-negative when the
  parenthesized number is the only PII in the text
- `test_paren_phone_no_false_positive_on_short_numbers` — guards against short
  parenthesized numbers (e.g. footnote references like `(12)`) being misdetected
  as phone numbers

**PLAN.md link:** https://github.com/Thenmani/pathreview/blob/fix/146-PII-US-phone-no-format-parenthesis/PLAN.md

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
- Identified the root cause: the `phone_us` separator class `[-.]?` did not
  include whitespace, so `(555) 123-4567` failed to match at the space after
  the closing parenthesis.
- Came up with an initial working fix — extended all three separator slots from
  `[-.]?` to `[-.\s]?` and replaced the leading `\b` with `(?<!\w)` for
  reliable boundary matching next to parentheses. Verified all reproduction
  tests passed.
- Optimized the fix further by removing unused capture groups `([0-9]{3})` →
  `[0-9]{3}` since `scrub()` uses a fixed replacement string and `detect()`
  reads the whole match — no backreferences needed.
- All reproduction tests now pass (`test_paren_phone_detect_reproduces_bug`,
  `test_paren_phone_scrub_reproduces_bug`).
- Added 3 additional edge-case tests: country code + parens combined,
  multiple phone numbers in one string, and a negative case for malformed
  2-digit area codes.
- Full suite: 37 passed, 1 failed — the one failure is a **pre-existing,
  unrelated bug** in the `street_address` pattern (`test_mixed_pii_and_text`):
  the pattern incorrectly matches the substring "Pl" inside the word
  "applications", partially redacting it as "[REDACTED]ications". This is out
  of scope for issue #146 and was present before this fix was applied.

**Fix implementation note:**
While committing `safety/pii_scrubber.py`, the pre-commit hooks (ruff) blocked
the commit due to 2 pre-existing errors in untouched lines:
- `B007` — unused loop variable `pii_type` in both `scrub()` and `detect()`
- `E501` — `street_address` pattern line too long (283 chars)

Fixed both as part of the commit:
- Renamed `pii_type` to `_pii_type` in both loops (underscore prefix signals
  intentionally unused to ruff).
- Split the `street_address` pattern across multiple lines using implicit
  string concatenation.

**Lesson learned:** During the rename, `detect()` accidentally referenced
the old `pii_type` name in the loop body while the loop variable was already
renamed to `_pii_type` — caused 11 test failures. Caught immediately by
running the full test suite. Also missed the same rename in `scrub()` on
the first attempt — ruff caught it on the next commit try. Run tests after
every change, even one-character renames.

**Self-review against contribution standards:**
Ran `make check` and `make test-unit` before opening the draft PR.

- `ruff check safety/pii_scrubber.py tests/unit/test_pii_scrubber.py` returned
  6 errors initially. Two were in my files (trailing whitespace in
  `test_pii_scrubber.py`) — fixed using `ruff check --fix`. The remaining 4
  errors are pre-existing in `safety/pii_scrubber.py` (unsorted imports,
  street_address pattern too long, unused loop variable, logger line too long)
  — none introduced by my changes.
- `make test-unit` result: 391 passed, 49 failed. All 49 failures are
  pre-existing across unrelated modules. Zero new failures introduced.
- In `tests/unit/test_pii_scrubber.py` specifically: 37 passed, 1 failed.
  The single failure (`test_mixed_pii_and_text`) is the pre-existing
  `street_address` false-positive bug — not related to this fix.

**Result:** My changes introduce no new failures in either `make check`
or `make test-unit`.

**Next steps:**
- Add remaining edge-case tests before final PR merge.
- Address any draft PR review feedback.
- Record Loom walkthrough video.
- Update `JOURNAL.md` with final Week 9 summary once all steps are complete.

**Blockers or open questions:**
None currently.
