# Design pre-review: Parallel heap vacuum

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7102](https://commitfest.postgresql.org/patch/7102/); [thread](https://www.postgresql.org/message-id/flat/CAD21AoAEfCNv-GgaDheDJ%2Bs-p_Lv1H24AiJeNoPGCmZNSwL1YA%40mail.gmail.com), 137 of 137 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and several committers in the thread accept the goal. The open points are fixable: heap scan workers start by default with no separate switch (standards-compat-1, standards-compat-2), autovacuum reach is not stated (demand-1), and v21 does not state why the merged relfrozenxid stays safe after a scan restart (replication-ondisk-1). The last point needs a committer who knows vacuumlazy.c. Most of v21's text is not visible, so confidence is medium.

Not checked: The v21 code, its documentation hunks and most of the 137 messages were not read. Coverage of every unskippable block after a restart, eager scanning in parallel mode, and what autovacuum can use cannot be judged without them.

## Prior attempts

- [Block level parallel vacuum](https://www.postgresql.org/message-id/flat/CAD21AoD1xAqp4zK-Vi1cuY3feq2oO8HcpJiz32UDUfe0BE31Xw@mail.gmail.com) (similar idea; Returned with feedback, 2016-10-03). "This doesn't seem like a good design, because it adds a lot of extra index scanning work." ([message](https://www.postgresql.org/message-id/CA%2BTgmobV6%2BZPTNE3Z%2B08D9Xp7UK%2BmSq-rztOW%2B%3DRGsr5-pKiUA%40mail.gmail.com)) Different now: P4 had each worker scan a fixed heap range and then vacuum every index. v21 parallelizes only phase 1 into one shared dead TID store, keeps one worker per index, and leaves phase 3 serial.
- [Block level parallel vacuum](https://www.postgresql.org/message-id/flat/CAD21AoD1xAqp4zK-Vi1cuY3feq2oO8HcpJiz32UDUfe0BE31Xw@mail.gmail.com) (similar idea; Returned with feedback, 2017-03-05). "I think there are too many issues here to address in the limited time we have remaining this cycle, and I believe that if we try to get them all solved in the next few weeks we're likely to end up getting backed into some choices by time pressure that we may later regret bitterly." ([message](https://www.postgresql.org/message-id/CA%2BTgmobLcu_q7-0-rqiBuzowZ1J-ywDHdgBJzXdSx978JPorKA%40mail.gmail.com)) Different now: P6 changed buffer locking so several processes could wait for one cleanup lock, and added a vacuum_parallel_workers GUC. v21 does neither; workers claim disjoint 32K-block chunks and reuse the PARALLEL option.
- [Block level parallel vacuum](https://www.postgresql.org/message-id/flat/CAD21AoD1xAqp4zK-Vi1cuY3feq2oO8HcpJiz32UDUfe0BE31Xw@mail.gmail.com) (similar idea; Returned with feedback, 2017-11-30). "So I propose to mark this patch as returned with feedback for now, and come back to it once the root problems are addressed." ([message](https://www.postgresql.org/message-id/CAB7nPqQiUYFEKnUUSMahDHA1bAqDLyMW0mjFHnUub%3DTZstr64A%40mail.gmail.com)) Different now: P2 was returned because it depended on a separate extension lock patch. v21 does not depend on another lock patch and keeps the phase 1 heap scan apart from index work.
- [Revert the commits related to allowing page lock to conflict among parallel group members.](https://www.postgresql.org/message-id/flat/CAJKUy5jffnRKNvRHKQ0LynRb0RJC-o4P8Ku3x9vGAVLwDBWumQ@mail.gmail.com) (related; Reverted, 2023-07-06). "We can acquire a lock on catalogs while doing catalog look up after acquring page lock." Different now: The revert concerns page locks among parallel group members and names several workers per index as the risk. v21's phase 1 works on heap pages only and does not put several workers on one index.

## Objections a reviewer is likely to raise

### 1. Existing VACUUM commands start heap scan workers without asking

A site runs nightly `vacuum` on large tables, or `vacuum (parallel 2)` sized for its indexes. After upgrade, each run also starts heap scan workers sized by table size. These workers take slots from max_parallel_maintenance_workers and add I/O during the maintenance window, and the DBA finds out from load graphs or new VACUUM VERBOSE lines.

The documentation hunk widens PARALLEL from index phases to heap scanning, and a bare `vacuum` already defaults to choosing a degree, so the new phase is on by default. In message #98 Andres notes that nightly vacuums are common and that the feature is enabled by default. In message #97 Melanie expected a separate switch for heap parallelism and did not find one. No visible version adds such a switch.

**Question:** Why is it right for existing VACUUM invocations to gain heap scan workers by default, and how does a user keep only the old index-only parallel behavior?

<sub>fixable, level 1b, desirability; messages [#97](https://www.postgresql.org/message-id/CAAKRu_atVy1RAOwRFwEAy0is-mRPay0cxGUO%2B%2BSdOCHgB-bSKQ%40mail.gmail.com), [#98](https://www.postgresql.org/message-id/qqguhicg3gvzs6z6enomx4mpcowvlknxuwtelbhmivmkdj622s%40bgo25mtkeoxe), [#100](https://www.postgresql.org/message-id/CAAKRu_ajnk7J4LjOXqh2V1%3DFZpTMkTpKZACpZ4tCt7UR3o9tqw%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Users with the clearest need run autovacuum, which this does not reach

An operator has autovacuum stuck for hours on one large table, holding back the freeze horizon. They upgrade expecting faster heap scans, but autovacuum still scans the heap with one process. They gain only if they replace autovacuum with scheduled manual VACUUM (PARALLEL n).

Most of the demand in the thread is about autovacuum. Andres Freund describes autovacuum bound by CPU or I/O on one large table (messages #129, #133), and he called parallel vacuum without autovacuum integration a mistake not to repeat (message #98). Message #99 deferred this, and message #128 points to a separate autovacuum patch for index vacuuming only. The v21 material does not say whether autovacuum can use the parallel heap scan, so the benefit goes to manual VACUUM users while the whole project carries the code.

**Question:** Can autovacuum, including the wraparound failsafe path, use the parallel heap scan in v21, and if not, which users who run manual VACUUM asked for it?

<sub>fixable, level 1b, demand; messages [#98](https://www.postgresql.org/message-id/qqguhicg3gvzs6z6enomx4mpcowvlknxuwtelbhmivmkdj622s%40bgo25mtkeoxe), [#99](https://www.postgresql.org/message-id/CAD21AoA9eJ0Qx%3D3h77__K5ssj8R3KoVY3Uw5P7vux8HmJMRKBg%40mail.gmail.com), [#128](https://www.postgresql.org/message-id/CAD21AoAR79ai9bXLoqKyYzcYEgpOvd1y7eGqO7OGHpC_qQ%2Bi1Q%40mail.gmail.com), [#129](https://www.postgresql.org/message-id/xymq6plhuujm6rpb3hyqpwqkqo3q6kdx2a6nnei6sgpbzzj7lb%40ejwunmymqgxb), [#133](https://www.postgresql.org/message-id/toxt6ppr2swqrclx5kni3hdy4c2xqcdbojup2xwjlgsui3mwp7%40wqg3liamu44k); `demand-1`</sub>

### 3. Measured gain is large only for tables without indexes or with warm cache

A DBA with a typical indexed table on network storage runs VACUUM (PARALLEL 8). Index vacuuming and phase 3 dominate the run time, so total time improves about 1.1x while eight workers compete with the user workload. They see little benefit and do not know which tables would gain more.

The v21 numbers in message #137 show total VACUUM gains of 1.1x to 1.5x once indexes exist, with warm caches and local NVMe. Tomas Vondra measured gains only without indexes and asked which workloads would benefit (message #122). Cold-cache phase 1 scaled less than expected (message #60), and message #137 has no cold-cache results. The remaining answer is INDEX_CLEANUP off for fast freezing (message #128), but no field report shows how often users run that by hand.

**Question:** Which production workloads, such as large index-free tables or manual INDEX_CLEANUP off freezing runs, are reported to need this, and what do cold-cache results on network storage show?

<sub>fixable, level 1b, demand; messages [#60](https://www.postgresql.org/message-id/CAD21AoAHoU_dj0AJcTVRY%3DzL1jOcnkcihh9Y_Xuw7%2BEhBT9a5Q%40mail.gmail.com), [#122](https://www.postgresql.org/message-id/df858858-72da-4649-aaf9-ab97e4fb2d6f%40vondra.me), [#128](https://www.postgresql.org/message-id/CAD21AoAR79ai9bXLoqKyYzcYEgpOvd1y7eGqO7OGHpC_qQ%2Bi1Q%40mail.gmail.com), [#137](https://www.postgresql.org/message-id/CALj2ACWtW0jCRddhpPG2t_ahX87fkbxZXj1ZKKPh3AnCT7gWjQ%40mail.gmail.com); `demand-2`</sub>

### 4. relfrozenxid advance across workers and scan restarts lacks a stated invariant

A large table fills maintenance_work_mem, so vacuum pauses phase 1 for index and heap vacuuming, then relaunches fewer workers. If a partly claimed 32K-block chunk is never finished, the leader can still store a merged relfrozenxid that is newer than unfrozen XIDs in those blocks. Nobody notices until clog truncation or wraparound, when those tuples read as invisible or fail.

The leader sets pg_class.relfrozenxid and relminmxid from the minimum of per-worker values, and the reported scanned_pages and skippedallvis decide whether it advances at all. Message #82 and message #108 show that resuming after a TID store cycle with fewer workers needed special takeover logic, and message #90 found a block-assignment bug in the skip path. Message #137 redesigns the scan with fixed chunks in vacuumlazy.c but does not say how every unskippable block is still covered. Melanie asked in message #90 for the reasoning to be written down because the merge is the place where a mistake leads to lost data.

**Question:** In v21, what guarantees that every block not skipped through the VM is scanned before the merged relfrozenxid is stored, including after a phase 1 restart with fewer workers?

<sub>fixable, level 2, design; messages [#82](https://www.postgresql.org/message-id/CAD21AoC%2Bb8An4%3DbiuSCurxv1cTgfDyQzmBvdR%2BgisRj-qq5U_w%40mail.gmail.com), [#90](https://www.postgresql.org/message-id/CAAKRu_aS_k751daj0QfO%2B5ZgLDz8LP1Wo_b5%3DRRKt6%3Df6OStLg%40mail.gmail.com), [#108](https://www.postgresql.org/message-id/CAD21AoB2zAeGjAARs5uExy66ZrYpfL5h_PQHQuCE6Dm%3D3rvSUw%40mail.gmail.com), [#137](https://www.postgresql.org/message-id/CALj2ACWtW0jCRddhpPG2t_ahX87fkbxZXj1ZKKPh3AnCT7gWjQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 5. Turning off heap scan workers also changes parallel seq scan plans

A DBA wants vacuum to stop using heap scan workers on one large table. The only knob given is min_parallel_table_scan_size, so the DBA raises it. Queries on all tables then lose parallel sequential scans, and the DBA finds out when reports run slower.

In message #99 the author names min_parallel_table_scan_size as the way to disable the parallel heap scan. In message #100 Melanie notes that this setting also drives parallel sequential scans and can only be set globally, and she asks for per-phase control. Nothing visible after that adds a reloption or a VACUUM option for heap scan parallelism, and the brief lists this as open. Reusing a planner setting to control a maintenance phase ties two unrelated behaviors to one knob.

**Question:** How can a user disable or limit parallel heap scanning for one table or one VACUUM run without changing parallel query planning?

<sub>fixable, level 2, new GUC; messages [#99](https://www.postgresql.org/message-id/CAD21AoA9eJ0Qx%3D3h77__K5ssj8R3KoVY3Uw5P7vux8HmJMRKBg%40mail.gmail.com), [#100](https://www.postgresql.org/message-id/CAAKRu_ajnk7J4LjOXqh2V1%3DFZpTMkTpKZACpZ4tCt7UR3o9tqw%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. In v21, what guarantees that every block not skipped through the VM is scanned before the merged relfrozenxid is stored, including after a phase 1 restart with fewer workers?
2. How does a user keep parallel index vacuuming but turn off the parallel heap scan for one table or one VACUUM run, without changing min_parallel_table_scan_size?
3. Why is it right for an existing plain `vacuum` or `vacuum (parallel n)` to start heap scan workers by default after upgrade?
4. Can autovacuum, including the wraparound failsafe path, use the v21 parallel heap scan, and if not, what in the design allows adding that later?
5. What do cold-cache results on network storage show for v21, with and without indexes?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-1: relfrozenxid advance across workers and scan restarts lacks a stated invariant
- **catalog changes:** see objection replication-ondisk-1: relfrozenxid advance across workers and scan restarts lacks a stated invariant
- **ABI / extension API:** v18 added five table AM callbacks in tableam.h; v21 drops them and splits helpers out of vacuumparallel.c instead.
- **new GUC:** No new GUC, but min_parallel_table_scan_size gains a second meaning as the threshold for the parallel heap scan in vacuum.
- **planner behaviour change:** see objection standards-compat-2: Turning off heap scan workers also changes parallel seq scan plans
- **backpatchability:** see objection standards-compat-1: Existing VACUUM commands start heap scan workers without asking
- **maintenance burden ("who maintains this in 10 years"):** vacuumlazy.c gains a parallel phase 1 path with a leader-side merge of relfrozenxid and relminmxid next to the serial path.
- **performance on the hot path:** Workers take an exclusive lock on the shared dead TID store; the author reports 11% to 17% of phase 1 at 4 to 8 workers.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Users with the clearest need run autovacuum, which this does not reach

## Who would commit it, and when

- Owner: qualified committer engaged. Masahiko Sawada, a committer, started the thread and co-authors the patch. A second reviewer who knows the phase 1 loop in vacuumlazy.c, eager scanning and relfrozenxid tracking should sign off on v21.
- Priority: proceed; audience a sizeable minority; review cost high. Large tables with slow vacuum are a known problem and reviewers in the thread accept the goal, but v21 shows large gains mainly without indexes. The patch adds a second phase 1 path in vacuumlazy.c with shared state, so review needs area experts; the control switch and the relfrozenxid invariant should be settled during that review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
