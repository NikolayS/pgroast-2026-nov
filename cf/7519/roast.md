# Design pre-review: Fix for bug #19773 : pg_dump differing generation expressions of inherited columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7519](https://commitfest.postgresql.org/patch/7519/); [thread](https://www.postgresql.org/message-id/flat/CAH5HC97y7PFE6rJh8kVDUszPxGQE030nZvyHu%3DwZT7FBkOV3qA%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is a reported bug fix (#19773) for wrong restores that report no error, and no surviving objection questions the goal or the v2 approach. The one surviving objection, replication-ondisk-1, concerns commit order. Messages #2 and #6 show that the emitted SET EXPRESSION drops a child's inherited CHECK and NOT NULL rows, and the thread does not say whether the backpatch to 17 waits for the backend fix. A committer who knows pg_dump and the stable branches has to decide that.

Not checked: Without the code, this review could not check that the decision made after sorting covers every case where loop repair makes an expression separate. It also could not check how the patch behaves for the multi-parent case.

## Prior attempts

None found before 2026-10-21.

## Objections a reviewer is likely to raise

### 1. Patch makes dump/restore produce inconsistent inherited-constraint catalog state

A v17 user with an inherited CHECK or NOT NULL on a generated column dumps with a patched pg_dump and restores. The restore reports no error, but the child's pg_constraint rows are missing, so rows that violate the CHECK are accepted. Later, pg_upgrade of that cluster passes --check and then fails with "child table is missing constraint", and DROP NOT NULL on the parent fails with "cache lookup failed".

Message #2 and message #6 show that the emitted ALTER TABLE ONLY ... SET EXPRESSION drops inherited CHECK and NOT NULL rows through ATPostAlterTypeCleanup(). The bug is in the backend, but today pg_dump never emits that command, so ordinary dump/restore does not hit it; the patch would make every such dump hit it. Today the child gets the wrong expression; with the patch it gets the right expression and a catalog that a later pg_upgrade rejects. The thread does not say whether the patch waits for the backend fix, or whether that fix will be backpatched to 17.

Precedent: [Revert "Improve compression and storage support with inheritance"](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org) (Reverted, 2024-02-20)

**Question:** Will this go in only together with a backend SET EXPRESSION fix, backpatched to 17, so that no released minor version restores into a catalog that pg_upgrade later rejects?

<sub>fixable, level 2, catalog changes; messages [#2](https://www.postgresql.org/message-id/CAAAe_zAZ%2BvjNaemjE5C3khRGjWa2d0ML%2Be%3D00-tqcXd%2BFOiFLw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAH5HC95SCKZKi%3Dwwcd5Kj2QqjMAwAb7%3DE5GV-3WcYF6Fz1yzXQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAAAe_zCWfn8Krssct-H%3DCZZ4huGDiGwypFZDyA9n1vw9c50gGw%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Will this patch be committed and backpatched to 17 only together with a backend fix that keeps inherited CHECK and NOT NULL constraints on SET EXPRESSION?
2. Will the backend fix for constraint loss on SET EXPRESSION itself be backpatched to 17, or only applied to master?
3. What is the plan for the multi-parent case, which still fails at restore with v2 and on master?
4. Do the SET EXPRESSION dependency problems that jian he linked affect any command that v2 now emits in a dump?
5. Can the cases from messages #2 and #6 stay in the regression database so that 002_pg_upgrade checks them, as message #7 suggests?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade uses pg_dump, and a restored cluster with missing inherited constraint rows later fails pg_upgrade with "child table is missing constraint".
- **catalog changes:** see objection replication-ondisk-1: Patch makes dump/restore produce inconsistent inherited-constraint catalog state
- **pg_dump / restore round-trip:** pg_dump now emits ALTER TABLE ONLY ... SET EXPRESSION for inherited generated columns, which changes what a restore produces.
- **backpatchability:** The commit message says "Backpatch-through: 17", and the emitted command triggers a backend bug that has no posted fix.
- **maintenance burden ("who maintains this in 10 years"):** v2 adds a decision after sortDumpableObjects() that only generation expressions use, next to the existing path for defaults.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows pg_dump's dependency sorting and inheritance handling, and who can judge backpatch risk for the stable branches. None has spoken in the visible thread.
- Priority: proceed; audience niche; review cost medium. The bug affects only inheritance trees with differing generation expressions, but the restore gives wrong values without an error, so it merits a fix. The patch is about 20 KiB in pg_dump only; the open point is commit order with the backend fix.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
