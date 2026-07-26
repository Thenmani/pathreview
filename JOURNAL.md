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

**Reproduction steps:**
1. Located the relevant file: `safety/pii_scrubber.py`, and the test file: `tests/unit/test_pii_scrubber.py`.
2. Added a new test, `test_paren_phone_reproduces_bug`, that calls `detect()` and `scrub()` on the input `"Call me at (324) 901-1234"`.
3. Ran the test:
4. Test failed, confirming the bug:



