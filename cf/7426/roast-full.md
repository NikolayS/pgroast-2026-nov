# Design pre-review: [PG19]Three bugs with a CHECK constraint that only the child enforces

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7426](https://commitfest.postgresql.org/patch/7426/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [\[PG19\] Three bugs with a CHECK constraint that only the child enforces](https://www.postgresql.org/message-id/flat/CAGRkXqSGE%3DfQrjCoNQsyiCZRKfQ4SrB015CAR%3D3pXYB5tB%3Dieg%40mail.gmail.com), started 2026-10-09
- Messages read: 16 of 16
- Patch prose read: v5-0001-Reject-enforcing-a-parent-CHECK-over-a-NOT-VALID-.patch, v5-0002-pg_dump-keep-ENFORCED-on-a-child-s-inherited-CHEC.patch, v5-0003-pg_dump-keep-ENFORCED-on-a-child-s-CHECK-that-no-.patch, v1-0003-pg_dump-keep-ENFORCED-on-a-child-s-inherited-CHEC.patch, v1-0001-Validate-NOT-VALID-children-when-a-parent-CHECK-b.patch, v1-0002-Keep-a-child-s-ENFORCED-CHECK-when-the-parent-s-c.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

v5-0001 is committed and no objection to it survived. For v5-0002 and v5-0003, standards-compat-1 and standards-compat-2 survived: the dump makes the child's constraint local, and ALTER CONSTRAINT ... ENFORCED would set conislocal for plain inheritance only. Message #14 leaves the plain inheritance case open and nobody answered. The goal is a valid bug fix, but the list has not agreed on this design.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Whether v5-0002 and v5-0003 change --binary-upgrade output, and what exactly v5-0002 changes in tablecmds.c, is known only from thread messages.

## The proposal, restated

**Problem.** In 19, a CHECK constraint can be ENFORCED on an inheritance child or partition while the parent keeps it NOT ENFORCED. Three paths lose that state: ALTER CONSTRAINT ... ENFORCED on the parent marks it validated over a NOT VALID child, ALTER COLUMN TYPE recreates the child's constraint from the parent, and pg_dump restores the child as NOT ENFORCED. The first can return wrong query results through constraint exclusion.

**Proposed design.** v5-0001 makes ALTER CONSTRAINT ... ENFORCED on a parent fail when a child holds the constraint as ENFORCED NOT VALID, with the error ALTER TABLE ... INHERIT already raises. v5-0002 and v5-0003 make pg_dump emit the child's ENFORCED CHECK as a local constraint when no parent enforces it. The ALTER COLUMN TYPE fix moved to a separate WIP patch from jian he.

**User-visible surface.**
- ALTER TABLE ... ALTER CONSTRAINT ... ENFORCED on a parent now errors when a child's constraint is ENFORCED NOT VALID, with an errhint.
- pg_dump output now creates a child's ENFORCED CHECK inline in CREATE TABLE when its parents keep it NOT ENFORCED.
- After restore, such a child constraint has conislocal = true, so dropping the parent's constraint no longer drops it.

**What the author claims.**
- Bug 1 is new in 19, and 0001 belongs on the 19 open items list.
- Bugs 2 and 3 also exist in 18; jian he says bug 2 goes back to at least 14.
- pg_upgrade keeps the child ENFORCED today; only pg_dump loses it.
- jian he says the local-constraint dump avoids an extra full scan under ACCESS EXCLUSIVE lock at restore.
- v5-0003 skips NOT VALID constraints, because restoring one would mark it valid without a scan.

**Thread state.** Andrew Dunstan committed v5-0001 on 2026-11-01 to close the 19 open item; v5-0002 and v5-0003 are left for later discussion. The open question is whether pg_dump may turn an inherited constraint into a local one, which jian he accepts for partitions but doubts for plain inheritance.

## 2. Prior attempts

### Revert "Improve compression and storage support with inheritance" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5)
- **Thread:** [https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org)
- **Date:** decided 2024-02-20; started 2024-02-16
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 0413a556990ba628a3de8a0b58be020fd9a14ed0. pg_dump cannot currently dump all the structures that are allowed by this patch. This needs more work in pg_dump and more test coverage.
- **Reason (quote):** "pg_dump cannot currently dump all the structures that are allowed by this patch."
- **What is different this time:** P12 reverted a new feature because pg_dump could not dump the inheritance states it allowed. Here the child-enforced state can already be created in 18, so the feature stays and pg_dump is changed.

### table inheritance versus column compression and storage settings (related)

- **Link:** [https://commitfest.postgresql.org/patch/4813/](https://commitfest.postgresql.org/patch/4813/)
- **Thread:** [https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org)
- **Date:** decided 2024-03-21; started 2023-12-04
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The committed patch was reverted after Tom Lane showed that pg_dump and pg_restore of the regression database fail: stchild4 does not restore and stchild5 gets the wrong attstorage. Ashutosh Bapat then found related existing dump bugs and proposed setting defaults when parents conflict. Peter Eisentraut declined to take on that change late in the cycle, citing unexpected pitfalls, missing test coverage and no patch.
- **Reason (quote):** "We have already discovered with the previous attempt that there are unexpected pitfalls and lacking test coverage. Also, there isn't even a patch yet." ([message](https://www.postgresql.org/message-id/0f3d5151-009d-452b-8652-ed6f1031f8c8%40eisentraut.org))
- **What is different this time:** P7 changed how inherited per-table settings merge and was returned after new dump failures appeared. This proposal is a bug fix with tests, though the thread also keeps finding related dump bugs.


## 3. Objections a reviewer is likely to raise

### 1. Dump and pg_upgrade turn an inherited child CHECK into a local one

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, on-disk format / pg_upgrade, catalog changes, backpatchability · `standards-compat-1`

A user dumps and restores, or runs pg_upgrade on, a database where child k enforces parent p's NOT ENFORCED check c. Later the user runs alter table p drop constraint c. Before the upgrade this dropped k's copy; after it, k keeps c as a local constraint and rejects rows, and the user finds out only when inserts into k fail.

v5-0002 and v5-0003 emit the child's constraint inline, so it comes back with conislocal = true (messages #7, #14). Message #7 notes that pg_upgrade keeps the constraint inherited today and would stop doing so, which makes a dump fix change catalog state for upgrades that currently work. jian he doubts this is right for plain inheritance (message #14), and nobody else has agreed to it. The v1-0003 design (alter constraint ... enforced after the parent) keeps conislocal intact, at the cost of a scan at restore time.

**Precedents.**
- [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (Reverted, 2024-02-20); thread: [link](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org)

Thread: [#7 (shihao zhong, 2026-10-15)](https://www.postgresql.org/message-id/CAGRkXqQQapfoGHpi2znkMfzkCk-%3D5GvmC16KKbVa6-BZn0yF6g%40mail.gmail.com), [#10 (jian he, 2026-10-19)](https://www.postgresql.org/message-id/CACJufxGkQSboSOj4PVA1S-2AA%2B17J%2BOM5YY2uxvBL%3DaDe9ES2Q%40mail.gmail.com), [#12 (jian he, 2026-10-23)](https://www.postgresql.org/message-id/CACJufxGjsWkmyhP8QTMTT6doWwuCjG9q%2Bfw2nkFiiAfLTxNMMg%40mail.gmail.com), [#14 (jian he, 2026-10-29)](https://www.postgresql.org/message-id/CACJufxE7ZYKrBnOP43dCED2mOQ2zDfpR8M-CBb_p_0e1gGqxYA%40mail.gmail.com)

**Question:** Why is a lossy conislocal change at restore and pg_upgrade better than the v1-0003 alter constraint form, at least under --binary-upgrade where no scan is needed?

Counter-check: held. Message #14 leaves the conislocal change for plain inheritance open, and nobody answered before 0002 and 0003 were deferred. The notnull_islocal case in message #12 is existing practice but did not settle it; [P12] must not be used as support.

### 2. ALTER CONSTRAINT ENFORCED on a child would detach it from the parent

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, catalog changes, backpatchability · `standards-compat-2`

On a plain inheritance child, a user runs alter table k3 alter constraint c enforced and then alter table p3 drop constraint c. Under v5-0002 the child keeps c as a local constraint, but a partition in the same situation loses it. If this is back-patched to 18, a minor release changes what existing scripts do.

v5-0002 touches tablecmds.c and, following message #10, makes alter constraint ... enforced set conislocal for regular inheritance but not for partitions. That gives one command two meanings depending on the inheritance kind, and makes a property-change command alter inheritance linkage, which is a different job. The precedent cited in message #10 (SET NOT NULL changing conislocal) is a single case, and message #12 calls the current state an inconsistency. The thread does not say whether this change goes to 18.

Thread: [#10 (jian he, 2026-10-19)](https://www.postgresql.org/message-id/CACJufxGkQSboSOj4PVA1S-2AA%2B17J%2BOM5YY2uxvBL%3DaDe9ES2Q%40mail.gmail.com), [#12 (jian he, 2026-10-23)](https://www.postgresql.org/message-id/CACJufxGjsWkmyhP8QTMTT6doWwuCjG9q%2Bfw2nkFiiAfLTxNMMg%40mail.gmail.com), [#13 (shihao zhong, 2026-10-24)](https://www.postgresql.org/message-id/CAGRkXqSnu-sUZQK9CB8K-cLbRQ45wLpC79wC2NuiRfL_0su-xQ%40mail.gmail.com), [#14 (jian he, 2026-10-29)](https://www.postgresql.org/message-id/CACJufxE7ZYKrBnOP43dCED2mOQ2zDfpR8M-CBb_p_0e1gGqxYA%40mail.gmail.com)

**Question:** Should alter constraint ... enforced change conislocal at all, why only for plain inheritance, and is this behaviour change intended for a back-patch to 18?

Counter-check: held. v5-0002 touches tablecmds.c, and message #10 says it makes ALTER CONSTRAINT ... ENFORCED set conislocal for plain inheritance only; no one agreed to that. The back-patch part does not apply, because 18 has no ALTER CONSTRAINT for CHECK (message #1).

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Message #7 says the local-constraint dump would also make the child's constraint local under pg_upgrade, which keeps it inherited today. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | After restore the child's pg_constraint row has conislocal = true; v5-0002 also changes tablecmds.c. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | ALTER CONSTRAINT ... ENFORCED on a parent now errors over an ENFORCED NOT VALID child; v5-0002 changes its effect on a child. |
| new GUC | no |  |
| planner behaviour change | **yes** | v5-0001 stops a parent being marked validated over a NOT VALID child, which constraint exclusion relied on. |
| pg_dump / restore round-trip | **yes** | v5-0002 and v5-0003 change how pg_dump emits a child's ENFORCED CHECK whose parents keep it NOT ENFORCED. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Bugs 2 and 3 also exist in 18, and the thread does not say whether or how the pg_dump fixes go there. |
| maintenance burden ("who maintains this in 10 years") | **yes** | pg_dump gains another case that rewrites inherited constraint state, next to the not-null case from message #12. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do v5-0002 or v5-0003 change conislocal in --binary-upgrade output, where pg_upgrade keeps the constraint inherited today?
2. For plain inheritance, why is restoring the child's constraint as local better than the v1-0003 form, which emits ALTER CONSTRAINT ... ENFORCED after the parent?
3. Should ALTER CONSTRAINT ... ENFORCED on a child set conislocal at all, and why for plain inheritance but not for partitions?
4. How will 18 get a fix for the pg_dump loss, given that v5-0002 and v5-0003 use ALTER CONSTRAINT syntax that 18 does not have?
5. Which thread will carry the ALTER COLUMN TYPE fix and the related dump losses from messages #8, #11 and #12?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andrew Dunstan committed v5-0001. The pg_dump patches need a committer who knows the inheritance handling in pg_dump and the conislocal rules in tablecmds.c; none has yet spoken on them in the visible thread.
- **Liability:** The owner signs for how dump, restore and possibly pg_upgrade set conislocal on inherited CHECK constraints, in 19 and in any back branch. Restored clusters keep that catalog state for good.
- **Author follow-up:** The author posted revisions up to v5 within about a month and reworked the design after review, so follow-up looks likely.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The bug affects only users whose child enforces a CHECK that the parent keeps NOT ENFORCED, but for them a restore silently loses enforcement. The two remaining patches are small; the conislocal question for plain inheritance needs an answer before code review.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Dump turns inherited child CHECK into local, diverging from pg_upgrade. Refuted (not specific): Duplicate of standards-compat-1. [P12] reverted a new feature, but the child-enforced state can already be created in 18 (message #1), so limiting ALTER CONSTRAINT would not remove it.
- `replication-ondisk-1` pg_upgrade would silently turn inherited child constraints into local ones. Refuted (not specific): Duplicate of standards-compat-1, which covers the pg_upgrade path and asks the same --binary-upgrade question.
- `replication-ondisk-2` v5-0002 changes the catalog effect of a DDL command, which conflicts with a back-patch. Refuted (wrong facts): The scenario needs ALTER CONSTRAINT ... ENFORCED on a CHECK in 18, but message #1 says that command is new in 19 and that the patches using it do not apply to 18. The remaining point about server behaviour duplicates standards-compat-2.
- `maintenance-fit-1` v5-0002 changes backend conislocal semantics for plain inheritance only. Refuted (not specific): Duplicate of standards-compat-2, which makes the same point about ALTER CONSTRAINT ... ENFORCED changing conislocal for plain inheritance only.
- `maintenance-fit-2` Per-property special cases in pg_dump with no general rule. Refuted (no failure scenario): The scenario depends on an unnamed future constraint property, and the other dump losses in messages #8 and #12 predate this patch. v5-0003 states its rule, follows the flagInhAttrs pattern, and moves pg_dump toward what [P12] asked for.

### Review angles

- prior-attempts: 1 objection(s). v5-0001 is committed (message #16) and was not challenged. The one objection covers the open pg_dump conislocal question in v5-0002 and v5-0003.
- standards-compat: 2 objection(s). v5-0001 is committed and its error matches the existing ALTER TABLE ... INHERIT error, so I raise no objection to it. The ALTER COLUMN TYPE fix has moved to another thread and is not evaluated here.
- replication-ondisk: 2 objection(s). v5-0001 is committed and adds only an error path, with no on-disk, WAL or catalog-format change. The objections cover the open v5-0002 and v5-0003.
- security: 0 objection(s). The proposal fixes how CHECK constraint state (enforced, validated, conislocal) is kept across ALTER CONSTRAINT, ALTER COLUMN TYPE and pg_dump. It adds no new privilege, role, function or setting, and it does not change who can run these commands, which already need table ownership. The wrong-result bug in 0001 is a correctness issue for the table owner's own data. It does not let one role see or change rows it could not reach before, so I see no security or privilege angle to object on.
- maintenance-fit: 2 objection(s). v5-0001 is committed and draws no objection from this angle; the objections concern only v5-0002 and v5-0003.
- already-exists: 0 objection(s). These are bug fixes for state that dump and restore lose; no existing feature, option or extension keeps a child-only ENFORCED CHECK through pg_dump. The local-constraint dump in v5-0003 follows the not-null precedent in flagInhAttrs (message #12), so it extends existing practice and adds no second mechanism.
- demand: 0 objection(s). These are correctness fixes, so the demand test does not apply. Bug 1 gives wrong query results through constraint exclusion. Bug 3 makes a dump and restore silently drop ENFORCED on a child's CHECK, and that hits every user of the 19 per-child enforceability feature. The bugs came from tool-assisted analysis, not field reports, but a fix for wrong results needs no demand evidence. Whether pg_dump may turn conislocal from false to true for plain inheritance is a design question for another angle.

### Precedents retrieved

- P1: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - used in the review
- P2: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P3: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P4: [pg_dump: fix NOT NULL constraint name comparison using makeObjectName](https://commitfest.postgresql.org/patch/6601/) (cf, Withdrawn, 2026-10-04) - judged not relevant; reason summarised from its thread
- P5: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant; reason summarised from its thread
- P6: [refactor AlterDomainAddConstraint (alter domain add constraint)](https://commitfest.postgresql.org/patch/5455/) (cf, Returned with feedback, 2026-04-02) - judged not relevant; reason summarised from its thread
- P7: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - used in the review; reason summarised from its thread
- P8: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P9: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant; reason summarised from its thread
- P10: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P11: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant
- P12: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - used in the review
- P13: [Add support for ON UPDATE/DELETE actions on ALTER CONSTRAINT](https://commitfest.postgresql.org/patch/1533/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P14: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P15: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant
- P16: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P17: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P18: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P19: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant
- P20: [Improving test coverage of extensions with pg_dump](https://commitfest.postgresql.org/patch/187/) (cf, Returned with feedback, 2015-12-02) - judged not relevant
- P21: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P22: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P23: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P24: [Remove deprecated exclusive backup mode](https://commitfest.postgresql.org/patch/1913/) (cf, Withdrawn, 2021-03-19) - judged not relevant

### Warnings

- review angle "standards-compat" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 155 s; cost reported by the CLI: $2.04
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 84 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:54:44.975Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
