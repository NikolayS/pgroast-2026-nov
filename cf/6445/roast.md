# Design pre-review: Set 1s WaitLatch timeout if standby limit has expired in ResolveRecoveryConflictWithBufferPin

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6445](https://commitfest.postgresql.org/patch/6445/); [thread](https://www.postgresql.org/message-id/flat/CALQc50gi-Kw9m1r6hytf12473-fCECy%3Dq9JtKS4ANeJFEyCBTw%40mail.gmail.com), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the race is stated step by step, and two production reports (messages #4 and #6) show demand. The one surviving objection, maintenance-fit-1, is a fixable level 2 point about commit order with the loop rewrite and the missing test. The fix changes how the startup process waits and targets stable branches, so a committer who knows recovery conflict handling has to sign for it.

Not checked: Without the code, we could not check that the timed wait rechecks the pin count correctly, keeps the same wait event in pg_stat_activity, or wakes only on unpin and its own timeout as P1 requires.

## Prior attempts

- [Revert "Get rid of the dedicated latch for signaling the startup process".](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com) (related; Reverted, 2020-12-17). "Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin()." Different now: P1 removed the dedicated startup latch, so unrelated events woke the startup process during a buffer pin wait. v3 adds a timeout set inside ResolveRecoveryConflictWithBufferPin() itself, which is one of the two wakeup sources the revert allows.

## Objections a reviewer is likely to raise

### 1. Fix and the pending loop rewrite of the same wait can drift apart

v3 is committed and backpatched to 17 and 18. Later, the other patch that rewrites this wait as a loop lands on master, and its expired path still sleeps with no timeout. The late-pinner stall comes back on master only, with no test to catch it, and users find it again through replay lag in production.

Message #5 says the two patches do not apply together and that the loop rewrite still needs this fix. The material does not say which patch goes first or who carries the timeout into the loop. The patch adds no test, and message #8 calls a TAP test possible but timing-sensitive, so a regression here would show up only as replay lag. Release branches already differ in signal and wait-event names (message #6), so master and the stable branches start out with different code.

**Question:** In which order will this fix and the loop rewrite be committed, and what test or check stops the expired path from losing its timeout again on master?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#5](https://www.postgresql.org/message-id/CAGRkXqSn-PU-KwU6GwwhhJxZ8yyjioRf-kxzR0UrS5y_Hbk3yA%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/A65E3EDC-92B5-4C69-818F-0089F7AEB428%40yandex-team.ru), [#8](https://www.postgresql.org/message-id/CADMs5OMJK%3DjaQJyePDfzDarvXo5i0gMLO8trZPgkbJJ6iJzGog%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. In which order will this fix and the loop rewrite of the same wait be committed, and who carries the 1s timeout into the loop?
2. What test or check would catch the expired path losing its timeout again, given that message #8 calls a TAP test timing-sensitive?
3. Which stable branches should get the fix, and does v3 apply to each of them given the differing signal and wait-event names in message #6?
4. What does the once-per-second broadcast cost on a standby with many backends during a long stall?
5. Which of CF entries 6444 and 6445 stays open for this thread?

## Touch points

- **WAL / replication / standby:** The patch changes how the startup process waits for a cleanup lock during WAL replay on a hot standby.
- **backpatchability:** The CF entry targets stable branches, and v3 avoids proc.c and proc.h to ease backpatching; no range is agreed.
- **maintenance burden ("who maintains this in 10 years"):** The fix conflicts with a pending loop rewrite of the same wait and has no regression test.
- **performance on the hot path:** The startup process resends the conflict signal to all backends once per second during an expired conflict; nobody measured the cost.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows recovery conflict handling and the startup process latch. Álvaro Herrera created the CF entry in message #2 but did not review the design in the visible thread.
- Priority: needs champion; audience a sizeable minority; review cost low. The fix is small, lives in standby.c, and has production reports on 17 and 18 plus three testers. It has waited since 2025-07 through two CommitFests without a committer review of the design.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
