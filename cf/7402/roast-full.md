# Design pre-review: Reduce SyncRepLock contention on the commit path

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7402](https://commitfest.postgresql.org/patch/7402/) (Needs review); authors: Yura Sokolov (funny_falcon), Vadim Ponomarev (vbp1)
- Thread: [Reduce SyncRepLock contention on the commit path](https://www.postgresql.org/message-id/flat/CANeUpr_dfdQQK6joN8STmpXy4N8cK2H_mK5cM6grwT%2BM28wzXA%40mail.gmail.com), started 2026-08-17
- Messages read: 7 of 7
- Patch prose read: v6-0001-Wake-the-released-sync-rep-waiters-after-the-queu.patch, v6-0002-Compute-the-synced-positions-before-taking-the-sy.patch, v6-0003-Release-the-sync-rep-waiters-once-per-drained-bat.patch, v6-0004-Let-a-committer-whose-acknowledgement-already-arr.patch, v6-0005-Add-stream-regress-test-for-synchronous-replicati.patch, v1-0001-Wake-the-released-sync-rep-waiters-after-the-queu.patch, v1-0002-Compute-the-synced-positions-before-taking-the-sy.patch, v1-0003-Release-the-sync-rep-waiters-once-per-drained-bat.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal, and two setups measured gains of about 9% and 4.6%. The remaining concerns are fixable design points. In 0003, a FATAL or proc_exit may drop a pending release (replication-ondisk-1), and five separate exit sites must each remember it (maintenance-fit-1). In 0004, a measured saving of about 2% pays for a new atomic (maintenance-fit-2), which the author already offered to drop. Signing off on the 0003 exit paths needs a committer who knows walsender shutdown.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the 0003 exit paths, the lock-free argument in 0002 and 0004, and the stats.sql change in 0005 are unchecked. Only 3 of 7 messages were shown.

## The proposal, restated

**Problem.** On a primary with synchronous replication, every commit that wrote WAL takes SyncRepLock exclusively in SyncRepWaitForLSN(), and every standby reply takes it in SyncRepReleaseWaiters(). Much of the work done under the lock does not need it, so committers queue on it at high commit rates.

**Proposed design.** Four independent patches move work out of the lock: wake released waiters after dropping it, compute synced positions before taking it, release waiters once per drained batch of standby replies, and let a committer skip the lock when an atomic lsn[] already covers its LSN. A fifth patch adds an opt-in stream regress test with two synchronous standbys.

**User-visible surface.**
- New PG_TEST_EXTRA keyword regress_sync_rep, which runs 027_stream_regress_sync.pl and is documented in regress.sgml.
- WalSndCtlData lsn[] becomes a 64-bit atomic, read and written through new SyncRepGetLSN() and SyncRepSetLSN() accessors.
- 0003 changes when waiters are released: once per drained batch of replies, not once per reply.

**What the author claims.**
- Two-host pgbench with 750 clients gained about 9% tps and lost about 9% mean latency with all four patches.
- LWLock/SyncRep wait samples fell by 43%; commits that blocked on the lock fell from about 10% to about 0.25% in an LWLOCK_STATS run.
- 0003 gives most of the gain; 0004 saves only 1.6-2.3% of backend lock acquisitions.
- Single-client latency did not change measurably, so deferring the release adds no delay.
- The co-author measured 67.3k to 70.4k tps on two 56-vcore virtual machines.

**Thread state.** Only the two authors have posted: Yura Sokolov reviewed, rebased and extended the series, and Vadim Ponomarev answered each round up to v6. No committer or third reviewer has replied, and both questions from the first message (missed exit paths in 0003, worth of 0004) have no outside answer.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-17; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Deferred release in 0003 may be lost on FATAL or proc_exit paths

fixable · level 2, rubric item 14 · WAL / replication / standby · `replication-ondisk-1`

A walsender stores the standby's flush position from a drained reply and marks a release as pending. The process then leaves through a FATAL or a proc_exit, for example pg_terminate_backend or postmaster death, before the drain ends. With one synchronous standby, committers whose WAL that standby already flushed keep waiting until a walsender sends a later reply.

The 0003 commit message covers four explicit returns, plus ERROR through WalSndErrorCleanup(). FATAL and proc_exit do not go through WalSndErrorCleanup(), and WalSndKill() clears the slot positions, so the acknowledgement is gone once the walsender exits. In message #1 the author asks whether an exit path was missed, and no outside reviewer has answered (messages #2 to #7). Today the release runs per reply, so this window does not exist.

Thread: [#1 (Vadim Ponomarev, 2026-08-17)](https://www.postgresql.org/message-id/CANeUpr_dfdQQK6joN8STmpXy4N8cK2H_mK5cM6grwT%2BM28wzXA%40mail.gmail.com), [#2 (Yura Sokolov, 2026-09-17)](https://www.postgresql.org/message-id/3024af42-d264-4bb8-aa58-377b286bd4b3%40postgrespro.ru), [#7 (Vadim Ponomarev, 2026-10-10)](https://www.postgresql.org/message-id/CANeUpr-5B7B%2BLnVeWeB2uJ%3DW4%3DfHKL_UahZFcakveMRrdOYhnQ%40mail.gmail.com)

**Question:** Does every way a walsender can leave the reply drain, including FATAL, proc_exit and postmaster death, either run the pending release or provably drop nothing a committer waits for?

Counter-check: held. FATAL goes straight to proc_exit without passing through WalSndErrorCleanup(), and the 0003 commit message lists no exit callback that runs the pending release. The wait ends when the standby reconnects and replies, so the cost is a delay, and fixable is the right severity.

### 2. 0003 adds an exit-path invariant that later walsender changes can break silently

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-1`

A later patch adds a message type or an early return to the reply drain in walsender.c without running the pending release. The acknowledged committers stay asleep until another reply happens to arrive. If the standby has disconnected, they hang, and the bug shows up as a rare field report of stuck commits.

Message #1 asks whether any exit path was missed, and no third reviewer has answered. The commit message lists goodbye, EOF, invalid type, unexpected type and WalSndErrorCleanup() as separate places that each must remember the release. The tests in 007_sync_rep.pl cover only today's paths and skip on Windows (message #7). So the rule depends on future authors knowing about it, and nothing fails loudly when one does not.

Thread: [#1 (Vadim Ponomarev, 2026-08-17)](https://www.postgresql.org/message-id/CANeUpr_dfdQQK6joN8STmpXy4N8cK2H_mK5cM6grwT%2BM28wzXA%40mail.gmail.com), [#6 (Vadim Ponomarev, 2026-10-06)](https://www.postgresql.org/message-id/CANeUpr_1LF1rEjPq3fEfRUSCdFkeivQv%2BGhcXSLnze4Ktq4BDA%40mail.gmail.com), [#7 (Vadim Ponomarev, 2026-10-10)](https://www.postgresql.org/message-id/CANeUpr-5B7B%2BLnVeWeB2uJ%3DW4%3DfHKL_UahZFcakveMRrdOYhnQ%40mail.gmail.com)

**Question:** Can the pending release run from one place that every exit from the drain passes through, so that a new exit path cannot skip it?

Counter-check: held. The 0003 commit message names five separate places that must each run the pending release, and the tests cover only today's exits. The proposal shows no assertion or single exit point that would catch a new early return.

### 3. 0004 adds lock-free shared state for a gain of about 2%

fixable · level 1b, rubric item 5 · maintenance burden ("who maintains this in 10 years"), performance on the hot path · `maintenance-fit-2`

On a platform with emulated 64-bit atomics, each synchronous commit reads lsn[] through a spinlock before it takes SyncRepLock, and the fast path almost never hits. Later changes to syncrep.c must keep the lock-free ordering argument correct for every lsn[] access.

Message #1 measured the fast path saving 1.6-2.3% of backend acquisitions, and the author planned to drop 0004. Message #2 kept it on the grounds that it "doesn't cost anything", but no measurement supports that. The commit message says that cost on platforms without native 64-bit atomics is untested. Ten such patches would leave many lock-free reads whose correctness each depends on a monotonicity argument, for little measured gain.

Thread: [#1 (Vadim Ponomarev, 2026-08-17)](https://www.postgresql.org/message-id/CANeUpr_dfdQQK6joN8STmpXy4N8cK2H_mK5cM6grwT%2BM28wzXA%40mail.gmail.com), [#2 (Yura Sokolov, 2026-09-17)](https://www.postgresql.org/message-id/3024af42-d264-4bb8-aa58-377b286bd4b3%40postgrespro.ru), [#3 (Vadim Ponomarev, 2026-09-22)](https://www.postgresql.org/message-id/CANeUpr9_gec4Vmck3zY%2Baw8g%3Dbfs1KvbbH_gOrisjGM1UeyA_g%40mail.gmail.com)

**Question:** Which workload makes the 0004 fast path hit often enough to justify a permanent atomic in WalSndCtlData, and what does it cost with emulated atomics?

Counter-check: held. The author measured a 1.6-2.3% saving and planned to drop 0004, and the commit message says the cost with emulated atomics is untested. The lock-free read in 2e57790836c weakens the correctness worry, but no workload shows a larger gain.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | 0001-0004 change when and how synchronous replication waiters are released in syncrep.c and walsender.c. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0004 changes WalSndCtlData lsn[] to a 64-bit atomic in walsender_private.h, a private header. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0003 adds a rule that every exit from the reply drain runs the pending release; 0004 adds lock-free shared state. |
| performance on the hot path | **yes** | SyncRepWaitForLSN() runs on every commit that wrote WAL; 0004 adds an atomic read there before the lock. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does a walsender that exits through FATAL, proc_exit or postmaster death during the reply drain run the pending release, or provably drop nothing a committer waits for?
2. Can the pending release run from one place that every exit from the reply drain passes through, so that a new early return cannot skip it?
3. Which workload or synchronous_commit level makes the 0004 fast path hit often enough to justify the atomic lsn[] in WalSndCtlData?
4. What does the 0004 read cost per commit on a platform where 64-bit atomics are emulated with a spinlock?
5. Why does 0005 change stats.sql and stats.out, which run in every regression run?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the walsender exit paths and the synchronous replication wait queue. Only the two authors have posted in the visible thread.
- **Liability:** The owner signs for no committer staying asleep after its standby acknowledged, on every walsender exit path, now and after later changes to the reply drain. With 0004, the owner also signs for the lock-free lsn[] read.
- **Author follow-up:** Good sign: the author answered each review round up to v6, and the co-author rebased and extended the series.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Users of synchronous replication at high commit rates gain about 5 to 9% throughput, and others pay nothing. The patches are small but change concurrency in syncrep.c and walsender.c, and 0003 adds an exit-path rule. 0004 can be dropped or split off without loss to 0001-0003.

## Appendix

### Refuted in the counter-check

- `demand-1` 0004 adds shared atomic state for a fast path rarely taken. Refuted (not specific): Duplicate of maintenance-fit-2. Its added claim of a torn read is weak, because lsn[] becomes an atomic type that code can only reach through the atomics API.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: [P18] records no reason and [P6] died over a WAL format change this series does not make. Message #1 says 0001 first appeared in the "Latches vs lwlock contention" thread, where the sync-rep part was not committed and, per the author, never got a separate review, so it records no rejection reason that could apply here.
- standards-compat: 0 objection(s). The series adds no SQL syntax, keyword, GUC, catalog entry, psql behaviour or dumped object, so there is nothing to round-trip or to clash with the standard. The PG_TEST_EXTRA keyword copies the existing regress_dump_restore pattern, and the lsn[] type change sits in the private walsender_private.h header. The 0003 timing change is internal. Not checked without the code: why 0005 edits stats.sql and stats.out, which run in every regression run.
- replication-ondisk: 1 objection(s). The series does not change on-disk format, WAL records, the wire protocol or catalogs. The lsn[] change in walsender_private.h is in shared memory only and lands in a major release, so it needs no pg_upgrade work and raises no backpatch question.
- security: 0 objection(s). The series has no privilege or exposure aspect. It adds no SQL surface, GUC, role, view or log output. It changes only lock handling inside syncrep.c and walsender.c, which run for replication connections that already need the REPLICATION attribute. The durability risk in 0003 and 0004, where a lost release could leave committers waiting, belongs to the correctness angle.
- maintenance-fit: 2 objection(s). 0001 and 0002 copy an existing practice (ProcArrayGroupClearXid) and keep their changes inside syncrep.c, so this angle finds no objection to them. No committer has engaged yet, but that is a committability concern, not a flaw in the design.
- already-exists: 0 objection(s). No existing feature, setting, contrib module or extension does this job: SyncRepLock is taken inside core code paths that an extension cannot reach. The local wake list in 0001 follows ProcArrayGroupClearXid(), and the general SetLatches() facility named in message #1 is not in core, so the series adds no second mechanism alongside an existing one.
- demand: 1 objection(s). Demand for the series as a whole is shown. SyncRepLock contention is a general cost of synchronous replication at high commit rates, two independent setups measured gains of about 9% and 4.6%, and the cost falls only on configurations that use synchronous replication.

### Precedents retrieved

- P1: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - judged not relevant; reason summarised from its thread
- P2: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant; reason summarised from its thread
- P3: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P4: [Improving the latch handling between logical replication launcher and worker processes.](https://commitfest.postgresql.org/patch/5095/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P5: [Report tranche ID and LWLock pointer in LWLock static tracepoints](https://commitfest.postgresql.org/patch/2927/) (cf, Returned with feedback, 2021-04-07) - judged not relevant; reason summarised from its thread
- P6: [Changing WAL Header to reduce contention during ReserveXLogInsertLocation()](https://commitfest.postgresql.org/patch/1455/) (cf, Returned with feedback, 2018-07-09) - used in the review; reason summarised from its thread
- P7: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant; reason summarised from its thread
- P8: [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (cf, Returned with feedback, 2021-10-06) - judged not relevant; reason summarised from its thread
- P9: [Add LWLock blocker(s) information](https://commitfest.postgresql.org/patch/2576/) (cf, Returned with feedback, 2020-11-18) - judged not relevant
- P10: [Quorum commit for multiple synchronous replication](https://commitfest.postgresql.org/patch/696/) (cf, Returned with feedback, 2016-10-02) - judged not relevant
- P11: [Additional LWLOCK_STATS statistics](https://commitfest.postgresql.org/patch/373/) (cf, Rejected, 2015-12-22) - judged not relevant
- P12: [LWLock optimization for Power processors](https://commitfest.postgresql.org/patch/984/) (cf, Returned with feedback, 2017-08-30) - judged not relevant
- P13: [Fix LWLock degradation on NUMA](https://commitfest.postgresql.org/patch/1166/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P14: [Make async slave to wait for lsn to be replayed](https://commitfest.postgresql.org/patch/772/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P15: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P16: [walsender.c fileheader comment](https://commitfest.postgresql.org/patch/5055/) (cf, Withdrawn, 2024-07-19) - judged not relevant
- P17: [Change walsender so that it applies per-role settings](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Standby_server_mode) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P18: [fix spinlock contention in LogwrtResult](https://commitfest.postgresql.org/patch/2716/) (cf, Withdrawn, 2022-09-23) - used in the review
- P19: [Adjust pg_stat_get_lock() prorows to match lock types](https://commitfest.postgresql.org/patch/6740/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P20: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1133/) (cf, Returned with feedback, 2018-03-30) - judged not relevant
- P21: [Optimize walsender handling invalid messages of 'drop publication'](https://commitfest.postgresql.org/patch/4393/) (cf, Withdrawn, 2023-10-20) - judged not relevant
- P22: [Fix lock contention for HASHHDR.mutex](https://commitfest.postgresql.org/patch/442/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P23: [Fix checkpointer sync request queue problems](https://commitfest.postgresql.org/patch/3583/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P24: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1666/) (cf, Returned with feedback, 2018-12-01) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0004-Let-a-committer-whose-acknowledgement-already-arr.patch not read
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (maintenance-fit-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 155 s; cost reported by the CLI: $1.81
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 66 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:50:32.624Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
