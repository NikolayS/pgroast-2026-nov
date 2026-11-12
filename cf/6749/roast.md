# Design pre-review: Disallow whole-row index references with virtual generated columns?

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6749](https://commitfest.postgresql.org/patch/6749/); [thread](https://www.postgresql.org/message-id/flat/CAJTYsWVmfHyooKCj30suW8%3DY6b8be7WcyYmuAKMD%3D2mTPFpDkg%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection disputes the goal: message #3 shows a unique index that keys on (1,) while SQL returns (1,2). Two fixable design gaps remain. The check runs only in DefineIndex, so `alter table ... add column ... virtual` on a table with a whole-row index reaches the same state (replication-ondisk-1). The proposal also does not say what pg_upgrade or a restore does with indexes of this kind that already exist (standards-compat-1).

Not checked: Without the code, I could not check whether the check catches every form of whole-row reference, or whether alter table add column already rejects this case elsewhere. The error message and error code are also not shown.

## Prior attempts

None found before 2026-05-08.

## Objections a reviewer is likely to raise

### 1. Adding a virtual generated column later still creates the broken index

A user has `create unique index ... on t (a) where t is not null` on a table with no virtual column, which passes the new check. They then run `alter table t add column b int generated always as (a * 2) virtual`. The index now evaluates the predicate on the heap tuple without b, and uniqueness gaps appear exactly as in message #1.

The patch touches only indexcmds.c, so the check runs at DefineIndex time. The invariant the patch protects is that no index evaluates a whole-row value that is missing a virtual column. ALTER TABLE ADD COLUMN reaches the same state without passing through DefineIndex. Matthias in message #4 already noted that adding columns can change whole-row IS NOT NULL results, so this path is known to matter.

**Question:** Does alter table add column of a virtual generated column fail when the table already has an index with a whole-row reference, and where is that enforced?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAJTYsWVmfHyooKCj30suW8%3DY6b8be7WcyYmuAKMD%3D2mTPFpDkg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAJTYsWV09r2OTJq690rm4CE1Qtg%3DfqDpWNo0PfHCfAbK199qqg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAEze2WhQWJgvjBCq0t4MDa1cscuO3NPO%3D7tt3fRJEDyLdhbvzQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Existing whole-row indexes on such tables block dump restore and pg_upgrade

A Postgres 18 user has a whole-row expression or partial index on a table with a virtual generated column. pg_upgrade to 19 replays that CREATE INDEX from the schema dump, the new check rejects it, and the upgrade stops partway. A plain pg_dump restore also fails at that index, and the user learns this only during the restore.

Virtual generated columns already shipped, so these indexes can exist in the field today, and the patch is tagged Bugfix. The proposal does not mention upgrade, restore, or a pg_upgrade pre-check. Message #1 says existing whole-row indexes "should not be affected", but that covers only tables without virtual generated columns. If the fix is back-patched, a minor release would also start rejecting a dump that the same major version produced.

**Question:** What does pg_upgrade or a restore do with an existing index of this kind, and should pg_upgrade --check report it before the upgrade starts?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#1](https://www.postgresql.org/message-id/CAJTYsWVmfHyooKCj30suW8%3DY6b8be7WcyYmuAKMD%3D2mTPFpDkg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAJTYsWV09r2OTJq690rm4CE1Qtg%3DfqDpWNo0PfHCfAbK199qqg%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Does `alter table ... add column ... generated always as (...) virtual` fail when the table already has an index with a whole-row reference, and where is that enforced?
2. What do pg_upgrade and a pg_dump restore do with an existing index of this kind, and should pg_upgrade --check report it?
3. Is the fix meant for back branches, and what should users who already have such an index do?
4. If CF 5667 is committed, is this patch dropped for master and kept for released branches only?
5. Does the patch address Matthias's point that whole-row IS [NOT] NULL predicates in indexes may be unsafe on any table when columns are added?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade replays create index for existing whole-row indexes on tables with virtual generated columns, and the new check rejects them.
- **catalog changes:** see objection replication-ondisk-1: Adding a virtual generated column later still creates the broken index
- **pg_dump / restore round-trip:** A dump that contains such an index fails to restore at the create index statement.
- **backpatchability:** The CF entry is tagged Bugfix and virtual generated columns already shipped, so a back-patch would add a new create index error in a minor release.
- **maintenance burden ("who maintains this in 10 years"):** The check must be removed or rewritten when index support for virtual generated columns arrives, as message #3 says.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera replied in message #2 but questioned the virtual-column-specific scope and did not offer to take the patch. The owner needs to know DefineIndex and the generated-column code.
- Priority: fix design first; audience niche; review cost low. The patch is 4.3 KB in indexcmds.c plus regression tests, and it affects only whole-row indexes on tables with virtual generated columns. It fixes a silent uniqueness failure, but the alter table path and the upgrade path need an answer first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
