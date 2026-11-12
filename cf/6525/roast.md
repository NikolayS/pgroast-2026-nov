# Design pre-review: Fix XLogFileReadAnyTLI silently applying divergent WAL from wrong timeline

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6525](https://commitfest.postgresql.org/patch/6525/); [thread](https://www.postgresql.org/message-id/flat/85386EF6-16B7-4D62-86BE-526A10F93825%40yandex-team.ru), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is stated and the demand is shown: a production switchover in message #1, an earlier report in message #2, and a reproduction back to v12 in message #5. One fixable level 2 objection survived, replication-ondisk-1: the back-patched wait is explained only at DEBUG1 and no text tells operators what to do when the segment never arrives. A committer who knows recovery must sign for a behaviour change in back branches.

Not checked: The code was not read. Whether the new rule is correct when several switch points fall in one segment, or when the segment exists only in pg_wal, could not be judged.

## Prior attempts

None found before 2026-02-20.

## Objections a reviewer is likely to raise

### 1. Back-patched stall can leave existing standbys waiting with no visible log message

A standby on v14-v18 takes a minor update. Later a cascading failover promotes a child timeline that dies before archiving its switch-point segment. Before the update, recovery replayed the parent copy and went on; after it, recovery waits forever, and the only sign at default log levels is that recovery stops moving.

Message #5 names this case and accepts the stall as the right default, but asks that it be stated. The v4 commit message does not mention it, and the patch adds no docs or release-note text. Message #10 keeps the only explanation at DEBUG1 and sends it on every retry, including on single-timeline standbys (message #11). In a minor release, a committer would likely want users to be able to tell this new wait from a hang, and to know what to do about it.

**Question:** How does an operator on a back branch learn that recovery is waiting for the owning timeline's switch-point segment, and what are they told to do when that segment will never be archived?

<sub>fixable, level 2, backpatchability; messages [#5](https://www.postgresql.org/message-id/CAOVWO5oaVqcm54w9LmY6JSx5Mk_XwUc4iJPDO8ZZYnHLoVX5JA%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/3238B48C-91CC-4029-AEEB-59AC561A25B0%40yandex-team.ru), [#11](https://www.postgresql.org/message-id/CAOVWO5rp%3Dxu_VUNDXzXXCc7bih2H8_NGct5U10-9pafef%3DSxMw%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. How does an operator on a back branch learn, at the default log level, that recovery is waiting for the owning timeline's switch-point segment?
2. What should an operator do when the child timeline died before archiving its switch-point segment, so the segment will never arrive?
3. Will the commit message and release notes state the cascading-failover case from message #5, where recovery now waits and used to continue?
4. Will the DEBUG1 message "not searching older timelines for WAL segment" fire only when an older timeline exists, as the reviewer asks?
5. Why did the 2022 thread with the functionally equivalent patch end without a commit, and does that reason apply here?

## Touch points

- **WAL / replication / standby:** The patch changes which WAL segment file archive recovery reads at a timeline switch point in xlogrecovery.c.
- **backpatchability:** The v4 commit message targets back-patch through 14, and recovery then waits where it used to continue.
- **maintenance burden ("who maintains this in 10 years"):** The patch changes the timeline fallback rule in XLogFileReadAnyTLI() and adds two TAP tests in src/test/recovery.

## Who would commit it, and when

- Owner: silent. Someone who knows archive recovery and timeline switching in xlogrecovery.c, and who is willing to back-patch a behaviour change. No committer has spoken in the visible thread.
- Priority: needs champion; audience a sizeable minority; review cost medium. The bug causes silent replay of divergent WAL for standbys that use a WAL archive across a timeline switch, and two reviewers call v4 ready for a committer. The patch is about 14 KiB in one recovery file plus tests, but no committer is engaged and back-patching needs care.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
