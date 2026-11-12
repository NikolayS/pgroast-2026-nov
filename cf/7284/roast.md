# Design pre-review: {heap,minimal}_expand_tuple() are unused

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7284](https://commitfest.postgresql.org/patch/7284/); [thread](https://www.postgresql.org/message-id/flat/gf6hokynqjp6g7o4zh6yy7ekbdbig5cnnd6ilk6iw3aodilj43%405x7xrz52jhve), 6 of 6 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at any level. The patch deletes four functions with no callers in core, and two committers who know heaptuple.c proposed and backed the removal in messages #1 to #3. The only cost is that an out-of-tree caller fails to build against the new major version, which is how Postgres normally removes C API. Confidence is medium because only three of six messages are visible and the code was not read.

Not checked: The code was not read, so this review did not confirm that the four functions have no remaining callers or that the reworded fast_default comments are accurate. Messages #4 to #6 were not shown.

## Prior attempts

None found before 2026-08-07.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which external code was searched for callers of the three exported functions, for example which extension repositories or code search service?
2. Will the release notes for the new major version list the three removed functions for extension authors?

## Touch points

- **ABI / extension API:** The patch removes heap_expand_tuple(), minimal_expand_tuple() and heap_copytuple_with_tuple() from htup_details.h.
- **maintenance burden ("who maintains this in 10 years"):** The patch lowers the burden by deleting nontrivial tuple-expansion code from heaptuple.c that nothing in core calls.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund proposed the removal in message #1 and Tom Lane backed it in messages #2 and #3; both work on the heap tuple code named in the thread.
- Priority: proceed; audience niche; review cost low. The patch only deletes uncalled functions in two files and rewords two test comments, so review is cheap and the result is less code to maintain. Two committers agree, a reviewer found no issues, and nobody has objected.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
