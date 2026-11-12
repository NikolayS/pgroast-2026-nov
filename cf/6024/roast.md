# Design pre-review: Improve read_local_xlog_page_guts by replacing polling with latch-based waiting

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6024](https://commitfest.postgresql.org/patch/6024/); [thread](https://www.postgresql.org/message-id/flat/CABPTF7UNft368x-RgOXkfj475OwEbp%2BVVO-wEXz7StgjD_%3D6sw%40mail.gmail.com), 22 of 22 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the goal is stated, a TODO in xlogutils.c asks for it, and the patch reuses the committed WaitForLSN() facility. One fixable level 2 objection survived, replication-ondisk-1: v12 waits with no timeout, so a flush path that omits the wakeup delays or stalls a reader. A committer who knows the WAL flush paths has to judge whether that dependency is acceptable as designed.

Not checked: Without the code, the walsender.c change, the wait event reported and the full set of flush wakeup sites could not be checked. The claim that an upstream timeline switch needs no separate wakeup rests on the commit message and the two new tests.

## Prior attempts

- [Reduce wakeup on idle for bgwriter & walwriter for >5s](https://www.postgresql.org/message-id/flat/CANbhV-HK8yvO_g4vwbmz__vZu_VZ4_jJWsTwmaNMXieTdzQCzQ@mail.gmail.com) (related; Withdrawn, 2023-03-30). "Someone would need to think about it and construct a complete argument for why it's OK to go to sleep for 60 seconds, or infinity, with that sloppiness in place." ([message](https://www.postgresql.org/message-id/CA%2BhUKG%2BB3FsKRVUv2pB%2BzEbq7WH8iqBs5%3DG%2B%2BeMUYn_Pu8AfuA%40mail.gmail.com)) Different now: P6 lengthened timed sleeps in bgwriter and walwriter and concerned a different race. This patch removes the 1 ms poll in a WAL page callback and waits with no timeout on the committed WaitForLSN() facility.

## Objections a reviewer is likely to raise

### 1. Unbounded flush wait depends on every flush path calling the wakeup

A later patch adds a WAL flush path on the primary that advances the flush LSN without calling WaitLSNWakeup(). A backend running pg_create_logical_replication_slot() waits with timeout -1 for an LSN that path flushed. It hangs until some unrelated flush happens, and on an idle primary that may never happen.

The old 1 ms poll recovered by itself from a missed wakeup. v12 calls WaitForLSN() with no timeout, so correctness now depends on a wakeup at every point that advances the flush LSN. Message #21 shows this has already moved once: d61f9c90 added implicit flushes and needed its own wakeup. Nothing in the proposal fails loudly when a future flush path omits the call, and [P6] records the same need for a complete argument before sleeping without a bound.

Precedent: [Reduce wakeup on idle for bgwriter & walwriter for >5s](https://www.postgresql.org/message-id/flat/CANbhV-HK8yvO_g4vwbmz__vZu_VZ4_jJWsTwmaNMXieTdzQCzQ@mail.gmail.com) (Withdrawn, 2023-03-30)

**Question:** Which code points advance the primary flush LSN today, and what would detect a new one that omits the wakeup: a bounded fallback timeout, an assertion, or a test?

<sub>fixable, level 2, WAL / replication / standby; messages [#20](https://www.postgresql.org/message-id/71D1DAFA-3A22-485D-9944-57BD1B91E23F%40gmail.com), [#21](https://www.postgresql.org/message-id/CABPTF7WNZeKw_tTUG3LRwdcR06%3D-AMZRVW76GgTmEXEFE7XMqg%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/CABPTF7W0s_XHGkbsWdVaGs5sBefePXmG64i3RY0FsdU%3Db_1xwA%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Which code points advance the primary flush LSN today, and what detects a new one that omits the wakeup: a bounded fallback timeout, an assertion, or a test?
2. What does the patch change in walsender.c, and why does a change to read_local_xlog_page_guts() need it?
3. Which wait event does a backend waiting in read_local_xlog_page_guts() now show in pg_stat_activity, and does its description fit a decoding backend?
4. What workload and setup produced the CPU numbers in the commit message, and can they be posted as text?
5. Do the SQL get/peek functions, slot advance and the slot sync worker need tests for the new wait, or do the two promotion tests cover the same code?

## Touch points

- **WAL / replication / standby:** The patch changes how backend WAL readers wait for flush on a primary and replay on a standby, and touches walsender.c.
- **maintenance burden ("who maintains this in 10 years"):** Every future path that advances the flush LSN must call the wakeup, because the reader no longer polls.
- **performance on the hot path:** The wait path of logical slot creation, SQL decoding and slot synchronization changes from a 1 ms sleep to WaitForLSN().

## Who would commit it, and when

- Owner: silent. Someone who knows the WAL flush and replay paths and the xlogwait.c facility. Michael Paquier commented in November 2025 but the brief shows no review from him after that.
- Priority: needs champion; audience a sizeable minority; review cost low. The patch is 3.9 KB plus tests, reuses committed infrastructure and adds no user-visible surface. It has one favorable non-committer review, and the brief shows no committer review since November 2025, across five CommitFests.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
