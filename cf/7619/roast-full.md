# Design pre-review: Avoid redundant BitmapAnd with a clauseless partial index

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7619](https://commitfest.postgresql.org/patch/7619/) (Needs review); authors: Krishna Sarabu (ksarabu)
- Thread: [Avoid redundant BitmapAnd with a clauseless partial index](https://www.postgresql.org/message-id/flat/CALdaZOikwmOS8_dSMm85xR1MZXPuHVfODJG7n1zS3nd2w%2B4VMw%40mail.gmail.com), started 2026-11-10
- Messages read: 1 of 1
- Patch prose read: v1-0001-Avoid-redundant-BitmapAnd-with-a-clauseless-partial-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

No objection to the goal or the design survived at level 1. The one surviving objection, maintenance-fit-2, is a fixable level 2 point: message #1 gives no planning-time number for the added proof. The check mirrors one that choose_bitmap_and() already makes, and it rejects only plans whose BitmapAnd cannot remove rows. Confidence is low because only one message is visible and no planner committer has replied.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code and repro.sql were not read. Whether the new check rejects only a BitmapAnd that cannot remove rows, and how often the extra proof runs, could not be confirmed.

## The proposal, restated

**Problem.** choose_bitmap_and() rejects a partial index whose predicate the already chosen quals imply, but misses the reverse order. When a clauseless partial index leads the AND group and a later index's quals imply its predicate, the BitmapAnd removes no rows but counts the predicate's selectivity twice. Users with a low random_page_cost can get this plan when the other index alone would be better.

**Proposed design.** The patch adds the mirror-image check to choose_bitmap_and() in indxpath.c. It rejects the combination only when the later index's quals imply the whole predicate, and only for plain IndexPaths, because find_indexpath_quals() flattens BitmapOr arms into one list. The patch adds regression tests in create_index.

**User-visible surface.**
- Plan choice: some bitmap AND plans with a clauseless partial index are no longer chosen; no new syntax, GUC or API.

**What the author claims.**
- In the repro, the BitmapAnd is estimated at about 15 rows against 2,000 actual.
- With random_page_cost = 1.1 the planner picks the BitmapAnd, including on the outer side of a nested-loop semi join.
- With the default random_page_cost the plan does not change.
- The BitmapAnd cannot remove any rows in this case.

**Thread state.** Only the first message is visible. Nobody has replied, and the CommitFest entry is in Needs review.

## 2. Prior attempts

### avoid bitmapOR-ing indexes for scan conditions implied by partition constraint (related)

- **Link:** [https://commitfest.postgresql.org/patch/2644/](https://commitfest.postgresql.org/patch/2644/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20200704004503.GA23805@telsasoft.com](https://www.postgresql.org/message-id/flat/20200704004503.GA23805@telsasoft.com)
- **Date:** decided 2021-02-01; started 2020-07-04
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane objected to fixing this in indxpath.c. The fix helps only bitmap scans, may repeat the same proof, cannot be turned off, leaves row estimates unchanged, and may break the loop that requires each OR arm to match an index. He proposed removing proven quals from baserestrictinfo instead, but measured about 2.5% slowdown on pgbench -S and left the direction open. The entry then sat in Waiting on Author for weeks, and Masahiko Sawada returned it with feedback.
- **Reason (quote):** "I really quite dislike solving this via a kluge in indxpath.c." ([message](https://www.postgresql.org/message-id/228337.1605212074%40sss.pgh.pa.us))
- **What is different this time:** P1 added a new proof step in generate_bitmap_or_paths() for partition constraints. This patch mirrors a redundancy check that choose_bitmap_and() already makes.

### Revert my best_inner_indexscan patch of yesterday, which turns out to have had a bad side-effect: (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a81e281636ac8c927528648d9eed04e8083fcdf5](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a81e281636ac8c927528648d9eed04e8083fcdf5)
- **Date:** decided 2006-04-09; started 2006-04-09
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Revert my best_inner_indexscan patch of yesterday, which turns out to have had a bad side-effect: it stopped finding plans that involved BitmapAnd combinations of indexscans using both join and non-join conditions. Instead, make choose_bitmap_and more aggressive about detecting redundancies between BitmapOr subplans.
- **Reason (quote):** "it stopped finding plans that involved BitmapAnd combinations of indexscans using both join and non-join conditions."
- **What is different this time:** P4 reverted a best_inner_indexscan change that dropped useful BitmapAnd plans. This patch rejects only a BitmapAnd whose later quals imply the whole predicate, so the rejected plan cannot remove rows.


## 3. Objections a reviewer is likely to raise

### 1. Extra predicate proofs add planning cost for every partial-index query

fixable · level 2, rubric item 8 · performance on the hot path, planner behaviour change · `maintenance-fit-2`

An OLTP schema has several partial indexes on one table and short queries that plan each time. After the change, choose_bitmap_and() runs another predicate_implied_by() proof for candidate pairs, and planning time goes up for every such query, including those whose plan never changes. The cost appears only after upgrade, in latency numbers.

The patch adds a second implication proof in a loop that already does one, and it runs for every query with a clauseless partial index among the bitmap candidates. Message #1 gives no planning-time measurement, and the brief marks this as unknown. The plan gain is limited to setups with a low random_page_cost, while the proof cost falls on every query with this index shape.

Thread: [#1 (Krishna Sarabu, 2026-11-10)](https://www.postgresql.org/message-id/CALdaZOikwmOS8_dSMm85xR1MZXPuHVfODJG7n1zS3nd2w%2B4VMw%40mail.gmail.com)

**Question:** What is the planning-time cost of the added proof on a table with many partial indexes and many bitmap candidates, measured on a query whose plan does not change?

Counter-check: held. Message #1 gives no planning-time number, so the request for one stands. The cost is narrower than stated: the new proof runs only when a clauseless partial index leads the AND group, in a loop that already runs predicate_implied_by().

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | choose_bitmap_and() stops choosing some BitmapAnd plans that lead with a clauseless partial index. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The proposal does not say whether it targets back branches, where a plan change in a minor release needs care. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a second redundancy rule to choose_bitmap_and() in indxpath.c, limited to plain IndexPaths. |
| performance on the hot path | **yes** | The patch adds a predicate proof during planning when a clauseless partial index leads the AND group. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What is the planning-time cost of the added proof on a table with many partial indexes, measured on a query whose plan does not change?
2. Can you post the repro schema and the EXPLAIN ANALYZE output before and after the patch in the thread body?
3. Does a partly implied predicate, or a predicate implied through a BitmapOr arm, still get the double-counted estimate after the patch?
4. Do you propose this for master only, or also for back branches?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows bitmap path selection and predicate proofs in indxpath.c would have to own it. Nobody has replied to message #1.
- **Liability:** The owner signs for plan changes in choose_bitmap_and() for users with a low random_page_cost, and for any planning-time cost of the extra proof. The liability lasts as long as the heuristic exists.
- **Author follow-up:** No sign either way; the thread has only the first message, posted two days before this review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 13 KiB, touches one planner file plus regression tests, and adds no user-visible surface. It affects only queries where a clauseless partial index leads a bitmap AND group and random_page_cost is low.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Narrow ordering special case leaves the double-counted estimate elsewhere. Refuted (no failure scenario): The scenario describes today's behaviour left unchanged, which the patch does not cause. The check mirrors the one choose_bitmap_and() already has, a partly implied predicate can still remove rows, and [P1] concerned partition constraints in generate_bitmap_or_paths().

### Review angles

- prior-attempts: 0 objection(s). Neither precedent's reason applies here. The patch mirrors an existing check in choose_bitmap_and(), and the plans it rejects cannot remove rows, so P4's lost-plan problem does not arise; the thread cites no earlier discussion.
- standards-compat: 0 objection(s). The proposal adds no syntax, GUC, catalog object, dump state or client-visible output. It changes plan choice only in a case where the BitmapAnd cannot remove rows, by mirroring a check choose_bitmap_and() already makes. If it were back-patched, plans would change in a minor release, but the proposal does not say it targets back branches, so I raise no objection on that.
- replication-ondisk: 0 objection(s). The proposal changes only plan choice in choose_bitmap_and() in indxpath.c, plus regression tests. It adds no on-disk, WAL, catalog, wire protocol or extension API state, so this angle has nothing to object to. Possible plan changes after a minor upgrade belong to the planner angle.
- security: 0 objection(s). The patch only changes which bitmap index paths choose_bitmap_and() combines. It adds no new privilege, object, setting or function, and it exposes no data or state. Predicate proofs use only immutable expressions, so no user-controlled code runs during planning. The proposal therefore has no security or privilege aspect to object to.
- maintenance-fit: 2 objection(s). Two fixable design points. Committability: no planner committer has replied yet. The change is small and stays in one function, so the maintenance cost after commit is low.
- already-exists: 0 objection(s). From this angle the proposal has no existing substitute. It extends the existing redundancy check in choose_bitmap_and() to the reverse order and adds no second mechanism. Per-query workarounds such as enable_bitmapscan = off or the pg_hint_plan extension can avoid the bad plan in a known query, but they do not fix the double-counted selectivity estimate, and no hook lets an extension change choose_bitmap_and().
- demand: 0 objection(s). No demand objection applies. The patch adds the mirror-image of an existing check in choose_bitmap_and() for a misestimate that removes no rows, and random_page_cost = 1.1 is a common setting on SSD-backed systems. The material has only a synthetic repro and no field report, and the planning-time cost is unmeasured, but both are design questions for other angles.

### Precedents retrieved

- P1: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - used in the review; reason summarised from its thread
- P2: [bitmap cost should account for correlated indexes](https://commitfest.postgresql.org/patch/2310/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P3: [On-disk bitmap indexes](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P4: [Revert my best_inner_indexscan patch of yesterday, which turns out to have had a bad side-effect:](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a81e281636ac8c927528648d9eed04e8083fcdf5) (revert, Reverted, 2006-04-09) - used in the review
- P5: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P6: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P7: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P8: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant; reason summarised from its thread
- P9: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant
- P10: [Predicate locking in SP-Gist index](https://commitfest.postgresql.org/patch/1215/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P11: [Revert "Show index search count in EXPLAIN ANALYZE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d00107cd63e780753aa25563fa37603369997d0c) (revert, Reverted, 2025-03-05) - judged not relevant
- P12: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P13: [Decoupling our alignment assumptions about int64 and double](https://commitfest.postgresql.org/patch/6441/) (cf, Withdrawn, 2026-02-03) - judged not relevant; reason summarised from its thread
- P14: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - judged not relevant; reason summarised from its thread
- P15: [Redundant qualifier elimination](https://commitfest.postgresql.org/patch/7127/) (cf, Returned with feedback, 2026-08-14) - judged not relevant
- P16: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant
- P17: [Remove redundant initializations](https://commitfest.postgresql.org/patch/3229/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P18: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P19: [Skip a redundant singleton GROUP BY node](https://commitfest.postgresql.org/patch/7419/) (cf, Withdrawn, 2026-10-09) - judged not relevant
- P20: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P21: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P22: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P23: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [Remove redundant NULL check in clause_selectivity_ext()](https://commitfest.postgresql.org/patch/5188/) (cf, Rejected, 2024-09-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 116 s; cost reported by the CLI: $0.93
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 55 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:08:38.813Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
