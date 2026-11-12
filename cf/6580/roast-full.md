# Design pre-review: Fix performance of REFRESH MATERIALIZED VIEW CONCURRENTLY

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6580](https://commitfest.postgresql.org/patch/6580/) (Needs review); authors: Surya Poondla (spoondla)
- Thread: [Two issues with REFRESH MATERIALIZED VIEW CONCURRENTLY](https://www.postgresql.org/message-id/flat/40d694df-39fd-4a4a-9459-9d6489165f60%40gogi.tv), started 2026-02-09
- Messages read: 22 of 22
- Patch prose read: 0001-Check-for-duplicate-rows-with-NULLs.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The slowdown in message #1 is reproducible, but the entry has no working design (prior-attempts-1). The author reports in message #20 that v7-0002 never skips a column, and the earlier unconditional skip gives wrong results for the (NULL, NULL) x2 case from message #16. Two level 1b questions also stand: a partial unique index already avoids the slow join (already-exists-1), and the only case is synthetic (demand-1). Confidence is medium because only 3 of 22 messages were read directly.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The v7-0002 patch and messages #4 to #22 were not read directly; the no-op and the (NULL, NULL) x2 failure rest on the brief and the objections' account of messages #16 and #20. A new design would need a fresh review at level 2.

## The proposal, restated

**Problem.** Refresh materialized view concurrently becomes much slower when a unique index covers a column full of NULLs, because the diff join compares that column with SQL equality and NULL = NULL never matches. In the reported case the refresh went from about 1700 ms to about 9000 ms after a second unique index on an all-NULL column. Users of concurrent refresh on materialized views with such indexes are affected.

**Proposed design.** The patch leaves nullable columns out of the full outer join condition in refresh_by_match_merge() and keeps the record equality operator (*=) for the whole row. A later version skips a nullable column only when its index also has a NOT NULL column. In message #20 the author says this guard is never true for materialized views, so the current patch does nothing.

**What the author claims.**
- The first version cut the reported refresh from about 9000 ms (11000 ms in a later run) to about 3600 ms.
- Cases where nullable unique-index columns never hold NULLs may get slightly slower; users can add NOT NULL to recover.
- Check and check-world pass with no regressions.
- No test is added because the timings are not stable.
- In message #20 the author says v7-0002 does not fix the regression, since materialized view columns are never marked NOT NULL.

**Thread state.** The author found in July that the performance patch is a no-op and proposed going ahead with the duplicate-row fix (CF 6579) alone while looking for another approach. Robert Haas and Yugo Nagata discussed only the correctness fix, and nobody has proposed a new design for the performance issue.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-02-09; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. The only two designs tried are either incorrect or a no-op

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 7 · design · `prior-attempts-1`

A committer reviews entry 6580 as it is listed, Needs review, and commits v7-0002. Concurrent refresh on the reported matview stays about 3 times slower, because the guard never fires. If v7-0002 is relaxed back to the first version to recover the speed, the (NULL, NULL) x2 case is computed wrong again.

Message #20 reports that v7-0002 skips a column only when its index has a NOT NULL column. Matview columns are never marked NOT NULL, so the patch does nothing. The first version skipped nullable columns without a condition, and message #16 shows that version breaks the (NULL, NULL) x2 refresh. Message #20 proposes dropping 0002 and finding another approach, but entry 6580 still asks for review of the dead design.

Thread: [#6 (surya poondla, 2026-02-13)](https://www.postgresql.org/message-id/CAOVWO5qF2_FV7M%3DUSdSgjjPO124YMpw7TY%2BM4%2BmnkNPOA2Bstg%40mail.gmail.com), [#16 (cca5507, 2026-03-21)](https://www.postgresql.org/message-id/tencent_2DDB9D84961482D40AA55CC88CE0E51D740A%40qq.com), [#17 (surya poondla, 2026-03-25)](https://www.postgresql.org/message-id/CAOVWO5o46gqX_%3DBkrzuTRyPijDBWaYAGFVqn7UXrww_4owhJ0w%40mail.gmail.com), [#20 (surya poondla, 2026-07-21)](https://www.postgresql.org/message-id/CAOVWO5pSxK9TZEEQ7xgNQOUFob_9F-u3YRRogXjDyRktgfaC%2BQ%40mail.gmail.com)

**Question:** Which new design removes NULL-key rows from the diff join without matching (NULL, NULL) rows by mistake, and how does it avoid the NOT NULL flag?

Counter-check: held. Message #20 confirms both points: the guarded version never skips a column, and the unguarded version is incorrect for the (NULL, NULL) x2 case. A new design in a later patch version could solve this, so the severity is fixable.

### 2. A partial unique index already avoids the slow diff join

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A user hits the slowdown from message #1 and waits for a core fix. They could write `create unique index on s(y) where y is not null` today: it enforces the same uniqueness, and refresh_by_match_merge() leaves it out of the join, so the refresh is fast again.

Concurrent refresh uses only unique indexes without a WHERE clause for the join condition, so a partial index takes no part in the diff join. Under the default NULLS DISTINCT, a unique index on y and one on y where y is not null accept the same rows. The thread offers adding NOT NULL as the remedy (message #6), but message #20 says matview columns cannot be NOT NULL, and v7-0002 changes nothing. A committer would likely ask whether a doc note covers the reported case with less risk than changing the join.

Thread: [#1 (Giuliano Gagliardi, 2026-02-09)](https://www.postgresql.org/message-id/40d694df-39fd-4a4a-9459-9d6489165f60%40gogi.tv), [#6 (surya poondla, 2026-02-13)](https://www.postgresql.org/message-id/CAOVWO5qF2_FV7M%3DUSdSgjjPO124YMpw7TY%2BM4%2BmnkNPOA2Bstg%40mail.gmail.com), [#20 (surya poondla, 2026-07-21)](https://www.postgresql.org/message-id/CAOVWO5pSxK9TZEEQ7xgNQOUFob_9F-u3YRRogXjDyRktgfaC%2BQ%40mail.gmail.com)

**Question:** Which users cannot replace the full unique index on the nullable column with a partial one, and what does a core change give them?

Counter-check: held. Concurrent refresh ignores unique indexes that have a predicate, and under NULLS DISTINCT a partial index `where y is not null` enforces the same uniqueness. The thread does not discuss this workaround, so the question stands.

### 3. Only a synthetic case from reading the code, with no field report

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer redesigns the diff join in refresh_by_match_merge() for a pattern that may be rare: a matview with a unique index on a mostly-NULL column, refreshed concurrently. Every concurrent refresh then runs the new join logic. If that logic has a correctness bug, all such users get wrong matview contents to speed up a case few of them have.

Message #1 found the slowdown by reading matview.c and showing it with generate_series. Neither it nor any later message reports a production matview hit by it. Message #21 says the delete-and-reinsert path is costly only when there are many such rows. Dropping the extra unique index works around it, since concurrent refresh needs only one usable unique index.

Thread: [#1 (Giuliano Gagliardi, 2026-02-09)](https://www.postgresql.org/message-id/40d694df-39fd-4a4a-9459-9d6489165f60%40gogi.tv), [#21 (Robert Haas, 2026-07-27)](https://www.postgresql.org/message-id/CA%2BTgmoaryPLAMNWhF8cP4L%2Bf%2BwQKkwS5OGCC4rUCwfpm%2Bu9-fg%40mail.gmail.com)

**Question:** Which real deployments have unique indexes on mostly-NULL matview columns, and why can they not drop that index or refresh without CONCURRENTLY?

Counter-check: held. The only case in the thread is the generate_series example from message #1. Message #21 agrees the path is slow with many such rows, but nobody reports a production matview affected, and the first version showed the correctness risk is real.

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
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The report is on pgsql-bugs, and the thread does not say whether the fix targets back branches or only version 20. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch changes the diff join in refresh_by_match_merge(), where a mistake gives wrong matview contents, and adds no test. |
| performance on the hot path | **yes** | Every concurrent refresh runs the changed join, and the author expects a small slowdown when nullable index columns hold no NULLs. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A partial unique index `where y is not null` already keeps the column out of the diff join. |

## 5. Questions the author must answer before anyone reviews code

1. Which design keeps rows with NULL keys out of the delete-and-reinsert path without matching (NULL, NULL) rows by mistake, given that matview columns are never marked NOT NULL?
2. Should entry 6580 stay at Needs review, or be withdrawn until a new performance design is posted?
3. Which users cannot replace the full unique index on the nullable column with a partial index `where y is not null`, and what does a core change give them?
4. Which real deployments refresh a matview concurrently with a unique index on a mostly-NULL column?
5. What does the new design cost a concurrent refresh when the nullable unique-index columns hold no NULLs?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The owner would need to know the diff logic in refresh_by_match_merge() and record equality semantics. Robert Haas took part in the thread, but only on the duplicate-row fix (CF 6579).
- **Liability:** The owner signs for the correctness of concurrent refresh for every matview, with no time limit. A wrong join condition leaves a matview that differs from its query without an error.
- **Author follow-up:** The author stayed through v7, reported in message #20 that v7-0002 does nothing, and said they are looking for another approach. No new version is announced for this entry.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The slowdown needs a unique index on a mostly-NULL matview column, and a partial index avoids it today. The change is confined to matview.c with no user-visible surface, but the current patch does nothing; review cost is a guess because the performance patch was not read.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` The current performance patch is dead code with no working design behind it. Refuted (not specific): Duplicate of prior-attempts-1. The added scenario about future NOT NULL support for materialized view columns rests on a commit of v7-0002, which the author has already asked to set aside in message #20.
- `maintenance-fit-2` Join conditions and duplicate pre-check are coupled with no loud failure. Refuted (no failure scenario): Dropping a column equality from the shared list makes the pre-check flag more rows as duplicates, as the first version did in message #16. The objection is also about v7-0001, which belongs to entry 6579.
- `demand-2` The entry has no working design left to evaluate. Refuted (not specific): Duplicate of prior-attempts-1, which states the same point about message #20 with more detail.

### Review angles

- prior-attempts: 1 objection(s). No precedent found in the corpus. The one objection rests on the earlier attempts inside this thread (messages #6, #16, #20), not on any outside rejection.
- standards-compat: 0 objection(s). The current v7-0002 changes no user-visible behaviour (message #20), and the earlier unconditional skip was withdrawn after the (NULL, NULL) case was raised. A redesign that changes refresh results for existing materialized views, or relies on NOT NULL on matview columns, would need a fresh review from this angle.
- replication-ondisk: 0 objection(s). The performance patch changes only the diff query that refresh_by_match_merge() builds. It adds no on-disk format, WAL record, catalog entry, protocol message or extension API, and the refresh still writes ordinary inserts and deletes that standbys replay as before. The one catalog change in the thread, the new built-in function in message #14, belongs to the duplicate-row fix and not to this entry.
- security: 0 objection(s). The proposal changes only the join condition that refresh_by_match_merge() builds for REFRESH MATERIALIZED VIEW CONCURRENTLY, which already requires ownership of the view. It adds no new privilege, object, setting or exposure, and the duplicate-row error detail shows only rows the owner can already read.
- maintenance-fit: 2 objection(s). Two objections only. The correctness fix in CF 6579 is outside this entry. Hot-path cost could not be judged because no working performance design exists.
- already-exists: 1 objection(s). One objection only: the partial-index idiom covers the reported case. Other angles cover the missing working design.
- demand: 2 objection(s). The slowdown is real and reproducible, but the material shows only one synthetic report and no production case.

### Precedents retrieved

- P1: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P2: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant; reason summarised from its thread
- P3: [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://commitfest.postgresql.org/patch/2664/) (cf, Returned with feedback, 2020-11-24) - judged not relevant; reason summarised from its thread
- P4: [Fix for REFRESH MATERIALIZED VIEW ownership error message](https://commitfest.postgresql.org/patch/1766/) (cf, Rejected, 2018-08-19) - judged not relevant; reason summarised from its thread
- P5: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P6: [Allow granting of permission to REFRESH MATERIALIZED VIEW, CLUSTER, and REINDEX](https://commitfest.postgresql.org/patch/1605/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P7: [Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet](https://commitfest.postgresql.org/patch/4077/) (cf, Withdrawn, 2022-12-27) - judged not relevant; reason summarised from its thread
- P8: [Triggers on Materialized Views](https://commitfest.postgresql.org/patch/1948/) (cf, Returned with feedback, 2019-04-01) - judged not relevant
- P9: [Temporary materialized views](https://commitfest.postgresql.org/patch/1951/) (cf, Returned with feedback, 2019-03-15) - judged not relevant
- P10: [Triggers on materialized views](https://commitfest.postgresql.org/patch/1953/) (cf, Withdrawn, 2019-01-04) - judged not relevant
- P11: [Range Merge Join](https://commitfest.postgresql.org/patch/3160/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P12: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P13: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant
- P14: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P15: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P16: [Range Merge Join](https://commitfest.postgresql.org/patch/1106/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P17: [Indexes with Included Columns (was Covering + unique indexes)](https://commitfest.postgresql.org/patch/961/) (cf, Returned with feedback, 2017-04-08) - judged not relevant
- P18: [Range Merge Join](https://commitfest.postgresql.org/patch/1449/) (cf, Returned with feedback, 2018-03-28) - judged not relevant
- P19: [Revert 95d737ff to add 'ignore_nulls'](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8a026e4f15e2063cd7fe6cb9ffe44f2a81398bd) (revert, Reverted, 2014-09-29) - judged not relevant
- P20: [Incremental sort](https://commitfest.postgresql.org/patch/1011/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P21: [Adding CommandID to heap xlog records](https://commitfest.postgresql.org/patch/3882/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P22: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P23: [Revert "Add some temporary code to record stack usage at server process exit."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96112ee7c60557bb192a9aa07b514db2400fd45e) (revert, Reverted, 2016-07-10) - judged not relevant
- P24: [Covering + unique indexes](https://commitfest.postgresql.org/patch/433/) (cf, Returned with feedback, 2016-04-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 121 s; cost reported by the CLI: $2.00
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 73 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:55:23.502Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
