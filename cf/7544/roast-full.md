# Design pre-review: Add amnotifydelete, an index AM callback for deleted tuples

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7544](https://commitfest.postgresql.org/patch/7544/) (Needs review); authors: Jan Bjorge Lovland (janbjorge)
- Thread: [Design proposal: grain, exact per-time-bucket aggregates from a summarizing index](https://www.postgresql.org/message-id/flat/CAA%2BRuFCYSbRr41E3Qt4cqKS%2B%2B_OQFiA%3DQUyidXL4f0c7zUiyKQ%40mail.gmail.com), started 2026-10-20
- Messages read: 4 of 4
- Patch prose read: v2-0001-Add-amnotifydelete-index-access-method-callback.patch, v1-0001-Add-amnotifydelete-index-access-method-callback.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The proposal fails at level 1b on already-exists-1 and already-exists-2. It does not say what amnotifydelete does that an AFTER ROW trigger inserting markers cannot, or what core placement adds over an extension beyond the planner path. Demand shown is one workload, and message #1 says TimescaleDB works. Confidence is low because only the author has posted, so the verdict rests on missing answers.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can check whether patch 1 covers every tuple-removing path or what the relcache check costs under concurrency. Message #4 was not shown, and the grain index type is not written.

## The proposal, restated

**Problem.** Index access methods learn of deleted tuples only when VACUUM calls ambulkdelete, so an index that keeps aggregates cannot stay correct within the deleting transaction. The author wants this for time-bucket rollups such as daily count and sum, whose query time grows with rows in the window under btree, BRIN or no index. Materialized views go stale, and trigger-based rollups lock the view on DELETE and UPDATE and fail under REPEATABLE READ.

**Proposed design.** Patch 1 adds an optional amnotifydelete callback. The executor passes it the recomputed index values of the old tuple version on DELETE, on non-HOT UPDATE, and on matching paths in MERGE, ON CONFLICT, logical replication apply and REPACK (CONCURRENTLY). Later patches add a grain index type that owns hidden summary and marker tables plus a metapage watermark W, a sealer that advances W after WaitForLockers, and a planner path.

**User-visible surface.**
- C API: new amnotifydelete member in IndexAmRoutine, NULL for all existing index AMs.
- Planned SQL: create index ... using grain (ts) include (cols) with (buckets = '1 minute, 1 hour, 1 day').
- Planned function pg_grain_buckets(index, from, to) to read the summaries.
- Planned hidden summary and marker tables owned by the index, visible to pg_dump and pg_upgrade.
- Planned planner path for group by date_bin(...) over a grain index.

**What the author claims.**
- Tables without an opted-in index pay one cached relcache check per row; one-row DELETE and UPDATE timings match master.
- DELETE of every row on 1 million rows took about 555 ms with dummy_index_am using the callback, against about 463 ms on master.
- On 119M NYC taxi rows, the grain prototype answers daily 30-day queries in 0.134 ms against 896 ms for plain GROUP BY.
- Results are exact, including after late writes, because they rely only on locks and MVCC visibility.
- The summary table is under 2% of the table size.

**Thread state.** Only the author has posted: the design, v1 and v2 of patch 1, and prototype numbers for patch 2. No one has replied, and his five design questions are unanswered.

## 2. Prior attempts

### Incremental Materialized View Maintenance (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2138/](https://commitfest.postgresql.org/patch/2138/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20181227215726.4d166b4874f8983a641123f5@sraoss.co.jp](https://www.postgresql.org/message-id/flat/20181227215726.4d166b4874f8983a641123f5@sraoss.co.jp)
- **Date:** decided 2022-10-12; started 2018-12-27
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier, huyajun (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier marked the entry Returned with Feedback after a month with no update. In July 2022 huyajun had reported bugs: a deadlock between TRUNCATE and concurrent INSERT, wrong counts after concurrent DELETEs, crashes on mismatched column lists, and an unwanted unique index on cross joins. The author said he was still working on them. Paquier also noted little design discussion in three years, and Greg Stark had earlier called the patch hard to review.
- **Reason (quote):** "There has been some feedback sent lately and no update for one month, so I am marking it as RwF for now." ([message](https://www.postgresql.org/message-id/Y0ZyiXIEVa9i39ng%40paquier.xyz))
- **What is different this time:** P1 kept aggregates current with AFTER triggers on a materialized view. This proposal keeps them in an index-owned summary table, using an executor callback, markers and a sealer watermark, so writers do not lock a shared rollup row.


## 3. Objections a reviewer is likely to raise

### 1. Row-level triggers already deliver old values within the deleting transaction

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), ABI / extension API, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A committer accepts amnotifydelete, which runs on every DELETE, non-HOT UPDATE, MERGE, ON CONFLICT, apply and REPACK path. Postgres now has two per-row notifications of deleted tuples, AFTER ROW triggers and this callback, and every future write path must remember to call both. If a new path misses the callback, nothing fails loudly, and a grain index silently drifts from the table.

An AFTER ROW DELETE or UPDATE trigger already receives the old row in the deleting transaction, which is the need the commit message states. Message #1 rejects triggers by pointing to the IVM patch, which locks the view and errors under REPEATABLE READ. But grain's own protocol only inserts marker rows when the time key is before W, and a trigger can do that without locking any summary row. The proposal does not show what the callback can do that such a trigger cannot.

Thread: [#1 (Jan Bjørge Løvland, 2026-10-20)](https://www.postgresql.org/message-id/CAA%2BRuFCYSbRr41E3Qt4cqKS%2B%2B_OQFiA%3DQUyidXL4f0c7zUiyKQ%40mail.gmail.com)

**Question:** Which step of the grain protocol cannot be done by a C row trigger that inserts markers, with the same WaitForLockers sealer, and why?

Counter-check: held. Message #1 rejects triggers because the IVM patch locks the view and fails under REPEATABLE READ. Inserting a marker row needs neither, and the proposal does not say what the callback does that an always-enabled C row trigger cannot.

### 2. The rollup can be built as an extension; core placement needs a reason

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), catalog changes, pg_dump / restore round-trip, maintenance burden ("who maintains this in 10 years") · `already-exists-2`

The project adds a grain index type with index-owned hidden heaps, new pg_dump and pg_upgrade handling, and a planner path. Every later catalog, dump or upgrade change must then account for a new kind of relation ownership. Users who need rollups today can already use TimescaleDB continuous aggregates, which merge stored buckets with recent raw rows past a watermark.

Message #1 says TimescaleDB works for this and calls it too heavy for one rollup. That makes it a convenience gap, since the job can be done today. A summary table, a marker table, a trigger, a sealer run by a background worker, and a pg_grain_buckets() function all fit in an extension. The thread has no demand beyond the author's own workload and no reply from anyone else.

Thread: [#1 (Jan Bjørge Løvland, 2026-10-20)](https://www.postgresql.org/message-id/CAA%2BRuFCYSbRr41E3Qt4cqKS%2B%2B_OQFiA%3DQUyidXL4f0c7zUiyKQ%40mail.gmail.com), [#4 (Jan Bjørge Løvland, 2026-11-07)](https://www.postgresql.org/message-id/CAA%2BRuFAnX%3Ds6aD0Oj7CeT93Uh0nUY5GBtKa-G6zRKpSCRpJt5w%40mail.gmail.com)

**Question:** What does core placement give grain that an extension built on triggers, a background worker and a set-returning function cannot have, apart from the planner path?

Counter-check: held. Message #1 says TimescaleDB works and objects only to its weight, which is a convenience argument. The material does not show what an extension with a trigger, a background worker and a function cannot do, apart from the planner path.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The planned grain index keeps watermark W in its metapage and owns hidden summary and marker tables that pg_upgrade must carry over. |
| WAL / replication / standby | **yes** | Patch 1 adds calls in the logical replication apply paths; question 5 leaves the metapage WAL logging open. |
| wire protocol | no |  |
| catalog changes | **yes** | The planned index-owned hidden tables are a new kind of relation ownership; patch 1 adds a relcache flag. |
| ABI / extension API | **yes** | Patch 1 adds the amnotifydelete member to IndexAmRoutine. |
| SQL-standard conflict or non-standard syntax | **yes** | Planned `create index ... using grain ... with (buckets = ...)` and pg_grain_buckets() add non-standard surface. |
| new GUC | no |  |
| planner behaviour change | **yes** | Patch 3 plans a planner path for group by date_bin(...) over a grain index. |
| pg_dump / restore round-trip | **yes** | Question 3 in message #1 leaves open how pg_dump handles the hidden summary and marker tables. |
| psql UX conventions | no |  |
| security / privileges | **yes** | The summaries aggregate all rows matching the index predicate, with no stated RLS or column privilege check on reads. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every current and future tuple-removing path must call amnotifydelete, and only dummy_index_am tests it. |
| performance on the hot path | **yes** | Every DELETE and non-HOT UPDATE on every table pays one cached relcache check per row. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #1 says TimescaleDB works, and the protocol may fit in an extension built on triggers. |

## 5. Questions the author must answer before anyone reviews code

1. Which step of the grain protocol cannot be done by a C row trigger that inserts markers, with the same WaitForLockers sealer?
2. What does core placement give grain that an extension with triggers, a background worker and a set-returning function cannot have, apart from the planner path?
3. Would the answers to your questions 1 and 2 change the amnotifydelete signature or call contract, and if so, why should patch 1 go in first?
4. What is the write cost of a grain index on inserts, updates and deletes, including marker inserts for rows before W?
5. How does the sealer run every minute, and how long does its WaitForLockers call wait when a long write transaction is open?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the index AM API and the executor's ModifyTable and logical apply paths. Nobody but the author has posted.
- **Liability:** The owner signs for a permanent IndexAmRoutine member and for every tuple-removing path calling it correctly. A missed call gives wrong aggregates with no error.
- **Author follow-up:** The author posted v1 two days after the design and a rebased v2 with a CI fix eight days later, which suggests he stays engaged.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** unclear
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The thread shows one workload and no replies, so the audience is not shown. The patch is about 51 KiB across the executor, relcache, repack and logical apply, and its only planned user still has five open design questions.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Index-owned hidden tables have no defined dump, upgrade or publication behaviour. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) A never-scanned index breaks what tools expect of an index. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Callback enters the extension API before its only consumer's design is accepted. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-1` (fixable, level 2) Grain summaries can expose rows hidden by row-level security or column privileges. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) Every future tuple-removing path must call amnotifydelete, with no loud failure. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `replication-ondisk-2` Metapage watermark and hidden summary tables have no pg_upgrade pairing rule. Refuted (not specific): Duplicate of standards-compat-1, whose failure scenario already covers W no longer matching the hidden tables after pg_upgrade.
- `maintenance-fit-2` Patch 1 adds a core API whose only planned user is unaccepted. Refuted (not specific): Duplicate of replication-ondisk-1. The added remark that no committer has replied is a committability point and does not strengthen the design objection.
- `maintenance-fit-3` Index-owned hidden heaps add a new relation kind to many subsystems. Refuted (not specific): Duplicate of standards-compat-1: both ask how existing commands and tools treat index-owned tables, which is the author's open question 3.
- `demand-1` Patch 1 adds an executor-wide hook with no accepted consumer. Refuted (not specific): Duplicate of replication-ondisk-1 and maintenance-fit-1. Its one separate point, the per-row relcache check, is answered by message #1, where one-row DELETE and UPDATE timings match master.
- `demand-2` Demand shown is one workload, not yet independent users. Refuted (not specific): Duplicate of already-exists-2, which already states that the thread shows no demand beyond the author's workload and that TimescaleDB does the job.

### Review angles

- prior-attempts: 0 objection(s). Only P1 is a close relative, and its recorded reason was inactivity, which does not apply here. Its summary also reports wrong counts after concurrent deletes, but that is an AI summary and does not support an objection on its own. Message #1 cites amtargetdelete and amupdate as earlier callback proposals but does not say why they were not accepted, and the material does not record it either. So no earlier rejection reason can be checked against this design. P21 (BRIN HOT revert) is in the same area, but this patch notifies summarizing indexes on updates that add entries, so the visibility-only gap behind that revert does not apply.
- standards-compat: 2 objection(s). Patch 1 alone adds a C callback that is NULL for existing AMs and changes no SQL, dump or client-visible behaviour. Both objections concern the planned surface that the callback is meant to serve.
- replication-ondisk: 2 objection(s). Patch 1 alone changes no on-disk format, WAL record or wire message. The nbtree VACUUM invariant does not apply, because nbtree only sets the new member to NULL.
- security: 1 objection(s). Patch 1, the amnotifydelete callback, is a C-level index AM hook with no privilege or exposure aspect. The only objection is about the planned grain index and its read paths.
- maintenance-fit: 3 objection(s). The nbtree invariant does not apply: the patch only adds a NULL callback entry in nbtree.c and does not change VACUUM behaviour. Hot-path cost for non-users looks small in the shown numbers, so no objection is raised on that.
- already-exists: 2 objection(s). Two objections raised. Both rest on existing mechanisms, row triggers and an external extension, that appear to cover the need with more effort.
- demand: 2 objection(s). Two demand objections raised. Both are 'not now, show demand', not a rejection of the idea.

### Precedents retrieved

- P1: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P2: [Add callback table access method to reset filenode when dropping relation](https://commitfest.postgresql.org/patch/3073/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P3: [Inconsistency in owner assignment between INDEX and STATISTICS](https://commitfest.postgresql.org/patch/6571/) (cf, Withdrawn, 2026-03-11) - judged not relevant; reason summarised from its thread
- P4: [Revert "Show index search count in EXPLAIN ANALYZE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d00107cd63e780753aa25563fa37603369997d0c) (revert, Reverted, 2025-03-05) - judged not relevant
- P5: [DELETE CASCADE](https://commitfest.postgresql.org/patch/3195/) (cf, Returned with feedback, 2022-01-31) - judged not relevant; reason summarised from its thread
- P6: [Free memory allocated by waitonlock_error_callback()](https://commitfest.postgresql.org/patch/6069/) (cf, Withdrawn, 2025-10-23) - judged not relevant; reason summarised from its thread
- P7: [Remove page-read callback from XLogReaderState](https://commitfest.postgresql.org/patch/2113/) (cf, Returned with feedback, 2021-10-04) - judged not relevant; reason summarised from its thread
- P8: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - used in the review; reason summarised from its thread
- P10: [explain to report bucket and memory stats for TupleHashTable nodes](https://commitfest.postgresql.org/patch/2428/) (cf, Returned with feedback, 2020-07-23) - judged not relevant
- P11: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [pg_buffercache: Add per-relation summary stats](https://commitfest.postgresql.org/patch/6555/) (cf, Returned with feedback, 2026-05-04) - judged not relevant
- P13: [Hidden/invisible columns](https://commitfest.postgresql.org/patch/3361/) (cf, Rejected, 2021-10-29) - judged not relevant
- P14: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant
- P15: [psql: --echo-hidden improvements](https://commitfest.postgresql.org/patch/4903/) (cf, Withdrawn, 2024-03-21) - judged not relevant
- P16: [Revert "Refactor CopyReadAttributes{CSV,Text}() to use a callback in COPY FROM"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=06bd311bce24083c76d9741ae89c98750aaf4b41) (revert, Reverted, 2024-02-14) - judged not relevant
- P17: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [Fix search_path for maintenance commands](https://commitfest.postgresql.org/patch/4367/) (cf, Withdrawn, 2024-01-18) - judged not relevant
- P19: [Incremental sort](https://commitfest.postgresql.org/patch/1011/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P20: [Improve management of SLRU statistics collection.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=81ca8686305c4c62d723ab224ad5c414f350a3a0) (revert, Partially reverted, 2020-05-13) - judged not relevant
- P21: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - used in the review
- P22: [Revert "Refactor ChangeVarNodesExtended() using the custom callback"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2782f3b8454676c9df408aa0bc960efca0f3eb22) (revert, Reverted, 2025-05-03) - judged not relevant
- P23: [TimestampTz->Text->TimestampTz casting fails with DateStyle 'Postgres'](https://commitfest.postgresql.org/patch/5371/) (cf, Returned with feedback, 2025-01-07) - judged not relevant
- P24: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 182 s; cost reported by the CLI: $2.13
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 115 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:07:36.749Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
