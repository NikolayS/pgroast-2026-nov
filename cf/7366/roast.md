# Design pre-review: ReplicationSlotRelease() clobbers another backend's statusFlags entry

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7366](https://commitfest.postgresql.org/patch/7366/); [thread](https://www.postgresql.org/message-id/flat/889a06dd-45fc-423b-9dd2-87d5b5dc60c6%40gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. Message #1 states the bug, the affected processes and the branch range, and Kuroda confirmed the analysis in message #2. v2-0001 changes one function and adds no user-visible surface. The open points are whether v2-0002 and the TAP test are committed, which the thread is already settling.

Not checked: The code was not read, so the claim that only StartupDecodingContext() sets PROC_IN_LOGICAL_DECODING is taken from the commit message. Messages #4 and #5 were not shown; the committer's plan is taken from the brief.

## Prior attempts

None found before 2026-09-28.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Is there any production report of a standby losing rows through this path, or does the PROC_AFFECTS_ALL_HORIZONS case rest on analysis only?
2. Will the TAP test be committed with the fix, and what does the back-branch version use in place of $node->advance_wal() before v17?
3. Does test number 058 conflict with an existing test in src/test/recovery on any branch from v14 to v18?
4. Is v2-0002 meant for master only, and does any path besides ReplicationSlotRelease() reach its new assertions from a process outside the proc array?

## Touch points

- **WAL / replication / standby:** The fix changes ReplicationSlotRelease() in slot.c, reached during slot invalidation by the checkpointer and the startup process on standbys.
- **ABI / extension API:** v2-0002 adds ProcArrayHasProc() to procarray.h; v2-0001 adds no API.
- **backpatchability:** v2-0001 is marked Backpatch-through: 14, and the test needs a local advance_wal() helper before v17.
- **maintenance burden ("who maintains this in 10 years"):** v2-0002 adds assertions in six files, and the new TAP test must be kept working on six branches.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier takes part in the thread and, per the brief, plans to commit v2-0001 down to v14. The owner needs to know replication slot invalidation and the proc array.
- Priority: proceed; audience a sizeable minority; review cost low. The bug affects installations where the checkpointer or startup process invalidates a slot, on v14 and later, and can weaken hot standby feedback. v2-0001 is a small change in slot.c with a committer engaged; CFbot reports that the patch needs a rebase.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
