# Design pre-review: Fix cumulative WAL statistics double-counting from parallel workers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7553](https://commitfest.postgresql.org/patch/7553/) (Needs review); authors: Lukas Fittl (lfittl)
- Thread: [Fix cumulative WAL statistics double-counting from parallel workers](https://www.postgresql.org/message-id/flat/CAP53PkzL-jF-Pm%2Bev2vBDvBYqDFNhcN4bpjGbGGbBfM_gtSdag%40mail.gmail.com), started 2026-10-29
- Messages read: 24 of 24
- Patch prose read: v1-0001-pgstat-Don-t-double-count-parallel-worker-activit.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is real and reproduced in message #1 and message #22, and no objection survived at level 1a or 1b. The surviving objections are fixable design points at level 2: maintenance-fit-1 (nothing enforces the worker subtraction), maintenance-fit-2 (whether master takes the same exported symbols as back branches) and standards-compat-1 (the per-backend attribution change in minor releases). Michael Paquier's objection for HEAD is still open in message #21, so a committer who knows the area must decide the shape per branch.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 24 messages and no code were read, and v2 is not posted. The review could not check whether the new globals are static, how the pgstat_backend.c hunk behaves, or whether every InstrAccumParallelQuery caller is covered.

## The proposal, restated

**Problem.** Since Postgres 14, pg_stat_wal counts WAL written by parallel workers twice, once by the worker and once by the leader through InstrAccumParallelQuery. The author's parallel VACUUM reproducer reports 131% of the WAL actually written. Parallel vacuum, pruning and hint bits set in parallel scans are affected, and so is pg_stat_get_backend_wal().

**Proposed design.** The leader adds worker counters both to pgWalUsage/pgBufferUsage and to new globals pgWalUsageFromWorkers/pgBufferUsageFromWorkers. Before reporting to cumulative stats, the leader subtracts the worker part using a helper, ComputeOwnWalUsage, and workers keep reporting their own activity. EXPLAIN and pg_stat_statements keep reading the combined pgWalUsage.

**User-visible surface.**
- pg_stat_wal stops counting parallel worker WAL twice.
- pg_stat_get_backend_wal() for a leader stops including its workers' WAL, about 31MB less in one reviewer's test.
- C API: new exported globals pgWalUsageFromWorkers and pgBufferUsageFromWorkers in instrument.h; a reviewer asked to make them static.
- C API: new function ComputeOwnWalUsage().

**What the author claims.**
- The bug exists since pg_stat_wal arrived in Postgres 14 and has a reproducer.
- The fix has no performance impact, because the subtraction runs at stats flush and not in a hot path.
- The fix is simple enough to backpatch to all branches.
- The same approach can later replace pgStatBlock(Read|Write)Time with diffs of pgBufferUsage for pg_stat_database.

**Thread state.** Bertrand Drouvot and Andres Freund accept approach (1) for back branches, but Michael Paquier objects that pgWalUsage should not contain worker data and prefers a leader-only counter, at least on HEAD. Lukas plans a v2 with Bertrand's comments and a commit message saying the pg_stat_get_backend_wal() change fixes attribution.

## 2. Prior attempts

### Parallel leader process info in EXPLAIN output (related)

- **Link:** [https://commitfest.postgresql.org/patch/2315/](https://commitfest.postgresql.org/patch/2315/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CA+hUKG+Z22=vkVkXtKRznzRdtj=MtygfZMYUJs8j7ObjzkG1Lw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKG+Z22=vkVkXtKRznzRdtj=MtygfZMYUJs8j7ObjzkG1Lw@mail.gmail.com)
- **Date:** decided 2020-03-17; started 2019-10-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane objected that 0001 adds another field to every PlanState and builds on ExecParallelRetrieveInstrumentation aggregating data early, so it should instead return raw leader and per-worker data for explain.c to aggregate. He also said it was unclear how the leader output maps to non-text formats without breaking which fields are present. The author dropped 0002 because of other refactoring work, agreed 0001 needed a rewrite, and marked the entry returned with feedback.
- **Reason (quote):** "I think this is doubling down on a fundamentally wrong decision to have ExecParallelRetrieveInstrumentation do some aggregation immediately." ([message](https://www.postgresql.org/message-id/18323.1580078950%40sss.pgh.pa.us))
- **What is different this time:** P7 added leader-only EXPLAIN output and a PlanState field, not a cumulative stats fix. It shares only the background that worker data is merged into leader state early.

### Revert "Fix accumulation of parallel worker instrumentation." (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAA4eK1LW2aFKzY3=vwvc=t-juzPPVWP2uT1bpx_MeyEqnM+p8g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAA4eK1LW2aFKzY3=vwvc=t-juzPPVWP2uT1bpx_MeyEqnM+p8g@mail.gmail.com), [https://www.postgresql.org/message-id/flat/20171127175631.GA405@depesz.com](https://www.postgresql.org/message-id/flat/20171127175631.GA405@depesz.com)
- **Date:** decided 2017-12-13; started 2017-12-05
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 2c09a5c12a66087218c7f8cba269cd3de51b9b82. Per further discussion, that doesn't seem to be the best possible fix.
- **Reason (quote):** "Per further discussion, that doesn't seem to be the best possible fix."
- **What is different this time:** P3 reverted a fix to how parallel worker executor instrumentation was accumulated, not WAL cumulative stats. The row gives no reason that applies to this patch.


## 3. Objections a reviewer is likely to raise

### 1. Unenforced rule: every pgWalUsage diff for cumulative stats must subtract workers

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch diffs pgBufferUsage to feed pg_stat_database, or adds another WAL-based stats kind, and uses the global directly instead of a ComputeOwn*Usage helper. Parallel worker activity is counted twice again. Nothing fails at build or test time, and users find out from inflated monitoring numbers years later, as happened with pg_stat_wal since version 14.

Message #3 says nothing guarantees the invariant, and the planned guard is a comment in instrument.h (messages #3, #7). The patch adds no regression test, and reviewers chose not to ask for one (message #7). The author already plans the next user of this pattern, buffer timing for pg_stat_database (message #1), so the same trap gets a second consumer soon.

Thread: [#1 (Lukas Fittl, 2026-10-29)](https://www.postgresql.org/message-id/CAP53PkzL-jF-Pm%2Bev2vBDvBYqDFNhcN4bpjGbGGbBfM_gtSdag%40mail.gmail.com), [#3 (Lukas Fittl, 2026-10-30)](https://www.postgresql.org/message-id/CAP53PkxYDUEa_cjRT9HWfzFrvQqYMfLngaEh7kWYkwLABTVfPw%40mail.gmail.com), [#7 (Bertrand Drouvot, 2026-11-02)](https://www.postgresql.org/message-id/auhKz%2FRirAnqHl3f%40bdtpg)

**Question:** What makes a future patch that diffs pgWalUsage or pgBufferUsage for cumulative stats, without the worker subtraction, fail in a test or at review instead of in the field?

Counter-check: held. The hazard predates the patch, but message #3 confirms that nothing enforces the rule and the only planned guard is a comment. Message #1 plans a second user of the pattern for pg_stat_database, so the scenario is concrete.

### 2. Master gets exported API the author calls an interim workaround

fixable · level 2, rubric item 6 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

Master ships ComputeOwnWalUsage() and pgWalUsageFromWorkers/pgBufferUsageFromWorkers, and an extension starts using them. Stack-based instrumentation then removes or redefines them one release later, and the extension breaks. Meanwhile pgBufferUsageFromWorkers is accumulated on every parallel query but no stats consumer reads it yet.

The author calls approach (1) "ugly" and a step before a more elegant solution in the stack-based thread (message #20). Michael objects to it on HEAD as an abuse of pgWalUsage (messages #13, #21), and Andres says the stack-based work removes the main problem with approach (2) (message #17). Back branches may need approach (1), but master could take a leader-only design or a fix with no new exported symbols. [P3] reverted a parallel instrumentation fix that "doesn't seem to be the best possible fix".

**Precedents.**
- [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (Reverted, 2017-12-13); thread: [#1](https://www.postgresql.org/message-id/flat/CAA4eK1LW2aFKzY3=vwvc=t-juzPPVWP2uT1bpx_MeyEqnM+p8g@mail.gmail.com), [#2](https://www.postgresql.org/message-id/flat/20171127175631.GA405@depesz.com)

Thread: [#13 (Michael Paquier, 2026-11-04)](https://www.postgresql.org/message-id/aurfeqkq2dgqvdZK%40paquier.xyz), [#17 (Andres Freund, 2026-11-04)](https://www.postgresql.org/message-id/cz3lbv54b477owlxu73vtyfsoomg3snvpgdb5jtiqogffeznpa%40x2ecg363asoc), [#20 (Lukas Fittl, 2026-11-05)](https://www.postgresql.org/message-id/CAP53Pkz%3DvE7X9BkSbxGJExERwHLqRHTsqgZo0hD_uQdO3bEFKA%40mail.gmail.com), [#21 (Michael Paquier, 2026-11-05)](https://www.postgresql.org/message-id/auwVNOik4oOhAIjv%40paquier.xyz)

**Question:** Why should master take the same exported globals and helper as the back branches, instead of a static-only fix or the leader-only design that the stack-based work allows?

Counter-check: held. Michael's objection to approach (1) on HEAD is still open in message #21, so the question stands. [P3] gives no usable reason and must not be used as support, and an extension break at a major release is a weak scenario.

### 3. Backpatch removes parallel worker WAL from per-backend stats with no replacement

fixable · level 2, rubric item 6 · backpatchability · `standards-compat-1`

A monitoring tool samples pg_stat_get_backend_wal() for a long-lived session that runs parallel VACUUM. After a minor release, that leader's wal_bytes drops by about a third in message #22's test. The worker WAL now shows up in no per-backend entry at all, because each worker's entry is dropped when it exits.

On stable branches, pg_stat_get_backend_wal() counts worker WAL once, in the leader's entry (message #22). The pgstat_backend.c hunk is an attribution change, not a double-counting fix, and message #23 and message #24 agree on that. It matches the documentation, but it moves WAL out of the only per-backend view that showed it, and message #24 leaves a per-leader worker view to future work. Fixing pg_stat_wal does not need this hunk, so a committer would likely ask why a reporting change goes into minor releases with the bug fix.

Thread: [#22 (shihao zhong, 2026-11-10)](https://www.postgresql.org/message-id/CAGRkXqQskUo2s539Wk%2BiYav1mp%3DcDcMPT%2BofVXHFRE4HTRxfBQ%40mail.gmail.com), [#23 (Bertrand Drouvot, 2026-11-10)](https://www.postgresql.org/message-id/avKaMOu4yUik38O2%40bdtpg), [#24 (Lukas Fittl, 2026-11-10)](https://www.postgresql.org/message-id/CAP53Pkxo%3DTT3iOXxinGGrQEozt1Ai%2BNjdAy7uWrPkjCJK6LzRg%40mail.gmail.com)

**Question:** Why should the pg_stat_get_backend_wal() attribution change go to stable branches rather than only to HEAD, where a per-leader worker view could come with it?

Counter-check: held. Messages #23 and #24 justify the change by the documentation, but they do not discuss stable branches. After a minor release, worker WAL appears in no per-backend entry, and the thread leaves the replacement to future work.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | instrument.h gains exported globals pgWalUsageFromWorkers and pgBufferUsageFromWorkers and the function ComputeOwnWalUsage(), also on stable branches. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The author proposes a backpatch to all branches, which changes pg_stat_wal and pg_stat_get_backend_wal() values in minor releases. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every future cumulative-stats user of pgWalUsage or pgBufferUsage must subtract the worker part, and only a comment guards that rule. |
| performance on the hot path | **yes** | InstrAccumParallelQuery does extra additions per worker; the subtraction runs only at stats flush. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What makes a future patch that diffs pgWalUsage or pgBufferUsage for cumulative stats, without the worker subtraction, fail in a test or at review?
2. Why should master take the same exported globals and helper as the back branches, instead of a static-only fix or the leader-only design?
3. Why should the pg_stat_get_backend_wal() attribution change go to stable branches, where no per-backend entry shows worker WAL afterwards?
4. Does the backpatch add exported symbols to stable branches, or will v2 make pgWalUsageFromWorkers and pgBufferUsageFromWorkers static?
5. Does any buffer-based cumulative statistic change with this patch, or is pgBufferUsageFromWorkers accumulated with no reader yet?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund and Michael Paquier take part in the thread but disagree on the shape for HEAD. The owner needs to know the cumulative stats code and parallel instrumentation.
- **Liability:** The owner signs for changed pg_stat_wal and pg_stat_get_backend_wal() numbers in minor releases on all branches since 14, and for new symbols in instrument.h that the stack-based instrumentation work may later change.
- **Author follow-up:** The author answers reviews within a day, plans a v2 with Bertrand's comments, and works on the related stack-based instrumentation thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 10 KiB in four files and fixes wrong pg_stat_wal totals for anyone running parallel vacuum or parallel scans. The open points are the shape on HEAD and the scope of the backpatch, which a v2 and the engaged committers can settle.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Patch adds compensating state instead of separating worker data on HEAD. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same unenforced invariant without relying on [P7]. [P7] concerned a new PlanState field and EXPLAIN output, and the HEAD question is covered by maintenance-fit-2.

### Review angles

- prior-attempts: 1 objection(s). For back branches no precedent argues against approach (1), and message #17 sees no alternative there. The one objection above concerns HEAD only.
- standards-compat: 1 objection(s). pg_stat_wal and pg_stat_statements semantics stay as users expect, and pgWalUsage keeps its meaning for extensions. From this angle, only the per-backend attribution change in back branches raises a concern.
- replication-ondisk: 0 objection(s). The fix changes only backend-local instrumentation counters and the values flushed to cumulative stats. It changes no on-disk format, stats file layout, WAL record, catalog, or wire protocol. The back-branch ABI risk is small: v1 adds two exported globals and keeps the meaning of pgWalUsage and the signature of InstrAccumParallelQuery. Bertrand suggests making the globals static, which removes even that risk.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes how a leader subtracts worker WAL counts before flushing to pg_stat_wal and pg_stat_get_backend_wal(), and it adds no new view, function, role check or data path, so no user can see or do anything new. Whether the new exported globals affect ABI belongs to other angles.
- maintenance-fit: 2 objection(s). The bug is real and the fix is cheap at flush time, so the objections are about long-run upkeep and the shape of master.
- already-exists: 0 objection(s). This is a bugfix for wrong numbers in pg_stat_wal and pg_stat_get_backend_wal(). No existing feature, setting or extension stops the double counting. Users can only compare pg_current_wal_lsn() deltas by hand, and that does not fix the cumulative view. A second mechanism (rubric item 12) is out of scope here because the patch fixes an existing mechanism, and the design dispute between the in-thread alternatives belongs to another angle.
- demand: 0 objection(s). This is a bug fix. pg_stat_wal reports about 131% of the WAL that parallel VACUUM writes, and both message #1 and message #22 reproduce it. Every user who reads pg_stat_wal while parallel vacuum or parallel scans run sees the wrong totals, so the demand angle has nothing to object to. Design disputes, such as whether pgWalUsage should hold worker data (messages #6 and #13), belong to other angles.

### Precedents retrieved

- P1: [speeding up GIN build with parallel workers](https://commitfest.postgresql.org/patch/527/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P2: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - judged not relevant; reason summarised from its thread
- P3: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - used in the review
- P4: [Allow setting parallel_workers on partitioned tables](https://commitfest.postgresql.org/patch/2987/) (cf, Returned with feedback, 2021-09-04) - judged not relevant; reason summarised from its thread
- P5: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P6: [Add a max_parallel_workers GUC](https://commitfest.postgresql.org/patch/703/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P7: [Parallel leader process info in EXPLAIN output](https://commitfest.postgresql.org/patch/2315/) (cf, Returned with feedback, 2020-03-17) - used in the review; reason summarised from its thread
- P8: [system load consideration hook for parallel workers](https://commitfest.postgresql.org/patch/725/) (cf, Rejected, 2016-09-22) - judged not relevant; reason summarised from its thread
- P9: [\[PG19\]pg_dump --statistics restores extended statistics by attribute number](https://commitfest.postgresql.org/patch/7446/) (cf, Withdrawn, 2026-10-19) - judged not relevant; reason summarised from its thread
- P10: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - used in the review; reason summarised from its thread
- P11: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant
- P12: [Add last failed connection error message to pg_stat_wal_receiver](https://commitfest.postgresql.org/patch/3666/) (cf, Withdrawn, 2022-10-11) - judged not relevant
- P13: [temporary file usage report with extended protocol and unnamed portals](https://commitfest.postgresql.org/patch/5711/) (cf, Withdrawn, 2025-08-21) - judged not relevant
- P14: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P15: [Vacuum: allow usage of more than 1GB of work mem](https://commitfest.postgresql.org/patch/844/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P16: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P17: [Undo worker and transaction rollback](https://commitfest.postgresql.org/patch/1828/) (cf, Withdrawn, 2019-09-09) - judged not relevant
- P18: [Add parallel counters to pg_stat_statements](https://commitfest.postgresql.org/patch/3787/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P19: [Improve logicalrep_worker_launch() logic](https://commitfest.postgresql.org/patch/5734/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P20: [Add accumulated statistics for wait event](https://commitfest.postgresql.org/patch/2332/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P21: [Add tracking of backend memory allocated to pg_stat_activity](https://commitfest.postgresql.org/patch/3865/) (cf, Withdrawn, 2023-01-05) - judged not relevant
- P22: [Revert: Get rid of WALBufMappingLock](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3fb58625d18fd226cb929c9700d0db72ac92c075) (revert, Reverted, 2025-02-17) - judged not relevant
- P23: [multivariate statistics](https://commitfest.postgresql.org/patch/120/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P24: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 99 s; cost reported by the CLI: $2.18
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 69 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:09:23.134Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
