# Design pre-review: Dropping a composite attribute causes data integrity violations

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7169](https://commitfest.postgresql.org/patch/7169/); [thread](https://www.postgresql.org/message-id/flat/CA%2BUBoq0F2ua2fQEAU3-6w0oxOgFc9BgEN4gRcchiUUZ%2BWouVrQ%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection disputes the goal: message #1 shows reproducible corruption, and the fix reuses the check that retyping already uses. The surviving objections are fixable and none stops review. They concern the target branches and the migration path (standards-compat-1, demand-1), whether the stored-expression cases go in together (standards-compat-3), and an owner blocked by another role's table (security-1). The backpatch decision is one a committer has to make.

Not checked: The code was not read, so the placement of the call in ATExecDropColumn(), its behaviour under inheritance recursion and typed tables, and any documentation changes could not be checked. The interaction with the reviewer's pgsql-bugs patch is known only from message #3.

## Prior attempts

- [Revert applying column aliases to the output of whole-row Vars.](https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us) (related; Reverted, 2022-03-17). "But corrupt data is worse, so back-patched it will be." Different now: P6 reverted a feature that labeled stored whole-row tuples with a blessed RECORD type. This patch refuses DDL that changes how stored composite values read. P6 shows a behaviour change was backpatched to stop corrupt data.

## Objections a reviewer is likely to raise

### 1. Accepted DDL becomes an error with no documented migration path

A team runs a schema migration that drops an attribute from a composite type used by a column, which works today. After upgrade, the migration fails with "cannot alter type ... because column ... uses it", and CASCADE does not help. The team finds out in the deploy step, and the docs give no supported way to finish the change.

The touched files are code and tests only, so the ALTER TYPE and ALTER TABLE reference pages do not mention the new refusal or the workaround. The proposal does not say whether the fix is backpatched, which matters because a minor release would break scripts that ran the day before. [P6] shows the project has backpatched a behaviour change to stop corrupt data, but that commit message stated the workaround and the scope.

Precedent: [Revert applying column aliases to the output of whole-row Vars.](https://www.postgresql.org/message-id/flat/2950001.1638729947@sss.pgh.pa.us) (Reverted, 2022-03-17)

**Question:** Which branches get this change, and what documented steps should a user follow to remove an attribute from a composite type that a table column uses?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CA%2BUBoq0F2ua2fQEAU3-6w0oxOgFc9BgEN4gRcchiUUZ%2BWouVrQ%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CA%2BUBoq3D5SPuDGFQN5MVp09AYEV2SZz2twgyuGnuwigMrN7BHA%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Narrow corruption case blocks a common schema change for all users

A user has a column of type ct with no unique index or equality-dependent object, and runs alter type ct drop attribute b to retire a field. After this patch the command fails, even with CASCADE. The user finds out at deploy time and has no documented path that keeps the stored data.

Corruption needs a unique index, a constraint, or another equality-sensitive object on the type, and the only evidence of it is the constructed example in message #1. The patch refuses every drop on a stored type, so users without such objects lose an operation that works for them today. The changed test_typex and test_tbl3 fixture show the drop is used on stored types. The thread does not describe the migration those users would follow, which the brief lists as unclear.

**Question:** Is there any field report of this corruption, and what supported migration do users who drop attributes of stored types follow once the drop is refused?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CA%2BUBoq0F2ua2fQEAU3-6w0oxOgFc9BgEN4gRcchiUUZ%2BWouVrQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CA%2BbCEdCbDjqwv1h%3D9P1ZG-y9baFe9_74t2wZobeR6GuK98K-AQ%40mail.gmail.com); `demand-1`</sub>

### 3. Contract changes twice if expression cases follow separately

Release N refuses the drop for columns but still allows it when a view, CHECK constraint or default holds a constant of the type. A later release adds those refusals too, so scripts break a second time. Between the two releases, a CHECK constraint can still make a table reject every row.

Message #3 shows six stored-expression cases with the same effect that v2 leaves open, and proposes combining with a separate pgsql-bugs patch. The author has not replied, so the proposal does not yet say whether one user-visible change or two is intended. One combined change is easier to document and to explain in release notes.

**Question:** Will this patch and the stored-expression patch go in together, so users see one change to when DROP ATTRIBUTE and DROP COLUMN are refused?

<sub>fixable, level 2, design; messages [#3](https://www.postgresql.org/message-id/CA%2BbCEdCbDjqwv1h%3D9P1ZG-y9baFe9_74t2wZobeR6GuK98K-AQ%40mail.gmail.com); `standards-compat-3`</sub>

### 4. Another role can block DROP COLUMN on a table it does not own

Role bob, who can use alice's types, creates a table with a column of alice's composite type or of her table's row type. Alice then runs ALTER TABLE ... DROP COLUMN or ALTER TYPE ... DROP ATTRIBUTE on her own object and gets an error, even with CASCADE. She cannot drop or alter bob's table, so the change is blocked until a superuser or bob acts.

The brief and the v1 tests both say CASCADE does not override the refusal. DROP TYPE ... CASCADE does remove dependent columns in other roles' tables, so the owner can always undo a dependency some other role created. Under this patch a dependency that a non-owner creates becomes a hard block on routine DDL by the owner. The same block already exists for retyping, but dropping a column is far more common, and neither the proposal nor messages #1 and #3 discuss this privilege aspect.

**Question:** How does an owner drop a column when a role they cannot control stores the row type, and why should CASCADE not drop or rewrite those dependent columns?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/CA%2BUBoq0F2ua2fQEAU3-6w0oxOgFc9BgEN4gRcchiUUZ%2BWouVrQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CA%2BbCEdCbDjqwv1h%3D9P1ZG-y9baFe9_74t2wZobeR6GuK98K-AQ%40mail.gmail.com); `security-1`</sub>

## Questions for the author

1. Which branches should get this change, given that it turns DDL accepted today into an error?
2. What documented steps should a user follow to remove an attribute from a composite type that a table column stores?
3. Should this patch go in together with the stored-expression patch from message #3, so that users see one change in when the drop is refused?
4. How does an owner drop a column when another role's table stores the row type, and why should CASCADE not handle those dependent columns?
5. Do the ALTER TYPE and ALTER TABLE reference pages need text about the new refusal?

## Touch points

- **on-disk format / pg_upgrade:** The patch concerns how stored composite values read after a drop, and it changes the test_tbl3 pg_upgrade fixture.
- **SQL-standard conflict or non-standard syntax:** ALTER TYPE ... DROP ATTRIBUTE and ALTER TABLE ... DROP COLUMN now fail in cases accepted today, and CASCADE does not override.
- **pg_dump / restore round-trip:** The bug makes a plain pg_dump fail to restore; the patch prevents new cases but not existing ones.
- **security / privileges:** A role that creates a column of another role's type can block the owner's drop, even with CASCADE.
- **backpatchability:** The entry is tagged Bugfix, but the thread does not say which branches get a change that turns accepted DDL into an error.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Narrow corruption case blocks a common schema change for all users

## Who would commit it, and when

- Owner: silent. Someone who knows ALTER TABLE in tablecmds.c and composite type dependencies. Message #3 copies Heikki because d78040a469b touched this call site, but he has not replied in the visible thread.
- Priority: proceed; audience niche; review cost low. The patch fixes reproducible corruption with one call to an existing check in tablecmds.c plus tests, and a reviewer has confirmed it. Only users who store composite types in columns and drop attributes are affected; the open points are scope and backpatch decisions.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
