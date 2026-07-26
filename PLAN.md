## Solution plan

**Issue:** #146 — PII scrubber fails to redact parenthesized US phone numbers

### Understand
The `phone_us` regex pattern in `safety/pii_scrubber.py` (line 15) is used by both
`scrub()` and `detect()` to find US phone numbers. It correctly matches dashed and
dotted formats like `555-123-4567` or `555.123.4567`, but fails on the parenthesized
format `(555) 123-4567`.

The root cause is in the separator character class. The pattern uses `[-.]?` immediately
after the optional closing parenthesis `\)?` — this class only allows a dash or a dot,
not whitespace. Since `(555) 123-4567` has a space after the `)`, the regex cannot
continue matching past that point, and the entire match fails.

As a result:
- `scrub()` leaves the number completely unredacted in the output.
- `detect()` returns an empty list when a parenthesized number is the only PII present
  — a false negative, which is the more serious half of the bug, since it means the
  scrubber reports "no PII found" when PII is actually present.

I reproduced this already with a failing test (`test_paren_phone_reproduces_bug` in
`tests/unit/test_pii_scrubber.py`), confirming both `detect()` returns `[]` and
`scrub()` leaves `(324) 901-1234` untouched.

**Root cause:** The `phone_us` regex's separator class `[-.]?` does not include `\s`,
so it can't match a space after a parenthesized area code.

### Map
Files I expect to touch:
- `safety/pii_scrubber.py` — the `phone_us` pattern in `PII_PATTERNS` (line 15). This is
  the only production code change needed.
- `tests/unit/test_pii_scrubber.py` — already contains my reproduction test and several
  new edge-case tests (no space after paren, dash/dot after paren, false-positive guard
  for short parenthesized numbers). I'll un-skip/confirm these pass once the fix lands.
- No other files should need changes — `PIIScrubber` is used elsewhere in the app, but
  I'm not changing its interface or the behavior of any other PII type, so callers are
  unaffected.

### Plan
1. Confirm the exact current regex and reproduce the bug (done — see committed failing
   test and `JOURNAL.md` reproduction notes).
2. Update the `phone_us` pattern to accept whitespace as a valid separator after the
   parenthesized area code, and tighten the boundary checks so `\(` doesn't interfere
   with `\b`:
   ```python
   r"(?<!\w)(?:\+?1[-.]?)?\(?([0-9]{3})\)?[-.\s]?([0-9]{3})[-.]?([0-9]{4})(?!\w)"
   ```
3. Run the existing reproduction test and confirm it now passes.
4. Run the full test file to confirm no regressions in email, SSN, or address detection.
5. Write additional unit tests covering edge cases beyond the originally reported format
   (e.g., no space after the closing paren, dash/dot after the paren, and a check that
   short parenthesized numbers like footnote references aren't misdetected as phone
   numbers). This is next week's main task, alongside the fix itself.
6. Run `make check` (or equivalent lint/format/type command) to confirm ruff, black, and
   mypy are clean before committing.
7. Update `JOURNAL.md` with the fix summary and before/after example.
8. Commit and push following the branch/commit conventions in `docs/CONTRIBUTING.md`.

### Inputs & outputs
**Pattern I'm changing:** `PII_PATTERNS["phone_us"]` in `safety/pii_scrubber.py`

**Existing happy path (unaffected):**
- Input: `"Call me at 555-123-4567"` → `scrub()` returns `"Call me at [REDACTED]"`

**Currently broken (what I'm fixing):**
- Input: `"Call me at (324) 901-1234"`
- Current: `scrub()` returns text unchanged; `detect()` returns `[]`
- Expected after fix: `scrub()` returns `"Call me at [REDACTED]"`;
  `detect()` returns one entry with `"type": "phone_us"`

### Risks & unknowns
1. **Overly permissive whitespace matching.** Allowing `\s` as a separator could
   theoretically make the pattern greedier than intended (e.g., matching across
   unrelated numbers separated by newlines, since `\s` includes `\n`). I'll check
   whether this causes any false positives in the mixed-content test and consider
   using `[ ]` (literal space) instead of `\s` if `\n` matching causes problems.
2. **Interaction with the boundary anchor change.** Switching `\b` to `(?<!\w)` /
   `(?!\w)` changes how the pattern behaves at the very start/end of a string, or next
   to punctuation like a trailing period. I need to re-run `test_phone_at_start_of_text`
   and `test_phone_at_end_of_text` specifically to confirm no regression.
3. **Unrelated pre-existing bug noticed during reproduction.** While running the full
   suite, I noticed `test_mixed_pii_and_text` fails independently — the `street_address`
   pattern partially matches inside the word "applications" (matching "Pl" as if it were
   "Place"). This is out of scope for #146 and I will not fix it as part of this issue,
   but I'll note it in `JOURNAL.md` in case it should become a separate issue.

### Edge cases
- `(555) 123-4567` (space after paren) — the originally reported case, must pass
- `(555)123-4567` (no space after paren) — must also pass
- `(555)-123-4567` / `(555).123.4567` (dash/dot after paren) — must also pass
- `555-123-4567`, `555.123.4567`, `+1-555-123-4567` — existing formats, must not regress
- `(12)` inside unrelated text (e.g., a footnote reference) — must NOT be misdetected as
  a phone number
- Parenthesized number at the very start or end of a string — must still match correctly
  with the updated boundary anchors
