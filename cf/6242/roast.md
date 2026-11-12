# Design pre-review: parallel pgbench -i

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6242](https://commitfest.postgresql.org/patch/6242/); [thread](https://www.postgresql.org/message-id/flat/cb014f00-66b2-4328-a65e-d11c681c9f45%40gmail.com), 20 of 20 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal, and the cost falls only on pgbench users who pass -j in init mode. The open points are fixable design points: standards-compat-1 and maintenance-fit-1 concern -j being accepted but silently ignored and the g step depending on the t step. already-exists-1 asks for a comparison with `-I G` that the thread does not have. Only 3 of 20 messages were shown, so confidence is medium.

Not checked: Without the code, this review could not check how workers map row ranges to partition bounds or what happens to the other workers and their partitions when one worker fails. Only 3 of 20 messages were read in full.

## Prior attempts

None found before 2025-11-17.

## Objections a reviewer is likely to raise

### 1. No comparison with existing server-side generation (-I G)

A user who wants a fast large pgbench database can already run `pgbench -i -I dtGvp --partitions=N`, which builds the rows inside the server and sends no COPY data. If G is already close to the parallel g timings, the patch adds thread and connection handling to pgbench but helps few users.

Every timing in messages #1, #9, #12 and #13 compares parallel g against serial client-side g only, and the v5 commit message does the same. Server-side generation (G) is the existing option built to avoid client-side generation cost, and the proposal does not mention it. G does not freeze rows, so parallel g with COPY FREEZE may still win on the vacuum phase, but the thread has no numbers that show this. A committer would likely want this comparison before accepting a second fast-load path in pgbench.

**Question:** At scale 100 and 2000 with range partitions, how does `-I dtGvp` compare in total time with v5's parallel `-I dtgvp`?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/cb014f00-66b2-4328-a65e-d11c681c9f45%40gmail.com), [#9](https://www.postgresql.org/message-id/10868918-cdf9-49dc-99af-8e8ccd6e368c%40gmail.com), [#12](https://www.postgresql.org/message-id/ddca6fbd-1e99-4756-8b02-38f7962d464d%40gmail.com), [#13](https://www.postgresql.org/message-id/CAEvyyTjt1_QXO_37h1hbVqWdONm%2BuopV74j3K2pS5VrLKmozsw%40mail.gmail.com); `already-exists-1`</sub>

### 2. -j in init mode is accepted but silently ignored in several cases

A user runs `pgbench -i -s 1000 -j 16 --partitions=16 --partition-method=hash`, or runs `-I g` against tables made by an earlier `-I dt`. pgbench accepts -j and loads serially without a message. The user finds out only by timing the run or by reading the source.

Before this patch pgbench rejected -j in init mode, so the option failed loudly when it could not apply. The v5 docs promise parallel loading for range partitions, but message #20 shows `-I g` alone loads serially even with --partitions repeated. The docs say nothing for hash or unpartitioned tables, and message #5 already asked why -j is accepted there. The proposal also makes the g step behave differently depending on whether t ran in the same invocation, which breaks the existing independence of -I steps.

**Question:** For each case where -j cannot parallelize (hash, no partitions, g without t), will pgbench error, warn, or load in parallel, and where is this documented?

<sub>fixable, level 2, psql UX conventions; messages [#5](https://www.postgresql.org/message-id/OS9PR01MB12149A6CEE200E1A5A88D9F14F563A%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#19](https://www.postgresql.org/message-id/89fa1f3c-bd2f-41b0-a9bf-efc3a52b8b19%40gmail.com), [#20](https://www.postgresql.org/message-id/TY5PR01MB1844572BFA5210A6942B8DE03FA9C2%40TY5PR01MB18445.jpnprd01.prod.outlook.com); `standards-compat-1`</sub>

### 3. Parallel load depends on t-step state in the same invocation

A user runs `pgbench -i -I dt --partitions=4`, then later `pgbench -i -I g -s 2 -j 2`. The second run loads serially and gives no message, so the user thinks -j works and only sees it from the timing. A later patch that changes how partition bounds are computed can break the row-to-partition split without any test failing.

Message #20 shows that v5 enables parallel loading only when tables_created is set in the same process, even when --partitions is given again. Koshi notes that the workers' row ranges must match the existing partition bounds, which v5 takes from command-line options and not from the catalog. That couples the g step to the t step. The init steps were meant to be independent, as message #18 points out. The author has not replied to message #20.

**Question:** Will the g step read partition count and bounds from the existing pgbench_accounts, and will it report when it falls back to serial loading?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#18](https://www.postgresql.org/message-id/TY5PR01MB18445D0C3B8EFAE960839112FFA892%40TY5PR01MB18445.jpnprd01.prod.outlook.com), [#20](https://www.postgresql.org/message-id/TY5PR01MB1844572BFA5210A6942B8DE03FA9C2%40TY5PR01MB18445.jpnprd01.prod.outlook.com); `maintenance-fit-1`</sub>

## Questions for the author

1. When -j is greater than 1 with hash partitioning, no partitioning, or `-I g` without t, does pgbench raise an error, warn, or load in parallel?
2. Will the g step read partition count and bounds from the existing pgbench_accounts, so that `-I g` alone can load in parallel?
3. At scale 100 and 2000 with range partitions, how does total time for `-I dtGvp` compare with v5's parallel `-I dtgvp`?
4. How much memory does pgbench use during a parallel load with many partitions, for example 1000?
5. Does v5 still contain the separate COPY batching change that Hayato raised as point 04?

## Touch points

- **psql UX conventions:** pgbench now accepts -j in init mode, with a meaning that depends on partition method and on which init steps run.
- **maintenance burden ("who maintains this in 10 years"):** pgbench.c gains a threaded loading path whose row ranges must stay in step with the partition bounds made by the t step.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** pgbench already has server-side generation (`-I G`) as a fast load path, and the thread does not compare against it.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas took part in the thread, and the range-only limit came from a committer in message #11. The owner needs to know pgbench's init steps and its thread handling.
- Priority: fix design first; audience niche; review cost low. The 12.7 KB patch touches only pgbench and helps only users who load range-partitioned pgbench_accounts. The rule for when -j applies and the dependency of the g step on the t step need an answer before more code review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
