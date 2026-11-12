# Design pre-review: NOT NULL NOT ENFORCED

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6029](https://commitfest.postgresql.org/patch/6029/) (Needs review); authors: Jian He (stayreal)
- Thread: [NOT NULL NOT ENFORCED](https://www.postgresql.org/message-id/flat/CACJufxFbH1_9BDow%3D4nMSdBfLSOAkiGD5hxO6bouWjZAyHbV%2BA%40mail.gmail.com), started 2025-09-04
- Messages read: 18 of 18
- Patch prose read: v6-0001-NOT-NULL-NOT-ENFORCED.patch, v1-0001-NOT-NULL-NOT-ENFORCED.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and no level fails. A committer who knows the not-null code reviewed the design, and the author followed him. The open points are fixable: the F492 remark (standards-compat-1), the SET NOT NULL error (standards-compat-2) and unproven dump round-trip for mixed hierarchies (prior-attempts-1). Demand (demand-1, demand-2) rests only on standard conformance, which lowers priority but does not block.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check whether information_schema views and other readers of not-null pg_constraint rows test conenforced, or whether v6 fixed the 002_pg_dump.pl failures reported for v4.

## The proposal, restated

**Problem.** Postgres supports NOT ENFORCED for foreign key and CHECK constraints but not for not-null constraints, so sql_features.txt lists F492 with the remark "except not-null constraints". The author wants to complete F492. The thread does not name users who asked for this or a use case beyond standard conformance.

**Proposed design.** A NOT ENFORCED not-null constraint is stored in pg_constraint with conenforced and convalidated both false, and pg_attribute.attnotnull stays false, so nulls are accepted and existing rows are not checked. A column keeps at most one not-null constraint, and the rules forbid an enforced parent with a not-enforced child. A column that inherits conflicting statuses from several parents gets an enforced constraint.

**User-visible surface.**
- SQL syntax: NOT NULL NOT ENFORCED in CREATE TABLE column and table constraints, and in ALTER TABLE ADD CONSTRAINT ... NOT NULL col NOT ENFORCED.
- Catalog semantics: pg_attribute.attnotnull now means an enforced (possibly invalid) not-null constraint exists.
- New errors: SET NOT NULL, VALIDATE CONSTRAINT and ALTER CONSTRAINT ... ENFORCED fail on a not-enforced not-null constraint.
- pg_dump emits NOT NULL NOT ENFORCED inline in CREATE TABLE (v4 onward).
- psql \d output changes (describe.c is touched).

**What the author claims.**
- The patch completes SQL feature F492 and removes the "except not-null constraints" remark.
- The regression tests cover the inheritance, partitioning and MERGE/SPLIT PARTITIONS corner cases.
- Changing a constraint's enforceability later is unsupported, so the complexity of commit 0cd17fdd3c does not apply yet.

**Thread state.** Álvaro Herrera reviewed the design and the author changed multiple-inheritance handling, the hint, and pg_dump output to follow him; a separate CREATE TABLE LIKE bug fix was committed by Peter Eisentraut. Solai v reported v5 tests passing, and Sebas Mannem plans to test v6; psql tab completion, raised by Haritabh Gupta, has no visible answer.

## 2. Prior attempts

### Revert "Improve compression and storage support with inheritance" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5)
- **Thread:** [https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org)
- **Date:** decided 2024-02-20; started 2024-02-16
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 0413a556990ba628a3de8a0b58be020fd9a14ed0. pg_dump cannot currently dump all the structures that are allowed by this patch. This needs more work in pg_dump and more test coverage.
- **Reason (quote):** "pg_dump cannot currently dump all the structures that are allowed by this patch."
- **What is different this time:** P20 was about compression and storage inheritance, a different feature. Its reason applies because this patch also allows new inheritance states that pg_dump has to reproduce.


## 3. Objections a reviewer is likely to raise

### 1. F492 marked complete while ALTER CONSTRAINT ENFORCED is rejected for not-null

fixable · level 1b, rubric item 12 · SQL-standard conflict or non-standard syntax · `standards-compat-1`

A user reads sql_features.txt, sees F492 listed as YES with no remark, and writes `alter table t alter constraint t_x_not_null enforced` for a not-null constraint, as they would for a foreign key. The server raises an error. The user learns from the error that support covers only part of the syntax the conformance table now claims.

The patch removes the "except not-null constraints" remark, but the v6 commit message (message #17) says ALTER CONSTRAINT ... [NOT] ENFORCED is not supported for not-null constraints. The commit message implies other constraint types accept it, and message #14 cites a separate commit that added enforceability changes elsewhere. So not-null constraints would remain the one type with a partial F492, and only the remark's wording changes. A committer would likely ask for the remark to be rewritten to name the gap rather than dropped.

Thread: [#14 (jian he, 2026-08-06)](https://www.postgresql.org/message-id/CACJufxHvGs7RR97-vN61Y2mZaF2YEgpHScw5SFG%3DTghFQAN9gA%40mail.gmail.com), [#17 (jian he, 2026-09-28)](https://www.postgresql.org/message-id/CACJufxECCgByCscWPCKCmHtW1f9Riz7S72sQ0HzDht3VQEoWfw%40mail.gmail.com)

**Question:** Does F492 in the standard include changing enforcement through ALTER CONSTRAINT, and if so, why should the remark go away before that works?

Counter-check: held. The v6 commit message both removes the remark and says ALTER CONSTRAINT ... [NOT] ENFORCED is unsupported for not-null constraints. Nothing in the thread says whether F492 covers that syntax, so the question stands and a reworded remark would answer it.

### 2. No user need shown beyond removing the F492 remark

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A user who wants an informational "never null" marker today can write CHECK (col IS NOT NULL) NOT ENFORCED. With the patch, that user gets a second spelling with the same runtime effect: nulls are accepted, attnotnull stays false and the proposal does not say the planner uses it. Postgres would then carry a new constraint state that adds nothing for this user.

Message #1 and the v6 commit message give one reason: completing F492 in sql_features.txt. No message in the thread names a user, a field report, an ETL or data-warehouse workflow, or a migration from another system that needs this syntax. Reviewers in messages #10, #15 and #16 test the patch and comment on its wording, but none of them asks for the feature. An unenforced CHECK constraint already gives the same documentation-only meaning.

Thread: [#1 (jian he, 2025-09-04)](https://www.postgresql.org/message-id/CACJufxFbH1_9BDow%3D4nMSdBfLSOAkiGD5hxO6bouWjZAyHbV%2BA%40mail.gmail.com), [#10 (Haritabh Gupta, 2026-01-06)](https://www.postgresql.org/message-id/176772428435.2452227.15663652319111720473.pgcf%40coridan.postgresql.org), [#15 (Sebas Mannem, 2026-09-11)](https://www.postgresql.org/message-id/E4FK57YYN4wTz9_ExrhZVlf9oCdmiIgMZk2av73brzfGnUlJb-cJt9c9sPhqpwnJeUvF9-5xTdWyFh5zWnzJvPITyfAfVAIW8dwPeS386BY%3D%40mannem.nl), [#16 (solai v, 2026-09-28)](https://www.postgresql.org/message-id/CAF0whuc--DUH%3DJ7aCpbC77id1Vq1CyFbK_xe4qPr8GAVRKuh8g%40mail.gmail.com), [#17 (jian he, 2026-09-28)](https://www.postgresql.org/message-id/CACJufxECCgByCscWPCKCmHtW1f9Riz7S72sQ0HzDht3VQEoWfw%40mail.gmail.com)

**Question:** Which users or migration tools need NOT NULL NOT ENFORCED specifically, and why does CHECK (col IS NOT NULL) NOT ENFORCED not serve them?

Counter-check: held. The thread gives standard conformance as the only reason and shows no user or tool that needs the syntax. Committer engagement and the existing NOT ENFORCED support for CHECK and foreign keys lower the weight, but they do not show demand.

### 3. SET NOT NULL errors and gives no route to enforce the constraint

fixable · level 2, rubric item 10 · design, SQL-standard conflict or non-standard syntax, psql UX conventions · `standards-compat-2`

A schema migration tool runs `alter table t alter column x set not null` on a column that has a NOT ENFORCED not-null constraint. Today SET NOT NULL either adds the constraint or does nothing. With the patch it raises an error, and the only way forward is to drop the constraint and run SET NOT NULL again.

Message #1 and message #17 make SET NOT NULL, VALIDATE CONSTRAINT and ALTER CONSTRAINT ... ENFORCED all fail on a not-enforced not-null constraint, so the state is one-way. Message #4 suggested `SET NOT NULL NOT ENFORCED` and letting SET NOT NULL do the expected thing, and the thread has no answer to it. SET NOT NULL is the usual Postgres verb for not-null, so a NOT ENFORCED form that only ADD CONSTRAINT can create, and that SET NOT NULL cannot turn into an enforced one, departs from existing practice without a stated reason.

Thread: [#1 (jian he, 2025-09-04)](https://www.postgresql.org/message-id/CACJufxFbH1_9BDow%3D4nMSdBfLSOAkiGD5hxO6bouWjZAyHbV%2BA%40mail.gmail.com), [#4 (Álvaro Herrera, 2025-09-05)](https://www.postgresql.org/message-id/202509051923.n4xc5vthqyhu%40alvherre.pgsql), [#17 (jian he, 2026-09-28)](https://www.postgresql.org/message-id/CACJufxECCgByCscWPCKCmHtW1f9Riz7S72sQ0HzDht3VQEoWfw%40mail.gmail.com)

**Question:** Why should SET NOT NULL on a not-enforced not-null constraint raise an error instead of enforcing and validating it, as it does for a column without one?

Counter-check: held. SET NOT NULL validates a NOT VALID not-null constraint today, so an error for the not-enforced case departs from that practice. Message #4 proposed SET NOT NULL forms and no visible reply addresses them.

### 4. Dump of mixed-enforcement inheritance trees is not shown to round-trip

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, catalog changes, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A user has a child table that inherits a NOT ENFORCED not-null from one parent and an enforced one from a second parent, so the child gets an enforced constraint. pg_dump decides whether to print the child's not-null from what the parents have, and could leave it out. After restore the child accepts nulls, and nobody sees it until bad rows appear.

[P20] was reverted because pg_dump could not dump every structure the patch allowed, and this patch adds new mixed states to the same inheritance logic. Message #11 shows that flagInhAttrs() decides what to print for inherited not-nulls and was written before enforcement could differ between parent and child. Message #13 reports 002_pg_dump.pl failing for v4, and message #16 tested v5 dumps only in general terms. No message lists the parent and child combinations that were dumped and restored.

**Precedents.**
- [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (Reverted, 2024-02-20); thread: [link](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org)

Thread: [#11 (jian he, 2026-02-23)](https://www.postgresql.org/message-id/CACJufxH0130OV4JymGviw8WkB5gehPztsYy3pu-0cmUtL%2BecgA%40mail.gmail.com), [#13 (jian he, 2026-03-03)](https://www.postgresql.org/message-id/CACJufxEBvceJ4Ju%3D7_LtfcL5ekuXK%3DAWgOB1vAN_svKRYetgyg%40mail.gmail.com), [#14 (jian he, 2026-08-06)](https://www.postgresql.org/message-id/CACJufxHvGs7RR97-vN61Y2mZaF2YEgpHScw5SFG%3DTghFQAN9gA%40mail.gmail.com), [#16 (solai v, 2026-09-28)](https://www.postgresql.org/message-id/CAF0whuc--DUH%3DJ7aCpbC77id1Vq1CyFbK_xe4qPr8GAVRKuh8g%40mail.gmail.com)

**Question:** For each allowed parent and child combination, including conflicting multiple parents and partitions, does a dump and restore give the same conenforced and attnotnull on every table?

Counter-check: held. Message #16 reports pg_dump testing only in general terms, and no reply confirms that v6 fixed the 002_pg_dump.pl failures from message #13. The reason recorded for [P20], pg_dump unable to dump every allowed structure, is the same kind of risk.

### 5. Every future not-null change pays for a state few will use

fixable · level 1b, rubric item 4 · maintenance burden ("who maintains this in 10 years"), catalog changes, pg_dump / restore round-trip · `demand-2`

A later patch changes not-null handling in inheritance, partition attach, MERGE/SPLIT PARTITIONS or pg_dump. It must also handle the not-enforced case and its parent/child rules. If it misses that case, mixed hierarchies dump or attach wrongly, and someone finds out at restore or attach time.

The rules span multiple inheritance, partitions, MERGE/SPLIT PARTITIONS (which v4 broke, message #14), pg_dump flagInhAttrs (message #11) and a changed meaning of attnotnull. The author notes that changing enforceability later would bring the complexity of an earlier commit, which future work must face (message #14). All later not-null work carries this cost, but the gain is limited to users of an unenforced not-null constraint, and the thread shows none. An engaged committer (messages #2, #12) lowers the risk but does not supply the demand.

Thread: [#11 (jian he, 2026-02-23)](https://www.postgresql.org/message-id/CACJufxH0130OV4JymGviw8WkB5gehPztsYy3pu-0cmUtL%2BecgA%40mail.gmail.com), [#12 (Álvaro Herrera, 2026-02-23)](https://www.postgresql.org/message-id/202602231223.yggz76dffiro%40alvherre.pgsql), [#14 (jian he, 2026-08-06)](https://www.postgresql.org/message-id/CACJufxHvGs7RR97-vN61Y2mZaF2YEgpHScw5SFG%3DTghFQAN9gA%40mail.gmail.com)

**Question:** What use of not-enforced not-null constraints justifies the extra inheritance, partitioning and pg_dump rules that every later not-null patch must keep correct?

Counter-check: held. Message #14 reports that v4 failed for MERGE/SPLIT PARTITIONS, which shows that not-null code paths must each learn the new state. The thread shows a committer engaged but no user who gains from that cost.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | pg_attribute.attnotnull now means an enforced not-null constraint exists; pg_constraint rows for not-null can have conenforced false. |
| ABI / extension API | **yes** | The patch touches tupdesc.h, heap.h and pg_constraint.h, which extensions include. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds NOT NULL NOT ENFORCED syntax in gram.y and removes the F492 remark in sql_features.txt. |
| new GUC | no |  |
| planner behaviour change | **yes** | relcache.c and tupdesc.h change how column nullability is computed; the thread does not say whether the planner uses a not-enforced constraint. |
| pg_dump / restore round-trip | **yes** | pg_dump.c emits NOT NULL NOT ENFORCED inline, and 002_pg_dump.pl failed for v4. |
| psql UX conventions | **yes** | describe.c changes \d output, and the tab completion question has no visible answer. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Inheritance, partitioning, MERGE/SPLIT PARTITIONS, LIKE and pg_dump code must each handle a not-enforced not-null state. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | CHECK (col IS NOT NULL) NOT ENFORCED already gives the same runtime behavior. |

## 5. Questions the author must answer before anyone reviews code

1. Does F492 in the standard include changing enforcement through ALTER CONSTRAINT, and if so, should the sql_features.txt remark name that gap instead of being removed?
2. Why should SET NOT NULL on a column with a not-enforced not-null constraint raise an error instead of enforcing and validating the constraint?
3. For each allowed parent and child combination, including conflicting multiple parents and partitions, does dump and restore give the same conenforced and attnotnull on every table?
4. Does the planner or executor ever use a not-enforced not-null constraint, for example to prove IS NOT NULL?
5. Which users or migration tools need NOT NULL NOT ENFORCED, and why does CHECK (col IS NOT NULL) NOT ENFORCED not serve them?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera is the listed committer and reviewed the design in message #2; the work needs someone who knows not-null constraint inheritance and pg_dump.
- **Liability:** The owner signs for the new attnotnull meaning and for a not-enforced state in inheritance, partitioning and pg_dump, with no end date.
- **Author follow-up:** The author posted six versions over 14 months, followed the committer's review, and moved the entry forward across CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The thread gives only standard conformance as the reason, so the audience is small. The patch is about 89 KB across tablecmds.c, the catalog code and pg_dump, but a qualified committer is engaged and the open points are fixable.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` attnotnull no longer tells whether a not-null constraint exists. Refuted (already handled): Message #2 raised this point, message #5 answered it, and in message #6 the committer agreed that attnotnull exists to tell applications a null insert will fail. That meaning is kept, and the scenario names no caller that breaks.
- `maintenance-fit-2` The hard part, changing enforceability, is deferred. Refuted (not specific): Duplicate of standards-compat-2 for the user-facing scenario. [P7] and [P1] record pg_upgrade and late structural problems, which say nothing about enforceability, and CHECK constraints also gained NOT ENFORCED before ALTER CONSTRAINT could change it.
- `maintenance-fit-3` Only standard conformance is offered against new inheritance complexity. Refuted (not specific): Duplicate of demand-2, which states the same cost against the same missing demand with more specific thread references.
- `already-exists-1` CHECK (col IS NOT NULL) NOT ENFORCED already gives the same behavior. Refuted (not specific): Duplicate of demand-1, which makes the same CHECK (col IS NOT NULL) NOT ENFORCED point. [P1] and [P7] concern cataloging not-null constraints and must not be used as support.

### Review angles

- prior-attempts: 1 objection(s). No precedent proposes NOT NULL NOT ENFORCED itself. The one objection rests on the pg_dump revert reason in [P20], which applies here and which the thread does not settle.
- standards-compat: 2 objection(s). pg_dump output was settled in messages #9 to #13, and the attnotnull meaning in messages #5 and #6, so neither is raised here.
- replication-ondisk: 0 objection(s). The proposal adds no WAL records, on-disk format, wire messages or catalog columns. It reuses pg_constraint.conenforced and convalidated, which already exist for CHECK and foreign key constraints, and pg_upgrade only meets clusters that hold no not-enforced not-null constraints. The patch narrows pg_attribute.attnotnull to mean an enforced constraint exists. This breaks the earlier rule that attnotnull is set whenever a not-null constraint row exists, but Herrera agreed in message #6 and v6 documents the change, so the point is settled. Not checked without the code: whether information_schema views and other code reading contype 'n' rows check conenforced.
- security: 0 objection(s). No privilege or exposure aspect: adding or dropping a not-null constraint already requires table ownership, and NOT ENFORCED only weakens a check the owner controls. Whether the planner or information_schema treats a not-enforced constraint as a guarantee is a correctness question for another angle, and it cannot be checked without the code.
- maintenance-fit: 3 objection(s). Not applicable. Ownership risk is lower than usual because Álvaro Herrera is the listed committer and reviewed the design in the thread.
- already-exists: 1 objection(s). One objection from this angle. The only alternative found is the existing NOT ENFORCED CHECK constraint, and it does not provide the standard NOT NULL syntax.
- demand: 2 objection(s). Objections are 'not now, show demand'. The design was reviewed by an engaged committer, so this angle does not claim the design is wrong.

### Precedents retrieved

- P1: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - used in the review
- P2: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P3: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P4: [Fix warnings under ENFORCE_REGRESSION_TEST_NAME_RESTRICTIONS](https://commitfest.postgresql.org/patch/4451/) (cf, Rejected, 2023-07-19) - judged not relevant; reason summarised from its thread
- P5: [Add ALWAYS DEFERRED option for CONSTRAINTs and CONSTRAINT TRIGGERs](https://commitfest.postgresql.org/patch/1319/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P6: [Enforce creation of destination folders for source files in pg_regress](https://commitfest.postgresql.org/patch/100/) (cf, Rejected, 2015-03-01) - judged not relevant; reason summarised from its thread
- P7: [Revert "Catalog NOT NULL constraints" and fallout](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9ce04b50e120275afbc03ca0b80839dde3da8308) (revert, Reverted, 2023-04-12) - used in the review
- P8: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - used in the review; reason summarised from its thread
- P9: [Enforce typmod for function inputs, function results and parameters for spi_prepare'd statements called from PLs](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Functions) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P10: [Revert "pg_dump: Lock all relations, not just plain tables".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d3adaabaf7d555ec8bb1d83c43f72e79f1bf0b7d) (revert, Reverted, 2020-11-06) - judged not relevant
- P11: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant; reason summarised from its thread
- P12: [Dump-restore loosing 'attnotnull' bit for DEFERRABLE PRIMARY KEY column(s)](https://commitfest.postgresql.org/patch/4729/) (cf, Rejected, 2023-12-28) - used in the review; reason summarised from its thread
- P13: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P14: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P15: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P16: [Revert "Adjust contrib/sepgsql regression test expected outputs."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21e6bf4ddd045bc718c99ae551f25957ae91cb4d) (revert, Reverted, 2023-04-12) - judged not relevant
- P17: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P18: [Add psql command to list constraints](https://commitfest.postgresql.org/patch/3468/) (cf, Rejected, 2022-03-28) - judged not relevant
- P19: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant
- P20: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - used in the review
- P21: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P22: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P23: [Avoid possible null pointer dereference](https://commitfest.postgresql.org/patch/5506/) (cf, Withdrawn, 2025-02-06) - judged not relevant
- P24: [Allow to warn or error on ... = NULL expressions](https://commitfest.postgresql.org/patch/1728/) (cf, Returned with feedback, 2018-12-01) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 194 s; cost reported by the CLI: $2.69
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 92 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:38:07.954Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
