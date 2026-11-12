# Design pre-review: Parallel heap vacuum

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7102](https://commitfest.postgresql.org/patch/7102/) (Needs review); authors: Masahiko Sawada (masahikosawada), Bharath Rupireddy (bharathrupireddy)
- Thread: [Parallel heap vacuum](https://www.postgresql.org/message-id/flat/CAD21AoAEfCNv-GgaDheDJ%2Bs-p_Lv1H24AiJeNoPGCmZNSwL1YA%40mail.gmail.com), started 2024-06-28
- Messages read: 137 of 137
- Patch prose read: v18-0005-Add-more-parallel-vacuum-tests.patch, v18-0004-Support-parallelism-for-collecting-dead-items-du.patch, v18-0002-vacuumparallel.c-Support-parallel-vacuuming-for-.patch, v18-0001-Introduces-table-AM-APIs-for-parallel-table-vacu.patch, v18-0003-Move-lazy-heap-scan-related-variables-to-new-str.patch, parallel_heap_vacuum_scan.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and several committers in the thread accept the goal. The open points are fixable: heap scan workers start by default with no separate switch (standards-compat-1, standards-compat-2), autovacuum reach is not stated (demand-1), and v21 does not state why the merged relfrozenxid stays safe after a scan restart (replication-ondisk-1). The last point needs a committer who knows vacuumlazy.c. Most of v21's text is not visible, so confidence is medium.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v21 code, its documentation hunks and most of the 137 messages were not read. Coverage of every unskippable block after a restart, eager scanning in parallel mode, and what autovacuum can use cannot be judged without them.

## The proposal, restated

**Problem.** Parallel vacuum today covers only index vacuuming and cleanup, so one process always scans the heap. Large tables without indexes, or vacuums run with INDEX_CLEANUP off for fast freezing, get no benefit from more workers.

**Proposed design.** Vacuum phase 1 (heap scan, pruning, freezing, dead TID collection) runs in the leader plus parallel workers. Each process claims fixed-size chunks of blocks, writes into a shared dead TID store, and the leader merges per-worker scan results, including relfrozenxid and relminmxid. In v21 the parallel heap scan lives in vacuumlazy.c and reuses helpers split out of vacuumparallel.c, with no table AM callbacks; phase 3 stays serial.

**User-visible surface.**
- The VACUUM PARALLEL option now also covers the heap scan phase, with the worker count based on table size.
- min_parallel_table_scan_size now decides whether a table gets a parallel heap scan in vacuum.
- VACUUM VERBOSE reports workers launched for collecting dead tuples.
- Earlier versions (v18) added table AM callbacks such as parallel_vacuum_compute_workers; v21 drops them.

**What the author claims.**
- In the v21 best-case benchmark, phase 1 is 2.4x to 4.3x faster on a 10GB table with 2 to 8 workers.
- On a 50GB storage-bound table, phase 1 gains 1.5x to 3.6x.
- Total VACUUM time improves 1.1x to 1.5x with indexes and up to 4.3x without.
- A 32K-block chunk, matching one VM page, avoids VM page contention that a 1K chunk causes.
- The shared TID store lock costs 11% to 17% of phase 1 at 4 to 8 workers.

**Thread state.** Bharath Rupireddy posted v21 on 2026-11-11 after moving the parallel scan into vacuumlazy.c; no replies to v20 or v21 are visible. Earlier objections from Andres Freund, Melanie Plageman, Robert Haas and Tomas Vondra on layering and the eager scanning interaction may or may not be settled.

## 2. Prior attempts

### Block level parallel vacuum (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/734/](https://commitfest.postgresql.org/patch/734/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAD21AoD1xAqp4zK-Vi1cuY3feq2oO8HcpJiz32UDUfe0BE31Xw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAD21AoD1xAqp4zK-Vi1cuY3feq2oO8HcpJiz32UDUfe0BE31Xw@mail.gmail.com)
- **Date:** decided 2016-10-03; started 2016-08-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Robert Haas, Claudio Freire, Michael Paquier, Pavan Deolasee (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers rejected the design because each worker scans every index, which adds index scanning work and makes workers contend for cleanup locks on the same index pages; the author's own benchmark showed index vacuum getting slower with more workers. Robert Haas proposed a parallel heap scan with a shared dead tuple array and one worker per index, and the author agreed. While reworking the patch, the author hit an unsolved problem: parallel workers in one lock group can extend the FSM or VM at the same time and fail. The entry was then closed for lack of activity after the feedback was given.
- **Reason (quote):** "This doesn't seem like a good design, because it adds a lot of extra index scanning work." ([message](https://www.postgresql.org/message-id/CA%2BTgmobV6%2BZPTNE3Z%2B08D9Xp7UK%2BmSq-rztOW%2B%3DRGsr5-pKiUA%40mail.gmail.com))
- **What is different this time:** P4 had each worker scan a fixed heap range and then vacuum every index. v21 parallelizes only phase 1 into one shared dead TID store, keeps one worker per index, and leaves phase 3 serial.

### Block level parallel vacuum (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/954/](https://commitfest.postgresql.org/patch/954/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAD21AoD1xAqp4zK-Vi1cuY3feq2oO8HcpJiz32UDUfe0BE31Xw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAD21AoD1xAqp4zK-Vi1cuY3feq2oO8HcpJiz32UDUfe0BE31Xw@mail.gmail.com)
- **Date:** decided 2017-03-05; started 2016-08-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Robert Haas proposed moving the patch to v11 because it still had too many open issues for the rest of the cycle, among them deadlock problems that need careful changes. The patch was also not in reviewable shape by March 1st. The author agreed that the design needed more discussion, and David Steele marked it Returned with Feedback.
- **Reason (quote):** "I think there are too many issues here to address in the limited time we have remaining this cycle, and I believe that if we try to get them all solved in the next few weeks we're likely to end up getting backed into some choices by time pressure that we may later regret bitterly." ([message](https://www.postgresql.org/message-id/CA%2BTgmobLcu_q7-0-rqiBuzowZ1J-ywDHdgBJzXdSx978JPorKA%40mail.gmail.com))
- **What is different this time:** P6 changed buffer locking so several processes could wait for one cleanup lock, and added a vacuum_parallel_workers GUC. v21 does neither; workers claim disjoint 32K-block chunks and reuse the PARALLEL option.

### Block level parallel vacuum (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1223/](https://commitfest.postgresql.org/patch/1223/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAD21AoD1xAqp4zK-Vi1cuY3feq2oO8HcpJiz32UDUfe0BE31Xw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAD21AoD1xAqp4zK-Vi1cuY3feq2oO8HcpJiz32UDUfe0BE31Xw@mail.gmail.com)
- **Date:** decided 2017-11-30; started 2016-08-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier returned the patch in November 2017 because it depended on a separate extension lock patch, and he proposed coming back once those root problems were addressed. A regression failure ("cache lookup failed for relation" from DROP SCHEMA ... CASCADE) was also still unexplained. The first round in 2016 was returned for lack of activity, after early tests showed index vacuum slowing down with more workers because of cleanup lock contention on index buffers.
- **Reason (quote):** "So I propose to mark this patch as returned with feedback for now, and come back to it once the root problems are addressed." ([message](https://www.postgresql.org/message-id/CAB7nPqQiUYFEKnUUSMahDHA1bAqDLyMW0mjFHnUub%3DTZstr64A%40mail.gmail.com))
- **What is different this time:** P2 was returned because it depended on a separate extension lock patch. v21 does not depend on another lock patch and keeps the phase 1 heap scan apart from index work.

### Revert the commits related to allowing page lock to conflict among parallel group members. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAJKUy5jffnRKNvRHKQ0LynRb0RJC-o4P8Ku3x9vGAVLwDBWumQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAJKUy5jffnRKNvRHKQ0LynRb0RJC-o4P8Ku3x9vGAVLwDBWumQ@mail.gmail.com), [https://www.postgresql.org/message-id/flat/CAD21AoCmT3cFQUN4aVvzy5chw7DuzXrJCbrjTU05B+Ss=Gn1LA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAD21AoCmT3cFQUN4aVvzy5chw7DuzXrJCbrjTU05B+Ss=Gn1LA@mail.gmail.com)
- **Date:** decided 2023-07-06; started 2020-03-19
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This commit reverts the work done by commits 3ba59ccc89 and 72e78d831a. Those commits were incorrect in asserting that we never acquire any other heavy-weight lock after acquring page lock other than relation extension lock. We can acquire a lock on catalogs while doing catalog look up after acquring page lock. This won't impact any existing feature but we need to think some other way to achieve this before parallelizing other write operations or even improving the parallelism in vacuum (like allowing multiple workers for an index).
- **Reason (quote):** "We can acquire a lock on catalogs while doing catalog look up after acquring page lock."
- **What is different this time:** The revert concerns page locks among parallel group members and names several workers per index as the risk. v21's phase 1 works on heap pages only and does not put several workers on one index.


## 3. Objections a reviewer is likely to raise

### 1. Existing VACUUM commands start heap scan workers without asking

fixable · level 1b, rubric item 4 · desirability, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A site runs nightly `vacuum` on large tables, or `vacuum (parallel 2)` sized for its indexes. After upgrade, each run also starts heap scan workers sized by table size. These workers take slots from max_parallel_maintenance_workers and add I/O during the maintenance window, and the DBA finds out from load graphs or new VACUUM VERBOSE lines.

The documentation hunk widens PARALLEL from index phases to heap scanning, and a bare `vacuum` already defaults to choosing a degree, so the new phase is on by default. In message #98 Andres notes that nightly vacuums are common and that the feature is enabled by default. In message #97 Melanie expected a separate switch for heap parallelism and did not find one. No visible version adds such a switch.

Thread: [#97 (Melanie Plageman, 2025-04-05)](https://www.postgresql.org/message-id/CAAKRu_atVy1RAOwRFwEAy0is-mRPay0cxGUO%2B%2BSdOCHgB-bSKQ%40mail.gmail.com), [#98 (Andres Freund, 2025-04-05)](https://www.postgresql.org/message-id/qqguhicg3gvzs6z6enomx4mpcowvlknxuwtelbhmivmkdj622s%40bgo25mtkeoxe), [#100 (Melanie Plageman, 2025-04-06)](https://www.postgresql.org/message-id/CAAKRu_ajnk7J4LjOXqh2V1%3DFZpTMkTpKZACpZ4tCt7UR3o9tqw%40mail.gmail.com)

**Question:** Why is it right for existing VACUUM invocations to gain heap scan workers by default, and how does a user keep only the old index-only parallel behavior?

Counter-check: held. The v18 documentation hunk puts heap scanning under the existing PARALLEL option, and a bare VACUUM already chooses a degree. Messages #97 and #98 raise the default-on concern, and no visible version adds a separate switch.

### 2. Users with the clearest need run autovacuum, which this does not reach

fixable · level 1b, rubric item 4 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

An operator has autovacuum stuck for hours on one large table, holding back the freeze horizon. They upgrade expecting faster heap scans, but autovacuum still scans the heap with one process. They gain only if they replace autovacuum with scheduled manual VACUUM (PARALLEL n).

Most of the demand in the thread is about autovacuum. Andres Freund describes autovacuum bound by CPU or I/O on one large table (messages #129, #133), and he called parallel vacuum without autovacuum integration a mistake not to repeat (message #98). Message #99 deferred this, and message #128 points to a separate autovacuum patch for index vacuuming only. The v21 material does not say whether autovacuum can use the parallel heap scan, so the benefit goes to manual VACUUM users while the whole project carries the code.

Thread: [#98 (Andres Freund, 2025-04-05)](https://www.postgresql.org/message-id/qqguhicg3gvzs6z6enomx4mpcowvlknxuwtelbhmivmkdj622s%40bgo25mtkeoxe), [#99 (Masahiko Sawada, 2025-04-06)](https://www.postgresql.org/message-id/CAD21AoA9eJ0Qx%3D3h77__K5ssj8R3KoVY3Uw5P7vux8HmJMRKBg%40mail.gmail.com), [#128 (Masahiko Sawada, 2025-09-17)](https://www.postgresql.org/message-id/CAD21AoAR79ai9bXLoqKyYzcYEgpOvd1y7eGqO7OGHpC_qQ%2Bi1Q%40mail.gmail.com), [#129 (Andres Freund, 2025-09-17)](https://www.postgresql.org/message-id/xymq6plhuujm6rpb3hyqpwqkqo3q6kdx2a6nnei6sgpbzzj7lb%40ejwunmymqgxb), [#133 (Andres Freund, 2025-09-18)](https://www.postgresql.org/message-id/toxt6ppr2swqrclx5kni3hdy4c2xqcdbojup2xwjlgsui3mwp7%40wqg3liamu44k)

**Question:** Can autovacuum, including the wraparound failsafe path, use the parallel heap scan in v21, and if not, which users who run manual VACUUM asked for it?

Counter-check: held. Manual-only parallelism copies the existing parallel index vacuum, and message #98 says nightly manual vacuums are common, so some demand exists. But message #98 asks not to extend manual-only parallelism, and the v21 material does not say whether autovacuum can use the heap scan.

### 3. Measured gain is large only for tables without indexes or with warm cache

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-2`

A DBA with a typical indexed table on network storage runs VACUUM (PARALLEL 8). Index vacuuming and phase 3 dominate the run time, so total time improves about 1.1x while eight workers compete with the user workload. They see little benefit and do not know which tables would gain more.

The v21 numbers in message #137 show total VACUUM gains of 1.1x to 1.5x once indexes exist, with warm caches and local NVMe. Tomas Vondra measured gains only without indexes and asked which workloads would benefit (message #122). Cold-cache phase 1 scaled less than expected (message #60), and message #137 has no cold-cache results. The remaining answer is INDEX_CLEANUP off for fast freezing (message #128), but no field report shows how often users run that by hand.

Thread: [#60 (Masahiko Sawada, 2025-03-07)](https://www.postgresql.org/message-id/CAD21AoAHoU_dj0AJcTVRY%3DzL1jOcnkcihh9Y_Xuw7%2BEhBT9a5Q%40mail.gmail.com), [#122 (Tomas Vondra, 2025-09-17)](https://www.postgresql.org/message-id/df858858-72da-4649-aaf9-ab97e4fb2d6f%40vondra.me), [#128 (Masahiko Sawada, 2025-09-17)](https://www.postgresql.org/message-id/CAD21AoAR79ai9bXLoqKyYzcYEgpOvd1y7eGqO7OGHpC_qQ%2Bi1Q%40mail.gmail.com), [#137 (Bharath Rupireddy, 2026-11-11)](https://www.postgresql.org/message-id/CALj2ACWtW0jCRddhpPG2t_ahX87fkbxZXj1ZKKPh3AnCT7gWjQ%40mail.gmail.com)

**Question:** Which production workloads, such as large index-free tables or manual INDEX_CLEANUP off freezing runs, are reported to need this, and what do cold-cache results on network storage show?

Counter-check: held. Message #128 answers the question in message #122 with the INDEX_CLEANUP off case, and v21 shows 1.5x to 3.6x on a storage-bound table. But v21 has no cold-cache results, and the earlier cold-cache run in message #60 scaled less than expected.

### 4. relfrozenxid advance across workers and scan restarts lacks a stated invariant

fixable · level 2, rubric item 7 · design, catalog changes, on-disk format / pg_upgrade · `replication-ondisk-1`

A large table fills maintenance_work_mem, so vacuum pauses phase 1 for index and heap vacuuming, then relaunches fewer workers. If a partly claimed 32K-block chunk is never finished, the leader can still store a merged relfrozenxid that is newer than unfrozen XIDs in those blocks. Nobody notices until clog truncation or wraparound, when those tuples read as invisible or fail.

The leader sets pg_class.relfrozenxid and relminmxid from the minimum of per-worker values, and the reported scanned_pages and skippedallvis decide whether it advances at all. Message #82 and message #108 show that resuming after a TID store cycle with fewer workers needed special takeover logic, and message #90 found a block-assignment bug in the skip path. Message #137 redesigns the scan with fixed chunks in vacuumlazy.c but does not say how every unskippable block is still covered. Melanie asked in message #90 for the reasoning to be written down because the merge is the place where a mistake leads to lost data.

Thread: [#82 (Masahiko Sawada, 2025-03-20)](https://www.postgresql.org/message-id/CAD21AoC%2Bb8An4%3DbiuSCurxv1cTgfDyQzmBvdR%2BgisRj-qq5U_w%40mail.gmail.com), [#90 (Melanie Plageman, 2025-03-26)](https://www.postgresql.org/message-id/CAAKRu_aS_k751daj0QfO%2B5ZgLDz8LP1Wo_b5%3DRRKt6%3Df6OStLg%40mail.gmail.com), [#108 (Masahiko Sawada, 2025-06-13)](https://www.postgresql.org/message-id/CAD21AoB2zAeGjAARs5uExy66ZrYpfL5h_PQHQuCE6Dm%3D3rvSUw%40mail.gmail.com), [#137 (Bharath Rupireddy, 2026-11-11)](https://www.postgresql.org/message-id/CALj2ACWtW0jCRddhpPG2t_ahX87fkbxZXj1ZKKPh3AnCT7gWjQ%40mail.gmail.com)

**Question:** In v21, what guarantees that every block not skipped through the VM is scanned before the merged relfrozenxid is stored, including after a phase 1 restart with fewer workers?

Counter-check: held. Earlier versions had the leader finish unclaimed work and tested it (message #108), but v21 replaces the scan with a new fixed-chunk design. The visible v21 text does not say why every unskippable block is covered after a restart.

### 5. Turning off heap scan workers also changes parallel seq scan plans

fixable · level 2, rubric item 12 · new GUC, planner behaviour change · `standards-compat-2`

A DBA wants vacuum to stop using heap scan workers on one large table. The only knob given is min_parallel_table_scan_size, so the DBA raises it. Queries on all tables then lose parallel sequential scans, and the DBA finds out when reports run slower.

In message #99 the author names min_parallel_table_scan_size as the way to disable the parallel heap scan. In message #100 Melanie notes that this setting also drives parallel sequential scans and can only be set globally, and she asks for per-phase control. Nothing visible after that adds a reloption or a VACUUM option for heap scan parallelism, and the brief lists this as open. Reusing a planner setting to control a maintenance phase ties two unrelated behaviors to one knob.

Thread: [#99 (Masahiko Sawada, 2025-04-06)](https://www.postgresql.org/message-id/CAD21AoA9eJ0Qx%3D3h77__K5ssj8R3KoVY3Uw5P7vux8HmJMRKBg%40mail.gmail.com), [#100 (Melanie Plageman, 2025-04-06)](https://www.postgresql.org/message-id/CAAKRu_ajnk7J4LjOXqh2V1%3DFZpTMkTpKZACpZ4tCt7UR3o9tqw%40mail.gmail.com)

**Question:** How can a user disable or limit parallel heap scanning for one table or one VACUUM run without changing parallel query planning?

Counter-check: held. The scenario is overstated, because min_parallel_table_scan_size can be set in the session that runs VACUUM. The request in message #100 for per-table or per-phase control is still unanswered, and reusing a planner setting needs a stated reason.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-1: relfrozenxid advance across workers and scan restarts lacks a stated invariant |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | see objection replication-ondisk-1: relfrozenxid advance across workers and scan restarts lacks a stated invariant |
| ABI / extension API | **yes** | v18 added five table AM callbacks in tableam.h; v21 drops them and splits helpers out of vacuumparallel.c instead. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but min_parallel_table_scan_size gains a second meaning as the threshold for the parallel heap scan in vacuum. |
| planner behaviour change | **yes** | see objection standards-compat-2: Turning off heap scan workers also changes parallel seq scan plans |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: Existing VACUUM commands start heap scan workers without asking |
| maintenance burden ("who maintains this in 10 years") | **yes** | vacuumlazy.c gains a parallel phase 1 path with a leader-side merge of relfrozenxid and relminmxid next to the serial path. |
| performance on the hot path | **yes** | Workers take an exclusive lock on the shared dead TID store; the author reports 11% to 17% of phase 1 at 4 to 8 workers. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Users with the clearest need run autovacuum, which this does not reach |

## 5. Questions the author must answer before anyone reviews code

1. In v21, what guarantees that every block not skipped through the VM is scanned before the merged relfrozenxid is stored, including after a phase 1 restart with fewer workers?
2. How does a user keep parallel index vacuuming but turn off the parallel heap scan for one table or one VACUUM run, without changing min_parallel_table_scan_size?
3. Why is it right for an existing plain `vacuum` or `vacuum (parallel n)` to start heap scan workers by default after upgrade?
4. Can autovacuum, including the wraparound failsafe path, use the v21 parallel heap scan, and if not, what in the design allows adding that later?
5. What do cold-cache results on network storage show for v21, with and without indexes?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Masahiko Sawada, a committer, started the thread and co-authors the patch. A second reviewer who knows the phase 1 loop in vacuumlazy.c, eager scanning and relfrozenxid tracking should sign off on v21.
- **Liability:** The owner signs for the merged relfrozenxid and relminmxid across workers and scan restarts, where a mistake loses data much later. The owner also keeps the parallel phase 1 path in step with every later change to the serial path.
- **Author follow-up:** Good sign: the thread has run since 2024-06, and Bharath Rupireddy posted v21 on 2026-11-11 after reworking the layering in response to review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Large tables with slow vacuum are a known problem and reviewers in the thread accept the goal, but v21 shows large gains mainly without indexes. The patch adds a second phase 1 path in vacuumlazy.c with shared state, so review needs area experts; the control switch and the relfrozenxid invariant should be settled during that review.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Two phase 1 code paths, and the parallel one rarely runs in tests. Refuted (wrong facts): The v18 patch touches the vacuum_parallel regression test, and message #136 says v20 includes tests, so injection-point tests are not the only coverage. A regression test can lower min_parallel_table_scan_size, as vacuum_parallel.sql already does for min_parallel_index_scan_size.
- `maintenance-fit-2` Existing VACUUM commands start heap workers without being asked. Refuted (not specific): Duplicate of standards-compat-1 (heap workers on by default) and standards-compat-2 (min_parallel_table_scan_size as the only switch).
- `maintenance-fit-3` Parallel heap scan helps only manual VACUUM, not autovacuum. Refuted (not specific): Duplicate of demand-1.
- `maintenance-fit-4` Heap-only placement leaves other table AMs to copy the code. Refuted (already handled): The thread weighed this: messages #114 and #123 reject table AM callbacks, and messages #122 and #127 accept shared helpers as one option. v21-0001 moves the reusable code in vacuumparallel.c into separate functions, which other table AMs can already call.

### Review angles

- prior-attempts: 0 objection(s). The 2016-2017 attempts [P4], [P6] and [P2] died on per-worker index scans, cleanup lock waiting changes and an extension lock dependency. v21 contains none of these, and [P5] concerns index-level parallelism, which v21 does not add. The open points in the thread are layering and the eager scanning interaction (messages #97, #98, #114, #123), which belong to other angles.
- standards-compat: 2 objection(s). Two objections from the standards and compatibility angle. Eager scanning support is not raised: v17 (message #108) and the v21 notes (message #137) say chunks serve as eager scan regions.
- replication-ondisk: 1 objection(s). No new WAL record types, on-disk format, catalog columns or protocol messages appear. Standby replay and pg_upgrade are unaffected. VM and pruning WAL come from several processes, but the record contents stay the same.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. VACUUM PARALLEL keeps its current ownership/MAINTAIN checks. Workers stay capped by max_parallel_maintenance_workers, the shared dead TID store is bounded by maintenance_work_mem, and heap pruning and freezing run no user-supplied code. Existing parallel index vacuum already allows the same worker use, so no new escalation, leak or unprivileged denial-of-service path is visible.
- maintenance-fit: 4 objection(s). Not applicable; objections are listed. A committer, Masahiko Sawada, is a co-author, so cleanup ownership is less of a concern than usual.
- already-exists: 0 objection(s). Postgres has no existing way to scan the heap of one table with several processes during vacuum. Parallel vacuum covers only index phases, and vacuumdb --jobs parallelizes across tables, so neither helps a single large table without indexes or a fast INDEX_CLEANUP off freeze. Partitioning a table so that several vacuums can run at once is a schema redesign, too large to count as an existing alternative. No extension can replace the phase 1 loop in vacuumlazy.c, and no hook exists for it. The v21 code split between vacuumlazy.c and vacuumparallel.c is a layering question for the design angle, and it does not add a second mechanism for a job that is already done.
- demand: 2 objection(s). The thread shows real interest from several committers in faster vacuum of large tables, so both objections ask for evidence that the demand reaches this design, not that the problem is missing.

### Precedents retrieved

- P1: [Skip vacuum log report code in lazy_scan_heap() if possible](https://commitfest.postgresql.org/patch/3404/) (cf, Rejected, 2021-12-04) - judged not relevant; reason summarised from its thread
- P2: [Block level parallel vacuum](https://commitfest.postgresql.org/patch/1223/) (cf, Returned with feedback, 2017-11-30) - used in the review; reason summarised from its thread
- P3: [Allow setting parallel_workers on partitioned tables](https://commitfest.postgresql.org/patch/2987/) (cf, Returned with feedback, 2021-09-04) - judged not relevant; reason summarised from its thread
- P4: [Block level parallel vacuum](https://commitfest.postgresql.org/patch/734/) (cf, Returned with feedback, 2016-10-03) - used in the review; reason summarised from its thread
- P5: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - used in the review
- P6: [Block level parallel vacuum](https://commitfest.postgresql.org/patch/954/) (cf, Returned with feedback, 2017-03-05) - used in the review; reason summarised from its thread
- P7: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P8: [Allow single table VACUUM in transaction block](https://commitfest.postgresql.org/patch/3969/) (cf, Returned with feedback, 2022-11-24) - judged not relevant; reason summarised from its thread
- P9: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P10: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - judged not relevant
- P11: [speeding up GIN build with parallel workers](https://commitfest.postgresql.org/patch/527/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P12: [Add a max_parallel_workers GUC](https://commitfest.postgresql.org/patch/703/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P13: [Allow VACUUM to have multiple target tables](https://commitfest.postgresql.org/patch/1275/) (cf, Rejected, 2017-09-05) - judged not relevant
- P14: [Revert "Avoid the creation of the free space map for small heap relations".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7db0cde6b58eef2ba0c70437324cbc7622230320) (revert, Reverted, 2019-05-07) - judged not relevant
- P15: [system load consideration hook for parallel workers](https://commitfest.postgresql.org/patch/725/) (cf, Rejected, 2016-09-22) - judged not relevant
- P16: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - judged not relevant
- P17: [heap_insert() and heap_update() optimization](https://commitfest.postgresql.org/patch/1830/) (cf, Rejected, 2018-10-17) - judged not relevant
- P18: [autovacuum: add possibility to change priority of vacuumed tables](https://commitfest.postgresql.org/patch/1519/) (cf, Returned with feedback, 2018-03-02) - judged not relevant
- P19: [optimizing vacuum truncation scans WIP](https://commitfest.postgresql.org/patch/220/) (cf, Returned with feedback, 2015-08-12) - judged not relevant
- P20: [Add callback table access method to reset filenode when dropping relation](https://commitfest.postgresql.org/patch/3073/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P21: [Remove size limitations of vacuums dead_tuples array](https://commitfest.postgresql.org/patch/2302/) (cf, Returned with feedback, 2019-11-30) - judged not relevant
- P22: [Revert "Avoid spurious deadlocks when upgrading a tuple lock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9d20b0ec8f2af43041b1a65e5fcd91acc47e9ace) (revert, Reverted, 2019-06-17) - judged not relevant
- P23: [Revert "Add eager and lazy freezing strategies to VACUUM."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6c6b4972664491de408670354fdbccc7c83a1c89) (revert, Reverted, 2023-01-26) - judged not relevant
- P24: [Trigger failsafe when VACUUM is run in single user mode](https://commitfest.postgresql.org/patch/3467/) (cf, Returned with feedback, 2022-04-07) - judged not relevant

### Warnings

- 45 long messages in the middle of the thread were left out of the prompts
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 182 s; cost reported by the CLI: $7.49
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 99 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:19:43.844Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
