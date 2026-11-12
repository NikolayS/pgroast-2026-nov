# Design pre-review: Whole-row fixes for DROP COLUMN, SET COLUMN DATA TYPE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6055](https://commitfest.postgresql.org/patch/6055/); [thread](https://www.postgresql.org/message-id/flat/CACJufxGA6KVQy7DbHGLVw9s9KKmpGyZt5ME6C7kEfjDpr2wZCw%40mail.gmail.com), 17 of 17 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The bug is real and reviewers reproduced it, so the goal stands. The design is contested: message #17 argues that whole-row indexes stay valid and should be rebuilt, and that pg_depend should record whole-row dependencies, and no reply is visible (standards-compat-2, prior-attempts-2, maintenance-fit-1). The Backport tag adds an undiscussed behavior change for stable branches (replication-ondisk-1). The list needs to choose one design before code review is useful.

Not checked: Without the code, this review could not check how the patch detects whole-row references in each catalog, how it handles partitions and policies that reference other tables, or whether the pg_depend entries written during DROP COLUMN are safe if the command fails.

## Prior attempts

- [Revert applying column aliases to the output of whole-row Vars.](https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us) (related; Reverted, 2022-03-17). "What *is* kind of awful is to make such a behavioral change in a back-patched bug fix." Different now: P1 changed the column names that whole-row Vars output, a different feature. It is relevant only as an example of a behavior change back-patched to prevent corrupt on-disk data, which this proposal does not claim.
- [Revert per-index collation version tracking feature.](https://www.postgresql.org/message-id/flat/CA+hUKGLhj5t1fcjqAu8iD9B3ixJtsTNqyCCD4V0aTO9kAKAjjA@mail.gmail.com) (related; Reverted, 2021-05-07). "Misgivings were also expressed about the use of the pg_depend catalog for this purpose." Different now: P20 tracked collation versions in pg_depend and missed composite and record types. This patch records pg_depend entries only at DROP COLUMN time, for a different purpose.

## Objections a reviewer is likely to raise

### 1. DROP COLUMN drops whole-row indexes that are still valid

A table has a partial index `on t (b) where t is not null`. A DBA drops an unrelated column `a`. The index on `b` is dropped with no notice and no CASCADE, so queries that used it turn into sequential scans, and nobody sees it until they look at plans.

An expression such as `t is null` stays well defined after a column is dropped. Only comparisons with a fixed-width row constant break. The patch treats every whole-row reference like a direct column reference, so it drops valid objects along with broken ones. In message #17 Matthias van de Meent proposed rebuilding these indexes, and the thread does not answer him.

**Question:** Which whole-row index or constraint definitions become invalid after DROP COLUMN, and why drop the ones that stay valid instead of rebuilding them?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CACJufxGA6KVQy7DbHGLVw9s9KKmpGyZt5ME6C7kEfjDpr2wZCw%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com); `standards-compat-2`</sub>

### 2. Competing fix in CF 7071 rebuilds indexes and has no reply

This patch ships and whole-row indexes are dropped on DROP COLUMN. A later release adopts the CF 7071 approach, which rebuilds them. Users then see the same command drop an index in one release and keep it in the next.

Message #17 says the whole-row index definition stays valid and only needs a rebuild. It points to CF 7071, which records whole-row Vars in pg_depend under their own objsubid. Message #14 ties this work to CF 6755, but no message answers message #17. Two fixes for one class of bug with opposite user-visible results would likely make a committer wait until one design is chosen.

**Question:** Why drop whole-row indexes on DROP COLUMN instead of rebuilding them as CF 7071 does, and can the two patches be reconciled into one design?

<sub>fixable, level 2, design; messages [#14](https://www.postgresql.org/message-id/CACJufxGv5zmiRyRB%2BbpEWjWKh0%2Bka_bVadnFKs8%2BspY7wSak4A%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com); `prior-attempts-2`</sub>

### 3. Per-catalog scans for whole-row references must be extended by hand

A later release adds a new kind of object that stores an expression over a table's row. Its author does not know that tablecmds.c keeps a hand-written list of catalogs to scan for whole-row Vars. DROP COLUMN then leaves the new object broken, and users find out only when inserts fail.

The patch scans pg_index, pg_constraint, pg_trigger and pg_policy one by one inside ALTER TABLE, and keeps pg_depend unchanged. Nothing fails loudly when an object class is missing from that list. Message #17 proposes recording whole-row Vars in pg_depend with their own objsubid, so the dependency code would cover every object class and the scan would cost O(whole-row dependencies), not O(all dependents). No reply to message #17 is visible.

**Question:** Why is a scan list in tablecmds.c preferable to recording whole-row dependencies in pg_depend, and what makes a future object class fail loudly if it is left out?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#8](https://www.postgresql.org/message-id/CACJufxEApboeVKP4jdYOkikOij22HrHWnuSb%3Du-cJ7AHMBcOxA%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CACJufxFeYySYRFOvUcvXeb6h0Q7FA6AAhn%2BxA%2BAaSbXU9c98Zw%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. SET DATA TYPE now fails for whole-row objects that would still work

A user widens `int` to `bigint` on a table that has an index on `(t is null)` or a policy using the row. Today the command succeeds and the index is rebuilt. With the patch, it errors and the HINT asks them to drop the object first, so a type change that used to be routine now needs manual drop and re-create.

The commit message justifies the error with record_eq, which only matters when a row is compared with another row value. The brief and message #15 describe the error as raised for any whole-row dependent object. That is broader than the failure it prevents. Message #17 argues for rebuilding instead, and no reply is visible.

**Question:** Can the error be limited to row comparisons that record_eq rejects, so that other whole-row objects are rebuilt as non-whole-row expression indexes are today?

<sub>fixable, level 2, design; messages [#4](https://www.postgresql.org/message-id/CACJufxHmmsg7GTD_LpGGam09%2BAq4KkYHe2dfPgtGjrUELqbK%3Dw%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CACJufxFeYySYRFOvUcvXeb6h0Q7FA6AAhn%2BxA%2BAaSbXU9c98Zw%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com); `standards-compat-3`</sub>

### 5. Backpatching silently drops user indexes and constraints in a minor release

A deployment script runs `alter table ... drop column` after a minor update. A unique index on a whole-row expression is now dropped without notice, so later inserts are no longer checked for uniqueness. Where a policy or trigger WHEN clause uses the row, the same script now fails without CASCADE, and the release stops.

The CommitFest entry tags this Backport with target version stable. The patch changes DDL results in two ways: some objects are dropped implicitly, and some commands that succeeded before now fail. Matthias in message #17 argues that whole-row indexes should be rebuilt, not dropped, which would avoid losing them, and no reply is visible. A committer would likely ask for master-only behavior, or for a narrower back branch fix such as an error.

Precedent: [Revert applying column aliases to the output of whole-row Vars.](https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us) (Reverted, 2022-03-17)

**Question:** Why is dropping or blocking safe to backpatch, and why is a back branch fix that only raises an error, leaving existing objects untouched, not enough?

<sub>fixable, level 1b, backpatchability; messages [#15](https://www.postgresql.org/message-id/CACJufxFeYySYRFOvUcvXeb6h0Q7FA6AAhn%2BxA%2BAaSbXU9c98Zw%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Which whole-row index and constraint definitions become invalid after DROP COLUMN, and why drop the ones that stay valid, such as `t is not null`, instead of rebuilding them?
2. How does this patch relate to CF 7071, and which of the two should be the long-term mechanism for whole-row dependencies?
3. Which parts of the patch are meant for stable branches, and why is a back-branch fix that only raises an error not enough?
4. Can the SET DATA TYPE error be limited to row comparisons that record_eq rejects, with other whole-row objects rebuilt as expression indexes are today?
5. What does DROP COLUMN cost with and without the patch on a partitioned table with many partitions, indexes, constraints and policies?

## Touch points

- **catalog changes:** The patch scans pg_index, pg_constraint, pg_trigger and pg_policy and writes pg_depend entries during DROP COLUMN.
- **ABI / extension API:** The patch changes src/include/optimizer/optimizer.h and var.c, so it adds or changes an exported helper.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: DROP COLUMN drops whole-row indexes that are still valid
- **planner behaviour change:** see objection standards-compat-2: DROP COLUMN drops whole-row indexes that are still valid
- **security / privileges:** DROP COLUMN ... CASCADE now drops row-level security policies that use a whole-row reference.
- **backpatchability:** The CommitFest entry is tagged Backport with target version stable, and the patch changes what DROP COLUMN and SET DATA TYPE do.
- **maintenance burden ("who maintains this in 10 years"):** tablecmds.c gets a hand-written list of four catalogs to scan, which each new expression-holding object class must extend.
- **performance on the hot path:** Every DROP COLUMN and SET DATA TYPE scans and parses dependent expressions while holding an ACCESS EXCLUSIVE lock.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the dependency code (pg_depend, find_expr_references_walker) and ALTER TABLE in tablecmds.c. None is identifiable in the visible thread.
- Priority: fix design first; audience niche; review cost medium. Whole-row references in constraints, indexes, triggers and policies are uncommon, but the bug leaves a table unusable for inserts. The single 33.6 KB patch is centered on tablecmds.c, and review is of limited use until the drop-or-rebuild question and the CF 7071 overlap are settled.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
