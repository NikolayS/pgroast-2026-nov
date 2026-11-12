# Design pre-review: GIN jsonb_ops misses root strings in strict jsonpath queries

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7489](https://commitfest.postgresql.org/patch/7489/); [thread](https://www.postgresql.org/message-id/flat/CAHWVJhHH0k3PxWmS%3D%3DV1p_PONOMUaSm6LjUV487fwnOqd0jTQw%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at any level. The proposal fixes wrong query results: a jsonb_ops index scan misses a root string that a sequential scan returns, and two people hit it independently (messages #1 and #2). The fix changes query-side key extraction only, so existing indexes match without REINDEX. Message #3 reports that it was pushed.

Not checked: The code was not read, so the review could not check root non-string scalars in strict mode or jsonb_path_ops. Message #3 does not say what the editorialization changed.

## Prior attempts

None found before 2026-10-20.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which back branches received the fix, given that the CommitFest target is "stable"?
2. Do root numbers, booleans and null in strict mode have the same mismatch between gin_extract_jsonb() and jsonb_ops__extract_nodes()?
3. Does jsonb_path_ops return the matching root string for the same two strict-mode queries?
4. Should the CommitFest entry be marked committed, since message #3 says the fix was pushed?

## Touch points

- **backpatchability:** The CommitFest target is "stable", and the thread does not say which back branches received the fix.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane pushed the fix with minor editorialization (message #3).
- Priority: proceed; audience niche; review cost low. The patch is a one-line wrong-results fix in jsonb_gin.c with two regression queries, and it affects only strict-mode jsonpath queries on jsonb_ops indexes. Message #3 says it was pushed, so the remaining work is to update the CommitFest entry, which still says "Needs review".

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
