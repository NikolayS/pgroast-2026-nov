# Design pre-review: Add ASCII fast path to Unicode normalization functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7388](https://commitfest.postgresql.org/patch/7388/); [thread](https://www.postgresql.org/message-id/flat/8cb366d6-a30f-49a4-b5d3-a5dfc0f2fea2%40dunslane.net), 16 of 16 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem and the workload are stated (`url IS NORMALIZED` over ClickBench data), the change adds no user-visible surface, and message #3 measured no regression on non-ASCII input. The patch reuses the existing is_valid_ascii() and stays inside varlena.c. Confidence is medium because the code was not read and no benchmark of v5 as a whole is shown.

Not checked: Without the code, I could not check that restarting at the last ASCII character gives the same result as a full decode, or that form-name matching and its error behaviour match the old path. Builds without SIMD support and big-endian machines are untested in the material.

## Prior attempts

None found before 2026-09-14.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. What does v5 as a whole measure against master on all-ASCII, mixed, non-ASCII and late-non-ASCII input?
2. What do the non-ASCII and late-non-ASCII cases measure on a build where is_valid_ascii() has no SIMD support?
3. Does an invalid form name still raise the same error when the input string is all ASCII?
4. Does the form-name matching in the text value accept and reject exactly the same names as the old pg_strcasecmp() path, including non-ASCII bytes in the name?
5. Has the patch been run on a big-endian machine?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** varlena.c gains an ASCII-prefix path and hand-written form-name matching that must stay in agreement with the full decode path.
- **performance on the hot path:** Every call to normalize(), IS NORMALIZED and unicode_assigned() now runs a byte scan first, including on non-ASCII input.

## Who would commit it, and when

- Owner: qualified committer engaged. Andrew Dunstan wrote the patch and, per the thread state, plans to commit it during the November CommitFest.
- Priority: proceed; audience a sizeable minority; review cost low. Users who run normalize() or IS NORMALIZED over mostly-ASCII text gain about 10 times on pure ASCII per message #1, confirmed in message #3. The patch is 18.4 KB in one backend file plus regression tests, and adds no user-visible surface.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
