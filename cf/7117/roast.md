# Design pre-review: ALTER COLUMN SET EXPRESSION on partitions not work in case of dependencies

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7117](https://commitfest.postgresql.org/patch/7117/); [thread](https://www.postgresql.org/message-id/flat/CACJufxEomSz3BwUGfk8A6MeAKj%3DCki5B%2BgYTbJO%3DACCJyytWZg%40mail.gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is real and no surviving objection questions the goal. The remaining points are fixable design points at level 2: standards-compat-1 (the back-branch error also rejects an ONLY case that works today), maintenance-fit-2 (set expression gets its own path for dependent constraints) and replication-ondisk-1 (no guidance for partitions already affected). A committer who knows the ALTER TABLE rebuild code has to decide the scope of the back-branch error.

Not checked: Without the code I could not check which dependency cases the new errors reject, or whether phase 3 re-validation covers every constraint kind for partitions, plain inheritance children and virtual generated columns. Only 3 of 6 messages were visible.

## Prior attempts

- [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://www.postgresql.org/message-id/flat/CACJufxEW9L279n=fpUg+qrXYP6g4oh_RLOTHVVphkWDVqaNGDw@mail.gmail.com) (related; Returned with feedback, 2026-04-02). The visible messages do not show why. Peter Eisentraut asked why tablecmds.c passes recurse = false when recursing, and suggested a more consistent use of the two flags would make the existing code work. Tom Lane said that if the flags matter here, DROP EXPRESSION should not use ATSimpleRecursion. The author replied in December 2025 that v3 is correct, and no later reply appears before the entry was returned with feedback. (AI summary of the thread) Different now: P1 changed DROP EXPRESSION recursion checks for multi-level inheritance; this proposal fixes dependent-object handling for set expression on partitions and with ONLY.

## Objections a reviewer is likely to raise

### 1. Back-branch fix makes working ALTER TABLE ONLY ... SET EXPRESSION fail

A migration script on 17.x runs alter table only t alter column b set expression on a partitioned parent that has an index on b and no foreign key. Today it succeeds. After a minor upgrade with the disallow patch it fails, and the user finds out when the deployment stops partway.

Message #2 says the ONLY case works when only an index depends on the column, yet v4 of the disallow patch rejects ONLY whenever any dependency exists (message #6). That turns a working command into an error in a stable release to fix a different, narrower failure. A committer would likely ask that a back-branch fix block only the cases that lose constraints or corrupt the rebuild, and leave working ones alone.

Precedent: [ALTER TABLE - check for column existence before starting operations](https://www.postgresql.org/message-id/flat/1897070.0DItnajBuJ@pierred-pdoc) (Returned with feedback, 2018-10-02)

**Question:** Which ALTER TABLE ONLY ... SET EXPRESSION cases that succeed today would the back-branch patch reject, and why must those be rejected rather than left working?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/CACJufxHp_6SepFvzsYd7UDsDmVTvrohApdQ1oxgfNEmQrNArdw%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CACJufxERxHfdx0QXJDxVvHnw%2BwfE%2BEVhFgBHK_ON490-9MVQWA%40mail.gmail.com); `standards-compat-1`</sub>

### 2. SET EXPRESSION gets its own dependent-handling path apart from ALTER TYPE

A later patch changes how ATPostAlterTypeCleanup rebuilds constraints or adds a dependent kind. It updates the rebuild path and misses the new SET EXPRESSION re-validation path. No test fails, and a partition is left with a constraint that was never checked against the new data.

Message #6 says v4 moves changedConstraintOids into tab->constraints for SET EXPRESSION only and removes that logic from ATPostAlterTypeCleanup. Dependents are also collected in phase 1 and then reset to NIL. AT_PASS_ALTER_TYPE and AT_PASS_SET_EXPRESSION used to share one cleanup step, and v4 splits it. Message #6 also found v3 incorrect because of pass ordering, which shows how easy it is to get this split wrong.

**Question:** Can the re-validation rule live inside RememberConstraintForRebuilding and ATPostAlterTypeCleanup, as message #5 suggests, so one place decides rebuild or re-check for both commands?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#4](https://www.postgresql.org/message-id/CACJufxE4PXVEtJ5t_W3FWz-Q-YPbVgne5chmu0BUoX5Sas8B%3Dw%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CACJufxERxHfdx0QXJDxVvHnw%2BwfE%2BEVhFgBHK_ON490-9MVQWA%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. No guidance for clusters that already lost inherited constraints

A PG17 user ran set expression on a partition, lost the inherited CHECK, then inserted rows that violate it. After the minor update the partition still has no pg_constraint row and no error appears. A later pg_dump restore re-inherits cc through partition of, and copy fails on the violating rows.

Message #1 shows the bug deletes pg_constraint rows for inherited CHECK and NOT NULL constraints. Both v4 patches only prevent new damage. Neither commit message says how to find affected partitions or repair them. For a back-branch fix that leaves bad catalog state behind, a committer would likely ask for a detection query and repair steps in the release notes before pushing.

**Question:** What query finds partitions whose inherited CHECK or NOT NULL constraint was dropped, and how should users repair them?

<sub>fixable, level 2, catalog changes; messages [#1](https://www.postgresql.org/message-id/CACJufxEomSz3BwUGfk8A6MeAKj%3DCki5B%2BgYTbJO%3DACCJyytWZg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CACJufxE4PXVEtJ5t_W3FWz-Q-YPbVgne5chmu0BUoX5Sas8B%3Dw%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CACJufxERxHfdx0QXJDxVvHnw%2BwfE%2BEVhFgBHK_ON490-9MVQWA%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Which alter table only ... set expression cases that succeed today does the back-branch patch reject, and why must each be rejected?
2. What is the root cause of the "could not read blocks" error in message #2, and does the new error cover every case that reaches it?
3. Can the re-validation rule live in RememberConstraintForRebuilding and ATPostAlterTypeCleanup, so that one place decides between rebuild and re-check?
4. What query finds partitions whose inherited CHECK or NOT NULL constraint was already dropped, and how should users repair them?
5. How does the second patch handle plain inheritance children, as opposed to partitions?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-1: No guidance for clusters that already lost inherited constraints
- **catalog changes:** The bug deletes inherited pg_constraint rows on partitions; the patches change when those rows are dropped or kept.
- **SQL-standard conflict or non-standard syntax:** The bugfix patch changes what alter table only ... set expression accepts, based on the meaning of ONLY.
- **pg_dump / restore round-trip:** A partition that already lost an inherited CHECK gets it back on restore, which can fail on rows that violate it.
- **backpatchability:** The bugfix patch targets stable branches and adds errors for commands that are accepted today.
- **maintenance burden ("who maintains this in 10 years"):** The second patch gives set expression a constraint re-validation path in tablecmds.c apart from ATPostAlterTypeCleanup.

## Who would commit it, and when

- Owner: silent. Someone who knows the ALTER TABLE phase 2 and phase 3 code in tablecmds.c, in particular ATPostAlterTypeCleanup and constraint inheritance on partitions. No committer has replied in the visible thread.
- Priority: needs champion; audience niche; review cost medium. Silent loss of inherited constraints is a correctness bug worth fixing, though only users who run set expression on a partition or with ONLY hit it. Both patches touch only tablecmds.c and tests, but the back-branch scope needs a decision from a committer, and none is engaged.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
