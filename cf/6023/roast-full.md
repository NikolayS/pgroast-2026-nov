# Design pre-review: Stack-based tracking of per-node WAL/buffer usage

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6023](https://commitfest.postgresql.org/patch/6023/) (Needs review); authors: Lukas Fittl (lfittl)
- Thread: [Stack-based tracking of per-node WAL/buffer usage](https://www.postgresql.org/message-id/flat/CAP53PkzdBK8VJ1fS4AZ481LgMN8f9mJiC39ZRHqkFUSYq6KWmg%40mail.gmail.com), started 2025-08-31
- Messages read: 49 of 49
- Patch prose read: v18-0001-instrumentation-Use-Instrumentation-instead-of-m.patch, v18-0002-instrumentation-Replace-direct-changes-of-pgBuff.patch, v18-0005-instrumentation-Retire-pgBufferUsage-in-favor-of.patch, v18-0003-instrumentation-Add-additional-regression-tests-.patch, v18-0004-instrumentation-Reduce-overhead-of-measuring-per.patch, v18-0007-Index-scans-Show-table-buffer-accesses-separatel.patch, v18-0006-instrumentation-Specialize-ExecProcNodeInstr-for.patch, v18-0008-Add-test_session_buffer_usage-test-module.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at level 1a or 1b: the problem is stated, measured, and wanted by several reviewers. The surviving objections are fixable design points at level 2. replication-ondisk-1 asks for a backstop when an extension calls InstrStart without abort handling, and maintenance-fit-1 asks for a committed test that catches lost abort-time totals. The abort model needs sign-off from a committer who knows the executor and resource owners.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the abort handling, the resource owner release order and the parallel worker accumulation could not be checked. The README hunk was truncated, and only 3 of 49 messages were shown, so later answers in the thread may already cover some questions.

## The proposal, restated

**Problem.** EXPLAIN (ANALYZE, BUFFERS) is slow because InstrStopNode diffs the global pgBufferUsage/pgWalUsage counters with BufferUsageAccumDiff for every node cycle. The author also wants a base for later work, such as estimated distinct buffer hits and separate index and table buffer counts in index scans.

**Proposed design.** Buffer and WAL counter updates write into the current entry of a per-backend instrumentation stack; each node pushes a zeroed entry on start, pops it on stop, and adds it to its parent at finalize. Aborts are handled by QueryInstrumentation through resource owners and a dedicated memory context, or by PG_TRY/PG_FINALLY. Later patches retire pgBufferUsage, specialize ExecProcNodeInstr per option set, and show table buffers for index scans.

**User-visible surface.**
- EXPLAIN ANALYZE shows a new "Table Buffers" line for Index Scan and Index Only Scan, counted in "Buffers" as well.
- Structured EXPLAIN output gets "Table Buffers" and "Table I/O Timings" groups.
- C API: global pgBufferUsage and BufferUsageAccumDiff are removed; extensions must use InstrStart/InstrStop instead.
- C API: QueryDesc totaltime is renamed and now allocated by ExecutorStart, with an options field set by extensions (already pushed).
- C API: new QueryInstrumentation, NodeInstrumentation, TriggerInstrumentation structs and INSTR_BUFUSAGE_*/INSTR_WALUSAGE_* macros.

**What the author claims.**
- The stack cuts EXPLAIN (ANALYZE, BUFFERS ON, TIMING OFF) overhead on a large COUNT(*) from about 35% to about 20% of runtime.
- Specialized ExecProcNodeInstr roughly halves TIMING OFF, BUFFERS OFF overhead, from about 13% to about 6%.
- Top-level totals stay correct after aborts, including caught errors in functions and triggers.
- Separate table buffer counts help find whether index or table access makes a query slow, a benefit Andres Freund also named.

**Thread state.** Andres Freund pushed several preparatory patches for 19 and deferred the stack patch to 20, asking for senior architectural review; Heikki Linnakangas still finds the stack and tree model confusing. v18 (October 2026) reworks the series and has no visible reply yet.

## 2. Prior attempts

### Reduce timing overhead of EXPLAIN ANALYZE using rdtsc (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3751/](https://commitfest.postgresql.org/patch/3751/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20200612232810.f46nbqkdhbutzqdg@alap3.anarazel.de](https://www.postgresql.org/message-id/flat/20200612232810.f46nbqkdhbutzqdg@alap3.anarazel.de)
- **Date:** decided 2024-02-01; started 2020-06-12
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The thread stopped after the v8 patch of February 2023, and nobody continued the work for about a year. vignesh C closed the entry as Returned with feedback because neither the author nor anyone else took it forward. Earlier, Andres Freund doubted the patch was ready on reliability and code complexity, citing the Linux-only restriction, but the entry closed from inactivity.
- **Reason (quote):** "Since the author or no one else showed interest in taking it forward and the patch had no activity for more than 1 year, I have changed the status to RWF." ([message](https://www.postgresql.org/message-id/CALDaNm2awLBioTdQH%3Daj8Yp7%2BMwR%3DvzpzHrKtgjAnUN1esLmvg%40mail.gmail.com))
- **What is different this time:** P1 cut the timing cost of EXPLAIN ANALYZE by reading the TSC. This proposal cuts the buffer and WAL counting cost by replacing the BufferUsageAccumDiff diff with a stack. P1 closed from inactivity, and this thread is active at v18.

### Sampling-based timing for EXPLAIN ANALYZE (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4100/](https://commitfest.postgresql.org/patch/4100/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAP53PkxXMk0j-+0=YwRti2pFR5UB2Gu4v2Oyk8hhZS0DRART6g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAP53PkxXMk0j-+0=YwRti2pFR5UB2Gu4v2Oyk8hhZS0DRART6g@mail.gmail.com)
- **Date:** decided 2023-07-03; started 2023-01-02
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P22 tried to lower EXPLAIN ANALYZE overhead by sampling timings. This proposal keeps exact per-node counts and changes how buffer and WAL usage reach each node. No reason is recorded for P22.


## 3. Objections a reviewer is likely to raise

### 1. Extension callers of the plain Instrumentation API can leave a dangling stack

fixable · level 2, rubric item 7 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

An extension that read pgBufferUsage switches to InstrStart/InstrStop as the 0005 commit message advises, but adds no PG_TRY/PG_FINALLY. An error between the two calls leaves instr_stack.current pointing at memory freed by the abort. Later buffer and WAL counter updates in the same backend write into that freed memory, and nothing reports it.

The 0005 commit message says plain Instrumentation "requires appropriate abort handling in all cases", and the 0004 message limits abort safety to QueryInstrumentation or explicit PG_TRY/PG_FINALLY. Message #26 confirms that without this reset the stack is corrupted after an abort. The material shows no backstop, such as a reset to instr_top or an assertion in AbortTransaction, for callers that get this wrong. Today an extension that diffs pgBufferUsage cannot corrupt memory this way, so the API change turns a counting mistake into a safety bug.

Thread: [#26 (Lukas Fittl, 2026-03-23)](https://www.postgresql.org/message-id/CAP53Pkx%2BHZ2OSGDHmNgSDisXdjGp4KFygaX%2BGaQcFEuXHRXg%3Dg%40mail.gmail.com), [#49 (Lukas Fittl, 2026-10-31)](https://www.postgresql.org/message-id/CAP53Pkwd6Gd4OL6cgh449wtdznWitz_V%3D-PRoswLHOzsMbcftA%40mail.gmail.com)

**Question:** What resets or checks instr_stack at transaction abort or error recovery when an extension calls InstrStart without PG_FINALLY or a resource owner?

Counter-check: held. The 0005 commit message sends extensions to InstrStart/InstrStop and leaves abort handling to the caller. The visible material shows no reset of the stack at transaction abort for a caller that uses neither a resource owner nor PG_FINALLY.

### 2. Lost abort-time totals are caught only by a test module not meant for commit

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch adds a new kind of instrumentation, for example per-trigger or per-receiver, and does not register it as a child of the QueryInstrumentation. When that code errors out, its buffer activity is not added to the parent. Regression tests pass, and session or pg_stat_statements totals undercount without any visible error.

The trigger case in messages #21 and #23 and the EXPLAIN (SERIALIZE) case in message #15 were both missing from abort accounting. Both were found only through test_session_buffer_usage, which the author calls test-only and not meant for commit (messages #11, #19, #32). Message #26 says a stack that is not reset does fail regression tests, but lost accumulation does not. Once pgBufferUsage is retired in 0005, these abort totals are the only session-level number, so a silent loss is harder to spot than it is today.

Thread: [#11 (Lukas Fittl, 2026-02-24)](https://www.postgresql.org/message-id/CAP53PkzGbyeJMLDAcvMRgzXPXYsYXZr3SBg0UwhfkYjqu8oK_g%40mail.gmail.com), [#15 (Lukas Fittl, 2026-03-09)](https://www.postgresql.org/message-id/CAP53PkxAS-rpp0kcOjg--q5NVNnRC%2B0kh1%2BgB8gnRNqacTqF5A%40mail.gmail.com), [#21 (Zsolt Parragi, 2026-03-18)](https://www.postgresql.org/message-id/CAN4CZFMk10YCixVTnt%2BuBaUH_BH1R%3DEr-PnPs2YDBhFjuE3K-Q%40mail.gmail.com), [#23 (Lukas Fittl, 2026-03-19)](https://www.postgresql.org/message-id/CAP53PkyEPa%3DaFZrd0crLgRJm1KGS0ykJ%2B1i2CwDfDNFKpCc%2BXQ%40mail.gmail.com), [#26 (Lukas Fittl, 2026-03-23)](https://www.postgresql.org/message-id/CAP53Pkx%2BHZ2OSGDHmNgSDisXdjGp4KFygaX%2BGaQcFEuXHRXg%3Dg%40mail.gmail.com), [#32 (Lukas Fittl, 2026-03-27)](https://www.postgresql.org/message-id/CAP53Pky2%3D31B1AS9vg%3DCa9308_hb0H4g968cSKxiFgT0moJfYg%40mail.gmail.com)

**Question:** Which committed test fails when a new instrumentation user skips registering with its QueryInstrumentation and then errors out?

Counter-check: held. The 0003 tests cover abort cases visible in EXPLAIN, but the lost trigger totals in message #23 showed up only through the test module. Messages #11 and #32 leave open whether that module is committed.

### 3. Two counting models remain, buffers on the stack and WAL on a global

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), performance on the hot path · `maintenance-fit-2`

A developer adds a new counter, or moves pgstat WAL accounting, and has to decide between the stack and pgWalUsage. INSTR_WALUSAGE_ADD writes both. If a fix updates only one of the two paths, pg_stat_wal and EXPLAIN WAL numbers drift apart, and nobody notices until a user compares them.

The 0005 commit message keeps pgWalUsage because pgstat and heap_page_prune_and_freeze depend on it, and message #3 calls removing it "interesting". The material gives no plan or release for retiring it. Every WAL record now updates two places, and the thread shows no measurements with WAL ON. Message #49 also cites an existing pg_stat_wal bug around these globals.

Thread: [#3 (Lukas Fittl, 2025-09-09)](https://www.postgresql.org/message-id/CAP53Pkw1RpyAVsvJwUOMgGnVoQUibMk%2BEB%2Bw-8WqY8kzsC4WjA%40mail.gmail.com), [#49 (Lukas Fittl, 2026-10-31)](https://www.postgresql.org/message-id/CAP53Pkwd6Gd4OL6cgh449wtdznWitz_V%3D-PRoswLHOzsMbcftA%40mail.gmail.com)

**Question:** What is the plan and target release for retiring pgWalUsage, and what does WAL ON overhead measure today?

Counter-check: held. The drift scenario is weak because INSTR_WALUSAGE_ADD updates both places in one macro. But the material gives no plan for retiring pgWalUsage and no WAL ON numbers for the stack patch, so those two requests stand.

### 4. "Table Buffers" label adds a third name next to "Heap" fields

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, psql UX conventions · `standards-compat-2`

Release 20 ships "Table Buffers" next to "Heap Fetches" and "Heap Blocks". Plan visualizers and log parsers then match on the text label and the JSON group key. A later rename to fix the inconsistency breaks those tools, so the project keeps the mixed naming for good.

In message #11 the author asks whether "Table Buffers" fits with "Heap Fetches" and "Heap Blocks", and message #12 says this is still open. v18 keeps the label, and structured output gets the new "Table Buffers" and "Table I/O Timings" groups (message #17). EXPLAIN labels and JSON keys are a contract that external tools read, so the naming should be settled before commit.

Thread: [#11 (Lukas Fittl, 2026-02-24)](https://www.postgresql.org/message-id/CAP53PkzGbyeJMLDAcvMRgzXPXYsYXZr3SBg0UwhfkYjqu8oK_g%40mail.gmail.com), [#12 (Lukas Fittl, 2026-03-08)](https://www.postgresql.org/message-id/CAP53PkxrmpECzVFpeeEEHDGe6u625s%2BYkmVv5-gw3L_NDSfbiA%40mail.gmail.com), [#17 (Lukas Fittl, 2026-03-14)](https://www.postgresql.org/message-id/CAP53PkxFP7i7-wy98ZmEJ11edYq-RrPvJoa4kzGhBBjERA4Nyw%40mail.gmail.com), [#49 (Lukas Fittl, 2026-10-31)](https://www.postgresql.org/message-id/CAP53Pkwd6Gd4OL6cgh449wtdznWitz_V%3D-PRoswLHOzsMbcftA%40mail.gmail.com)

**Question:** Which naming will EXPLAIN use for table access across nodes, and will "Heap Fetches" and "Heap Blocks" be renamed or kept?

Counter-check: held. Message #12 says the naming question is still open, and v18 ships "Table Buffers" next to "Heap Fetches" and "Heap Blocks". Andres Freund's "s/heap/table/" in message #7 is about a code comment and does not settle the EXPLAIN label.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | 0002 changes how xlog.c updates WAL usage counters, and 0004 touches replication/logical/worker.c; no WAL record changes. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0005 removes pgBufferUsage and BufferUsageAccumDiff and adds QueryInstrumentation, NodeInstrumentation and the INSTR_* macros. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: "Table Buffers" label adds a third name next to "Heap" fields |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | 0007 adds a "Table Buffers" line and structured "Table Buffers" and "Table I/O Timings" groups to EXPLAIN ANALYZE output. |
| security / privileges | no |  |
| backpatchability | **yes** | The series rewrites instrument.c and the counter updates in bufmgr.c, so later back-patched fixes in those areas will not apply cleanly. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every new instrumentation user must handle aborts through a resource owner or PG_TRY/PG_FINALLY, and pgWalUsage stays as a second counting model. |
| performance on the hot path | **yes** | Buffer and WAL counter updates in bufmgr.c, localbuf.c and xlog.c now write through the current instrumentation stack entry. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What resets or checks instr_stack at transaction abort when an extension calls InstrStart without PG_TRY/PG_FINALLY or a resource owner?
2. Which committed test fails when a new instrumentation user does not register with its QueryInstrumentation and then errors out?
3. Is test_session_buffer_usage meant for commit, and if not, what replaces its coverage of top-level totals after aborts?
4. What is the plan and target release for retiring pgWalUsage, and what overhead does the stack patch show with WAL ON?
5. Which naming will EXPLAIN use for table access across nodes, and will "Heap Fetches" and "Heap Blocks" be renamed or kept?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund pushed the preparatory patches and reviewed the stack design, and Heikki Linnakangas reviewed an earlier version. The stack patch still needs a committer who knows executor instrumentation, resource owners and abort handling; v18 has no reply yet.
- **Liability:** The owner signs for correct buffer and WAL totals after every abort path, in EXPLAIN, pg_stat_statements, VACUUM and parallel workers, and for the extension API break. That liability lasts as long as the stack model exists, and it grows with each new instrumentation user.
- **Author follow-up:** Strong sign that the author stays: 18 versions over 14 months, feedback addressed in each round, and the entry moved forward by the author in October 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** EXPLAIN (ANALYZE, BUFFERS) users and tools like pg_stat_statements gain from the lower overhead, and two reviewers want the table buffer split. The series is eight patches across the executor, buffer manager, WAL insertion and several access methods, and the abort model adds rules that future instrumentation users must follow.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Removing pgBufferUsage breaks extensions and splits buffer and WAL APIs. Refuted (not specific): Duplicate of replication-ondisk-1 (port without PG_FINALLY) and maintenance-fit-2 (pgWalUsage stays a global). The remaining point, a compile break in a major release, is loud and is normal practice for extension APIs.
- `maintenance-fit-3` Removing pgBufferUsage moves abort-safety work onto every extension. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same failure scenario with more detail on the missing reset.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent was rejected for a reason that applies here. P1 closed from inactivity, while this author has posted 18 versions and had parts committed. P22 and P24 have no recorded reason. Message #13 points to an earlier thread only as the source of the count(*) benchmark.
- standards-compat: 2 objection(s). There is no SQL-level syntax, GUC or pg_dump change. The EXPLAIN option set is unchanged, and "Buffers" still includes table access, so existing totals stay comparable.
- replication-ondisk: 1 objection(s). The proposal changes no on-disk format, WAL record, wire protocol message or catalog, so pg_upgrade, standbys and subscribers are not affected. The pointer in shared parallel instrumentation (message #43) was answered in message #44. The nbtree VACUUM invariant does not apply, because nbtsort.c is only touched for instrumentation.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes how per-backend buffer and WAL counters are collected, and EXPLAIN ANALYZE shows them only to a user who could already run the query and see its total Buffers. The new instrumentation memory and stack are per backend, freed at query end or abort, and limited by the nesting depth that the executor already enforces.
- maintenance-fit: 3 objection(s). The nbtree VACUUM invariant does not apply, because nbtsort.c is touched only for parallel build instrumentation. The stop on understandability (message #37, Heikki) and the open request for senior review (message #34) belong to other angles.
- already-exists: 0 objection(s). Postgres has no existing way to get per-node buffer or WAL counts without the BufferUsageAccumDiff cost, and an extension cannot replace how the buffer manager counts. pg_statio_user_tables splits heap and index blocks only per table and cumulatively, so it cannot replace per-node Table Buffers under concurrency.
- demand: 0 objection(s). The material shows real demand. EXPLAIN ANALYZE overhead is a long-standing pain that a committer has pursued since 2020 ([P1]), and Andres Freund calls the design a large improvement (message #34). Tomas Vondra and Andres both want separate table buffer counts for index scans (messages #2, #18). Costs on non-users, extension API churn and the abort model belong to other angles.

### Precedents retrieved

- P1: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - used in the review; reason summarised from its thread
- P2: [Revert "Include information on buffer usage during planning phase, in EXPLAIN output."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19db23bcbda99e93321cb0636677ec9c6e121a2a) (revert, Reverted, 2020-04-03) - judged not relevant
- P3: [minimizing pg_stat_statements.track='none' performance overhead](https://commitfest.postgresql.org/patch/2092/) (cf, Withdrawn, 2019-04-15) - judged not relevant; reason summarised from its thread
- P4: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P5: [explain_regress, explain(MACHINE), and default to explain(BUFFERS)](https://commitfest.postgresql.org/patch/3409/) (cf, Withdrawn, 2023-01-15) - judged not relevant; reason summarised from its thread
- P6: [Subtrans performance smoothing](https://commitfest.postgresql.org/patch/3792/) (cf, Withdrawn, 2022-09-06) - judged not relevant; reason summarised from its thread
- P7: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant; reason summarised from its thread
- P8: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - judged not relevant; reason summarised from its thread
- P9: [explain analyze rows=%.0f](https://commitfest.postgresql.org/patch/3704/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P10: [KnownAssignedXidsGetAndSetXmin performance](https://commitfest.postgresql.org/patch/3271/) (cf, Returned with feedback, 2022-11-15) - judged not relevant; reason summarised from its thread
- P11: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant
- P12: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P13: [Add parallel counters to pg_stat_statements](https://commitfest.postgresql.org/patch/3787/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P14: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant
- P15: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - judged not relevant
- P16: [JIT compilation per plan node](https://commitfest.postgresql.org/patch/4837/) (cf, Withdrawn, 2024-08-21) - judged not relevant
- P17: [Optimization for lazy_scan_heap](https://commitfest.postgresql.org/patch/699/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P18: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P19: [Revert "Add some temporary instrumentation to pgstat.c."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5a2bb060121f574d6587e5fc6c7f1d4e3a759351) (revert, Reverted, 2012-05-15) - judged not relevant
- P20: [Revert "Temporarily instrument postgres_fdw test to look for statistics changes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=04e7ecadf64d18f67ca1d0632d8ab71f120ca5e3) (revert, Reverted, 2018-03-08) - judged not relevant
- P21: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P22: [Sampling-based timing for EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/4100/) (cf, Returned with feedback, 2023-07-03) - used in the review
- P23: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P24: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - used in the review

### Warnings

- attachment limit (8) reached; v18-0009-pgstat-Derive-pg_stat_database-I-O-timings-from-.patch not read
- attachment limit (8) reached; v1-0002-Replace-direct-changes-of-pgBufferUsage-pgWalUsag.patch not read
- attachment limit (8) reached; v1-0001-Separate-node-instrumentation-from-other-use-of-I.patch not read
- attachment limit (8) reached; v1-0003-Introduce-stack-for-tracking-per-node-WAL-buffer-.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 176 s; cost reported by the CLI: $6.55
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 80 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:38:05.664Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
