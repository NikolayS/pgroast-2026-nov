# Design pre-review: Avoid redundant BitmapAnd with a clauseless partial index

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7619](https://commitfest.postgresql.org/patch/7619/); [thread](https://www.postgresql.org/message-id/flat/CALdaZOikwmOS8_dSMm85xR1MZXPuHVfODJG7n1zS3nd2w%2B4VMw%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). No objection to the goal or the design survived at level 1. The one surviving objection, maintenance-fit-2, is a fixable level 2 point: message #1 gives no planning-time number for the added proof. The check mirrors one that choose_bitmap_and() already makes, and it rejects only plans whose BitmapAnd cannot remove rows. Confidence is low because only one message is visible and no planner committer has replied.

Not checked: The code and repro.sql were not read. Whether the new check rejects only a BitmapAnd that cannot remove rows, and how often the extra proof runs, could not be confirmed.

## Prior attempts

- [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://www.postgresql.org/message-id/flat/20200704004503.GA23805@telsasoft.com) (related; Returned with feedback, 2021-02-01). "I really quite dislike solving this via a kluge in indxpath.c." ([message](https://www.postgresql.org/message-id/228337.1605212074%40sss.pgh.pa.us)) Different now: P1 added a new proof step in generate_bitmap_or_paths() for partition constraints. This patch mirrors a redundancy check that choose_bitmap_and() already makes.
- [Revert my best_inner_indexscan patch of yesterday, which turns out to have had a bad side-effect:](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a81e281636ac8c927528648d9eed04e8083fcdf5) (related; Reverted, 2006-04-09). "it stopped finding plans that involved BitmapAnd combinations of indexscans using both join and non-join conditions." Different now: P4 reverted a best_inner_indexscan change that dropped useful BitmapAnd plans. This patch rejects only a BitmapAnd whose later quals imply the whole predicate, so the rejected plan cannot remove rows.

## Objections a reviewer is likely to raise

### 1. Extra predicate proofs add planning cost for every partial-index query

An OLTP schema has several partial indexes on one table and short queries that plan each time. After the change, choose_bitmap_and() runs another predicate_implied_by() proof for candidate pairs, and planning time goes up for every such query, including those whose plan never changes. The cost appears only after upgrade, in latency numbers.

The patch adds a second implication proof in a loop that already does one, and it runs for every query with a clauseless partial index among the bitmap candidates. Message #1 gives no planning-time measurement, and the brief marks this as unknown. The plan gain is limited to setups with a low random_page_cost, while the proof cost falls on every query with this index shape.

**Question:** What is the planning-time cost of the added proof on a table with many partial indexes and many bitmap candidates, measured on a query whose plan does not change?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/CALdaZOikwmOS8_dSMm85xR1MZXPuHVfODJG7n1zS3nd2w%2B4VMw%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. What is the planning-time cost of the added proof on a table with many partial indexes, measured on a query whose plan does not change?
2. Can you post the repro schema and the EXPLAIN ANALYZE output before and after the patch in the thread body?
3. Does a partly implied predicate, or a predicate implied through a BitmapOr arm, still get the double-counted estimate after the patch?
4. Do you propose this for master only, or also for back branches?

## Touch points

- **planner behaviour change:** choose_bitmap_and() stops choosing some BitmapAnd plans that lead with a clauseless partial index.
- **backpatchability:** The proposal does not say whether it targets back branches, where a plan change in a minor release needs care.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a second redundancy rule to choose_bitmap_and() in indxpath.c, limited to plain IndexPaths.
- **performance on the hot path:** The patch adds a predicate proof during planning when a clauseless partial index leads the AND group.

## Who would commit it, and when

- Owner: silent. A committer who knows bitmap path selection and predicate proofs in indxpath.c would have to own it. Nobody has replied to message #1.
- Priority: proceed; audience niche; review cost low. The patch is about 13 KiB, touches one planner file plus regression tests, and adds no user-visible surface. It affects only queries where a clauseless partial index leads a bitmap AND group and random_page_cost is low.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
