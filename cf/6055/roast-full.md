# Design pre-review: Whole-row fixes for DROP COLUMN, SET COLUMN DATA TYPE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6055](https://commitfest.postgresql.org/patch/6055/) (Needs review); authors: Jian He (stayreal)
- Thread: [let ALTER TABLE DROP COLUMN drop whole-row referenced object](https://www.postgresql.org/message-id/flat/CACJufxGA6KVQy7DbHGLVw9s9KKmpGyZt5ME6C7kEfjDpr2wZCw%40mail.gmail.com), started 2025-09-08
- Messages read: 17 of 17
- Patch prose read: v14-0001-Whole-row-fixes-for-DROP-COLUMN-SET-COLUMN-DATA-TYPE.patch, v1-0001-let-ALTER-TABLE-DROP-COLUMN-drop-wholerow-referenced-object.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The bug is real and reviewers reproduced it, so the goal stands. The design is contested: message #17 argues that whole-row indexes stay valid and should be rebuilt, and that pg_depend should record whole-row dependencies, and no reply is visible (standards-compat-2, prior-attempts-2, maintenance-fit-1). The Backport tag adds an undiscussed behavior change for stable branches (replication-ondisk-1). The list needs to choose one design before code review is useful.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check how the patch detects whole-row references in each catalog, how it handles partitions and policies that reference other tables, or whether the pg_depend entries written during DROP COLUMN are safe if the command fails.

## The proposal, restated

**Problem.** ALTER TABLE DROP COLUMN leaves behind CHECK constraints and indexes that use a whole-row reference such as `check (ts = row(1,1,1))`, because no pg_depend entry links them to the columns. After the drop, inserts fail with errors like "cannot compare record types with different numbers of columns". ALTER COLUMN SET DATA TYPE leaves such objects in the same broken state.

**Proposed design.** During DROP COLUMN, the patch finds indexes, CHECK constraints, trigger WHEN clauses and row-level security policies with whole-row references and records a dependency on the dropped column, so performMultipleDeletions handles them. Indexes and constraints are then dropped like other column dependents, and triggers and policies need CASCADE. SET DATA TYPE raises an error when such objects exist, because record_eq cannot compare records whose columns have different types.

**User-visible surface.**
- ALTER TABLE ... DROP COLUMN now drops indexes and CHECK constraints that use a whole-row reference to the table.
- ALTER TABLE ... DROP COLUMN fails without CASCADE when a trigger WHEN clause or a policy uses a whole-row reference.
- ALTER TABLE ... ALTER COLUMN ... SET DATA TYPE fails with a new error when a whole-row dependent object exists.
- New error message: "cannot alter table column ... because constraint ... uses table ... row type", with a HINT to drop the constraint.

**What the author claims.**
- The documentation says indexes and table constraints involving the column are dropped, so dropping whole-row dependents is the expected behavior.
- Without the fix, inserts fail after DROP COLUMN or SET DATA TYPE and the cause is hard to find.
- An explicit error in SET DATA TYPE helps users find the problem faster than the later record_eq error.
- The CommitFest entry is tagged as a bug fix and backport candidate, with target version stable.

**Thread state.** Chao Li, Kirill Reshke and Jinbinge reviewed the code, and lakshmi tested v8 and v14 and reported correct behavior. Matthias van de Meent objected that scanning catalogs is slow and that indexes should be rebuilt, not dropped, and pointed to his own patch (CF 7071). No reply to that objection is visible.

## 2. Prior attempts

### Revert applying column aliases to the output of whole-row Vars. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac)
- **Thread:** [https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us)
- **Date:** decided 2022-03-17; started 2014-11-10
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** In commit bf7ca1587, I had the bright idea that we could make the result of a whole-row Var (that is, foo.*) track any column aliases that had been applied to the FROM entry the Var refers to. However, that's not terribly logically consistent, because now the output of the Var is no longer of the named composite type that the Var claims to emit. bf7ca1587 tried to handle that by changing the output tuple values to be labeled with a blessed RECORD type, but that's really pretty disastrous: we can wind up storing such tuples onto disk, whereupon they're not readable by other sessions. The only practical fix I can see is to give up on what bf7ca1587 tried to do, and say that the column names of tuples produced by a whole-row Var are always those of the underlying named composite type, query aliases or no. While this introduces some inconsistencies, it removes others, so it's not that awful in the abstract. What *is* kind of awful is to make such a behavioral change in a back-patched bug fix. But corrupt data is worse, so back-patched it will be. (A workaround available to anyone who's unhappy about this is to introduce an extra level of sub-SELECT, so that the whole-row Var is referring to the sub-SELECT's output and not to a named table type. Then the Var is of type RECORD to begin with and there's no issue.) Per report from Miles Delahunty. The faulty commit dates to 9.5, so back-patch to all supported branches.
- **Reason (quote):** "What *is* kind of awful is to make such a behavioral change in a back-patched bug fix."
- **What is different this time:** P1 changed the column names that whole-row Vars output, a different feature. It is relevant only as an example of a behavior change back-patched to prevent corrupt on-disk data, which this proposal does not claim.

### Revert per-index collation version tracking feature. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721)
- **Thread:** [https://www.postgresql.org/message-id/flat/CA+hUKGLhj5t1fcjqAu8iD9B3ixJtsTNqyCCD4V0aTO9kAKAjjA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKGLhj5t1fcjqAu8iD9B3ixJtsTNqyCCD4V0aTO9kAKAjjA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/CA+hUKGJ_hk3rU==g2FpAMChb_4i+TJacpjjqFsinY-tRM3FBmA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKGJ_hk3rU==g2FpAMChb_4i+TJacpjjqFsinY-tRM3FBmA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/3044030.1618261159@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/3044030.1618261159@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/HK0PR01MB22744393C474D503E16C8509F4709@HK0PR01MB2274.apcprd01.prod.exchangelabs.com](https://www.postgresql.org/message-id/flat/HK0PR01MB22744393C474D503E16C8509F4709@HK0PR01MB2274.apcprd01.prod.exchangelabs.com), [https://www.postgresql.org/message-id/flat/3564817.1618420687@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/3564817.1618420687@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/20210117215940.GE8560@telsasoft.com](https://www.postgresql.org/message-id/flat/20210117215940.GE8560@telsasoft.com)
- **Date:** decided 2021-05-07; started 2020-11-15
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Design problems were discovered in the handling of composite types and record types that would cause some relevant versions not to be recorded. Misgivings were also expressed about the use of the pg_depend catalog for this purpose. We're out of time for this release so we'll revert and try again. Commits reverted: 1bf946bd: Doc: Document known problem with Windows collation versions. cf002008: Remove no-longer-relevant test case. ef387bed: Fix bogus collation-version-recording logic. 0fb0a050: Hide internal error for pg_collation_actual_version(<bad OID>). ff942057: Suppress "warning: variable 'collcollate' set but not used". d50e3b1f: Fix assertion in collation version lookup. f24b1569: Rethink extraction of collation dependencies. 257836a7: Track collation versions for indexes. cd6f479e: Add pg_depend.refobjversion. 7d1297df: Remove pg_collation.collversion.
- **Reason (quote):** "Misgivings were also expressed about the use of the pg_depend catalog for this purpose."
- **What is different this time:** P20 tracked collation versions in pg_depend and missed composite and record types. This patch records pg_depend entries only at DROP COLUMN time, for a different purpose.


## 3. Objections a reviewer is likely to raise

### 1. DROP COLUMN drops whole-row indexes that are still valid

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, planner behaviour change · `standards-compat-2`

A table has a partial index `on t (b) where t is not null`. A DBA drops an unrelated column `a`. The index on `b` is dropped with no notice and no CASCADE, so queries that used it turn into sequential scans, and nobody sees it until they look at plans.

An expression such as `t is null` stays well defined after a column is dropped. Only comparisons with a fixed-width row constant break. The patch treats every whole-row reference like a direct column reference, so it drops valid objects along with broken ones. In message #17 Matthias van de Meent proposed rebuilding these indexes, and the thread does not answer him.

Thread: [#1 (jian he, 2025-09-08)](https://www.postgresql.org/message-id/CACJufxGA6KVQy7DbHGLVw9s9KKmpGyZt5ME6C7kEfjDpr2wZCw%40mail.gmail.com), [#17 (Matthias van de Meent, 2026-08-05)](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com)

**Question:** Which whole-row index or constraint definitions become invalid after DROP COLUMN, and why drop the ones that stay valid instead of rebuilding them?

Counter-check: held. The author's reply is the doc sentence that indexes involving the column are dropped, but `t is not null` stays well defined after the drop. Message #17 makes this point and the thread does not answer it.

### 2. Competing fix in CF 7071 rebuilds indexes and has no reply

fixable · level 2, rubric item 12 · design, catalog changes, maintenance burden ("who maintains this in 10 years") · `prior-attempts-2`

This patch ships and whole-row indexes are dropped on DROP COLUMN. A later release adopts the CF 7071 approach, which rebuilds them. Users then see the same command drop an index in one release and keep it in the next.

Message #17 says the whole-row index definition stays valid and only needs a rebuild. It points to CF 7071, which records whole-row Vars in pg_depend under their own objsubid. Message #14 ties this work to CF 6755, but no message answers message #17. Two fixes for one class of bug with opposite user-visible results would likely make a committer wait until one design is chosen.

Thread: [#14 (jian he, 2026-05-25)](https://www.postgresql.org/message-id/CACJufxGv5zmiRyRB%2BbpEWjWKh0%2Bka_bVadnFKs8%2BspY7wSak4A%40mail.gmail.com), [#17 (Matthias van de Meent, 2026-08-05)](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com)

**Question:** Why drop whole-row indexes on DROP COLUMN instead of rebuilding them as CF 7071 does, and can the two patches be reconciled into one design?

Counter-check: held. Message #17 names CF 7071 as a different fix for the same class of bug, and no reply is visible. Message #14 covers only CF 6755, so which design should win is still unanswered.

### 3. Per-catalog scans for whole-row references must be extended by hand

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), catalog changes · `maintenance-fit-1`

A later release adds a new kind of object that stores an expression over a table's row. Its author does not know that tablecmds.c keeps a hand-written list of catalogs to scan for whole-row Vars. DROP COLUMN then leaves the new object broken, and users find out only when inserts fail.

The patch scans pg_index, pg_constraint, pg_trigger and pg_policy one by one inside ALTER TABLE, and keeps pg_depend unchanged. Nothing fails loudly when an object class is missing from that list. Message #17 proposes recording whole-row Vars in pg_depend with their own objsubid, so the dependency code would cover every object class and the scan would cost O(whole-row dependencies), not O(all dependents). No reply to message #17 is visible.

Thread: [#8 (jian he, 2025-12-23)](https://www.postgresql.org/message-id/CACJufxEApboeVKP4jdYOkikOij22HrHWnuSb%3Du-cJ7AHMBcOxA%40mail.gmail.com), [#15 (jian he, 2026-07-14)](https://www.postgresql.org/message-id/CACJufxFeYySYRFOvUcvXeb6h0Q7FA6AAhn%2BxA%2BAaSbXU9c98Zw%40mail.gmail.com), [#17 (Matthias van de Meent, 2026-08-05)](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com)

**Question:** Why is a scan list in tablecmds.c preferable to recording whole-row dependencies in pg_depend, and what makes a future object class fail loudly if it is left out?

Counter-check: held. The patch does record pg_depend entries, but only at ALTER time after scanning four catalogs by hand. The proposal does not make a new expression-holding object class fail loudly, and message #17's alternative has no reply.

### 4. SET DATA TYPE now fails for whole-row objects that would still work

fixable · level 2, rubric item 10 · design, SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-3`

A user widens `int` to `bigint` on a table that has an index on `(t is null)` or a policy using the row. Today the command succeeds and the index is rebuilt. With the patch, it errors and the HINT asks them to drop the object first, so a type change that used to be routine now needs manual drop and re-create.

The commit message justifies the error with record_eq, which only matters when a row is compared with another row value. The brief and message #15 describe the error as raised for any whole-row dependent object. That is broader than the failure it prevents. Message #17 argues for rebuilding instead, and no reply is visible.

Thread: [#4 (jian he, 2025-09-12)](https://www.postgresql.org/message-id/CACJufxHmmsg7GTD_LpGGam09%2BAq4KkYHe2dfPgtGjrUELqbK%3Dw%40mail.gmail.com), [#15 (jian he, 2026-07-14)](https://www.postgresql.org/message-id/CACJufxFeYySYRFOvUcvXeb6h0Q7FA6AAhn%2BxA%2BAaSbXU9c98Zw%40mail.gmail.com), [#17 (Matthias van de Meent, 2026-08-05)](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com)

**Question:** Can the error be limited to row comparisons that record_eq rejects, so that other whole-row objects are rebuilt as non-whole-row expression indexes are today?

Counter-check: held. The commit message justifies the error with record_eq but raises it for all whole-row dependent objects. An index on `(t is null)` does not go through record_eq, so the error is broader than its stated reason.

### 5. Backpatching silently drops user indexes and constraints in a minor release

fixable · level 1b, rubric item 6 · backpatchability, catalog changes · `replication-ondisk-1`

A deployment script runs `alter table ... drop column` after a minor update. A unique index on a whole-row expression is now dropped without notice, so later inserts are no longer checked for uniqueness. Where a policy or trigger WHEN clause uses the row, the same script now fails without CASCADE, and the release stops.

The CommitFest entry tags this Backport with target version stable. The patch changes DDL results in two ways: some objects are dropped implicitly, and some commands that succeeded before now fail. Matthias in message #17 argues that whole-row indexes should be rebuilt, not dropped, which would avoid losing them, and no reply is visible. A committer would likely ask for master-only behavior, or for a narrower back branch fix such as an error.

**Precedents.**
- [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (Reverted, 2022-03-17); thread: [link](https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us)

Thread: [#15 (jian he, 2026-07-14)](https://www.postgresql.org/message-id/CACJufxFeYySYRFOvUcvXeb6h0Q7FA6AAhn%2BxA%2BAaSbXU9c98Zw%40mail.gmail.com), [#17 (Matthias van de Meent, 2026-08-05)](https://www.postgresql.org/message-id/CAEze2Wgg-GbHNJ7iijUoRX7yWPwTJSwZ%2Bq9kQK%2B9-yVw%3DjLucg%40mail.gmail.com)

**Question:** Why is dropping or blocking safe to backpatch, and why is a back branch fix that only raises an error, leaving existing objects untouched, not enough?

Counter-check: held. The CommitFest entry says Backport with target version stable, and no message discusses whether stable branches can take new implicit drops and new DROP COLUMN failures. [P1] is only an illustration; the objection stands without it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch scans pg_index, pg_constraint, pg_trigger and pg_policy and writes pg_depend entries during DROP COLUMN. |
| ABI / extension API | **yes** | The patch changes src/include/optimizer/optimizer.h and var.c, so it adds or changes an exported helper. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: DROP COLUMN drops whole-row indexes that are still valid |
| new GUC | no |  |
| planner behaviour change | **yes** | see objection standards-compat-2: DROP COLUMN drops whole-row indexes that are still valid |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | DROP COLUMN ... CASCADE now drops row-level security policies that use a whole-row reference. |
| backpatchability | **yes** | The CommitFest entry is tagged Backport with target version stable, and the patch changes what DROP COLUMN and SET DATA TYPE do. |
| maintenance burden ("who maintains this in 10 years") | **yes** | tablecmds.c gets a hand-written list of four catalogs to scan, which each new expression-holding object class must extend. |
| performance on the hot path | **yes** | Every DROP COLUMN and SET DATA TYPE scans and parses dependent expressions while holding an ACCESS EXCLUSIVE lock. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which whole-row index and constraint definitions become invalid after DROP COLUMN, and why drop the ones that stay valid, such as `t is not null`, instead of rebuilding them?
2. How does this patch relate to CF 7071, and which of the two should be the long-term mechanism for whole-row dependencies?
3. Which parts of the patch are meant for stable branches, and why is a back-branch fix that only raises an error not enough?
4. Can the SET DATA TYPE error be limited to row comparisons that record_eq rejects, with other whole-row objects rebuilt as expression indexes are today?
5. What does DROP COLUMN cost with and without the patch on a partitioned table with many partitions, indexes, constraints and policies?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the dependency code (pg_depend, find_expr_references_walker) and ALTER TABLE in tablecmds.c. None is identifiable in the visible thread.
- **Liability:** The owner signs for new implicit drops and new errors in DROP COLUMN and SET DATA TYPE, in every stable branch if back-patched. The owner also signs for a catalog scan list that later object classes must extend.
- **Author follow-up:** The author posted 14 versions and kept the CommitFest entry moving for over a year. No reply to message #17 is visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Whole-row references in constraints, indexes, triggers and policies are uncommon, but the bug leaves a table unusable for inserts. The single 33.6 KB patch is centered on tablecmds.c, and review is of limited use until the drop-or-rebuild question and the CF 7071 overlap are settled.

## Appendix

### Held up in the counter-check, not listed above

- `demand-1` (fixable, level 1b) Every DROP COLUMN pays catalog scans for a rare whole-row case. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` Back-patched DROP COLUMN behavior change without data corruption to justify it. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same back-patch concern with a more concrete scenario and a narrower alternative for back branches.
- `standards-compat-1` Back-patching changes DROP COLUMN and SET DATA TYPE behavior in stable branches. Refuted (not specific): Duplicate of replication-ondisk-1. The SET DATA TYPE part of the scenario is covered by standards-compat-3.
- `replication-ondisk-2` Clusters that already ran DROP COLUMN keep broken objects. Refuted (no failure scenario): The broken objects exist today without the patch, and the patch does not make that state worse. Release-note advice for existing clusters is outside levels 1-2 and does not count against the design.
- `maintenance-fit-2` Back-patching a change that drops user indexes and blocks DDL. Refuted (not specific): Duplicate of replication-ondisk-1. Its point about back branches diverging from master is a variant of the same back-patch concern.
- `maintenance-fit-3` Three patches change whole-row dependency handling without a shared plan. Refuted (not specific): Duplicate of prior-attempts-2. Message #14 already says how the patch relates to CF 6755, so only the CF 7071 question is open, and prior-attempts-2 covers it.
- `already-exists-1` Ad hoc catalog scans duplicate what pg_depend is meant to record. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about per-catalog scans versus pg_depend and adds the missing loud failure for new object classes.

### Review angles

- prior-attempts: 2 objection(s). No precedent found for the same idea. Both objections come from the backport tag and from the competing approach in message #17.
- standards-compat: 3 objection(s). Three objections raised. The touched files include no documentation changes, but that is outside this angle.
- replication-ondisk: 2 objection(s). The patch adds no on-disk format, WAL record or wire protocol change. Its pg_depend entries are written inside the DDL transaction, so this angle sees no replay or standby concern.
- security: 0 objection(s). The patch adds no new privilege, role or default, and every new path either errors out or follows the CASCADE rules that column-referencing policies and triggers already follow. A broken whole-row policy left by an unhandled case (for example, a reference to another table) fails at query time with a record_eq error, so it denies access and does not leak rows. This check rests on the prose only, since the code is not shown.
- maintenance-fit: 3 objection(s). Objections are listed above. No hot-path cost was found, because the scans run only during ALTER TABLE.
- already-exists: 1 objection(s). There is no existing feature or extension that gives the right behavior today. The only workaround is to drop the affected objects by hand after the failure, so the goal itself is not already met.
- demand: 1 objection(s). The underlying bug is real and leaves tables unusable for inserts, so a lack of demand alone would not justify rejecting a fix. The only demand-side objection is that every user pays for the check while few are affected.

### Precedents retrieved

- P1: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - used in the review
- P2: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P3: [Revert "Temporarily make pg_ctl and server shutdown a whole lot chattier."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c5e9b771275b93b09eec6b760677fe6c5e726ab2) (revert, Reverted, 2016-02-10) - judged not relevant
- P4: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P5: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P6: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P7: [Bug: Reading from single byte character column type may cause out of bounds memory reads](https://commitfest.postgresql.org/patch/3767/) (cf, Returned with feedback, 2022-09-08) - judged not relevant; reason summarised from its thread
- P8: [Avoid orphaned dependencies](https://commitfest.postgresql.org/patch/3106/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P9: [Remove dependence on integer wrapping](https://commitfest.postgresql.org/patch/5035/) (cf, Withdrawn, 2024-06-17) - judged not relevant; reason summarised from its thread
- P10: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P11: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P12: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P13: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - judged not relevant
- P14: [Revert "Fix isolation test to be less timing-dependent"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6c8be5962aea2eee8c366c01bbbcf5bf5ddf5294) (revert, Reverted, 2018-01-03) - judged not relevant
- P15: [Revert "Have dtrace depend on object files directly, not objfiles.txt"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=08fbad0afd62690cc82990c0504529ef238ac24d) (revert, Reverted, 2015-10-15) - judged not relevant
- P16: [Optimize walsender handling invalid messages of 'drop publication'](https://commitfest.postgresql.org/patch/4393/) (cf, Withdrawn, 2023-10-20) - judged not relevant; reason summarised from its thread
- P17: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant; reason summarised from its thread
- P18: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P19: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant
- P20: [Revert per-index collation version tracking feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721) (revert, Reverted, 2021-05-07) - used in the review
- P21: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P22: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant
- P23: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant
- P24: [Referential Integrity Checks with Statement Level Triggers](https://commitfest.postgresql.org/patch/1975/) (cf, Returned with feedback, 2019-08-01) - judged not relevant

### Warnings

- tone lint: 1 finding(s) remain after the repair call; see tone_lint in roast.json before forwarding this text

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 175 s; cost reported by the CLI: $2.35
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 109 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 1 left
  - `refuted[1].argument`: hyperbole ("awful")
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:40:56.279Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
