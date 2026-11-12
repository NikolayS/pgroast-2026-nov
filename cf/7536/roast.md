# Design pre-review: JSON_SERIALIZE for JSONB returns parts of the internal JSONB representation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7536](https://commitfest.postgresql.org/patch/7536/); [thread](https://www.postgresql.org/message-id/flat/03583FE7-A5EE-407C-907D-67865F3F8BFC%40gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is reproduced on master, the docs allow a jsonb argument, and no objection to fixing it survived. The two surviving objections, standards-compat-1 and replication-ondisk-1, are fixable and concern expressions already stored when the parser-only fix is backpatched through 17. A committer who knows the SQL/JSON parser code has to choose between the parser fix and an executor-side fix for back branches. No one but the author has replied yet.

Not checked: The code was not read, so the review could not confirm that the new coercion is right for every RETURNING type. It also could not check whether other callers of makeJsonConstructorExpr() change behaviour.

## Prior attempts

None found before 2026-03-03.

## Objections a reviewer is likely to raise

### 1. Parser-only fix leaves existing views wrong after a minor update

A user on 17 or 18 has a view that calls JSON_SERIALIZE() on a jsonb column. After the minor update the view still returns header bytes. A copy restored from pg_dump re-parses the view and returns JSON text, so the two servers give different results for the same view.

Message #2 and the 0001 commit message say stored expressions keep the old coercion until recreated, and the patch targets back branches through 17. A minor release is expected to fix bugs without user action, but a minor update does not re-parse stored rules. Message #2 rejects the executor-side fix because it adds a step and keeps the wrong coercion in the tree, but it is the only option that also fixes existing views. The proposal does not give release-note text or a query that finds affected views.

**Question:** How will users on 17 and 18 find and fix existing views after the minor update, or why is an executor-side fix for old trees not worth its cost?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/CAGRkXqQXwh_gJmQE3PZj0v13omMPgFGBbb8TJoBFH8ECsAFCUg%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Expression indexes and stored generated columns keep old values after upgrade

A user has an expression index or a stored generated column on JSON_SERIALIZE(jsonb_col). pg_upgrade or dump and restore re-parses the expression with the fixed parser, but pg_upgrade copies the index files and table data unchanged. Index lookups with the new expression then miss rows, and generated column values no longer match their expression, with no error.

Message #2 and the commit message say the fix lives in the parser and that stored expressions must be recreated. Re-parsing happens silently whenever an expression is deparsed and restored, so the stored expression and the stored data can disagree after pg_upgrade. The proposal does not say whether JSON_SERIALIZE() on jsonb is accepted in index and generated column expressions. It gives no reindex or rewrite instruction.

**Question:** Can JSON_SERIALIZE() on jsonb appear in an index or stored generated column, and if so, what must users run after pg_upgrade or a minor update?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#2](https://www.postgresql.org/message-id/CAGRkXqQXwh_gJmQE3PZj0v13omMPgFGBbb8TJoBFH8ECsAFCUg%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. How do users on 17 and 18 find the views, rules and defaults that call JSON_SERIALIZE() on jsonb and must be recreated after the minor update?
2. For back branches, why is the parser-only fix better than an executor-side conversion that also corrects stored expressions?
3. Can JSON_SERIALIZE() on jsonb appear in an expression index or a stored generated column, and if so, what must users run after pg_upgrade?
4. Do other SQL/JSON constructors that go through makeJsonConstructorExpr() build the same wrong coercion for a jsonb input?
5. What release-note text do you propose for the minor releases?

## Touch points

- **on-disk format / pg_upgrade:** An expression index or stored generated column on JSON_SERIALIZE(jsonb) could hold old values after pg_upgrade re-parses the expression.
- **catalog changes:** see objection replication-ondisk-1: Expression indexes and stored generated columns keep old values after upgrade
- **pg_dump / restore round-trip:** A view restored from pg_dump is re-parsed and returns JSON text, while the original view still returns header bytes.
- **backpatchability:** The commit message asks for a backpatch through 17, and stored expressions keep the old coercion after a minor update.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the SQL/JSON constructor code in the parser and executor, and the rules for backpatching fixes that affect stored expressions.
- Priority: proceed; audience niche; review cost low. A documented function returns jsonb header bytes, sometimes invalid in the server encoding, and the fix touches one parser file and adds tests. Only callers of JSON_SERIALIZE() with jsonb are affected, and the open point is the back-branch handling of stored expressions.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
