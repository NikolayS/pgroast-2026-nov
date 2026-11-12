# Design pre-review: LWLock granular partition lock memory layout

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7429](https://commitfest.postgresql.org/patch/7429/) (Needs review); authors: Alexandre Felipe (afelipe)
- Thread: [LWLock granular partition lock memory layout](https://www.postgresql.org/message-id/flat/CAE8JnxOGU4pT19z6JRVW9W0inS1cXt_pRPpCN0dAdTueo_GVkg%40mail.gmail.com), started 2026-10-06
- Messages read: 5 of 5
- Patch prose read: benchmark.txt, v1-0002-introduce-LWLockAligned.patch, v1-0001-LWLock-struct-layout.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The proposal fails at level 1b on evidence. demand-1: message #3 asked for numbers from realistic workloads, and message #4 says the effect is not visible on one on the author's machine. already-exists-1: the thread never compares the patch with unpatched Postgres that has S times more padded partitions, which costs a few KiB. The idea may be sound, but the list needs both answers before design review.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the change in pg_config_manual.h, the value of S and the fate of the old offset macros are unknown. The level 2 objections on exclusive-heavy false sharing and on the second lock stride were not evaluated.

## The proposal, restated

**Problem.** Partitioned hash tables in shared memory use an array of LWLocks, each padded to a full cache line to avoid false sharing. The author argues that the same memory could hold more partition locks, giving more concurrency and less contention. The thread does not name a workload or user that suffers from the current layout.

**Proposed design.** Patch 0001 declares the main LWLock partition groups as struct fields so offsets come from the struct layout instead of recursive macros, without changing allocation. Patch 0002 adds an LWLockAligned type smaller than LWLockPadded, sized as a divisor of the cache line, so S partition locks share one cache line. Individual (unpartitioned) LWLocks keep full padding.

**User-visible surface.**
- New LWLockAligned type in storage/lwlock.h, visible to extensions using the C API.
- Changed declaration of partition lock groups in lwlocklist.h, lock.h and buf_internals.h, which code computing partition lock offsets may depend on.
- Changes in pg_config_manual.h; the material does not say which setting is added or changed.

**What the author claims.**
- Packing S partition locks per cache line raises LW_EXCLUSIVE concurrency by about a factor S, with no added cache-line traffic per operation.
- It reduces contention on LWLockWaitListLock, CAS retries in LWLockAttemptLock, and the number of backends that sleep.
- "In theory this has benefits with no compromises."
- pgbench on an 8-core laptop with 128 partitions: about 26% more throughput at 8 clients, no change at 1 client.
- The author expects larger gains on multi-socket machines but did not measure them.

**Thread state.** Nathan Bossart asked for numbers from realistic workloads and could not tell what benchmark.txt shows. The author replied with one pgbench result at 1 and 8 clients; no one has replied to it.

## 2. Prior attempts

### Cache line alignment of PGXACT (related)

- **Link:** [https://commitfest.postgresql.org/patch/974/](https://commitfest.postgresql.org/patch/974/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAPpHfdtJY4zOEDsjad6J5AyZMqZcv6gSY9AkKpA7qN3jyQ2+1Q@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAPpHfdtJY4zOEDsjad6J5AyZMqZcv6gSY9AkKpA7qN3jyQ2+1Q@mail.gmail.com)
- **Date:** decided 2017-09-22; started 2016-08-19
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Alexander Korotkov, Ashutosh Sharma (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Benchmarks showed read-only gains but regressions on read-write and mixed workloads, and the author judged the read-write loss about as large as the read-only gain. The author also found that padding PGPROC, or another unrelated alignment change, gave the same read-only effect, so it was unclear the patch itself was needed. The author returned the patch for more investigation.
- **Reason (quote):** "It became clear that this patch causes regression." ([message](https://www.postgresql.org/message-id/CAPpHfdtrbEFfT9BQ5D762QSdKhkFTewJ_%2BvEC%3DL6oHCu5JcE8w%40mail.gmail.com))
- **What is different this time:** P6 added cache-line alignment to PGXACT entries; this proposal reduces padding of partition LWLocks. P6 supports only the request for read-write runs and a layout control.


## 3. Objections a reviewer is likely to raise

### 1. No user or workload shown to suffer from padded partition locks

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer changes the layout of shared lock arrays, the buffer mapping and lock manager locks among them, based on one laptop pgbench run. No user has reported contention on these locks, so nothing shows that anyone in the field gains. If the multi-socket gain never appears, the project keeps the extra layout complexity for nothing.

In message #3 Nathan Bossart asked for numbers from realistic workloads. In message #4 the author says the effect cannot be seen on a realistic workload on the test machine and only expects it on multi-socket hosts. Message #5 gives one run at 1 and 8 clients, without the script, scale, mode or which partitioned locks changed. The thread has no field report, no wait-event data and no second person asking for this.

Thread: [#3 (Nathan Bossart, 2026-10-07)](https://www.postgresql.org/message-id/asaVvxCLAu0t372f%40nathan), [#4 (Alexandre Felipe, 2026-10-07)](https://www.postgresql.org/message-id/CAE8JnxPhhbyajhri9X6YRXX0WuR%2BtHt1D0AOea-Rk3j_i5YJKA%40mail.gmail.com), [#5 (Alexandre Felipe, 2026-10-10)](https://www.postgresql.org/message-id/CAE8JnxO699xhiRx%3DQjBdO3V8wYz32yTkFdhq0aiqa4iXTXwV6w%40mail.gmail.com)

**Question:** On which production-like workload and hardware do partition LWLock waits show up in pg_stat_activity wait events, and how much does the patch reduce them?

Counter-check: held. Message #4 says the effect is not visible on a realistic workload on the author's machine, and the thread has no field report or wait-event data. The single pgbench result in message #5 shows a gain but not a user who needs it.

### 2. Raising the existing partition-count constants may give the same gain

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), ABI / extension API · `already-exists-1`

A committer compares the patch with a one-line change to LOG2_NUM_LOCK_PARTITIONS or NUM_BUFFER_PARTITIONS, which keeps every lock on its own padded cache line. If that matches the 26% gain, the project has added a second LWLock layout type and a new struct layout for a few KiB of shared memory.

Partition counts are already compile-time constants in lwlock.h. The author's model compares S packed locks against one padded lock, but the plain alternative is S padded locks on S cache lines, which has no false sharing. With 128 partitions that alternative costs about 8 KiB more per S step. Message #5 compares patched against base at the same count and never against base with S times more partitions.

**Precedents.**
- [Cache line alignment of PGXACT](https://commitfest.postgresql.org/patch/974/) (Returned with feedback, 2017-09-22); thread: [link](https://www.postgresql.org/message-id/flat/CAPpHfdtJY4zOEDsjad6J5AyZMqZcv6gSY9AkKpA7qN3jyQ2+1Q@mail.gmail.com)

Thread: [#1 (Alexandre Felipe, 2026-10-06)](https://www.postgresql.org/message-id/CAE8JnxOGU4pT19z6JRVW9W0inS1cXt_pRPpCN0dAdTueo_GVkg%40mail.gmail.com), [#4 (Alexandre Felipe, 2026-10-07)](https://www.postgresql.org/message-id/CAE8JnxPhhbyajhri9X6YRXX0WuR%2BtHt1D0AOea-Rk3j_i5YJKA%40mail.gmail.com), [#5 (Alexandre Felipe, 2026-10-10)](https://www.postgresql.org/message-id/CAE8JnxO699xhiRx%3DQjBdO3V8wYz32yTkFdhq0aiqa4iXTXwV6w%40mail.gmail.com)

**Question:** With the same pgbench run, how does the patch compare with unpatched Postgres where the partition count is raised by a factor of S, and what is the memory difference?

Counter-check: held. The thread compares patched against base only, never against base with S times more padded partitions, which costs a few KiB. [P6] is about PGXACT alignment and must not be used as support here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Patch 0002 adds LWLockAligned to lwlock.h, and 0001 changes partition lock declarations in lock.h, buf_internals.h and lwlocklist.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second lock stride sits next to LWLockPadded, and each future partitioned lock group must choose one. |
| performance on the hot path | **yes** | Partition LWLocks, which sit on the lock acquire and release path, would share cache lines. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Raising the existing compile-time partition counts with padded locks may give the same gain. |

## 5. Questions the author must answer before anyone reviews code

1. On which realistic workload and hardware do partition LWLock waits show up in wait events, and how much does the patch reduce them?
2. With the same pgbench run, how does the patch compare with unpatched Postgres whose partition count is raised by a factor of S, and what is the memory difference?
3. What were the pgbench script, scale, mode and number of runs behind the 26% result at 8 clients?
4. Which partitioned locks change (buffer mapping, lock manager, predicate lock), and does the number of partitions change or only the memory layout?
5. What do read-write and exclusive-heavy runs show on a multi-socket machine, where locks on one cache line may cause false sharing?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the LWLock code and the layout of the main lock array. The one reviewer in the thread only asked for numbers.
- **Liability:** The owner signs for the performance of the buffer mapping and lock manager partition locks on all hardware, including multi-socket machines that nobody measured, for every later release.
- **Author follow-up:** The author posted a corrected patch one day after the first and answered the request for numbers with a pgbench result.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: Repeatable results on a many-core or multi-socket machine, with script, scale, mode and run count stated, for read-only and read-write loads. They must show a gain over both unpatched Postgres and unpatched Postgres with S times more padded partitions.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** park
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patches are small, but they change the layout of core lock arrays, and judging them needs benchmark work on large machines. The thread shows one laptop pgbench result and no workload that suffers today, so review time is better spent once the measurements exist.

## Appendix

### Held up in the counter-check, not listed above

- `prior-attempts-1` (fixable, level 2) Layout-only gain shown on one workload, read-write and exclusive cases unmeasured. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Second lock stride makes partition offsets easy to get wrong. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-1` Packed partition locks add cross-partition cache line traffic under write load. Refuted (not specific): Duplicate of prior-attempts-1 for the unmeasured exclusive-heavy case and of already-exists-1 for the comparison with more padded partitions. Its extra-traffic claim assumes fewer cache lines, but the 0002 commit message describes the same memory holding more locks.
- `demand-2` Cost lands on all users and extensions, gain is unmeasured. Refuted (not specific): Duplicate of prior-attempts-1 for the unmeasured write-heavy risk and of maintenance-fit-2 for the extension cost. The who-pays framing also fits poorly, because a layout change that is not opt-in gives any gain to all users too.

### Review angles

- prior-attempts: 1 objection(s). P16 shares the contention topic but records no reason, so it supports nothing here. The thread does not point to any earlier discussion of packing partition locks.
- standards-compat: 0 objection(s). The proposal changes only the shared-memory layout of partition LWLocks, with no SQL syntax, GUC, catalog object, dump content or client-visible behaviour, so nothing in the standards and compatibility angle fails. Changing the C-level lock array declarations is normal in a major release; whether extensions that index MainLWLockArray directly must change, and what pg_config_manual.h gains, cannot be checked without the code.
- replication-ondisk: 0 objection(s). The proposal changes only the shared memory layout of partition LWLocks, which is rebuilt at every server start. It touches no on-disk format, WAL record, catalog, replication path or wire protocol, so pg_upgrade and standbys are unaffected. Extensions that index MainLWLockArray directly with the partition offset macros may need source changes in a major release. That is normal for master-only changes, and the material does not show such a dependency.
- security: 0 objection(s). The proposal changes only the shared-memory layout of partition LWLocks. It adds no SQL surface, privilege, role, setting reachable by users, monitoring view or authentication path, so no new action or information becomes available to a non-superuser. False sharing between packed locks is a performance question for other panel members, and it gives an unprivileged user no denial-of-service path beyond the lock contention ordinary queries already cause.
- maintenance-fit: 2 objection(s). Two objections apply from this angle. No committer who knows the LWLock area has engaged beyond asking for numbers, but that is a committability point and not counted as an objection.
- already-exists: 1 objection(s). One objection from this angle. No extension or hook covers LWLock memory layout, so the only existing alternative is raising the partition-count constants.
- demand: 2 objection(s). Not applicable; objections raised.

### Precedents retrieved

- P1: [Revert "Don't lock partitions pruned by initial pruning"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1722d5eb05d8e5d2e064cd1798abcae4f296ca9d) (revert, Reverted, 2025-05-22) - judged not relevant
- P2: [Delay locking partitions during query execution](https://commitfest.postgresql.org/patch/1897/) (cf, Withdrawn, 2019-03-05) - judged not relevant; reason summarised from its thread
- P3: [Revert lowering of lock level for ATTACH PARTITION](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8ca984b2c739e096c08f260f477792495f4dfe4) (revert, Reverted, 2018-04-12) - judged not relevant
- P4: [Revert "Move PartitionPruneInfo out of plan nodes into PlannedStmt"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5472743d9e8583638a897b47558066167cc14583) (revert, Reverted, 2023-05-04) - judged not relevant
- P5: [Lazy JIT IR code generation to increase JIT speed with partitions](https://commitfest.postgresql.org/patch/3071/) (cf, Withdrawn, 2023-01-27) - judged not relevant; reason summarised from its thread
- P6: [Cache line alignment of PGXACT](https://commitfest.postgresql.org/patch/974/) (cf, Returned with feedback, 2017-09-22) - used in the review; reason summarised from its thread
- P7: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P8: [Revert an ill-considered portion of my patch of 12-Mar, which tried to save a few lines in sql_exec_error_callback() by using the function source string...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6775c0108039a13458004d893e4a17b4ae3fae9d) (revert, Reverted, 2007-06-17) - judged not relevant
- P9: [Fix pg_rewind false positives caused by shutdown-only WAL](https://commitfest.postgresql.org/patch/6041/) (cf, Withdrawn, 2026-05-01) - judged not relevant; reason summarised from its thread
- P10: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P11: [hashjoin - gracefully increasing NTUP_PER_BUCKET instead of batching](https://commitfest.postgresql.org/patch/28/) (cf, Rejected, 2015-02-13) - judged not relevant; reason summarised from its thread
- P12: [Optimize numeric comparison and aggregation via packed-datum extraction](https://commitfest.postgresql.org/patch/6651/) (cf, Rejected, 2026-05-10) - judged not relevant; reason summarised from its thread
- P13: [Revert workarounds for -Wmissing-braces false positives on old GCC](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=618c64ffd3967cb5313b4b11e1e1043a074f2139) (revert, Reverted, 2025-03-20) - judged not relevant
- P14: [Reducing serialized size of Nodes from nodeToString output](https://commitfest.postgresql.org/patch/4685/) (cf, Returned with feedback, 2024-04-04) - judged not relevant; reason summarised from its thread
- P15: [Reduce size of fmgr_builtins array by 1/3](https://commitfest.postgresql.org/patch/2411/) (cf, Rejected, 2020-01-18) - judged not relevant
- P16: [Fix lock contention for HASHHDR.mutex](https://commitfest.postgresql.org/patch/442/) (cf, Returned with feedback, 2016-02-08) - used in the review
- P17: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P18: [Allow has_table_previlege to return false when user has no previlege on namespace](https://commitfest.postgresql.org/patch/1783/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P19: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1133/) (cf, Returned with feedback, 2018-03-30) - judged not relevant
- P20: [XLog size reductions: smaller XLogRecordBlockHeader](https://commitfest.postgresql.org/patch/4324/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P21: [Revert idea of zer-padding padding session id in log_line_prefix](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b06e6aba8773df8040b357a1cb058879ed2d94f) (revert, Reverted, 2013-05-06) - judged not relevant
- P22: [Revert "Add missing padding from MemoryChunk struct"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5495796ad12aae971137d184ff25add8f6b03968) (revert, Reverted, 2022-08-30) - judged not relevant
- P23: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1666/) (cf, Returned with feedback, 2018-12-01) - judged not relevant
- P24: [Revert "Don't truncate database and user names in startup packets."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0ff56e2d3ff1db3de727b33b2dac985ccc43ef8) (revert, Reverted, 2024-12-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 131 s; cost reported by the CLI: $1.52
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 80 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:54:48.033Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
