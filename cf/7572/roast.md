# Design pre-review: Standby stuck after walreceiver restart just before a WAL segment boundary

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7572](https://commitfest.postgresql.org/patch/7572/); [thread](https://www.postgresql.org/message-id/flat/CADstiJuSX%3D0KZX%2B99q%3DLH4TtV0my%2Bnr3m93asc1bzn_vOiHqug%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The proposal states a reproducible standby hang, its cause, and a fix that changes behavior only when RecPtr is exactly on a segment boundary. It adds no user-visible surface, format or API. Confidence is medium because the code was not read and no committer has spoken in the visible thread.

Not checked: The code was not read, so the claim that RecPtr - 1 always falls in the segment of the page being read is unchecked, as is the stability of the test. Messages #4 and #5 were not shown in full.

## Prior attempts

None found before 2026-10-31.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Do other callers of RequestXLogStreaming() pass an end position, and can they skip the tail of a segment in the same way?
2. Will a v2 use plain RecPtr - 1 as message #3 suggests, or is there a reason to keep the conversions in WaitForWALToBecomeAvailable()?
3. Can the test build the state directly, by truncating or zeroing the segment tail on the standby, so that the 5-attempt padding retry loop is not needed?
4. Which branches is the fix meant for, given that the CommitFest entry targets 14 and the posted patch is for master with no REL_14_STABLE version?
5. Is the new TAP test meant to be backpatched with the fix, and does it pass unchanged on each back branch?

## Touch points

- **WAL / replication / standby:** The patch changes the start LSN the startup process gives the walreceiver in xlogrecovery.c.
- **backpatchability:** The author asks for a backpatch to all supported branches; no REL_14_STABLE version is posted.
- **maintenance burden ("who maintains this in 10 years"):** A new TAP test with a WAL padding retry loop runs on every supported branch and the buildfarm.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows the recovery and walreceiver start-up code in xlogrecovery.c; none has spoken in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost low. The fix addresses a silent hang that any streaming standby can hit after a walreceiver restart, and it touches one backend file plus a test. Two reviewers agree the fix is correct; the remaining points are a simpler expression and a deterministic test.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
