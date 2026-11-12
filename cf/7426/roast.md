# Design pre-review: [PG19]Three bugs with a CHECK constraint that only the child enforces

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7426](https://commitfest.postgresql.org/patch/7426/); [thread](https://www.postgresql.org/message-id/flat/CAGRkXqSGE%3DfQrjCoNQsyiCZRKfQ4SrB015CAR%3D3pXYB5tB%3Dieg%40mail.gmail.com), 16 of 16 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). v5-0001 is committed and no objection to it survived. For v5-0002 and v5-0003, standards-compat-1 and standards-compat-2 survived: the dump makes the child's constraint local, and ALTER CONSTRAINT ... ENFORCED would set conislocal for plain inheritance only. Message #14 leaves the plain inheritance case open and nobody answered. The goal is a valid bug fix, but the list has not agreed on this design.

Not checked: The code was not read. Whether v5-0002 and v5-0003 change --binary-upgrade output, and what exactly v5-0002 changes in tablecmds.c, is known only from thread messages.

## Prior attempts

- [Revert "Improve compression and storage support with inheritance"](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org) (related; Reverted, 2024-02-20). "pg_dump cannot currently dump all the structures that are allowed by this patch." Different now: P12 reverted a new feature because pg_dump could not dump the inheritance states it allowed. Here the child-enforced state can already be created in 18, so the feature stays and pg_dump is changed.
- [table inheritance versus column compression and storage settings](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org) (related; Returned with feedback, 2024-03-21). "We have already discovered with the previous attempt that there are unexpected pitfalls and lacking test coverage. Also, there isn't even a patch yet." ([message](https://www.postgresql.org/message-id/0f3d5151-009d-452b-8652-ed6f1031f8c8%40eisentraut.org)) Different now: P7 changed how inherited per-table settings merge and was returned after new dump failures appeared. This proposal is a bug fix with tests, though the thread also keeps finding related dump bugs.

## Objections a reviewer is likely to raise

### 1. Dump and pg_upgrade turn an inherited child CHECK into a local one

A user dumps and restores, or runs pg_upgrade on, a database where child k enforces parent p's NOT ENFORCED check c. Later the user runs alter table p drop constraint c. Before the upgrade this dropped k's copy; after it, k keeps c as a local constraint and rejects rows, and the user finds out only when inserts into k fail.

v5-0002 and v5-0003 emit the child's constraint inline, so it comes back with conislocal = true (messages #7, #14). Message #7 notes that pg_upgrade keeps the constraint inherited today and would stop doing so, which makes a dump fix change catalog state for upgrades that currently work. jian he doubts this is right for plain inheritance (message #14), and nobody else has agreed to it. The v1-0003 design (alter constraint ... enforced after the parent) keeps conislocal intact, at the cost of a scan at restore time.

Precedent: [Revert "Improve compression and storage support with inheritance"](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org) (Reverted, 2024-02-20)

**Question:** Why is a lossy conislocal change at restore and pg_upgrade better than the v1-0003 alter constraint form, at least under --binary-upgrade where no scan is needed?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#7](https://www.postgresql.org/message-id/CAGRkXqQQapfoGHpi2znkMfzkCk-%3D5GvmC16KKbVa6-BZn0yF6g%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CACJufxGkQSboSOj4PVA1S-2AA%2B17J%2BOM5YY2uxvBL%3DaDe9ES2Q%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CACJufxGjsWkmyhP8QTMTT6doWwuCjG9q%2Bfw2nkFiiAfLTxNMMg%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CACJufxE7ZYKrBnOP43dCED2mOQ2zDfpR8M-CBb_p_0e1gGqxYA%40mail.gmail.com); `standards-compat-1`</sub>

### 2. ALTER CONSTRAINT ENFORCED on a child would detach it from the parent

On a plain inheritance child, a user runs alter table k3 alter constraint c enforced and then alter table p3 drop constraint c. Under v5-0002 the child keeps c as a local constraint, but a partition in the same situation loses it. If this is back-patched to 18, a minor release changes what existing scripts do.

v5-0002 touches tablecmds.c and, following message #10, makes alter constraint ... enforced set conislocal for regular inheritance but not for partitions. That gives one command two meanings depending on the inheritance kind, and makes a property-change command alter inheritance linkage, which is a different job. The precedent cited in message #10 (SET NOT NULL changing conislocal) is a single case, and message #12 calls the current state an inconsistency. The thread does not say whether this change goes to 18.

**Question:** Should alter constraint ... enforced change conislocal at all, why only for plain inheritance, and is this behaviour change intended for a back-patch to 18?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#10](https://www.postgresql.org/message-id/CACJufxGkQSboSOj4PVA1S-2AA%2B17J%2BOM5YY2uxvBL%3DaDe9ES2Q%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CACJufxGjsWkmyhP8QTMTT6doWwuCjG9q%2Bfw2nkFiiAfLTxNMMg%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAGRkXqSnu-sUZQK9CB8K-cLbRQ45wLpC79wC2NuiRfL_0su-xQ%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CACJufxE7ZYKrBnOP43dCED2mOQ2zDfpR8M-CBb_p_0e1gGqxYA%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Do v5-0002 or v5-0003 change conislocal in --binary-upgrade output, where pg_upgrade keeps the constraint inherited today?
2. For plain inheritance, why is restoring the child's constraint as local better than the v1-0003 form, which emits ALTER CONSTRAINT ... ENFORCED after the parent?
3. Should ALTER CONSTRAINT ... ENFORCED on a child set conislocal at all, and why for plain inheritance but not for partitions?
4. How will 18 get a fix for the pg_dump loss, given that v5-0002 and v5-0003 use ALTER CONSTRAINT syntax that 18 does not have?
5. Which thread will carry the ALTER COLUMN TYPE fix and the related dump losses from messages #8, #11 and #12?

## Touch points

- **on-disk format / pg_upgrade:** Message #7 says the local-constraint dump would also make the child's constraint local under pg_upgrade, which keeps it inherited today.
- **catalog changes:** After restore the child's pg_constraint row has conislocal = true; v5-0002 also changes tablecmds.c.
- **SQL-standard conflict or non-standard syntax:** ALTER CONSTRAINT ... ENFORCED on a parent now errors over an ENFORCED NOT VALID child; v5-0002 changes its effect on a child.
- **planner behaviour change:** v5-0001 stops a parent being marked validated over a NOT VALID child, which constraint exclusion relied on.
- **pg_dump / restore round-trip:** v5-0002 and v5-0003 change how pg_dump emits a child's ENFORCED CHECK whose parents keep it NOT ENFORCED.
- **backpatchability:** Bugs 2 and 3 also exist in 18, and the thread does not say whether or how the pg_dump fixes go there.
- **maintenance burden ("who maintains this in 10 years"):** pg_dump gains another case that rewrites inherited constraint state, next to the not-null case from message #12.

## Who would commit it, and when

- Owner: qualified committer engaged. Andrew Dunstan committed v5-0001. The pg_dump patches need a committer who knows the inheritance handling in pg_dump and the conislocal rules in tablecmds.c; none has yet spoken on them in the visible thread.
- Priority: fix design first; audience niche; review cost medium. The bug affects only users whose child enforces a CHECK that the parent keeps NOT ENFORCED, but for them a restore silently loses enforcement. The two remaining patches are small; the conislocal question for plain inheritance needs an answer before code review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
