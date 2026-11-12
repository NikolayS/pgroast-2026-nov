# Design pre-review: ALTER COLUMN SET EXPRESSION on partitions not work in case of dependencies

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7117](https://commitfest.postgresql.org/patch/7117/) (Needs review); authors: Jian He (stayreal)
- Thread: [ALTER COLUMN SET EXPRESSION on partitions not work in case of constraint dependencies](https://www.postgresql.org/message-id/flat/CACJufxEomSz3BwUGfk8A6MeAKj%3DCki5B%2BgYTbJO%3DACCJyytWZg%40mail.gmail.com), started 2026-08-01
- Messages read: 6 of 6
- Patch prose read: v4-0001-Fix-dependency-issue-when-directly-ALTER-TABLE-SET-EXPRESSION-o.nocfbot, v4-0001-Disallow-directly-ALTER-TABLE-SET-EXPRESSION-on-child-table-if-de.patch, v1-0001-Disallow-ALTER-TABLE-SET-EXPRESSION-if-dependent-cannot-rebuilt.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is real and no surviving objection questions the goal. The remaining points are fixable design points at level 2: standards-compat-1 (the back-branch error also rejects an ONLY case that works today), maintenance-fit-2 (set expression gets its own path for dependent constraints) and replication-ondisk-1 (no guidance for partitions already affected). A committer who knows the ALTER TABLE rebuild code has to decide the scope of the back-branch error.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code I could not check which dependency cases the new errors reject, or whether phase 3 re-validation covers every constraint kind for partitions, plain inheritance children and virtual generated columns. Only 3 of 6 messages were visible.

## The proposal, restated

**Problem.** Running alter table ... alter column ... set expression directly on a partition silently drops inherited CHECK and NOT NULL constraints on the generated column without recreating them. With alter table only on a partitioned parent, an index on the column plus a foreign key leads to a "could not read blocks" error. The bug affects users of generated columns on partitioned or inherited tables since PG17.

**Proposed design.** There are two alternative patches. The bugfix patch, meant for back branches, errors out when set expression runs directly on a child table or with only and the column has dependent indexes or constraints. The second patch, a proof of concept, lets set expression on a partition succeed when the column has only NOT NULL, CHECK or FOREIGN KEY dependents, and re-validates them in phase 3 without rebuilding them.

**User-visible surface.**
- New error when alter table ... alter column ... set expression runs on a child table whose generated column has dependent indexes or constraints.
- New error when alter table only ... alter column ... set expression runs and the generated column has dependent objects.
- With the second patch, set expression on a partition re-validates CHECK, NOT NULL and FOREIGN KEY constraints without dropping and recreating them.

**What the author claims.**
- The bug has existed since PG17.
- Changing a generation expression changes only column data, not its type, so re-validating constraints is enough.
- Re-validation avoids an ACCESS EXCLUSIVE lock on the whole partition hierarchy when only one partition changes.
- Rebuilding dependents under alter table only contradicts the meaning of ONLY.
- The fix is too complex for back branches, so they should get the error.

**Thread state.** Alberto Piai is the only reviewer and withdrew his idea of also rebuilding the parent constraint. He prefers the re-validation approach for constraints and says that set expression with ONLY failing depending on dependencies would surprise users; v4 answers his points, and the CommitFest entry needs a rebase.

## 2. Prior attempts

### Fix ALTER TABLE DROP EXPRESSION with ONLY option (related)

- **Link:** [https://commitfest.postgresql.org/patch/5992/](https://commitfest.postgresql.org/patch/5992/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CACJufxEW9L279n=fpUg+qrXYP6g4oh_RLOTHVVphkWDVqaNGDw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CACJufxEW9L279n=fpUg+qrXYP6g4oh_RLOTHVVphkWDVqaNGDw@mail.gmail.com)
- **Date:** decided 2026-04-02; started 2025-08-24
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. Peter Eisentraut asked why tablecmds.c passes recurse = false when recursing, and suggested a more consistent use of the two flags would make the existing code work. Tom Lane said that if the flags matter here, DROP EXPRESSION should not use ATSimpleRecursion. The author replied in December 2025 that v3 is correct, and no later reply appears before the entry was returned with feedback.
- **Reason (quote):** quote unavailable
- **What is different this time:** P1 changed DROP EXPRESSION recursion checks for multi-level inheritance; this proposal fixes dependent-object handling for set expression on partitions and with ONLY.


## 3. Objections a reviewer is likely to raise

### 1. Back-branch fix makes working ALTER TABLE ONLY ... SET EXPRESSION fail

fixable · level 2, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A migration script on 17.x runs alter table only t alter column b set expression on a partitioned parent that has an index on b and no foreign key. Today it succeeds. After a minor upgrade with the disallow patch it fails, and the user finds out when the deployment stops partway.

Message #2 says the ONLY case works when only an index depends on the column, yet v4 of the disallow patch rejects ONLY whenever any dependency exists (message #6). That turns a working command into an error in a stable release to fix a different, narrower failure. A committer would likely ask that a back-branch fix block only the cases that lose constraints or corrupt the rebuild, and leave working ones alone.

**Precedents.**
- [ALTER TABLE - check for column existence before starting operations](https://commitfest.postgresql.org/patch/1621/) (Returned with feedback, 2018-10-02); thread: [link](https://www.postgresql.org/message-id/flat/1897070.0DItnajBuJ@pierred-pdoc)

Thread: [#2 (jian he, 2026-08-07)](https://www.postgresql.org/message-id/CACJufxHp_6SepFvzsYd7UDsDmVTvrohApdQ1oxgfNEmQrNArdw%40mail.gmail.com), [#6 (jian he, 2026-08-26)](https://www.postgresql.org/message-id/CACJufxERxHfdx0QXJDxVvHnw%2BwfE%2BEVhFgBHK_ON490-9MVQWA%40mail.gmail.com)

**Question:** Which ALTER TABLE ONLY ... SET EXPRESSION cases that succeed today would the back-branch patch reject, and why must those be rejected rather than left working?

Counter-check: held. Message #2 says the ONLY case works once the index is removed, yet the v4 disallow patch rejects ONLY with any dependency. The thread gives no stable-branch reason beyond the meaning of ONLY. P6 must not be used as support.

### 2. SET EXPRESSION gets its own dependent-handling path apart from ALTER TYPE

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A later patch changes how ATPostAlterTypeCleanup rebuilds constraints or adds a dependent kind. It updates the rebuild path and misses the new SET EXPRESSION re-validation path. No test fails, and a partition is left with a constraint that was never checked against the new data.

Message #6 says v4 moves changedConstraintOids into tab->constraints for SET EXPRESSION only and removes that logic from ATPostAlterTypeCleanup. Dependents are also collected in phase 1 and then reset to NIL. AT_PASS_ALTER_TYPE and AT_PASS_SET_EXPRESSION used to share one cleanup step, and v4 splits it. Message #6 also found v3 incorrect because of pass ordering, which shows how easy it is to get this split wrong.

Thread: [#4 (jian he, 2026-08-10)](https://www.postgresql.org/message-id/CACJufxE4PXVEtJ5t_W3FWz-Q-YPbVgne5chmu0BUoX5Sas8B%3Dw%40mail.gmail.com), [#6 (jian he, 2026-08-26)](https://www.postgresql.org/message-id/CACJufxERxHfdx0QXJDxVvHnw%2BwfE%2BEVhFgBHK_ON490-9MVQWA%40mail.gmail.com)

**Question:** Can the re-validation rule live inside RememberConstraintForRebuilding and ATPostAlterTypeCleanup, as message #5 suggests, so one place decides rebuild or re-check for both commands?

Counter-check: held. Message #6 confirms v4 handles constraints for set expression apart from the shared ATPostAlterTypeCleanup logic. Message #5 asked for the rule to live in the Remember*ForRebuilding functions, and v4 does not say why it did not.

### 3. No guidance for clusters that already lost inherited constraints

fixable · level 2, rubric item 14 · catalog changes, pg_dump / restore round-trip, on-disk format / pg_upgrade, backpatchability · `replication-ondisk-1`

A PG17 user ran set expression on a partition, lost the inherited CHECK, then inserted rows that violate it. After the minor update the partition still has no pg_constraint row and no error appears. A later pg_dump restore re-inherits cc through partition of, and copy fails on the violating rows.

Message #1 shows the bug deletes pg_constraint rows for inherited CHECK and NOT NULL constraints. Both v4 patches only prevent new damage. Neither commit message says how to find affected partitions or repair them. For a back-branch fix that leaves bad catalog state behind, a committer would likely ask for a detection query and repair steps in the release notes before pushing.

Thread: [#1 (jian he, 2026-08-01)](https://www.postgresql.org/message-id/CACJufxEomSz3BwUGfk8A6MeAKj%3DCki5B%2BgYTbJO%3DACCJyytWZg%40mail.gmail.com), [#4 (jian he, 2026-08-10)](https://www.postgresql.org/message-id/CACJufxE4PXVEtJ5t_W3FWz-Q-YPbVgne5chmu0BUoX5Sas8B%3Dw%40mail.gmail.com), [#6 (jian he, 2026-08-26)](https://www.postgresql.org/message-id/CACJufxERxHfdx0QXJDxVvHnw%2BwfE%2BEVhFgBHK_ON490-9MVQWA%40mail.gmail.com)

**Question:** What query finds partitions whose inherited CHECK or NOT NULL constraint was dropped, and how should users repair them?

Counter-check: held. Message #1 shows the bug removes pg_constraint rows, and neither patch finds or repairs partitions that are already affected. The patch does not cause this damage, so this is a release-note request and does not block the design.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-1: No guidance for clusters that already lost inherited constraints |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The bug deletes inherited pg_constraint rows on partitions; the patches change when those rows are dropped or kept. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The bugfix patch changes what alter table only ... set expression accepts, based on the meaning of ONLY. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | A partition that already lost an inherited CHECK gets it back on restore, which can fail on rows that violate it. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bugfix patch targets stable branches and adds errors for commands that are accepted today. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The second patch gives set expression a constraint re-validation path in tablecmds.c apart from ATPostAlterTypeCleanup. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which alter table only ... set expression cases that succeed today does the back-branch patch reject, and why must each be rejected?
2. What is the root cause of the "could not read blocks" error in message #2, and does the new error cover every case that reaches it?
3. Can the re-validation rule live in RememberConstraintForRebuilding and ATPostAlterTypeCleanup, so that one place decides between rebuild and re-check?
4. What query finds partitions whose inherited CHECK or NOT NULL constraint was already dropped, and how should users repair them?
5. How does the second patch handle plain inheritance children, as opposed to partitions?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the ALTER TABLE phase 2 and phase 3 code in tablecmds.c, in particular ATPostAlterTypeCleanup and constraint inheritance on partitions. No committer has replied in the visible thread.
- **Liability:** A behaviour change in stable branches from 17 on: any script that the new error breaks is reported against the minor release. On master, the owner also signs for a second constraint-handling path that later ALTER TABLE patches must keep correct.
- **Author follow-up:** The author posted four patch versions, answered the reviewer's points in v4 and moved the entry to the next CommitFest. The entry needs a rebase since 2026-08-27.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Silent loss of inherited constraints is a correctness bug worth fixing, though only users who run set expression on a partition or with ONLY hit it. Both patches touch only tablecmds.c and tests, but the back-branch scope needs a decision from a committer, and none is engaged.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Back-branch error breaks ONLY set expression that works today. Refuted (not specific): Duplicate of standards-compat-1. P6 concerned a new check on master and should not be used as support for a back-branch concern.
- `standards-compat-2` Success of SET EXPRESSION depends on which dependent object types exist. Refuted (no failure scenario): The v4 commit message says set expression on a partition whose index belongs to an index hierarchy already fails today, so the index case is not a new failure. The rest is documentation, which is out of scope, and the branch question in maintenance-fit-3.
- `maintenance-fit-1` Back-branch fix turns a working ALTER TABLE ONLY case into an error. Refuted (not specific): Duplicate of standards-compat-1. The added point that ALTER TYPE may share the bug has no support in the thread.
- `maintenance-fit-3` Master and back branches would differ in SET EXPRESSION semantics. Refuted (already handled): Message #4 says back branches get the error, and the re-validation patch is a proof of concept for later work. An error on stable branches with a fuller fix on master is existing practice, and a missing committer is a committability point.

### Review angles

- prior-attempts: 1 objection(s). No precedent proposes the same fix. P19 is close in area but has no recorded reason, so it is not used.
- standards-compat: 2 objection(s). The core bug, silent loss of inherited constraints, deserves a fix; the objections cover only the user-facing contract of the proposed errors.
- replication-ondisk: 1 objection(s). Beyond the catalog cleanup point, neither patch changes on-disk format, WAL records, replication, wire protocol or extension API. The re-validation patch keeps constraint rows and their stored expressions, which stays valid because the column type does not change.
- security: 0 objection(s). Both patches change how set expression handles constraints and indexes on a generated column; neither adds a privilege, role, view or protocol path. Ownership checks on the altered table stay as they are, and the bugfix narrows what a partition owner can do by stopping the silent drop of inherited CHECK and NOT NULL constraints. FK re-validation in the second patch uses the existing phase 3 RI path, so I see no new escalation, leak or unbounded resource use.
- maintenance-fit: 3 objection(s). The bug is real, the fix has no hot-path cost, and it adds no new dependency. These objections are about the scope of the back-branch error and the long-run shape of the code.
- already-exists: 0 objection(s). The proposal fixes silent loss of inherited CHECK and NOT NULL constraints, plus a read error, inside core ALTER TABLE. No extension, setting or documented idiom gives correct behaviour here. The proof-of-concept patch reuses the existing phase 3 constraint validation and adds no second mechanism. Running SET EXPRESSION on the parent is not equivalent, because it changes every partition.
- demand: 0 objection(s). This is a bug fix: since PG17, set expression on a partition drops inherited CHECK and NOT NULL constraints without telling the user, and message #2 shows an internal read error. Silent loss of a constraint needs no demand evidence. Any user of generated columns on partitioned tables can hit it, and the costs fall only on that command.

### Precedents retrieved

- P1: [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://commitfest.postgresql.org/patch/5992/) (cf, Returned with feedback, 2026-04-02) - used in the review; reason summarised from its thread
- P2: [Revert support for ALTER TABLE ... MERGE/SPLIT PARTITION(S) commands](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3890d90c1508125729ed20038d90513694fc3a7b) (revert, Reverted, 2024-08-24) - judged not relevant
- P3: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - judged not relevant; reason summarised from its thread
- P4: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant; reason summarised from its thread
- P5: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P6: [ALTER TABLE - check for column existence before starting operations](https://commitfest.postgresql.org/patch/1621/) (cf, Returned with feedback, 2018-10-02) - used in the review; reason summarised from its thread
- P7: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant; reason summarised from its thread
- P8: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant; reason summarised from its thread
- P9: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P10: [Allow AddForeignUpdateTargets to add extra target expressions other than columns](https://commitfest.postgresql.org/patch/1819/) (cf, Returned with feedback, 2019-11-27) - judged not relevant
- P11: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P12: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P13: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P14: [ALTER TABLE SET LOGGED/UNLOGGED on a partitioned table does nothing silently](https://commitfest.postgresql.org/patch/2857/) (cf, Withdrawn, 2021-04-07) - judged not relevant
- P15: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant
- P16: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant
- P17: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - judged not relevant
- P18: [Tab completion for ALTER COLUMN SET STATISTICS](https://commitfest.postgresql.org/patch/377/) (cf, Rejected, 2015-12-21) - judged not relevant
- P19: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - used in the review
- P20: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P21: [refactor AlterDomainAddConstraint (alter domain add constraint)](https://commitfest.postgresql.org/patch/5455/) (cf, Returned with feedback, 2026-04-02) - judged not relevant
- P22: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P23: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant
- P24: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 123 s; cost reported by the CLI: $1.47
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 83 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:20:45.683Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
