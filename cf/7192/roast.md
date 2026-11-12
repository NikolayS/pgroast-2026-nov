# Design pre-review: GiST multirange consistent fix

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7192](https://commitfest.postgresql.org/patch/7192/); [thread](https://www.postgresql.org/message-id/flat/CAH2-Wzm1GpQ9qixP%3DXs2TYdfnLM88OZ%3DyUFHNuajS_W7U90-tQ%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible case in message #1 (501 rows from the sequential scan, 500 from the index scan), and the fix changes one scan-time test in an existing opclass with no new user-visible surface. Confidence is medium because only 3 of 8 messages and no fix code were visible.

Not checked: The fix code and messages #4 to #8 were not visible, so the correctness of the v4 predicate for empty and unbounded multiranges and on internal pages could not be checked. The claim that stored keys do not change also rests on the description only.

## Prior attempts

None found before 2026-08-01.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Is the fix meant for back-patching to 14 and later, and does the same patch apply to each branch?
2. Can you confirm that the stored keys do not change, so existing multirange GiST indexes need no REINDEX?
3. What argument or test shows that the v4 predicate gives no false negatives for empty multiranges and for unbounded ones?
4. How many more heap rechecks does the v4 test cause compared with the v2 approach on the regression data?
5. Do internal pages use the same contained-by test, and is it correct there as well as on leaf pages?

## Touch points

- **backpatchability:** Message #1 says the bug affects 14 and up, so the fix is a back-patch candidate for all supported branches.
- **performance on the hot path:** The new contained-by test in multirange_gist_consistent() runs on every such GiST scan and may send more tuples to heap recheck.

## Who would commit it, and when

- Owner: qualified committer engaged. Per the brief, Alexander Korotkov wrote v3 and v4 and Peter Geoghegan reported the bug; the owner needs to know GiST opclass consistent functions and the range and multirange code.
- Priority: proceed; audience niche; review cost low. The bug gives silent wrong results for contained-by scans on multirange GiST indexes on 14 and later, and the fix is limited to one consistent function. Few users index multiranges with GiST, but wrong results justify attention now.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
