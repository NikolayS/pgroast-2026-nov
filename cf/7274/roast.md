# Design pre-review: Report changes discarded for relations not in the subscription

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7274](https://commitfest.postgresql.org/patch/7274/); [thread](https://www.postgresql.org/message-id/flat/CAB5wL7ZH%3DhOoJexa56S8Hh%2B1772DJT2P-hrieh5bohs5sZ3dBw%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). Nothing that survived stops the proposal at levels 1a, 1b or 2. The problem is stated with a field case, apply behavior does not change, and only subscribers that hit the condition see the new messages. The one surviving objection, standards-compat-1, is a fixable design point: the commit message names a missing conflict and statistics counter, and v2 adds neither. Nobody has commented on the design in the thread yet.

Not checked: Without the code, we could not check the cost of the new check per change, how the tracking survives relation map invalidation, or what parallel apply workers report.

## Prior attempts

- [Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet](https://www.postgresql.org/message-id/flat/CAGPVpCQtF8LTM8svXdqMcMw8EmX4cNH=tGjwBW6ukJo2PEQcng@mail.gmail.com) (similar idea; Withdrawn, 2022-12-27). "I see your point but not sure if it is worth changing the current behavior without any specific use case which we want to address with this change." ([message](https://www.postgresql.org/message-id/CAA4eK1KSEL%2Bb81L47MpdVCY79n0QgboxF6XTEjSc0ZcLkDzyWQ%40mail.gmail.com)) Different now: P1 changed apply behavior, replacing an ERROR with a warning and a skip when a table was missing on the subscriber, and gave no use case. This proposal changes no behavior, only logs a skip that already happens silently, and gives a field case in message #1.

## Objections a reviewer is likely to raise

### 1. Ad hoc LOG line where conflict logging and subscription stats exist

A monitoring setup that tracks apply problems through pg_stat_subscription_stats sees no counter move while rows for an unrefreshed table are discarded. The single LOG line per worker lifetime is easy to miss in log rotation. The administrator finds the loss only when the subscriber table is compared with the publisher.

Postgres already reports apply-side anomalies through conflict types that are logged with a fixed format and counted in pg_stat_subscription_stats. The commit message names the missing counter and the missing conflict as part of the problem, but v2 adds neither. It adds a free-form message reported once per relation and sync state, so the log shows no ongoing count of discarded changes. A committer would likely ask why the existing reporting mechanism does not fit this case, or why it is deferred.

**Question:** Why is a once-per-worker LOG message preferred over a pg_stat_subscription_stats counter or a conflict type for permanently discarded changes?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAB5wL7ZH%3DhOoJexa56S8Hh%2B1772DJT2P-hrieh5bohs5sZ3dBw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAB5wL7b74bpW3KhKxY-8_%2BXsyGRudirBR5-HuVAgjnR1kR53kw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Why is a LOG message reported once per apply worker lifetime preferred over a pg_stat_subscription_stats counter or a conflict type for permanently discarded changes?
2. Is the empty pg_subscription_rel after a 14-to-18 pg_upgrade expected, or is it a bug that needs a separate fix?
3. Is LOG the intended level for a message that repeats for each affected relation after every apply worker restart?
4. Does a parallel apply worker share the once-per-state tracking with the leader, or does each worker report the same relation again?
5. Will the logical replication chapter document that changes for a published table are discarded until REFRESH PUBLICATION runs?

## Touch points

- **WAL / replication / standby:** The patch adds reporting to the logical replication apply worker in worker.c; apply behavior stays the same.
- **ABI / extension API:** The patch adds a field to LogicalRepRelMapEntry in logicalrelation.h, which C code can see.
- **backpatchability:** The author limits the patch to master because it changes LogicalRepRelMapEntry, so released branches keep the silent discard.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a user-visible LOG message with a hint, plus once-per-state tracking in the relation map entry.
- **performance on the hot path:** should_apply_changes_for_rel() runs once per applied change, and the new check sits in it.

## Who would commit it, and when

- Owner: silent. A committer who knows the logical replication apply worker and table synchronization code would have to own it. None has spoken in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is about 11 KiB, touches worker.c, one header and two TAP tests, and makes a permanent silent discard visible to logical replication users. The open point in standards-compat-1 can be settled in normal review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
