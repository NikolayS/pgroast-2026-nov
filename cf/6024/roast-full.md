# Design pre-review: Improve read_local_xlog_page_guts by replacing polling with latch-based waiting

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6024](https://commitfest.postgresql.org/patch/6024/) (Needs review); authors: Xuneng Zhou (dylanzhou)
- Thread: [Improve read_local_xlog_page_guts by replacing polling with latch-based waiting](https://www.postgresql.org/message-id/flat/CABPTF7UNft368x-RgOXkfj475OwEbp%2BVVO-wEXz7StgjD_%3D6sw%40mail.gmail.com), started 2025-08-27
- Messages read: 22 of 22
- Patch prose read: v12-0001-Replace-polling-with-LSN-waits-in-read_local_xlo.patch, v12-0002-Test-logical-decoding-LSN-waits-across-promotion.patch, v8-0001-Implement-WAIT-FOR-command.patch, v1-0001-Improve-read_local_xlog_page_guts-by-replacing-po.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the goal is stated, a TODO in xlogutils.c asks for it, and the patch reuses the committed WaitForLSN() facility. One fixable level 2 objection survived, replication-ondisk-1: v12 waits with no timeout, so a flush path that omits the wakeup delays or stalls a reader. A committer who knows the WAL flush paths has to judge whether that dependency is acceptable as designed.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the walsender.c change, the wait event reported and the full set of flush wakeup sites could not be checked. The claim that an upstream timeline switch needs no separate wakeup rests on the commit message and the two new tests.

## The proposal, restated

**Problem.** read_local_xlog_page_guts() sleeps 1 ms between checks while it waits for WAL to be flushed on a primary or replayed on a standby. The author saw heavy polling in a performance run. The users are backend WAL readers: logical slot creation, SQL decoding and slot synchronization, including on cascading standbys.

**Proposed design.** v12 calls the committed WaitForLSN() facility (3b4e53a, 447aae1): flush waits on a primary, replay waits on a standby. Waiters wake on WAL flush, on replay progress and on promotion, and the loop then rechecks timeline and read limit as before. A second patch adds TAP tests for local and upstream promotion during slot creation and during pg_sync_replication_slots().

**What the author claims.**
- In one sampled run, CPU cycles fell about 39%, instructions 14%, cache misses 28% and context switches 23%, with similar elapsed time.
- An upstream timeline switch needs no separate wakeup, because replay must pass the segment before it can be read as historical WAL.
- Local promotion releases the replay wait, and the next iteration uses the flush position.
- The patch keeps the existing no-wait behavior and the timeline and read-limit checks.
- Heikki's TODO comment in the code already suggests replacing the check/sleep/repeat loop.

**Thread state.** Rui Zhao reviewed v11 favorably with minor comments, and v12 (2026-10-18) addresses them and adds tests. Michael Paquier called the reworked patch much simpler in November 2025 but posted no review after that, and consolidation with syncrep.c is left open.

## 2. Prior attempts

### Reduce wakeup on idle for bgwriter & walwriter for >5s (related)

- **Link:** [https://commitfest.postgresql.org/patch/4029/](https://commitfest.postgresql.org/patch/4029/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CANbhV-HK8yvO_g4vwbmz__vZu_VZ4_jJWsTwmaNMXieTdzQCzQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CANbhV-HK8yvO_g4vwbmz__vZu_VZ4_jJWsTwmaNMXieTdzQCzQ@mail.gmail.com)
- **Date:** decided 2023-03-30; started 2022-02-19
- **Outcome:** Withdrawn (cf)
- **Who objected:** Thomas Munro, Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Thomas Munro asked why these processes need a timeout at all. A 60s timeout would still need proof that a missed bgwriter wakeup, from the race with StrategyGetBuffer(), is harmless, and with that proof the timeout could be dropped. Tom Lane agreed and said the real goal is explicit wakeups for every event of interest. Simon Riggs said removing the timeout was too much work for that cycle and stopped working on it. Bharath Rupireddy then posted a separate walwriter patch with no timeout, but the visible messages do not show why the entry was withdrawn.
- **Reason (quote):** "Someone would need to think about it and construct a complete argument for why it's OK to go to sleep for 60 seconds, or infinity, with that sloppiness in place." ([message](https://www.postgresql.org/message-id/CA%2BhUKG%2BB3FsKRVUv2pB%2BzEbq7WH8iqBs5%3DG%2B%2BeMUYn_Pu8AfuA%40mail.gmail.com))
- **What is different this time:** P6 lengthened timed sleeps in bgwriter and walwriter and concerned a different race. This patch removes the 1 ms poll in a WAL page callback and waits with no timeout on the committed WaitForLSN() facility.


## 3. Objections a reviewer is likely to raise

### 1. Unbounded flush wait depends on every flush path calling the wakeup

fixable · level 2, rubric item 9 · WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A later patch adds a WAL flush path on the primary that advances the flush LSN without calling WaitLSNWakeup(). A backend running pg_create_logical_replication_slot() waits with timeout -1 for an LSN that path flushed. It hangs until some unrelated flush happens, and on an idle primary that may never happen.

The old 1 ms poll recovered by itself from a missed wakeup. v12 calls WaitForLSN() with no timeout, so correctness now depends on a wakeup at every point that advances the flush LSN. Message #21 shows this has already moved once: d61f9c90 added implicit flushes and needed its own wakeup. Nothing in the proposal fails loudly when a future flush path omits the call, and [P6] records the same need for a complete argument before sleeping without a bound.

**Precedents.**
- [Reduce wakeup on idle for bgwriter & walwriter for >5s](https://commitfest.postgresql.org/patch/4029/) (Withdrawn, 2023-03-30); thread: [link](https://www.postgresql.org/message-id/flat/CANbhV-HK8yvO_g4vwbmz__vZu_VZ4_jJWsTwmaNMXieTdzQCzQ@mail.gmail.com)

Thread: [#20 (Rui Zhao, 2026-06-08)](https://www.postgresql.org/message-id/71D1DAFA-3A22-485D-9944-57BD1B91E23F%40gmail.com), [#21 (Xuneng Zhou, 2026-10-12)](https://www.postgresql.org/message-id/CABPTF7WNZeKw_tTUG3LRwdcR06%3D-AMZRVW76GgTmEXEFE7XMqg%40mail.gmail.com), [#22 (Xuneng Zhou, 2026-10-18)](https://www.postgresql.org/message-id/CABPTF7W0s_XHGkbsWdVaGs5sBefePXmG64i3RY0FsdU%3Db_1xwA%40mail.gmail.com)

**Question:** Which code points advance the primary flush LSN today, and what would detect a new one that omits the wakeup: a bounded fallback timeout, an assertion, or a test?

Counter-check: held. Messages #18 and #20 trace today's wakeup sites, but message #21 shows the set already grew once with d61f9c90. v12 waits with timeout -1 and names no check that catches a new flush path without the wakeup. [P6] concerns a different race, so it supports only by analogy.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes how backend WAL readers wait for flush on a primary and replay on a standby, and touches walsender.c. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every future path that advances the flush LSN must call the wakeup, because the reader no longer polls. |
| performance on the hot path | **yes** | The wait path of logical slot creation, SQL decoding and slot synchronization changes from a 1 ms sleep to WaitForLSN(). |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which code points advance the primary flush LSN today, and what detects a new one that omits the wakeup: a bounded fallback timeout, an assertion, or a test?
2. What does the patch change in walsender.c, and why does a change to read_local_xlog_page_guts() need it?
3. Which wait event does a backend waiting in read_local_xlog_page_guts() now show in pg_stat_activity, and does its description fit a decoding backend?
4. What workload and setup produced the CPU numbers in the commit message, and can they be posted as text?
5. Do the SQL get/peek functions, slot advance and the slot sync worker need tests for the new wait, or do the two promotion tests cover the same code?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the WAL flush and replay paths and the xlogwait.c facility. Michael Paquier commented in November 2025 but the brief shows no review from him after that.
- **Liability:** The owner signs for the rule that every flush and replay advance wakes LSN waiters, for as long as backend WAL readers wait without a timeout. A missed wakeup shows up as a stalled slot creation or decoding call in production.
- **Author follow-up:** The author posted twelve versions over 14 months, addressed the v11 review in v12 and added tests, and keeps moving the entry between CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 3.9 KB plus tests, reuses committed infrastructure and adds no user-visible surface. It has one favorable non-committer review, and the brief shows no committer review since November 2025, across five CommitFests.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Indefinite LSN wait turns any missed flush wakeup into a hang. Refuted (not specific): Duplicate of replication-ondisk-1. That one states the outcome more accurately: the waiter sleeps until a later flush that does call the wakeup, which is not the same as never waking.
- `maintenance-fit-1` Unbounded wait depends on every future flush path calling the wakeup. Refuted (not specific): Duplicate of replication-ondisk-1, which makes the same point and also asks for the list of code points that advance the flush LSN today.

### Review angles

- prior-attempts: 1 objection(s). The pg_wal_replay_wait reverts ([P1], [P2]) were about a procedure that needs a snapshot-less backend. They do not apply to this internal callback, which already waits in the same context today.
- standards-compat: 0 objection(s). The proposal adds no syntax, GUC, catalog object or dumped state, and v12 keeps the no-wait and timeline behavior, so nothing in the user contract changes. The one visible change is in pg_stat_activity: waiting decoding backends may now show the WAIT FOR wait events where pg_usleep showed none, which needs checking against the code and the event descriptions.
- replication-ondisk: 1 objection(s). The proposal adds no on-disk format, WAL record, catalog or protocol change. The only cross-process state is the shared LSN waiter queue, which is already committed.
- security: 0 objection(s). The patch changes only how existing backend WAL readers wait for WAL: a 1 ms sleep loop becomes a WaitForLSN() wait on the committed facility. It adds no new SQL interface, privilege, setting or monitoring output, so no new user can reach it. The callers (slot creation, SQL decoding, slot sync) already need replication privileges, and the existing per-backend waiter slots bound the memory use. I found no new information leak, escalation path or denial-of-service route by an unprivileged user.
- maintenance-fit: 1 objection(s). The change is small and reuses committed infrastructure, so it adds little code to maintain. Committability is weaker: no committer has reviewed the patch since message #16 in November 2025.
- already-exists: 0 objection(s). Postgres has no existing feature, GUC or extension that removes this 1 ms poll, and an extension cannot change an internal page-read callback. The patch reuses the committed WaitForLSN() facility, so it adds no second mechanism of its own. The syncrep.c consolidation question in message #10 concerns that committed facility, so it is not an objection to this patch.
- demand: 0 objection(s). No demand objection: this is an internal change with no user-visible surface, Michael Paquier called the goal a fine idea in message #6, and a TODO in xlogutils.c already asks for it. The cost falls on the feature path, since the flush and replay wakeups are already committed; the CPU-only gain and the benchmark shown only as an image are questions for the performance angle.

### Precedents retrieved

- P1: [Revert: Implement pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=772faafca1b288c4dd66b7150a7831c27b768003) (revert, Reverted, 2024-04-11) - used in the review
- P2: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - used in the review
- P3: [Improvements for pg_wal_replay_wait()](https://commitfest.postgresql.org/patch/5167/) (cf, Returned with feedback, 2024-12-10) - used in the review; reason summarised from its thread
- P4: [Make async slave to wait for lsn to be replayed](https://commitfest.postgresql.org/patch/772/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P5: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P6: [Reduce wakeup on idle for bgwriter & walwriter for >5s](https://commitfest.postgresql.org/patch/4029/) (cf, Withdrawn, 2023-03-30) - used in the review; reason summarised from its thread
- P7: [suppressing useless wakeups in logical/worker.c](https://commitfest.postgresql.org/patch/4035/) (cf, Returned with feedback, 2023-09-01) - used in the review; reason summarised from its thread
- P8: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant; reason summarised from its thread
- P9: [Add LSN <-> time conversion facility](https://commitfest.postgresql.org/patch/4754/) (cf, Withdrawn, 2024-10-22) - judged not relevant; reason summarised from its thread
- P10: [Re: MarkBufferDirtyHint() and LSN update](https://commitfest.postgresql.org/patch/2352/) (cf, Withdrawn, 2019-11-25) - judged not relevant; reason summarised from its thread
- P11: [LSN Map](https://commitfest.postgresql.org/patch/89/) (cf, Returned with feedback, 2015-07-06) - judged not relevant
- P12: [Remove page-read callback from XLogReaderState](https://commitfest.postgresql.org/patch/2113/) (cf, Returned with feedback, 2021-10-04) - judged not relevant
- P13: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - judged not relevant
- P14: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P15: [Revert "When WalSndCaughtUp, sleep only in WalSndWaitForWal()."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=72a3dc321d76c93842d502793f93b9dc2d2305b2) (revert, Reverted, 2020-04-25) - used in the review
- P16: [Revert indexed and enlargable binary heap implementation.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=810f64a01567610af7b92b0de930f16f3e20064e) (revert, Reverted, 2024-04-11) - judged not relevant
- P17: [walsender.c fileheader comment](https://commitfest.postgresql.org/patch/5055/) (cf, Withdrawn, 2024-07-19) - judged not relevant
- P18: [Lockless queue of waiters based on atomic operations for LWLock](https://commitfest.postgresql.org/patch/3983/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P19: [Revert "Get rid of WALBufMappingLock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c13070a27b63d9ce4850d88a63bf889a6fde26f0) (revert, Reverted, 2025-08-22) - judged not relevant
- P20: [Make relfile tombstone files conditional on WAL level](https://commitfest.postgresql.org/patch/3030/) (cf, Withdrawn, 2021-10-29) - judged not relevant
- P21: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P22: [Use latch instead of pg_usleep for standby recovery conflicts](https://commitfest.postgresql.org/patch/977/) (cf, Returned with feedback, 2017-04-03) - used in the review
- P23: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P24: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 143 s; cost reported by the CLI: $3.06
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 8 of 24 kept
- Tone check: 60 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:38:04.418Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
