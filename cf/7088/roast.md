# Design pre-review: Batch fsyncs when recycling WAL segments

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7088](https://commitfest.postgresql.org/patch/7088/); [thread](https://www.postgresql.org/message-id/flat/CAJTYsWUy7ycUswwHeWdbQpKboRj9Cd1rfm2TwcV5WC_1T28pSw%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). No objection survived against 0001, which batches only unlinks and does not touch the WAL write path. For 0002, demand-1 and maintenance-fit-3 stand: every number measures background checkpoint cleanup time, and message #3 says it is "not a TPS result". 0002 adds a durability invariant to the WAL write path for that gain, and the author doubts in message #3 that it is worth the plumbing. Confidence is low because only the author has posted and the verdict rests on missing evidence.

Not checked: Without the code, nobody checked whether the frontier and the active-batch marker cover every segment installer, or whether 0001's retry through the kept archive-status files works when archiving is off. Four level 2 objections against 0002 survived but were not evaluated here.

## Prior attempts

- [Speed up WAL file removal during recovery](https://www.postgresql.org/message-id/flat/0A3221C70F24FB45833433255569204D1F81B0C8@G01JPEXMBYT05) (similar idea; Returned with feedback, 2018-09-30). "In short, if a crash happens in the code paths calling RemoveXlogFile with durable = false before fsync'ing pg_wal, then a rename has no guarantee to be durable, so you could finish again with a file that as an old name, but new contents." ([message](https://www.postgresql.org/message-id/20180221075250.GB1632%40paquier.xyz)) Different now: P9 dropped per-file fsyncs in timeline-switch cleanup, where renames were mixed in, with no guard. 0001 batches only unlinks and leaves timeline-switch cleanup unbatched; 0002 batches renames but adds a write-path fallback fsync.

## Objections a reviewer is likely to raise

### 1. Gain is shown only as background checkpoint cleanup time

A committer merges the recycle batching on the strength of a 139 ms to 27 ms cleanup saving. The checkpointer does this work in the background, so most users see no change in TPS or commit latency. The project keeps the added code with no measured user benefit.

Every number in messages #1, #2 and #3 measures checkpoint cleanup time. Message #3 says it is "not a TPS result". The only demand in the material is a second-hand note in message #1 that an earlier thread called these fsyncs a bottleneck. No field report, no -general or -bugs case, and no user stall caused by this cleanup appears.

**Question:** Which workload shows a user-visible stall, such as commit latency, WAL segment shortage or recovery time, caused by per-segment pg_wal fsyncs, and how large is it?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAJTYsWUy7ycUswwHeWdbQpKboRj9Cd1rfm2TwcV5WC_1T28pSw%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAJTYsWW5nRTuPkU%2BmwM2ohN3L4SxN4xwca-5a7s99dOr49Roug%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAJTYsWU0ozR-SA3WJQK1eMR%2B65rv3x98KYYi29V9xQ0HKkzFFg%40mail.gmail.com); `demand-1`</sub>

### 2. Write-path complexity paid by everyone for checkpointer-only time savings

A user on default settings upgrades and sees no TPS or latency change, because recycling already runs in the background. The project still carries the frontier logic on the WAL write path and must keep it correct in every later WAL change.

All numbers measure checkpoint cleanup time, and message #3 says "This is not a TPS result". The large gains appear with 1 MiB segments or 128 removed segments, which are unusual setups. 0001 gets part of the benefit without touching the write path. 0002's extra cost lands on shared code that all users run.

**Question:** Which workload shows lower commit latency or higher throughput from 0002 beyond what 0001 already gives?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAJTYsWUy7ycUswwHeWdbQpKboRj9Cd1rfm2TwcV5WC_1T28pSw%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAJTYsWW5nRTuPkU%2BmwM2ohN3L4SxN4xwca-5a7s99dOr49Roug%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAJTYsWU0ozR-SA3WJQK1eMR%2B65rv3x98KYYi29V9xQ0HKkzFFg%40mail.gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. Which workload shows a user-visible effect, such as commit latency, throughput or recovery time, from per-segment pg_wal fsyncs, and how large is it?
2. What does 0002 add in commit latency or throughput beyond what 0001 alone gives on the same workload?
3. Can the safety rule for recycled segments live in one function, so that a future segment installer cannot bypass it without failing a test?
4. Which processes can create or write a WAL segment during a restartpoint, archive recovery or promotion, and how does each one reach the frontier check?
5. After a failed batch fsync, why is a later pg_wal fsync trusted, and why does the marker stay set until restart?

## Touch points

- **on-disk format / pg_upgrade:** Both patches change when pg_wal directory entries for removed and renamed segments become durable.
- **WAL / replication / standby:** 0002 changes how recycled WAL segments are installed and adds a timeline-qualified frontier in XLogCtl.
- **maintenance burden ("who maintains this in 10 years"):** 0002 adds shared state that the checkpointer, every segment installer and XLogFileInit() must keep consistent.
- **performance on the hot path:** 0002 adds a frontier and batch-marker check, and a possible file and pg_wal fsync, to XLogFileInit().
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Gain is shown only as background checkpoint cleanup time

## Who would commit it, and when

- Owner: silent. It needs a committer who knows WAL segment installation and crash durability in xlog.c. Message #1 cc's two people from the earlier thread, but nobody other than the author has replied.
- Not now, show demand first: This applies to 0002 only. A benchmark or field report where per-segment pg_wal fsyncs cause a measurable commit-latency, throughput or recovery-time problem, and where 0002 removes it beyond what 0001 does, would change the answer.
- Priority: needs design rethink; audience unclear; review cost medium. 0001 is small, limited to unlinks, and drew no objection, so it can go ahead by itself. 0002 is marked WIP, puts a new invariant on the WAL write path, and the author asks in messages #1 and #3 for a simpler one, while no user-visible gain is shown.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
