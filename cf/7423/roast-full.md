# Design pre-review: Detoast a column once per row instead of once per reference

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7423](https://commitfest.postgresql.org/patch/7423/) (Needs review); authors: Johannes Edmeier (jedmeier)
- Thread: [Detoast a column once per row instead of once per reference](https://www.postgresql.org/message-id/flat/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), started 2026-09-30
- Messages read: 3 of 3
- Patch prose read: v2-0002-Decide-at-plan-time-which-columns-to-detoast-once.patch, v2-0001-Executor-support-for-detoasting-a-column-once-per.patch, v2-0004-Test-once-per-row-detoasting.patch, v2-0003-Share-detoasted-values-with-subplan-and-nestloop-.patch, v1-0001-Executor-support-for-detoasting-a-column-once-per.patch, v1-0003-Share-detoasted-values-with-subplan-and-nestloop-.patch, v1-0002-Decide-at-plan-time-which-columns-to-detoast-once.patch, v1-0004-Test-once-per-row-detoasting.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem is clear and demand is shown, so levels 1a and 1b pass. Message #3 reports wrong results on default settings in two cases, and the author has not yet replied. A custom scan returns 529 rows instead of 500 (replication-ondisk-1), and pg_column_size in a LATERAL call returns 320004 instead of 3679 (standards-compat-1). Both concern the safety rule the design rests on, and maintenance-fit-1 shows the hardcoded lists make such misses silent.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody could check that the executor's argument-position list is complete or that the JIT steps match the interpreter. The same holds for whether a parallel worker can follow the ParamExecData slot back-reference.

## The proposal, restated

**Problem.** Each function or operator applied to a TOASTed column fetches and decompresses it again, so a query naming a wide jsonb or text column in several predicates pays several times. The plan does not show this and no query rewrite avoids it. Users with wide out-of-line columns and repeated references are affected.

**Proposed design.** The planner marks, per plan node, the toastable columns that two or more argument positions read, counting across one node boundary. The executor keeps one detoasted copy beside the slot in tts_detoasted for as long as the slot holds the row, and only argument positions read it; tts_values keeps the stored datum. 0003 lets subplan and nestloop parameters find the copy through a back-reference to the source slot.

**User-visible surface.**
- New GUC detoast_reuse (boolean, default on, developer option), read at plan time.
- New C API field TupleTableSlotOps.resets_detoasted; slot implementations that leave it false keep no copies.
- New TupleTableSlot field tts_detoasted and new fields on Plan, PlannedStmt and ParamExecData.
- pg_column_size, pg_column_compression and pg_column_toast_chunk_id keep seeing the stored form only when applied to the column directly.
- New injection points in detoast_attr() and a new test module detoast_reuse.

**What the author claims.**
- Buffer reads drop about 50% to 75% on queries reading a column two to four times; instruction counts drop up to 55%.
- Worst measured overhead is +0.22% on JSONBench, and +0.5% for a plan-cached statement in a loop with nothing to share.
- A second-hand application workload ran 3.31x faster with 7.48x fewer buffer accesses.
- Planner decisions cannot affect results; a wrong decision costs only a detoast.
- No catalog version bump is needed, because the new fields live only on plan nodes.

**Thread state.** Denis Smirnov tested v2 and asks for three changes: slice-reading functions must not force a full copy, pg_column_size in a LATERAL FROM call returns a wrong value, and extensions refilling a virtual slot in place get stale copies. The author has not replied, and CFbot reports the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. In-tree slots promise copy resets that extensions refilling slots in place break

fixable · level 2, rubric item 7 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

An extension's custom scan uses the in-tree virtual slot ops and rewrites tts_values in place for each row, as was legal before. The four in-tree slot implementations declare resets_detoasted, so later rows read the first row's detoasted copy. Users get wrong query results in release builds and see no error.

Message #3 shows this case: 529 rows instead of 500 from a custom scan in a release build, and an assert failure in an assert build. The slot's promise is made by core, but extension code that never opted in decides whether it holds, and no current API forbids in-place refills. The reviewer's fix keeps tts_detoast_src in all builds and compares one pointer. The v1-0001 commit message says a matching address proves nothing because addresses are reused, so a buffer reused at the same address would still yield a stale copy.

Thread: [#1 (Johannes Edmeier, 2026-09-30)](https://www.postgresql.org/message-id/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), [#3 (Denis Smirnov, 2026-11-08)](https://www.postgresql.org/message-id/71CC110B-6BDB-4CF5-B0D4-13378FCB8DFA%40gmail.com)

**Question:** What check, valid in release builds, makes a stale copy impossible when out-of-tree code refills an in-tree slot in place and reuses the same datum address?

Counter-check: held. Message #3 reports 529 rows instead of 500 in a release build from code that was legal before. The 0001 commit message itself says a matching address proves nothing, so the pointer compare suggested in message #3 is not yet shown to be enough.

### 2. Stored-form functions return different values depending on call position

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A DBA runs a monitoring query with `lateral pg_column_size(t.col)` or wraps pg_column_compression in an extension function. After upgrade, with the default settings, it reports 320004 instead of 3679 and NULL compression. Nothing in the plan shows the change, so the DBA finds out only by comparing with a direct call.

Message #3 shows the LATERAL FROM case returning the decompressed size and a NULL compression method on v2 defaults. The config.sgml hunk documents that a function receiving the column as an argument now sees the fetched value. So whether a function sees the stored form depends on syntax, which no existing function contract has had. Open question 1 in message #1 says extensions cannot declare such functions, and the pg_proc flag or support request is not written.

Thread: [#1 (Johannes Edmeier, 2026-09-30)](https://www.postgresql.org/message-id/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), [#3 (Denis Smirnov, 2026-11-08)](https://www.postgresql.org/message-id/71CC110B-6BDB-4CF5-B0D4-13378FCB8DFA%40gmail.com)

**Question:** Will one check at every place that compiles function arguments, plus a declarable property for extension functions, be part of the series before it is committed?

Counter-check: held. Message #3 reproduces the wrong pg_column_size and pg_column_compression results for a LATERAL call on default settings, and the author has not replied. Message #1 confirms that extensions cannot declare a stored-form function.

### 3. Hardcoded function lists turn omissions into wrong answers and slower plans

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API, planner behaviour change · `maintenance-fit-1`

A later release adds a function that reports the stored form, or one that reads only a slice, and no one adds it to ExecFuncReadsStoredForm() or ExecFuncReadsSliceOrSize(). The first returns a wrong value with detoast_reuse on. The second reads whole values where it read a few chunks. Users find out from wrong numbers or slower queries, and no test fails.

Message #1 admits that a function missing from the stored-form list silently returns a wrong answer, and that extensions cannot declare one. Message #3 shows the check is already bypassed for a LATERAL call, where pg_column_size returns 320004 instead of 3679. It also shows that slice readers such as "=" on text read 1843 buffers instead of 3. A second copy of the argument-position list in the planner is kept in step only by a comment, so each new expression node becomes a chore that reviewers are likely to miss.

Thread: [#1 (Johannes Edmeier, 2026-09-30)](https://www.postgresql.org/message-id/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), [#3 (Denis Smirnov, 2026-11-08)](https://www.postgresql.org/message-id/71CC110B-6BDB-4CF5-B0D4-13378FCB8DFA%40gmail.com)

**Question:** Can the stored-form and slice rules live in pg_proc or a support request, checked in one place, before this goes in?

Counter-check: held. Message #1 states that a function missing from the stored-form list silently returns a wrong answer. Message #3 measures 1843 buffer hits instead of 3 for slice readers, so a miss in the slice list costs more than a lost optimization.

### 4. Developer option defaults on and is read only at plan time

fixable · level 2, rubric item 10 · new GUC · `standards-compat-3`

A user hits a wrong pg_column_size result or a slice-read regression and sets detoast_reuse = off as the documented escape. Prepared statements and cached PL/pgSQL plans keep the old plans and the old behaviour. The user concludes the setting does nothing.

Developer options are normally hidden debugging switches whose default is the plain behaviour. This one defaults on, changes what some functions return (message #3), and is the only user control. The config.sgml hunk says it applies only to plans made after the change. Message #1 does not say why a developer option is the right category for a default-on feature.

Thread: [#1 (Johannes Edmeier, 2026-09-30)](https://www.postgresql.org/message-id/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), [#3 (Denis Smirnov, 2026-11-08)](https://www.postgresql.org/message-id/71CC110B-6BDB-4CF5-B0D4-13378FCB8DFA%40gmail.com)

**Question:** Why is a default-on developer option the right category, and how does a user with cached plans turn the behaviour off?

Counter-check: held. The proposal does not say why a default-on feature is filed as a developer option. The plan-time part should carry less weight, because the hunk documents it and enable_* settings behave the same way with cached plans.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | Open question 1 proposes a pg_proc flag or support request for stored-form functions; the series does not include it yet. |
| ABI / extension API | **yes** | New fields on TupleTableSlot, TupleTableSlotOps, Plan, PlannedStmt and ParamExecData, plus a reset promise that in-tree slot types make for extensions. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Stored-form functions return different values depending on call position |
| new GUC | **yes** | Adds detoast_reuse, a boolean developer option that defaults to on and is read at plan time. |
| planner behaviour change | **yes** | set_plan_detoast_reuse() in setrefs.c marks columns per plan node; the plan shape does not change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | RLS and security-barrier quals are argument positions, so a stale copy after an in-place slot refill could reach them. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Three hardcoded lists, a planner copy of the argument-position list, and new expression steps in the interpreter and JIT. |
| performance on the hot path | **yes** | A test runs at every argument position of every compiled expression; message #1 measures +0.22% to +0.5%. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What check, valid in release builds, prevents a stale copy when extension code refills an in-tree slot in place and the datum address is reused?
2. Will one stored-form check cover every place that compiles function arguments, including calls in FROM, and can extension functions declare this property before commit?
3. Will functions that read only a slice or the size get the stored datum unless a copy already exists, as message #3 asks?
4. Why is detoast_reuse a developer option when it defaults to on and is the only way for a user to turn the behaviour off?
5. How much more memory does a plan hold when several slots keep copies of wide values until the slot is refilled?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows expression evaluation, the JIT expression code and the TupleTableSlot API. The visible thread has the author and one reviewer.
- **Liability:** The owner signs for a correctness rule that depends on every slot refill path, every argument position and three function lists staying complete, in core and in extensions, with no end date.
- **Author follow-up:** The author rebased within two weeks and says the design and measurements are his to answer for. Message #3 has no reply yet and CFbot reports that a rebase is needed.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Users with wide jsonb or text columns read several times per row gain 50% to 75% fewer buffer reads. The four-patch series, about 165 KiB, touches the executor, JIT and planner, and the wrong results in message #3 need a design answer first.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` In-place refill of TTSOpsVirtual slots now gives wrong results. Refuted (not specific): Duplicate of replication-ondisk-1, which covers the same in-place refill failure and also asks whether the proposed pointer compare is sufficient. The typlen point from message #3 is a cheap check that belongs with that objection.
- `security-1` Stale detoasted copy can let a row pass an RLS or barrier qual. Refuted (not specific): Duplicate of replication-ondisk-1. The RLS case is one consequence of the same stale copy after an in-place refill, with no separate mechanism, and the same fix closes it.
- `maintenance-fit-2` New slot contract gives wrong results in release builds when broken. Refuted (not specific): Duplicate of replication-ondisk-1, which describes the same custom scan failure and the same assert-only check. Its question about the cost of an all-build check fits under that objection.
- `maintenance-fit-3` Per-execution cost paid by every query, including those with nothing to share. Refuted (already handled): Message #1 measures the cost on the path in question (+0.22% worst case, +0.5% for a short cached statement) and says a node that marks nothing settles each position in three null tests. The author also built the alternative design and withdrew it at about 2%.

### Review angles

- prior-attempts: 0 objection(s). The only listed precedent, [P10], covers numeric packed-datum extraction and records no reason, so it does not match. In message #1 the author reports that "Shared detoast Datum" stalled because a query-lifespan cache was "likely never going to be committed" and a planner-level decision was preferred; this series makes that decision at plan time and keeps no cache, so the reported reason does not carry over.
- standards-compat: 3 objection(s). Three objections raised. pg_dump, catalog and SQL syntax are not touched, and the author's view that no catalog version bump is needed fits plan-only fields.
- replication-ondisk: 1 objection(s). There is no on-disk, WAL, catalog or wire-protocol change. The new Plan and PlannedStmt fields reach only parallel workers of the same server version, so no catversion bump is needed. Without the code, I could not check that parallel workers never follow the ParamExecData slot back-reference across processes.
- security: 1 objection(s). Apart from the stale-copy case, the design does not change who can read which values. The copy holds the same value that functions already receive, and no new privilege is involved. Memory held longer in slots is bounded by values the user can already detoast.
- maintenance-fit: 3 objection(s). Not applicable; three objections raised.
- already-exists: 0 objection(s). No existing feature, GUC or extension gives one detoast per row for repeated references; ALTER COLUMN SET STORAGE and materialized CTEs keep or store the stored datum. Query rewrites that force a copy are uncertain and need application changes, and expression evaluation and slots have no hook that an extension could use for this.
- demand: 0 objection(s). The material shows lasting, independent demand. Message #1 cites Tom Lane's 2008 and 2016 work on this, Andy Fan's ten-revision patch, and Oleg Bartunov's "curse of TOAST" tests, plus a second-hand production workload that ran 3.31x faster. The cost on queries with nothing to share is measured at +0.22% to +0.5%, so the trade-off is on the table; whether it is acceptable belongs to the performance angle.

### Precedents retrieved

- P1: [Fix slotsync worker busy loop causing repeated log messages](https://commitfest.postgresql.org/patch/6546/) (cf, Withdrawn, 2026-05-01) - judged not relevant; reason summarised from its thread
- P2: [Support for 8-byte TOAST values (aka the TOAST infinite loop problem)](https://commitfest.postgresql.org/patch/5830/) (cf, Returned with feedback, 2026-02-10) - judged not relevant; reason summarised from its thread
- P3: [Refuse setting toast.* reloptions when TOAST relation does not exist](https://commitfest.postgresql.org/patch/1486/) (cf, Returned with feedback, 2018-04-10) - judged not relevant; reason summarised from its thread
- P4: [Pluggable toaster](https://commitfest.postgresql.org/patch/3490/) (cf, Rejected, 2023-04-03) - judged not relevant; reason summarised from its thread
- P5: [Revert "isolationtester: don't repeat the is-it-waiting query when retrying a step."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dca369320f6023b55feb49f281d394181fc57903) (revert, Reverted, 2016-02-12) - judged not relevant
- P6: [pg_stat_toast](https://commitfest.postgresql.org/patch/3457/) (cf, Withdrawn, 2022-07-01) - judged not relevant; reason summarised from its thread
- P7: [Toast compression method options](https://commitfest.postgresql.org/patch/3102/) (cf, Withdrawn, 2021-10-01) - judged not relevant; reason summarised from its thread
- P8: [format_datum debugging function](https://commitfest.postgresql.org/patch/5165/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [Revert "Optimize pg_visibility with read streams."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ddfc556a644404a8942e77651f75f09aa5188782) (revert, Reverted, 2024-09-04) - judged not relevant
- P10: [Optimize numeric comparison and aggregation via packed-datum extraction](https://commitfest.postgresql.org/patch/6651/) (cf, Rejected, 2026-05-10) - used in the review; reason summarised from its thread
- P11: [Reuse data from readRecordBuf in XLogDecodeNextRecord](https://commitfest.postgresql.org/patch/6343/) (cf, Withdrawn, 2026-02-09) - judged not relevant
- P12: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - judged not relevant
- P13: [JIT expression evaluation improvements](https://commitfest.postgresql.org/patch/2501/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P14: [Revert "Fix corner case with PGP decompression in pgcrypto"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=38f60f174e3279069b5547b5f4015eb4a8492037) (revert, Reverted, 2020-07-22) - judged not relevant
- P15: [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (cf, Rejected, 2019-03-19) - judged not relevant
- P16: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant
- P17: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P18: [Revert "Convert *GetDatum() and DatumGet*() macros to inline functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e8d78581bb864369035c736a82ecfb57482a1a09) (revert, Reverted, 2022-09-12) - judged not relevant
- P19: [Document the limitations of row-level security](https://commitfest.postgresql.org/patch/2027/) (cf, Returned with feedback, 2019-04-04) - judged not relevant
- P20: [Add a feature that enables row-wise comparison partition pruning](https://commitfest.postgresql.org/patch/2654/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P21: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P22: [Revert MAINTAIN privilege and pg_maintain predefined role.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=151c22deee66a3390ca9a1c3675e29de54ae73fc) (revert, Reverted, 2023-07-07) - judged not relevant
- P23: [Revert non-text output formats for pg_dumpall](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7ca548f23a60887541493b2da47f2b2720c35ee3) (revert, Reverted, 2026-06-18) - judged not relevant
- P24: [minimizing pg_stat_statements.track='none' performance overhead](https://commitfest.postgresql.org/patch/2092/) (cf, Withdrawn, 2019-04-15) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 165 s; cost reported by the CLI: $2.88
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 88 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:53:53.414Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
