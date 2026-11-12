# Design pre-review: parallel pgbench -i

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6242](https://commitfest.postgresql.org/patch/6242/) (Needs review); authors: Mircea Cadariu (mcadariu)
- Thread: [parallel data loading for pgbench -i](https://www.postgresql.org/message-id/flat/cb014f00-66b2-4328-a65e-d11c681c9f45%40gmail.com), started 2025-11-17
- Messages read: 20 of 20
- Patch prose read: v5-0001-pgbench-parallelize-account-loading-for-range-par.patch, v1-0001-parallel-pgbench-wip.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal, and the cost falls only on pgbench users who pass -j in init mode. The open points are fixable design points: standards-compat-1 and maintenance-fit-1 concern -j being accepted but silently ignored and the g step depending on the t step. already-exists-1 asks for a comparison with `-I G` that the thread does not have. Only 3 of 20 messages were shown, so confidence is medium.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check how workers map row ranges to partition bounds or what happens to the other workers and their partitions when one worker fails. Only 3 of 20 messages were read in full.

## The proposal, restated

**Problem.** pgbench -i at large scale factors takes a long time, mostly in client-side data generation and vacuum. The author wants faster initialization for people who build benchmark databases. The idea comes from a blog post by Tomas Vondra.

**Proposed design.** With -j > 1 and --partition-method=range, pgbench -i loads pgbench_accounts with one thread per partition at most. The t step creates parent and partitions as usual. In the g step each worker truncates its assigned partitions and loads them with COPY FREEZE in its own transaction, so vacuum stays cheap.

**User-visible surface.**
- pgbench -j is now accepted in initialization mode and sets the number of loading threads.
- Parallel loading applies only with --partitions and --partition-method=range, at most one thread per partition.

**What the author claims.**
- At scale 100 with 10 partitions, v3 cut total init time from about 29 s to about 13 s.
- COPY FREEZE per partition keeps the vacuum phase small (about 1.6 s versus 7.8 s on master).
- v5 fixes the --tablespace, -I t then -I G, and repeated -I g failures Koshi reported.

**Thread state.** Reviewers lakshmi, Hayato Kuroda, Heikki Linnakangas and Koshi Shibagaki took part; v5 is the latest version. Koshi's last message asks to read partition count and bounds from the existing table so -I g alone runs in parallel, and to fall back to serial loading with a message; the author has not replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-11-17; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No comparison with existing server-side generation (-I G)

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A user who wants a fast large pgbench database can already run `pgbench -i -I dtGvp --partitions=N`, which builds the rows inside the server and sends no COPY data. If G is already close to the parallel g timings, the patch adds thread and connection handling to pgbench but helps few users.

Every timing in messages #1, #9, #12 and #13 compares parallel g against serial client-side g only, and the v5 commit message does the same. Server-side generation (G) is the existing option built to avoid client-side generation cost, and the proposal does not mention it. G does not freeze rows, so parallel g with COPY FREEZE may still win on the vacuum phase, but the thread has no numbers that show this. A committer would likely want this comparison before accepting a second fast-load path in pgbench.

Thread: [#1 (Mircea Cadariu, 2025-11-17)](https://www.postgresql.org/message-id/cb014f00-66b2-4328-a65e-d11c681c9f45%40gmail.com), [#9 (Mircea Cadariu, 2026-03-13)](https://www.postgresql.org/message-id/10868918-cdf9-49dc-99af-8e8ccd6e368c%40gmail.com), [#12 (Mircea Cadariu, 2026-04-10)](https://www.postgresql.org/message-id/ddca6fbd-1e99-4756-8b02-38f7962d464d%40gmail.com), [#13 (lakshmi, 2026-04-13)](https://www.postgresql.org/message-id/CAEvyyTjt1_QXO_37h1hbVqWdONm%2BuopV74j3K2pS5VrLKmozsw%40mail.gmail.com)

**Question:** At scale 100 and 2000 with range partitions, how does `-I dtGvp` compare in total time with v5's parallel `-I dtgvp`?

Counter-check: held. No message compares the patch with `-I G`, and server-side generation can produce the same tables. General knowledge suggests G is single-threaded and leaves rows unfrozen, but the thread has no numbers to settle it.

### 2. -j in init mode is accepted but silently ignored in several cases

fixable · level 2, rubric item 10 · psql UX conventions · `standards-compat-1`

A user runs `pgbench -i -s 1000 -j 16 --partitions=16 --partition-method=hash`, or runs `-I g` against tables made by an earlier `-I dt`. pgbench accepts -j and loads serially without a message. The user finds out only by timing the run or by reading the source.

Before this patch pgbench rejected -j in init mode, so the option failed loudly when it could not apply. The v5 docs promise parallel loading for range partitions, but message #20 shows `-I g` alone loads serially even with --partitions repeated. The docs say nothing for hash or unpartitioned tables, and message #5 already asked why -j is accepted there. The proposal also makes the g step behave differently depending on whether t ran in the same invocation, which breaks the existing independence of -I steps.

Thread: [#5 ("Hayato Kuroda (Fujitsu)", 2026-02-11)](https://www.postgresql.org/message-id/OS9PR01MB12149A6CEE200E1A5A88D9F14F563A%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#19 (Mircea Cadariu, 2026-10-14)](https://www.postgresql.org/message-id/89fa1f3c-bd2f-41b0-a9bf-efc3a52b8b19%40gmail.com), [#20 ("Koshi Shibagaki (Fujitsu)", 2026-10-15)](https://www.postgresql.org/message-id/TY5PR01MB1844572BFA5210A6942B8DE03FA9C2%40TY5PR01MB18445.jpnprd01.prod.outlook.com)

**Question:** For each case where -j cannot parallelize (hash, no partitions, g without t), will pgbench error, warn, or load in parallel, and where is this documented?

Counter-check: held. Message #20 confirms that `-I g` with -j 2 loads serially without a message, and the author has not replied. The v5 docs do not say what -j does with hash partitioning or no partitions.

### 3. Parallel load depends on t-step state in the same invocation

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A user runs `pgbench -i -I dt --partitions=4`, then later `pgbench -i -I g -s 2 -j 2`. The second run loads serially and gives no message, so the user thinks -j works and only sees it from the timing. A later patch that changes how partition bounds are computed can break the row-to-partition split without any test failing.

Message #20 shows that v5 enables parallel loading only when tables_created is set in the same process, even when --partitions is given again. Koshi notes that the workers' row ranges must match the existing partition bounds, which v5 takes from command-line options and not from the catalog. That couples the g step to the t step. The init steps were meant to be independent, as message #18 points out. The author has not replied to message #20.

Thread: [#18 ("Koshi Shibagaki (Fujitsu)", 2026-10-02)](https://www.postgresql.org/message-id/TY5PR01MB18445D0C3B8EFAE960839112FFA892%40TY5PR01MB18445.jpnprd01.prod.outlook.com), [#20 ("Koshi Shibagaki (Fujitsu)", 2026-10-15)](https://www.postgresql.org/message-id/TY5PR01MB1844572BFA5210A6942B8DE03FA9C2%40TY5PR01MB18445.jpnprd01.prod.outlook.com)

**Question:** Will the g step read partition count and bounds from the existing pgbench_accounts, and will it report when it falls back to serial loading?

Counter-check: held. Message #20 shows v5 derives worker row ranges from command-line options and the tables_created flag, so the g step depends on the t step. The silent serial load overlaps with standards-compat-1, but the open question of reading bounds from the existing table is separate.

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
| psql UX conventions | **yes** | pgbench now accepts -j in init mode, with a meaning that depends on partition method and on which init steps run. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | pgbench.c gains a threaded loading path whose row ranges must stay in step with the partition bounds made by the t step. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | pgbench already has server-side generation (`-I G`) as a fast load path, and the thread does not compare against it. |

## 5. Questions the author must answer before anyone reviews code

1. When -j is greater than 1 with hash partitioning, no partitioning, or `-I g` without t, does pgbench raise an error, warn, or load in parallel?
2. Will the g step read partition count and bounds from the existing pgbench_accounts, so that `-I g` alone can load in parallel?
3. At scale 100 and 2000 with range partitions, how does total time for `-I dtGvp` compare with v5's parallel `-I dtgvp`?
4. How much memory does pgbench use during a parallel load with many partitions, for example 1000?
5. Does v5 still contain the separate COPY batching change that Hayato raised as point 04?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas took part in the thread, and the range-only limit came from a committer in message #11. The owner needs to know pgbench's init steps and its thread handling.
- **Liability:** The owner signs for a threaded load path in pgbench.c, its behaviour on Windows, and the rule for when -j applies in init mode. The change is client-only, so the liability is limited to pgbench.
- **Author follow-up:** The author posted five versions and moved the entry to a new CommitFest in September 2026. Message #20 from Koshi has no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The 12.7 KB patch touches only pgbench and helps only users who load range-partitioned pgbench_accounts. The rule for when -j applies and the dependency of the g step on the t step need an answer before more code review.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` -j in init mode can be silently ignored. Refuted (not specific): Duplicate of standards-compat-1, which covers the same accepted-but-ignored -j cases and adds the confirmed report from message #20.
- `demand-1` Speedup reaches only users who already benchmark range-partitioned accounts. Refuted (already handled): The thread weighed the scope: messages #4, #6 and #10 measured the non-partitioned case as no faster overall, and a committer proposed the range-only limit in message #11. Users of plain tables pay no cost, and the `-I G` comparison duplicates already-exists-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P4] (pgbench COPY FREEZE) was withdrawn only because it duplicated another queued patch. [P18] and [P19] are server-side parallel COPY with no recorded reason, and [P1] partitions pgbench_history. None of these reasons applies to client-side parallel loading of pgbench_accounts partitions, and the thread cites no earlier discussion of this idea apart from the blog post in message #3.
- standards-compat: 1 objection(s). The option reuses the existing -j rather than adding a new one, and it was rejected in init mode before, so no existing script changes behaviour. No SQL, catalog or dump surface is involved.
- replication-ondisk: 0 objection(s). The proposal changes only how the pgbench client loads pgbench_accounts. It adds no on-disk format, WAL record, catalog, wire protocol or ABI change, and COPY FREEZE after TRUNCATE in one transaction is an existing server path, so this angle has nothing to object to.
- security: 0 objection(s). The proposal changes no privilege or exposure. Each worker opens a normal client connection with the caller's credentials and runs TRUNCATE and COPY FREEZE on tables that the same user just created. No new privilege, role, or server-side code path is involved, and the connection count is set by the caller through -j and limited to one connection per partition.
- maintenance-fit: 2 objection(s). Fit and hot path are fine: pgbench is a client tool, and the change adds no server-side cost for users who do not use it. Heikki took part in the thread, so the area has a qualified reviewer.
- already-exists: 1 objection(s). No extension or external tool that does parallel pgbench initialization is known with certainty. The only existing alternative is pgbench's own server-side generation step, covered above.
- demand: 1 objection(s). Demand for faster pgbench -i in general is plausible and costs fall only on pgbench users, so the only demand concern is the narrow scope of v5.

### Precedents retrieved

- P1: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - used in the review; reason summarised from its thread
- P2: [pgbench - - break out timing data for initialization phases](https://commitfest.postgresql.org/patch/1308/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P3: [pgbench - break out initialization timing info by init phase](https://commitfest.postgresql.org/patch/1493/) (cf, Returned with feedback, 2018-03-07) - judged not relevant; reason summarised from its thread
- P4: [pgbench using COPY FREEZE](https://commitfest.postgresql.org/patch/3201/) (cf, Withdrawn, 2021-07-01) - used in the review; reason summarised from its thread
- P5: [Remove redundant initializations](https://commitfest.postgresql.org/patch/3229/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [system load consideration hook for parallel workers](https://commitfest.postgresql.org/patch/725/) (cf, Rejected, 2016-09-22) - judged not relevant; reason summarised from its thread
- P7: [Lazy JIT IR code generation to increase JIT speed with partitions](https://commitfest.postgresql.org/patch/3071/) (cf, Withdrawn, 2023-01-27) - judged not relevant; reason summarised from its thread
- P8: [Revert fix missing call to table_finish_bulk_insert during COPY](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5db56fc4d6e95c582b61c99328ea0702b869fa0) (revert, Reverted, 2019-07-01) - judged not relevant
- P9: [pgbench: faster version of tpcb-like transaction](https://commitfest.postgresql.org/patch/1257/) (cf, Returned with feedback, 2017-10-01) - judged not relevant; reason summarised from its thread
- P10: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P11: [accounting for memory used for BufFile during hash joins](https://commitfest.postgresql.org/patch/2109/) (cf, Returned with feedback, 2019-11-26) - judged not relevant
- P12: [pgbench: add multiconnect support](https://commitfest.postgresql.org/patch/3227/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P13: [COPY FREEZE and PD_ALL_VISIBLE](https://commitfest.postgresql.org/patch/401/) (cf, Returned with feedback, 2015-12-24) - judged not relevant
- P14: [Memory Accounting (towards memory-bounded HashAgg)](https://commitfest.postgresql.org/patch/269/) (cf, Rejected, 2015-07-26) - judged not relevant
- P15: [pgbench allow '=' in \set](https://commitfest.postgresql.org/patch/246/) (cf, Rejected, 2015-05-13) - judged not relevant
- P16: [Revert "Fix partition pruning setup during DETACH CONCURRENTLY"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b0ea16528cda1f3a993bbfeb2b93eed9762af4e6) (revert, Reverted, 2024-06-24) - judged not relevant
- P17: [pgbench log file headers](https://commitfest.postgresql.org/patch/4660/) (cf, Returned with feedback, 2024-04-07) - judged not relevant
- P18: [Support parallelism for copy command](https://commitfest.postgresql.org/patch/2610/) (cf, Withdrawn, 2021-03-01) - used in the review
- P19: [Parallel COPY FROM](https://commitfest.postgresql.org/patch/1266/) (cf, Returned with feedback, 2017-10-02) - used in the review
- P20: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - judged not relevant
- P21: [Faster pglz compression](https://commitfest.postgresql.org/patch/2897/) (cf, Withdrawn, 2023-02-28) - judged not relevant
- P22: [Allow pg_upgrade to copy segments of the same relfilenode in parallel](https://commitfest.postgresql.org/patch/3525/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P23: [Revert "Improve accounting for memory used by shared hash tables"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aff1dc8dfa7584ddbc695e1aae0f767db66326f) (revert, Reverted, 2025-04-04) - judged not relevant
- P24: [Speed up LockReleaseAll](https://commitfest.postgresql.org/patch/3220/) (cf, Returned with feedback, 2021-12-03) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 128 s; cost reported by the CLI: $1.87
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 68 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:46:50.255Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
