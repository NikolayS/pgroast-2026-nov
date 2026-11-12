# Design pre-review: Fix recovery connection detail in EXEC_BACKEND builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7487](https://commitfest.postgresql.org/patch/7487/); [thread](https://www.postgresql.org/message-id/flat/CAMm1aWYtiKDv7uzPWJZ2Gre2973bRV%3DGTOhd6BU_-VxdeSkGVg%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated in message #1, the fix makes EXEC_BACKEND builds report the same DETAIL and HINT as fork() builds, and v2 reuses the existing CAC_state mechanism as Heikki Linnakangas suggested in message #2. Confidence is medium because the code was not read and nobody has replied to v2.

Not checked: Without the code, nobody checked whether all CAC_state consumers handle CAC_NOTCONSISTENT or where the postmaster reads the consistency state. The author did not test on Windows.

## Prior attempts

None found before 2026-10-12.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which branches should get the fix, and if back-patched, is CAC_NOTCONSISTENT added at the end of CAC_state so existing values keep their numbers?
2. Can the postmaster's view of consistency be out of date when it picks CAC_NOTCONSISTENT or CAC_NOTHOTSTANDBY, and does that differ from fork() builds today?
3. Does every remaining consumer of CAC_state handle the restored CAC_NOTCONSISTENT value?
4. Can a TAP test cover the DETAIL and HINT for a connection rejected after consistency but before the hot standby snapshot is ready?
5. Has anyone run v2 on Windows, the main platform that uses EXEC_BACKEND?

## Touch points

- **WAL / replication / standby:** The patch changes the DETAIL and HINT of connections rejected on a standby during hot standby startup; recovery behaviour is unchanged.
- **ABI / extension API:** The patch adds CAC_NOTCONSISTENT to the CAC_state enum in src/include/tcop/backend_startup.h.
- **backpatchability:** The entry is tagged Bugfix but the thread does not say which branches get it; a back-patch changes an enum in a header.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas reviewed v1 in message #2 and suggested the design that v2 uses. Nobody has replied to v2 yet.
- Priority: proceed; audience niche; review cost low. The fix affects only EXEC_BACKEND builds, such as Windows, during hot standby startup. The patch is about 4 KB across three files and removes the backend's reliance on an inherited postmaster variable.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
