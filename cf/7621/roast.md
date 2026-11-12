# Design pre-review: Fix jsonpath printing of operations followed by an accessor

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7621](https://commitfest.postgresql.org/patch/7621/); [thread](https://www.postgresql.org/message-id/flat/CA%2BY2BmewUC0FwoZ_ktQFc%3D%3DG5OBXVqmQjG3T6x%2B%3DYVfCNO%3DHrA%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The first message states the bug with reproducible examples: a printed jsonpath that returns a different result or does not parse, which reaches users through pg_dump and pg_get_viewdef(). The fix extends the rule printJsonPathItem() already uses for numeric literals. Confidence is medium because only one message is visible and the code was not read.

Not checked: The code was not read, so the claim that output changes only for paths that print wrong today is unchecked. Whether the new rule covers every operation and predicate kind is also unchecked.

## Prior attempts

None found before 2026-11-10.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does every printed form that the patch changes still parse, with the same meaning, on each supported older release that might load a newer dump?
2. What should users do with dumps already taken that hold the broken printed forms, and does that belong in the release notes?
3. If the fix is back-patched, can any index or stored value built on jsonpath text output differ after the minor upgrade?
4. Which parts of the jsonpath grammar does the fuzzer generate, and can you post it so reviewers can repeat the runs?
5. Does a jsonpath operator or predicate added later get parentheses under the new rule without a further change to printJsonPathItem()?

## Touch points

- **pg_dump / restore round-trip:** pg_dump and pg_get_viewdef() emit jsonpath output text, which the patch changes for paths that print wrong today.
- **backpatchability:** The bug dates from v12, and the author offers versions for REL_14 to REL_16 if a back-patch is wanted.

## Who would commit it, and when

- Owner: silent. Someone who knows the jsonpath parser and output code in jsonpath.c, and who can judge a back-patch that changes output text.
- Priority: proceed; audience niche; review cost low. The bug gives wrong results and failed restores, but only for paths with a parenthesized operation followed by an accessor. The patch is 8.8 KiB, touches jsonpath.c and its regression test, and adds no new user-visible surface.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
