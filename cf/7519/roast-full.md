# Design pre-review: Fix for bug #19773 : pg_dump differing generation expressions of inherited columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7519](https://commitfest.postgresql.org/patch/7519/) (Needs review); authors: Nitin Motiani (nitinmotiani)
- Thread: [\[PATCH v1\] Fix for bug #19773 : pg_dump differing generation expressions of inherited columns](https://www.postgresql.org/message-id/flat/CAH5HC97y7PFE6rJh8kVDUszPxGQE030nZvyHu%3DwZT7FBkOV3qA%40mail.gmail.com), started 2026-10-21
- Messages read: 8 of 8
- Patch prose read: v2-0001-pg_dump-Dump-differing-generation-expressions-of-.patch, v1-0001-pg_dump-Dump-differing-generation-expressions-of-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is a reported bug fix (#19773) for wrong restores that report no error, and no surviving objection questions the goal or the v2 approach. The one surviving objection, replication-ondisk-1, concerns commit order. Messages #2 and #6 show that the emitted SET EXPRESSION drops a child's inherited CHECK and NOT NULL rows, and the thread does not say whether the backpatch to 17 waits for the backend fix. A committer who knows pg_dump and the stable branches has to decide that.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check that the decision made after sorting covers every case where loop repair makes an expression separate. It also could not check how the patch behaves for the multi-parent case.

## The proposal, restated

**Problem.** pg_dump drops a child table's generation expression when it differs from the parent's, so the restored child silently uses the parent's expression and computes wrong values (bug #19773). The state became possible in v16, and since v17 ALTER TABLE ... SET EXPRESSION can produce it on an inherited column. Users of table inheritance with generated columns are affected.

**Proposed design.** pg_dump emits a separate ALTER TABLE ONLY ... SET EXPRESSION for a child whose column is not printed, as it does for defaults. It does so when the child's expression differs from the parent's, or when it matches but the parent's expression is itself dumped separately. v2 makes that second decision after sorting the dumpable objects instead of adding dependency edges, because v1's edges caused sort hangs.

**User-visible surface.**
- pg_dump output now contains ALTER TABLE ONLY ... ALTER COLUMN ... SET EXPRESSION AS (...) for inherited generated columns.

**What the author claims.**
- Fixes the reported cases for STORED and VIRTUAL columns; the reviewer confirmed this for v1.
- v2 fixes the sort hang and the multi-level cases where the child silently got the old parent expression.
- Checking after the sort is cleaner and easier to maintain than adding new loop-repair rules.
- Always emitting SET EXPRESSION at the end was rejected because restore to v16 and older lacks that command.
- Backpatch through v17.

**Thread state.** Henson Choi reviewed v1 and v2, found the v2 approach sound, and leaves the multi-parent case and the loss of inherited CHECK and NOT NULL constraints on SET EXPRESSION as separate backend bugs, which the author will take up. Ashutosh Bapat suggests testing through 002_pg_upgrade, and jian he points to known SET EXPRESSION dependency problems; neither point has an answer yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-21; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Patch makes dump/restore produce inconsistent inherited-constraint catalog state

fixable · level 2, rubric item 7 · catalog changes, on-disk format / pg_upgrade, backpatchability · `replication-ondisk-1`

A v17 user with an inherited CHECK or NOT NULL on a generated column dumps with a patched pg_dump and restores. The restore reports no error, but the child's pg_constraint rows are missing, so rows that violate the CHECK are accepted. Later, pg_upgrade of that cluster passes --check and then fails with "child table is missing constraint", and DROP NOT NULL on the parent fails with "cache lookup failed".

Message #2 and message #6 show that the emitted ALTER TABLE ONLY ... SET EXPRESSION drops inherited CHECK and NOT NULL rows through ATPostAlterTypeCleanup(). The bug is in the backend, but today pg_dump never emits that command, so ordinary dump/restore does not hit it; the patch would make every such dump hit it. Today the child gets the wrong expression; with the patch it gets the right expression and a catalog that a later pg_upgrade rejects. The thread does not say whether the patch waits for the backend fix, or whether that fix will be backpatched to 17.

**Precedents.**
- [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (Reverted, 2024-02-20); thread: [link](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org)

Thread: [#2 (Henson Choi, 2026-10-22)](https://www.postgresql.org/message-id/CAAAe_zAZ%2BvjNaemjE5C3khRGjWa2d0ML%2Be%3D00-tqcXd%2BFOiFLw%40mail.gmail.com), [#4 (Nitin Motiani, 2026-10-26)](https://www.postgresql.org/message-id/CAH5HC95SCKZKi%3Dwwcd5Kj2QqjMAwAb7%3DE5GV-3WcYF6Fz1yzXQ%40mail.gmail.com), [#6 (Henson Choi, 2026-11-03)](https://www.postgresql.org/message-id/CAAAe_zCWfn8Krssct-H%3DCZZ4huGDiGwypFZDyA9n1vw9c50gGw%40mail.gmail.com)

**Question:** Will this go in only together with a backend SET EXPRESSION fix, backpatched to 17, so that no released minor version restores into a catalog that pg_upgrade later rejects?

Counter-check: held. Messages #2 and #6 show that v2 restores drop inherited CHECK constraints without an error, and the thread does not say whether the backpatch waits for the backend fix. P18 reverted a backend feature and must not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade uses pg_dump, and a restored cluster with missing inherited constraint rows later fails pg_upgrade with "child table is missing constraint". |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | see objection replication-ondisk-1: Patch makes dump/restore produce inconsistent inherited-constraint catalog state |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump now emits ALTER TABLE ONLY ... SET EXPRESSION for inherited generated columns, which changes what a restore produces. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message says "Backpatch-through: 17", and the emitted command triggers a backend bug that has no posted fix. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v2 adds a decision after sortDumpableObjects() that only generation expressions use, next to the existing path for defaults. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will this patch be committed and backpatched to 17 only together with a backend fix that keeps inherited CHECK and NOT NULL constraints on SET EXPRESSION?
2. Will the backend fix for constraint loss on SET EXPRESSION itself be backpatched to 17, or only applied to master?
3. What is the plan for the multi-parent case, which still fails at restore with v2 and on master?
4. Do the SET EXPRESSION dependency problems that jian he linked affect any command that v2 now emits in a dump?
5. Can the cases from messages #2 and #6 stay in the regression database so that 002_pg_upgrade checks them, as message #7 suggests?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows pg_dump's dependency sorting and inheritance handling, and who can judge backpatch risk for the stable branches. None has spoken in the visible thread.
- **Liability:** The owner signs for changed pg_dump output in 17 and later, including pg_upgrade runs. Until the backend fix lands, the owner also signs for restores that drop inherited CHECK and NOT NULL rows without an error.
- **Author follow-up:** The author posted v2 after the v1 review, and the brief says the author will take up the two backend bugs.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The bug affects only inheritance trees with differing generation expressions, but the restore gives wrong values without an error, so it merits a fix. The patch is about 20 KiB in pg_dump only; the open point is commit order with the backend fix.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Inheritance dump fix tested only in pg_dump TAP, not pg_upgrade. Refuted (no failure scenario): The scenario depends on an unnamed future backend change, and where the tests live is a level 4 matter outside this review. P18 and P1 concern a backend feature that pg_dump could not dump, so they do not support this objection about a pg_dump fix.
- `standards-compat-1` Restores with v2 lose inherited CHECK and NOT NULL constraints. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same constraint loss and the same commit-order question more completely.
- `maintenance-fit-1` Backpatched dumps would trade wrong expressions for lost inherited constraints. Refuted (not specific): Duplicate of replication-ondisk-1, which covers the same constraint loss and the same question about the order of the two fixes.
- `maintenance-fit-2` Post-sort decision adds a second, generation-only path beside defaults. Refuted (already handled): The commit message says the decision runs after sorting because loop repair can make an expression separate, so the check sees that case by design. Message #4 weighed new loop-repair rules and rejected them as harder to maintain, and it ties the defaults limitation to the multi-parent case only.

### Review angles

- prior-attempts: 1 objection(s). No precedent rejects this idea. The one objection rests on the recorded coverage concern in P18 and P1, which message #7 raises and nobody has answered yet.
- standards-compat: 1 objection(s). Other points from this angle do not hold up. The new ALTER TABLE ONLY ... SET EXPRESSION follows the existing pattern for separate defaults, and the multi-parent failure exists on master already.
- replication-ondisk: 1 objection(s). Only pg_dump output changes. There is no new WAL, on-disk format, catalog column or protocol change. The one persistent-state concern is the catalog state that restore and pg_upgrade produce.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. pg_dump only emits ALTER TABLE ONLY ... SET EXPRESSION, which the table owner can already run, and a restore already evaluates generation expressions, so no new code runs as the restoring role. The loss of inherited CHECK and NOT NULL constraints is a data-integrity issue for other angles.
- maintenance-fit: 2 objection(s). The goal itself, fixing silent loss of child generation expressions in pg_dump, is a clear bug fix and belongs in core. The objections above concern sequencing and maintainability only.
- already-exists: 0 objection(s). This is a pg_dump bugfix for silently wrong restores, and no existing feature, option or extension gives a correct dump of a child's differing generation expression. The patch extends the existing separate-ALTER mechanism used for defaults, so it does not add a second mechanism.
- demand: 0 objection(s). Demand is shown: this fixes a reported bug (#19773) where pg_dump restores a wrong generation expression without any error. Correct dump and restore is a core duty of pg_dump, a second developer was tracking the same defect (message #2), and the cost falls only on dumps that contain inherited generated columns.

### Precedents retrieved

- P1: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - used in the review; reason summarised from its thread
- P2: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant; reason summarised from its thread
- P3: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P4: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P5: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P6: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P7: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant; reason summarised from its thread
- P8: [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://commitfest.postgresql.org/patch/5992/) (cf, Returned with feedback, 2026-04-02) - used in the review; reason summarised from its thread
- P9: [Infinite Autovacuum loop caused by failing virtual generated column expression](https://commitfest.postgresql.org/patch/7100/) (cf, Withdrawn, 2026-08-19) - judged not relevant
- P10: [Improving test coverage of extensions with pg_dump](https://commitfest.postgresql.org/patch/187/) (cf, Returned with feedback, 2015-12-02) - judged not relevant
- P11: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P12: [Add --include-table-data-where option to pg_dump, to export only a subset of table data](https://commitfest.postgresql.org/patch/1644/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P13: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant
- P14: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P15: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant
- P16: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant
- P17: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant
- P18: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - used in the review
- P19: [Revert "pg_dump: Lock all relations, not just plain tables".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d3adaabaf7d555ec8bb1d83c43f72e79f1bf0b7d) (revert, Reverted, 2020-11-06) - judged not relevant
- P20: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P21: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P22: [\[PG19\]pg_dump --statistics restores extended statistics by attribute number](https://commitfest.postgresql.org/patch/7446/) (cf, Withdrawn, 2026-10-19) - judged not relevant
- P23: [foreign key on virtual generated column](https://commitfest.postgresql.org/patch/5782/) (cf, Withdrawn, 2026-09-23) - judged not relevant
- P24: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 121 s; cost reported by the CLI: $1.62
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 69 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:03:25.828Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
