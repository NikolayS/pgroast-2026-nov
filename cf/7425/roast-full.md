# Design pre-review: Unlocked path for GetSnapshotDataReuse

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7425](https://commitfest.postgresql.org/patch/7425/) (Needs review); authors: Matthias van de Meent (mmeent)
- Thread: [POC: Unlocked path for GetSnapshotDataReuse](https://www.postgresql.org/message-id/flat/CAEze2Wjj5LPNpuq6fNy-wv552wbk61dGRJ52NdV%2BWddq0Uf33A%40mail.gmail.com), started 2026-08-03
- Messages read: 7 of 7
- Patch prose read: v2-0003-procarray-Implement-full-lockless-snapshot-reuse.patch, v2-0002-procarray-Implement-optimistic-lockless-snapshot-.patch, v2-0001-Split-out-TransamVariablesData-from-transam.h.patch, v1-0002-nocfbot.patch, v1-0001-Draft-Unlocked-path-for-GetSnapshotDataReuse.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal, and a committer who knows this code supports it in message #3. Three fixable design points remain: the hot standby argument for 0003 (replication-ondisk-1), the undocumented pairing of a plain atomic write with the barrier in LWLockRelease (maintenance-fit-1), and the missing benchmarks (maintenance-fit-2). None of them stops review; each needs sign-off from someone who knows procarray.c.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the barrier placement, the order of the xmin write and the second counter read, and the hot standby paths could not be checked. The safety of 0003 rests on arguments in messages #5 and #6 that this review saw only through the brief and the objections.

## The proposal, restated

**Problem.** GetSnapshotData() takes ProcArrayLock in shared mode even when it only checks that a cached snapshot is still valid. That shared acquisition writes to the lock's cache line, which costs time on busy systems with many backends. The author's test-suite count shows about half of calls could skip the lock.

**Proposed design.** An atomic read of TransamVariables->xactCompletionCount replaces the locked check when a backend reuses its cached snapshot. 0002 does this only when MyProc->xmin is already set. 0003 also installs the snapshot's xmin without the lock, then reads the counter again and falls back to the locked path if it changed. 0001 moves TransamVariablesData into a new varsup.h so it can hold atomic types.

**User-visible surface.**
- New header src/include/access/varsup.h holds TransamVariablesData, and extensions that use TransamVariables must include it.
- TransamVariables->xactCompletionCount becomes an atomic type, so C code that reads it must change.

**What the author claims.**
- About 50% of GetSnapshotData() calls in the regression suite take the lockless path in v1, and 37% miss it only because no xmin is set.
- The change saves write traffic on the ProcArrayLock cache line in some workloads.
- Setting xmin without an exclusive lock is safe, because only the owning backend writes its xmin, already under LW_SHARED today.
- With 0003, vacuum horizons can move backward in more cases, and the author treats this as acceptable because other code already does it.
- Commit 623a9ba79b already pointed to this optimization.

**Thread state.** Andres Freund and Bertrand Drouvot support the idea and accept the xmin reasoning behind 0003. Open points: documenting the pairing of a plain atomic write with a membarrier read, a cheaper first read, and benchmarks.

## 2. Prior attempts

### POC: Cache data in GetSnapshotData() (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/553/](https://commitfest.postgresql.org/patch/553/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAD__OugYQmNP5dFkGdSjP48B8WjD0+rtff3iupThnQxTXLEUog@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAD__OugYQmNP5dFkGdSjP48B8WjD0+rtff3iupThnQxTXLEUog@mail.gmail.com)
- **Date:** decided 2017-09-29; started 2016-01-15
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** No closing message is shown. In 2016 Robert Haas returned the patch until the CLOG contention work was settled. At the 2017 decision three issues were open: the read-only gain appeared only on machines with more than two sockets, Robert Haas called the small regression at 1 client concerning, and the author had no way yet to skip caching when quick write transactions invalidate the cache before it is reused.
- **Reason (quote):** "As Andres has suggested previously I need a mechanism to identify and avoid caching for such scenarios." ([message](https://www.postgresql.org/message-id/CAD__OujRZEjE5y3vfmmZmSSr3oYGZSHRxwDwF7kyhBHB2BpW_g%40mail.gmail.com))
- **What is different this time:** P3 shared one snapshot across backends in shared memory and paid for the cache when writes invalidated it. This proposal keeps the existing per-backend cache and only drops the lock from the validity check, with a fallback to the locked path.

### Move PinBuffer and UnpinBuffer to atomics (related)

- **Link:** [https://commitfest.postgresql.org/patch/370/](https://commitfest.postgresql.org/patch/370/)
- **Thread:** [https://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell](https://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell)
- **Date:** decided 2015-12-16; started 2015-09-11
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Andres Freund (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Andres Freund judged the first patch incorrect. PinBuffer no longer took the header lock, but StrategyGetBuffer, SyncOneBuffer and BufferAlloc still relied on that lock for mutual exclusion. Work moved to the single atomic state version, which Alexander Korotkov refactored and others benchmarked, and it was still being cleaned up when the entry closed. No visible message states why the entry was returned.
- **Reason (quote):** "Which you made ineffective because PinBuffer() doesn't take a lock anymore. Mutual exclusion through locks only works if all participants take the locks." ([message](https://www.postgresql.org/message-id/20150914154142.GB9666%40alap3.anarazel.de))
- **What is different this time:** P2 removed a lock that other code still relied on for mutual exclusion. Here only the owning backend writes its xmin, and a second read of the counter replaces the exclusion against committers.

### Revert changes to CONCURRENTLY that "sped up" Xmin advance (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b)
- **Thread:** [https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org), [https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql](https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql)
- **Date:** decided 2022-05-31; started 2021-02-23
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit d9d076222f5b "VACUUM: ignore indexing operations with CONCURRENTLY". These changes caused indexes created with the CONCURRENTLY option to miss heap tuples that were HOT-updated and HOT-pruned during the index creation. Before these changes, HOT pruning would have been prevented by the Xmin of the transaction creating the index, but because this change was precisely to allow the Xmin to move forward ignoring that backend, now other backends scanning the table can prune them. This is not a problem for VACUUM (which requires a lock that conflicts with a CREATE INDEX CONCURRENTLY operation), but HOT-prune can definitely occur. In other words, Xmin advancement was sped up, but at the cost of corrupting the resulting index. Regrettably, this means that the new feature in PG14 that RIC/CIC on very large tables no longer force VACUUM to retain very old tuples goes away. We might try to implement it again in a later release, but for now the risk of indexes missing tuples is too high and there's no easy fix. Backpatch to 14, where this change appeared. Diagnosys-by: Andrey Borodin <x4mmm@yandex-team.ru> Diagnosys-by: Michael Paquier <michael@paquier.xyz> Diagnosys-by: Andres Freund <andres@anarazel.de>
- **Reason (quote):** "In other words, Xmin advancement was sped up, but at the cost of corrupting the resulting index."
- **What is different this time:** P11 let horizons ignore a backend's xmin. 0003 installs the xmin without the lock and rechecks the counter, which is a different mechanism.


## 3. Objections a reviewer is likely to raise

### 1. Lockless xmin install in 0003 has no stated argument for hot standby

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-1`

On a hot standby, a backend with a cached snapshot and no xmin installs that snapshot's old xmin without ProcArrayLock while the startup process replays a prune record. The startup process checks for conflicts, sees no xmin on that backend, and removes the tuples. The backend's counter check passes because no completion was replayed, so the query reads pages missing rows it should see.

The safety argument in message #5 covers only primary-side writers: owner-only writes to xmin, plus transaction completion bumping xactCompletionCount. On a standby, other events advance horizons: prune replay with recovery conflicts, and KnownAssignedXids expiry from running-xacts records for xids that never got an abort record. The lockless path is safe only if every such event either bumps xactCompletionCount or cannot pass an xmin that a valid reused snapshot still needs. Message #6 accepts the reasoning without discussing standby, and the patch adds no rule that future standby code must follow.

**Precedents.**
- [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (Reverted, 2022-05-31); thread: [#1](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org), [#2](https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql)

Thread: [#5 (Matthias van de Meent, 2026-10-09)](https://www.postgresql.org/message-id/CAEze2WgQsX%3Ds%3DXpmSf7q7-rHhWmMJaSBea7xF%2BOevXE29KaxEg%40mail.gmail.com), [#6 (Bertrand Drouvot, 2026-10-12)](https://www.postgresql.org/message-id/asym8dZmZM3vd7AV%40bdtpg)

**Question:** On a hot standby, which replay events can remove data a reused snapshot needs without bumping xactCompletionCount, and why does 0003 still never return wrong results?

Counter-check: held. The thread gives no argument for hot standby, so the request stands. The scenario is weaker than stated, because the conflict check and GetSnapshotData both take ProcArrayLock in shared mode today, and [P11] concerns a different mechanism and must not be used as support.

### 2. Counter correctness depends on an undocumented barrier in LWLockRelease

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch moves an xactCompletionCount increment outside the ProcArrayLock exclusive section, for example into group clear or a new completion path, or changes LWLockRelease ordering. The lockless reader can then see a stale count and reuse a snapshot that misses a commit. No test fails; users see wrong visibility results long after release.

Message #6 notes that IncrementXactCompletionCount() uses a plain pg_atomic_write_u64() and relies on the full barrier in the following LWLockRelease(ProcArrayLock), and that no other code in the tree uses this pairing. Message #3 asks for the same pairing to be documented. Today the Assert on LW_EXCLUSIVE guards the lock, but nothing ties the read side's correctness to that release barrier. Each future completion path must preserve an invariant that is visible only in comments.

**Precedents.**
- [Move PinBuffer and UnpinBuffer to atomics](https://commitfest.postgresql.org/patch/370/) (Returned with feedback, 2015-12-16); thread: [link](https://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell)

Thread: [#3 (Andres Freund, 2026-10-09)](https://www.postgresql.org/message-id/lgelgkjrkvnrqb2mfbgcx3fnr3cdr6f2tyanfvhsiyfduw5njz%40okoelncmxfmo), [#6 (Bertrand Drouvot, 2026-10-12)](https://www.postgresql.org/message-id/asym8dZmZM3vd7AV%40bdtpg)

**Question:** What fails loudly, by assertion or by API shape, if a future increment site stops being followed by a barrier-carrying release?

Counter-check: held. Message #6 asks for the pairing of the plain write with the barrier in LWLockRelease to be documented, and the thread has no answer yet. The assert on LW_EXCLUSIVE covers part of the scenario, and [P2] is about a different error and adds no support.

### 3. No measurement of the new path, including membarrier reads on ARM

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-2`

On a weakly ordered platform, every snapshot taken by every backend now does one or two pg_atomic_read_membarrier_u64() calls. A workload with frequent commits fails the counter check and takes the lock anyway, so it pays for both. An OLTP user on ARM finds a regression only after upgrading.

The only numbers are call counts from the regression suite (message #1); in message #5 the author asserts that membarrier reads cost less than the lock acquisition but shows no measurement. Messages #2 and #6 ask for benchmarks, and message #6 proposes a cheaper first read. GetSnapshotData() is on the path of nearly every query, so the cost falls on all users. In [P3], snapshot reuse work stalled on a regression at low client counts and on workloads where commits invalidate the cache before it is reused.

**Precedents.**
- [POC: Cache data in GetSnapshotData()](https://commitfest.postgresql.org/patch/553/) (Returned with feedback, 2017-09-29); thread: [link](https://www.postgresql.org/message-id/flat/CAD__OugYQmNP5dFkGdSjP48B8WjD0+rtff3iupThnQxTXLEUog@mail.gmail.com)

Thread: [#1 (Matthias van de Meent, 2026-08-03)](https://www.postgresql.org/message-id/CAEze2Wjj5LPNpuq6fNy-wv552wbk61dGRJ52NdV%2BWddq0Uf33A%40mail.gmail.com), [#2 (Bertrand Drouvot, 2026-10-09)](https://www.postgresql.org/message-id/asiCYBUj0RQ6FA5C%40bdtpg), [#5 (Matthias van de Meent, 2026-10-09)](https://www.postgresql.org/message-id/CAEze2WgQsX%3Ds%3DXpmSf7q7-rHhWmMJaSBea7xF%2BOevXE29KaxEg%40mail.gmail.com), [#6 (Bertrand Drouvot, 2026-10-12)](https://www.postgresql.org/message-id/asym8dZmZM3vd7AV%40bdtpg)

**Question:** What throughput do x86 and ARM show at 1 client and at high client counts, for read-only and commit-heavy pgbench runs?

Counter-check: held. The thread shows only call counts, and messages #2 and #6 ask for benchmarks that no later message provides. [P3] cached snapshots in shared memory, a different design, so it must not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | 0003 installs xmin without ProcArrayLock, and the thread gives no safety argument for hot standby backends. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0001 moves TransamVariablesData to a new varsup.h, and xactCompletionCount becomes an atomic type. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The lockless read relies on every xactCompletionCount increment being followed by the barrier in LWLockRelease(ProcArrayLock). |
| performance on the hot path | **yes** | GetSnapshotData() runs for nearly every query, and the new path adds membarrier reads of the counter. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. On a hot standby, which replay events can remove data a reused snapshot needs without bumping xactCompletionCount, and why does 0003 still never return wrong results?
2. What fails loudly, by assertion or by API shape, if a future xactCompletionCount increment is no longer followed by a barrier-carrying LWLockRelease?
3. What throughput do x86 and ARM show at 1 client and at high client counts, for read-only and commit-heavy pgbench runs?
4. How often do vacuum horizons move backward with 0003, and which horizon consumers were checked against that?
5. What does moving only xactCompletionCount out of TransamVariables cost committing backends, compared with the full split into varsup.h?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund, who worked on the original xactCompletionCount code, reviews the patch in message #3 and says the optimization ought to be done.
- **Liability:** The owner signs for snapshot correctness on primary and hot standby under a lockless protocol, with no time limit. Errors would show as wrong query results or missing tuples, and no test is likely to catch them.
- **Author follow-up:** The author posted a v2 series in message #5 that splits the header move into its own commit, as message #3 asked.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The gain goes to busy systems with many backends, and a committer who knows the area is engaged. The patches are small, but lockless reasoning in procarray.c needs careful review, and the memory-ordering rule must be kept by all later completion paths.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` 0003 installs xmin locklessly without a stated horizon invariant. Refuted (already handled): Messages #5 and #6 state the rule for the primary: an unchanged counter means every xid the snapshot saw as running still runs, and the second read catches a completion. The hot standby part is a duplicate of replication-ondisk-1, and [P11] removed a backend from horizons, a different mechanism.
- `maintenance-fit-3` Moving all of TransamVariablesData forces extension changes for one counter. Refuted (no failure scenario): An extension that must add one include on a new major version is routine and is not a failure. Message #2 says only procarray.c uses xactCompletionCount, and the cache-line cost of the narrower move belongs to the benchmark request in maintenance-fit-2.
- `demand-1` Gain is shown only as call counts, not as throughput. Refuted (not specific): Duplicate of maintenance-fit-2, which asks for the same benchmarks. As a desirability objection it is weak, because in message #3 a committer who knows this code says the change is a win in several workloads and should be done.

### Review angles

- prior-attempts: 1 objection(s). 0002 (lockless reuse with xmin already set) has no prior rejection whose reason still applies, and P16 has no recorded reason, so it is not used.
- standards-compat: 0 objection(s). The proposal adds no SQL syntax, GUC, catalog object, psql feature or dump content, and existing queries return the same results. The only user-facing change is the new varsup.h header, which extensions that use TransamVariables must include. That kind of header reorganization is routine in a major release and easy to adapt to. Vacuum horizon retreat and memory ordering are safety questions outside this angle.
- replication-ondisk: 1 objection(s). The proposal adds no on-disk format, WAL record, catalog or wire-protocol change. The header split and atomic type change in 0001 and 0002 touch extension source code only on master, which is routine across major versions. The one open point from this angle is hot standby correctness for 0003. P11 is cited only as an example of an xmin-advance change that was safe for vacuum but not for HOT pruning.
- security: 0 objection(s). The proposal changes how a backend checks its own cached snapshot inside procarray.c. It adds no SQL surface, privilege, setting, view or log output, and no user can reach or observe the new path in a new way. Correctness of the lockless xmin publication and of backward vacuum horizons is a data-safety question for the design angle, not a privilege or exposure issue.
- maintenance-fit: 3 objection(s). No fit objection: the change is core procarray work, and Andres Freund, who wrote the original snapshot scalability code, supports it (message #3).
- already-exists: 0 objection(s). This is an internal locking change in GetSnapshotData(), so no setting, extension or hook could give the same result. It extends the existing xactCompletionCount reuse mechanism instead of adding a second one, so this angle has no objection.
- demand: 1 objection(s). Demand for the goal is credible: it affects every backend that takes snapshots, Andres Freund planned it in 623a9ba79b, and two reviewers support it. The one open point is the size of the gain, which is not yet measured.

### Precedents retrieved

- P1: [Global snapshots](https://commitfest.postgresql.org/patch/1662/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P2: [Move PinBuffer and UnpinBuffer to atomics](https://commitfest.postgresql.org/patch/370/) (cf, Returned with feedback, 2015-12-16) - used in the review; reason summarised from its thread
- P3: [POC: Cache data in GetSnapshotData()](https://commitfest.postgresql.org/patch/553/) (cf, Returned with feedback, 2017-09-29) - used in the review; reason summarised from its thread
- P4: [snapshot too old, configured by time](https://commitfest.postgresql.org/patch/348/) (cf, Returned with feedback, 2016-02-01) - judged not relevant
- P5: [Atomic pgrename on Windows](https://commitfest.postgresql.org/patch/1386/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P6: [Fix typo 586/686 in atomics/arch-x86.h](https://commitfest.postgresql.org/patch/6295/) (cf, Returned with feedback, 2026-03-12) - judged not relevant; reason summarised from its thread
- P7: [Atomic rename feature for Windows](https://commitfest.postgresql.org/patch/3347/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P8: [old_snapshot: add test for coverage](https://commitfest.postgresql.org/patch/3805/) (cf, Returned with feedback, 2022-11-25) - judged not relevant; reason summarised from its thread
- P9: [Reuse data from readRecordBuf in XLogDecodeNextRecord](https://commitfest.postgresql.org/patch/6343/) (cf, Withdrawn, 2026-02-09) - judged not relevant; reason summarised from its thread
- P10: [Fixes for non-atomic read of read of control file on ext4 + ntfs](https://commitfest.postgresql.org/patch/4025/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P11: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - used in the review
- P12: [Split xlog.c](https://commitfest.postgresql.org/patch/3173/) (cf, Returned with feedback, 2021-11-15) - judged not relevant
- P13: [Lockless queue of waiters based on atomic operations for LWLock](https://commitfest.postgresql.org/patch/3983/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P14: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P15: [Revert "Move page initialization from RelationAddExtraBlocks() to use."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=684200543b4cbfe1ac002c9962e90683d4ea4691) (revert, Reverted, 2019-01-29) - judged not relevant
- P16: [Improving connection scalability](https://commitfest.postgresql.org/patch/3651/) (cf, Rejected, 2022-08-01) - used in the review
- P17: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P18: [Move backup-related code to xlogbackup.c/.h](https://commitfest.postgresql.org/patch/3919/) (cf, Rejected, 2023-03-24) - judged not relevant
- P19: [Make installcheck-world in a clean environment](https://commitfest.postgresql.org/patch/1672/) (cf, Withdrawn, 2019-07-30) - judged not relevant
- P20: [Fix lock contention for HASHHDR.mutex](https://commitfest.postgresql.org/patch/442/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P21: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P22: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4739/) (cf, Rejected, 2024-04-08) - judged not relevant
- P23: [fix spinlock contention in LogwrtResult](https://commitfest.postgresql.org/patch/2716/) (cf, Withdrawn, 2022-09-23) - judged not relevant
- P24: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4750/) (cf, Withdrawn, 2024-10-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 173 s; cost reported by the CLI: $1.88
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 80 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:53:56.166Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
