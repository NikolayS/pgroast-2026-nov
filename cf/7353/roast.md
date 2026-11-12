# Design pre-review: BUG #19723: CREATE INDEX racing with ALTER INDEX ATTACH PARTITION triggers unexpected internal error

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7353](https://commitfest.postgresql.org/patch/7353/); [thread](https://www.postgresql.org/message-id/flat/19723-8a47b625c2c64f75%40postgresql.org), 4 of 4 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducer and a bisect in message #1, and the fix is local to DefineIndex(). The one design concern, new blocking or deadlock in v1, was raised in message #4 and answered there with a fast-path check plus a recheck after locking. Which version to commit and the back-patch range are still open.

Not checked: The commit message, touched files and code of the recheck patch in message #4 were not shown, so its locking behaviour and its isolation test could not be checked. The back-patch range also depends on where 8b08f7d48 landed.

## Prior attempts

None found before 2026-09-26.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which branches carry commit 8b08f7d48, and so which branches need this fix?
2. Does the author agree to replace v1 with the fast-path check plus recheck from message #4, given the deadlock that message reports for v1?
3. Can the recheck version still block or deadlock when ALTER INDEX ... ATTACH PARTITION commits during the lock wait on the child index?
4. Do other callers of has_superclass(), or other partition attach paths, make the same check without holding a lock on the child?

## Touch points

- **catalog changes:** The fix stops DefineIndex() from attempting a conflicting pg_inherits row; it adds no catalog change.
- **backpatchability:** The entry targets stable branches, but nobody says which branches carry commit 8b08f7d48.
- **maintenance burden ("who maintains this in 10 years"):** The patch changes lock order in DefineIndex() in src/backend/commands/indexcmds.c, which future changes to that loop must preserve.

## Who would commit it, and when

- Owner: silent. A committer who knows partitioned index DDL and the lock rules in DefineIndex() would have to own it; none has commented in the visible thread.
- Priority: proceed; audience niche; review cost low. The bug needs CREATE INDEX on a partitioned table to race with ALTER INDEX ... ATTACH PARTITION, so few users hit it, but it is a reproducible internal error bisected to a recent commit. The fix touches one file and adds no user-visible surface.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
