# Design pre-review: Dropping a composite attribute causes data integrity violations

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7169](https://commitfest.postgresql.org/patch/7169/) (Needs review); authors: Nikhil Sontakke (nsontakke)
- Thread: [Dropping a composite attribute causes data integrity violations](https://www.postgresql.org/message-id/flat/CA%2BUBoq0F2ua2fQEAU3-6w0oxOgFc9BgEN4gRcchiUUZ%2BWouVrQ%40mail.gmail.com), started 2026-08-17
- Messages read: 3 of 3
- Patch prose read: v2-0001-Refuse-to-drop-a-column-whose-row-type-is-stored-.patch, 0001-Refuse-to-drop-a-column-whose-row-rowtype-drop-guard.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection disputes the goal: message #1 shows reproducible corruption, and the fix reuses the check that retyping already uses. The surviving objections are fixable and none stops review. They concern the target branches and the migration path (standards-compat-1, demand-1), whether the stored-expression cases go in together (standards-compat-3), and an owner blocked by another role's table (security-1). The backpatch decision is one a committer has to make.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the placement of the call in ATExecDropColumn(), its behaviour under inheritance recursion and typed tables, and any documentation changes could not be checked. The interaction with the reviewer's pgsql-bugs patch is known only from message #3.

## The proposal, restated

**Problem.** A composite value stores only its type OID and typmod, so ALTER TYPE ... DROP ATTRIBUTE changes how existing stored values read. Distinct values can become equal under a unique index, after which REINDEX fails and a plain pg_dump of the database fails to restore. ALTER TABLE ... DROP COLUMN has the same problem for a table's row type.

**Proposed design.** ATExecDropColumn() calls the existing find_composite_type_dependencies(), which ALTER COLUMN TYPE already uses but only when a rewrite is queued. The patch makes the call after the droppability check and before recursion, so it runs at each inheritance level. The drop is refused when another relation stores the row type, and CASCADE does not override this.

**User-visible surface.**
- ALTER TYPE ... DROP ATTRIBUTE now fails when a column of another relation uses the type.
- ALTER TABLE ... DROP COLUMN now fails when another relation stores the table's row type.
- The refusal uses the existing "cannot alter type ... because column ... uses it" error, and CASCADE does not bypass it.

**What the author claims.**
- The drop corrupts committed data: a unique index ends up violated, REINDEX fails and a dump does not restore.
- The fix reuses an existing check and covers both commands and all inheritance levels.
- Regression, isolation and pg_upgrade suites pass.
- Three existing tests relied on the old behaviour and were changed: test_typex, the test_tbl3 pg_upgrade fixture, and test_decoding oldest_xmin.

**Thread state.** Manuel Reyes Bravo reviewed v2, confirmed the fix and its tests, and showed that constants of the type in views, CHECK constraints and defaults still allow the drop. He proposes combining it with his separate pgsql-bugs patch and asked Heikki about overlap with d78040a469b; the author has not replied.

## 2. Prior attempts

### Revert applying column aliases to the output of whole-row Vars. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac)
- **Thread:** [https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us)
- **Date:** decided 2022-03-17; started 2014-11-10
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** In commit bf7ca1587, I had the bright idea that we could make the result of a whole-row Var (that is, foo.*) track any column aliases that had been applied to the FROM entry the Var refers to. However, that's not terribly logically consistent, because now the output of the Var is no longer of the named composite type that the Var claims to emit. bf7ca1587 tried to handle that by changing the output tuple values to be labeled with a blessed RECORD type, but that's really pretty disastrous: we can wind up storing such tuples onto disk, whereupon they're not readable by other sessions. The only practical fix I can see is to give up on what bf7ca1587 tried to do, and say that the column names of tuples produced by a whole-row Var are always those of the underlying named composite type, query aliases or no. While this introduces some inconsistencies, it removes others, so it's not that awful in the abstract. What *is* kind of awful is to make such a behavioral change in a back-patched bug fix. But corrupt data is worse, so back-patched it will be. (A workaround available to anyone who's unhappy about this is to introduce an extra level of sub-SELECT, so that the whole-row Var is referring to the sub-SELECT's output and not to a named table type. Then the Var is of type RECORD to begin with and there's no issue.) Per report from Miles Delahunty. The faulty commit dates to 9.5, so back-patch to all supported branches.
- **Reason (quote):** "But corrupt data is worse, so back-patched it will be."
- **What is different this time:** P6 reverted a feature that labeled stored whole-row tuples with a blessed RECORD type. This patch refuses DDL that changes how stored composite values read. P6 shows a behaviour change was backpatched to stop corrupt data.


## 3. Objections a reviewer is likely to raise

### 1. Accepted DDL becomes an error with no documented migration path

fixable · level 1b, rubric item 6 · desirability, SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-1`

A team runs a schema migration that drops an attribute from a composite type used by a column, which works today. After upgrade, the migration fails with "cannot alter type ... because column ... uses it", and CASCADE does not help. The team finds out in the deploy step, and the docs give no supported way to finish the change.

The touched files are code and tests only, so the ALTER TYPE and ALTER TABLE reference pages do not mention the new refusal or the workaround. The proposal does not say whether the fix is backpatched, which matters because a minor release would break scripts that ran the day before. [P6] shows the project has backpatched a behaviour change to stop corrupt data, but that commit message stated the workaround and the scope.

**Precedents.**
- [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (Reverted, 2022-03-17); thread: [link](https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us)

Thread: [#1 (Nikhil Sontakke, 2026-08-17)](https://www.postgresql.org/message-id/CA%2BUBoq0F2ua2fQEAU3-6w0oxOgFc9BgEN4gRcchiUUZ%2BWouVrQ%40mail.gmail.com), [#2 (Nikhil Sontakke, 2026-08-19)](https://www.postgresql.org/message-id/CA%2BUBoq3D5SPuDGFQN5MVp09AYEV2SZz2twgyuGnuwigMrN7BHA%40mail.gmail.com)

**Question:** Which branches get this change, and what documented steps should a user follow to remove an attribute from a composite type that a table column uses?

Counter-check: held. The thread does not say which branches get the change or how a user removes an attribute from an in-use type. P6 is about whole-row Vars and supports only the general point that corruption fixes with behaviour changes were backpatched.

### 2. Narrow corruption case blocks a common schema change for all users

fixable · level 1b, rubric item 4 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A user has a column of type ct with no unique index or equality-dependent object, and runs alter type ct drop attribute b to retire a field. After this patch the command fails, even with CASCADE. The user finds out at deploy time and has no documented path that keeps the stored data.

Corruption needs a unique index, a constraint, or another equality-sensitive object on the type, and the only evidence of it is the constructed example in message #1. The patch refuses every drop on a stored type, so users without such objects lose an operation that works for them today. The changed test_typex and test_tbl3 fixture show the drop is used on stored types. The thread does not describe the migration those users would follow, which the brief lists as unclear.

Thread: [#1 (Nikhil Sontakke, 2026-08-17)](https://www.postgresql.org/message-id/CA%2BUBoq0F2ua2fQEAU3-6w0oxOgFc9BgEN4gRcchiUUZ%2BWouVrQ%40mail.gmail.com), [#3 (Manuel Reyes Bravo, 2026-09-21)](https://www.postgresql.org/message-id/CA%2BbCEdCbDjqwv1h%3D9P1ZG-y9baFe9_74t2wZobeR6GuK98K-AQ%40mail.gmail.com)

**Question:** Is there any field report of this corruption, and what supported migration do users who drop attributes of stored types follow once the drop is refused?

Counter-check: held. The refusal covers stored types with no unique index, where the drop works as intended today, and the thread does not weigh that. A reproducible corruption case needs no field report, and the migration-path part duplicates standards-compat-1.

### 3. Contract changes twice if expression cases follow separately

fixable · level 2, rubric item 13 · design, SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-3`

Release N refuses the drop for columns but still allows it when a view, CHECK constraint or default holds a constant of the type. A later release adds those refusals too, so scripts break a second time. Between the two releases, a CHECK constraint can still make a table reject every row.

Message #3 shows six stored-expression cases with the same effect that v2 leaves open, and proposes combining with a separate pgsql-bugs patch. The author has not replied, so the proposal does not yet say whether one user-visible change or two is intended. One combined change is easier to document and to explain in release notes.

Thread: [#3 (Manuel Reyes Bravo, 2026-09-21)](https://www.postgresql.org/message-id/CA%2BbCEdCbDjqwv1h%3D9P1ZG-y9baFe9_74t2wZobeR6GuK98K-AQ%40mail.gmail.com)

**Question:** Will this patch and the stored-expression patch go in together, so users see one change to when DROP ATTRIBUTE and DROP COLUMN are refused?

Counter-check: held. Message #3 shows the CHECK constraint, view and default cases still allow the drop under v2, and the author has not answered the proposal to combine the patches. The scope of the user-visible change is still open.

### 4. Another role can block DROP COLUMN on a table it does not own

fixable · level 2, rubric item 7 · security / privileges · `security-1`

Role bob, who can use alice's types, creates a table with a column of alice's composite type or of her table's row type. Alice then runs ALTER TABLE ... DROP COLUMN or ALTER TYPE ... DROP ATTRIBUTE on her own object and gets an error, even with CASCADE. She cannot drop or alter bob's table, so the change is blocked until a superuser or bob acts.

The brief and the v1 tests both say CASCADE does not override the refusal. DROP TYPE ... CASCADE does remove dependent columns in other roles' tables, so the owner can always undo a dependency some other role created. Under this patch a dependency that a non-owner creates becomes a hard block on routine DDL by the owner. The same block already exists for retyping, but dropping a column is far more common, and neither the proposal nor messages #1 and #3 discuss this privilege aspect.

Thread: [#1 (Nikhil Sontakke, 2026-08-17)](https://www.postgresql.org/message-id/CA%2BUBoq0F2ua2fQEAU3-6w0oxOgFc9BgEN4gRcchiUUZ%2BWouVrQ%40mail.gmail.com), [#3 (Manuel Reyes Bravo, 2026-09-21)](https://www.postgresql.org/message-id/CA%2BbCEdCbDjqwv1h%3D9P1ZG-y9baFe9_74t2wZobeR6GuK98K-AQ%40mail.gmail.com)

**Question:** How does an owner drop a column when a role they cannot control stores the row type, and why should CASCADE not drop or rewrite those dependent columns?

Counter-check: held. USAGE on types is granted to PUBLIC by default, so another role can create the dependent column, and CASCADE does not remove it. The retyping check already blocks the owner this way, which lowers the concern, but the thread does not discuss extending it to drops.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The patch concerns how stored composite values read after a drop, and it changes the test_tbl3 pg_upgrade fixture. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | ALTER TYPE ... DROP ATTRIBUTE and ALTER TABLE ... DROP COLUMN now fail in cases accepted today, and CASCADE does not override. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | The bug makes a plain pg_dump fail to restore; the patch prevents new cases but not existing ones. |
| psql UX conventions | no |  |
| security / privileges | **yes** | A role that creates a column of another role's type can block the owner's drop, even with CASCADE. |
| backpatchability | **yes** | The entry is tagged Bugfix, but the thread does not say which branches get a change that turns accepted DDL into an error. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Narrow corruption case blocks a common schema change for all users |

## 5. Questions the author must answer before anyone reviews code

1. Which branches should get this change, given that it turns DDL accepted today into an error?
2. What documented steps should a user follow to remove an attribute from a composite type that a table column stores?
3. Should this patch go in together with the stored-expression patch from message #3, so that users see one change in when the drop is refused?
4. How does an owner drop a column when another role's table stores the row type, and why should CASCADE not handle those dependent columns?
5. Do the ALTER TYPE and ALTER TABLE reference pages need text about the new refusal?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows ALTER TABLE in tablecmds.c and composite type dependencies. Message #3 copies Heikki because d78040a469b touched this call site, but he has not replied in the visible thread.
- **Liability:** The owner signs for a behaviour change in two common DDL commands, the backpatch decision, and reports from users whose migration scripts start to fail. The code added is one call to an existing function.
- **Author follow-up:** The author posted v2 two days after v1, but has not answered the review in message #3 of 2026-09-21.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch fixes reproducible corruption with one call to an existing check in tablecmds.c plus tests, and a reviewer has confirmed it. Only users who store composite types in columns and drop attributes are affected; the open points are scope and backpatch decisions.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Refusal also hits drops that cannot change any stored value. Refuted (wrong facts): Retyping an attribute of a type that another relation stores has no rewrite path either. Message #1 shows it is refused with the same error, regardless of table contents, so the patch copies existing practice.
- `replication-ondisk-1` Fixture change drops pg_upgrade coverage of stored values with dropped attributes. Refuted (no failure scenario): The scenario depends on an unnamed future change to pg_upgrade, and test coverage is outside levels 1-2. The changed fixture still creates a type with a dropped attribute and a table that uses it.
- `replication-ondisk-2` No guidance for clusters already holding values damaged by a drop. Refuted (no failure scenario): The damaged values and the later restore failure exist today without the patch, so the proposal neither causes nor worsens them. Detection advice is release-note text, and item 14 concerns state that the patch itself creates.

### Review angles

- prior-attempts: 0 objection(s). No precedent in the list proposed or rejected this guard, so no earlier rejection reason applies. P6 accepted a back-patched behaviour change to prevent corrupt composite data, which supports this patch on the open backpatch question. Message #3 points to a related pgsql-bugs patch by the reviewer that extends find_composite_type_dependencies() to stored expressions; the thread does not say that patch was rejected.
- standards-compat: 3 objection(s). The bug is real and the fix reuses an existing check, so these objections concern the user-facing contract only.
- replication-ondisk: 2 objection(s). The patch adds no WAL records, catalog columns, protocol messages or on-disk format changes. Both objections concern data that already exists in the field, not new state that this patch creates.
- security: 1 objection(s). The only privilege issue found is that a non-owner can block the owner's DDL. The error message names the other relation, but the existing ALTER COLUMN TYPE refusal already does this, so it is not raised.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The patch adds a call to the existing find_composite_type_dependencies() from ATExecDropColumn(). That matches how ALTER ... ALTER ATTRIBUTE TYPE already refuses, adds no new mechanism, GUC or API, and costs only a catalog scan during DROP COLUMN and DROP ATTRIBUTE. Nothing runs per tuple or per query, and it fixes data corruption that has to live in core. The patch also does not say whether it should go in alone or together with the reviewer's patch for constants inside expressions (message #3), nor whether it is to be backpatched. Those are scope and backpatch questions for the thread, and no failure scenario follows from them in this angle. No precedent applies: [P6] concerns whole-row Var labeling, and [P4] and [P9] concern enum ADD VALUE.
- already-exists: 0 objection(s). Postgres has no feature, setting or extension that stops this data corruption, and the patch extends the existing find_composite_type_dependencies() check without adding a second mechanism. The reviewer's pgsql-bugs patch in message #3 adds to the same function and covers a different case, so it does not duplicate this one.
- demand: 1 objection(s). Demand for fixing silent data corruption is not in doubt. The one objection asks for the cost to existing users to be weighed and a migration path to be stated.

### Precedents retrieved

- P1: [Revert badly-broken patch to dump comments on composite-type columns.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=18661f282bae5599abb68f53bc8b451b2d67bd75) (revert, Reverted, 2004-08-06) - judged not relevant
- P2: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant; reason summarised from its thread
- P3: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P4: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - used in the review
- P5: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P6: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - used in the review
- P7: [Bug: Reading from single byte character column type may cause out of bounds memory reads](https://commitfest.postgresql.org/patch/3767/) (cf, Returned with feedback, 2022-09-08) - judged not relevant; reason summarised from its thread
- P8: [Add IMMUTABLE column attribute](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Data_Types) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [Revert to 9.6 treatment of ALTER TYPE enumtype ADD VALUE.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=28e07270768518524291d7d7906668eb67f6b8a5) (revert, Reverted, 2017-09-27) - used in the review
- P10: [Revert "Stabilize output of new regression test case".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cbadba8dd632fc0d4162f7d686fec631bce7dfd0) (revert, Reverted, 2018-02-14) - judged not relevant
- P11: [Revert addition of third argument to format_type().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7564654adf07ec26b83c7effc7f54f7183e04519) (revert, Reverted, 2011-03-10) - judged not relevant
- P12: [Revert: Allow table AM to store complex data structures in rd_amcache](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=922c4c461d213a422ee7eb6c38e399607539210a) (revert, Reverted, 2024-04-11) - judged not relevant
- P13: [Role Attribute Bitmask Representation](https://commitfest.postgresql.org/patch/85/) (cf, Rejected, 2015-01-12) - judged not relevant; reason summarised from its thread
- P14: [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://commitfest.postgresql.org/patch/5992/) (cf, Returned with feedback, 2026-04-02) - judged not relevant; reason summarised from its thread
- P15: [format_datum debugging function](https://commitfest.postgresql.org/patch/5165/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P16: [Optimize numeric comparison and aggregation via packed-datum extraction](https://commitfest.postgresql.org/patch/6651/) (cf, Rejected, 2026-05-10) - judged not relevant; reason summarised from its thread
- P17: [Stores short varlena in an array](https://commitfest.postgresql.org/patch/5461/) (cf, Rejected, 2025-01-25) - judged not relevant
- P18: [Revert per-index collation version tracking feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721) (revert, Reverted, 2021-05-07) - judged not relevant
- P19: [Store FullTransactionId in TwoPhaseFileHeader/GlobalTransactionData](https://commitfest.postgresql.org/patch/2226/) (cf, Withdrawn, 2019-09-04) - judged not relevant
- P20: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - judged not relevant
- P21: [Revert "TupleHashTable: store additional data along with tuple."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4a07f532b40a64fb4714a3f7ab6063435411edb) (revert, Reverted, 2025-01-13) - judged not relevant
- P22: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant
- P23: [Enforce typmod for function inputs, function results and parameters for spi_prepare'd statements called from PLs](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Functions) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P24: [Revert "Convert *GetDatum() and DatumGet*() macros to inline functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e8d78581bb864369035c736a82ecfb57482a1a09) (revert, Reverted, 2022-09-12) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 127 s; cost reported by the CLI: $1.38
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 81 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:24:51.272Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
