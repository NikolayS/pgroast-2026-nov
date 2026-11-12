# Design pre-review: Set 1s WaitLatch timeout if standby limit has expired in ResolveRecoveryConflictWithBufferPin

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6444](https://commitfest.postgresql.org/patch/6444/); [thread](https://www.postgresql.org/message-id/flat/CALQc50gi-Kw9m1r6hytf12473-fCECy%3Dq9JtKS4ANeJFEyCBTw%40mail.gmail.com), 12 of 12 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the panel at levels 1a, 1b or 2. The problem is stated with a concrete race, two independent field reports support it (messages #4 and #6), and the fix is a bounded timeout confined to standby.c. The open items are a test, the backpatch branch list, the unmeasured broadcast cost and the order relative to the parallel loop rewrite; they belong to committability and later review levels.

Not checked: The code was not read, and v3 is known only from the brief. The recheck of the pin count after a timeout, the latch reset order and the interaction with the deadlock-check timeout could not be judged.

## Prior attempts

- [Revert "Get rid of the dedicated latch for signaling the startup process".](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com) (related; Reverted, 2020-12-17). "Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin()." Different now: P1 removed the dedicated wakeup latch, so unrelated events woke the startup process. v3 keeps that latch and adds only a timeout inside ResolveRecoveryConflictWithBufferPin(), which is one of the two wakeup sources the revert message allows.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which stable branches should receive the fix, and what does the adapted patch look like on 17, where the signal and wait-event names differ?
2. What does one conflict broadcast per second cost on a standby with many backends while replay stays blocked?
3. Should this fix go in before or after the parallel patch that rewrites the same wait as a loop, and which one gets rebased?
4. Can a test reproduce the late pin deterministically, for example with an injection point, without depending on timing?
5. Which of the CommitFest entries 6444 and 6445 stays open for this thread?

## Touch points

- **WAL / replication / standby:** The patch changes how the startup process waits during a buffer pin recovery conflict on a hot standby.
- **ABI / extension API:** v1 and v2 touch proc.c and proc.h; v3 is reported to leave both unchanged.
- **security / privileges:** The patch narrows an existing way for any standby session to stall replay with a late buffer pin.
- **backpatchability:** The CF entry targets stable branches, and v3 needs signal and wait-event names adapted on 17.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a timed wait in recovery conflict code that few people know, and a parallel patch rewrites the same wait.
- **performance on the hot path:** The startup process resends the conflict signal to backends every second while replay is blocked after the limit expires.

## Who would commit it, and when

- Owner: silent. Someone who knows the startup process, recovery conflict handling and the buffer pin wakeup path. Álvaro Herrera created the CF entry in message #2 but has not reviewed the patch in the visible thread.
- Priority: needs champion; audience a sizeable minority; review cost low. The fix is small and confined to standby.c, and two field reports show replay stalls of up to 5h41m on hot standbys. Three reviewers tested v3, but no committer who knows the area has reviewed it, and the entry has been open across two CommitFests.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
