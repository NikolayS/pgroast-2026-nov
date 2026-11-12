# Design pre-review: Support "ALTER FOREIGN TABLE ... RENAME CONSTRAINT ..."

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7558](https://commitfest.postgresql.org/patch/7558/); [thread](https://www.postgresql.org/message-id/flat/CAB8KJ%3DgBwpo%2BiXR5n_NQ1xude3Ms2P3n%3DOmL6F%3DB%3DVSNgCXgfw%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). One objection survived, already-exists-1, and it is a fixable question of value against cost. Message #1 says `alter table ... rename constraint` already works on foreign tables, so the new form is a second spelling. It matches the existing rename column and add/drop constraint forms and reuses the existing rename path. The three command tag objections were refuted, so nothing that survived stops the proposal at levels 1-2.

Not checked: Without the code, the scope of the AlterObjectTypeCommandTag() refactor cannot be checked, so the claim that no existing command changes its tag is unverified. The behaviour of `only` and `*` for inherited constraints also could not be judged.

## Prior attempts

None found before 2026-10-30.

## Objections a reviewer is likely to raise

### 1. Rename constraint on foreign tables already works through alter table

A user today runs `alter table ft_foo rename constraint alice to bob` and gets exactly the result the new syntax gives. The patch adds a second spelling, plus a command tag refactor in utility.c. If that refactor changes the tag of any existing column or constraint command, event triggers or tools that filter on the tag can stop matching after an upgrade.

Message #1 states that alter table ... rename constraint works on all table-like objects, so the gap is that users cannot find it, not that the job cannot be done. A documentation cross-reference in alter_foreign_table.sgml would cover the same need without grammar or command tag changes. The author's consistency case is real, since alter foreign table already has rename column and add/drop constraint. Still, [P1] shows a small ease-of-use gain being weighed against even modest added code.

Precedent: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://www.postgresql.org/message-id/flat/CADrsxdYj2SMquRgf=DS9d_qLMW_gnF7PxzRp0VKTM7OfBsPpVg@mail.gmail.com) (Rejected, 2025-03-26)

**Question:** Why is a cross-reference to alter table rename constraint in the alter foreign table docs not enough, and does the tag refactor change the tag of any existing command?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAB8KJ%3DgBwpo%2BiXR5n_NQ1xude3Ms2P3n%3DOmL6F%3DB%3DVSNgCXgfw%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Why is a cross-reference to `alter table ... rename constraint` in the alter foreign table documentation not enough?
2. Does any existing statement return a different command tag after the AlterObjectTypeCommandTag() refactor, and if so, which?
3. Which constraint types on a foreign table can the new form rename?
4. How do `only` and `*` behave when the renamed constraint is inherited by or from another table?
5. Do the regression tests check the command tag returned for the new form and for the existing column and constraint forms?

## Touch points

- **SQL-standard conflict or non-standard syntax:** The patch adds a non-standard `alter foreign table ... rename constraint` form in gram.y.
- **psql UX conventions:** The patch extends tab completion for alter foreign table rename constraint and add/drop column/constraint.
- **maintenance burden ("who maintains this in 10 years"):** The patch refactors AlterObjectTypeCommandTag() in tcop/utility.c and adds a second spelling to keep documented and tested.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #1 says `alter table ... rename constraint` already does this on foreign tables.

## Who would commit it, and when

- Owner: silent. A committer who knows the utility command grammar and the command tag code in tcop/utility.c would have to own it.
- Priority: proceed; audience niche; review cost low. The author calls the need niche in message #1, and a workaround exists. The patch is about 11 KiB, reuses the existing rename path, and touches only the grammar, utility.c, psql completion, docs and tests.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
