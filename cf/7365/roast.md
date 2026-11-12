# Design pre-review: ATTACH PARTITION cost grows linearly with pg_constraint size (seqscan in CloneFkReferenced), much worse since not-null constraints are in pg_constraint (PG 18)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7365](https://commitfest.postgresql.org/patch/7365/); [thread](https://www.postgresql.org/message-id/flat/trinity-08d3329b-f6b2-4c11-91d7-2198e50bf4fc-1790681922354%40trinity-msg-rest-gmx-gmx-live-58cc8f554d-c56lm), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection stops the proposal at levels 1a, 1b or 2. The problem is stated with a reproducer, demand is shown by a production report, and the design copies the existing contypid index. The one surviving objection, maintenance-fit-1, is a measurement request about index maintenance cost on bulk DDL. A committer who knows the catalog code must weigh that cost and sign the catalog change.

Not checked: The v4 patch and the Tom Lane message that shaped it are not in the visible material. The four converted lookups, the index name and the contype check in each loop could not be checked.

## Prior attempts

None found before 2026-09-29.

## Objections a reviewer is likely to raise

### 1. Unreconciled cost of the full index on every constraint insert

A user restores a large schema with pg_restore, or runs a migration that creates many tables with NOT NULL columns. Each not-null row in pg_constraint now also inserts an index entry with confrelid = 0. Users without partitions or foreign keys pay this cost and gain nothing.

Message #4 measured about 5% extra on bulk DDL for v1. Message #11 says constraint creation costs the same, but shows no run that reconciles the two numbers. Message #7 notes that about 90% of the index entries are not used, so nearly all of the maintenance cost lands on DDL that never uses the index. The shape copies the existing contypid index and has support in the thread, so this is a measurement gap, not a design flaw.

**Question:** What is the v4 overhead on a schema-only pg_restore with about 1M not-null constraints, and why did the 5% seen in v1 go away?

<sub>fixable, level 2, performance on the hot path; messages [#4](https://www.postgresql.org/message-id/179069810491.316007.4145107668471017354%40gmail.com), [#7](https://www.postgresql.org/message-id/arv-UHG6c-raryI-%40alvherre.pgsql), [#11](https://www.postgresql.org/message-id/179211577746.206608.6783271124892214599%40gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. What is the v4 overhead on a schema-only pg_restore with about 1M not-null constraints, and why did the 5% seen in v1 go away?
2. How do the pg_depend lookup and the trigger-based early exit from message #1 compare with the index on the same reproducer?
3. Do pg_upgrade or tools that list catalog indexes need any change for the new index?
4. Is any fix planned for PG 18 and 19, where the catalog index cannot be added?

## Touch points

- **on-disk format / pg_upgrade:** A new system index on pg_constraint changes the initial catalog contents and needs a catalog version bump.
- **catalog changes:** The patch adds a non-unique index on pg_constraint(confrelid).
- **backpatchability:** The slowdown is a PG 18 regression, but a new catalog index cannot go into released branches; the entry targets version 20.
- **maintenance burden ("who maintains this in 10 years"):** One more catalog index to maintain, and four lookup sites now depend on it.
- **performance on the hot path:** Every pg_constraint insert, including each not-null row, also inserts an index entry with confrelid = 0.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera, who takes part in the thread, agreed with the approach but deferred review until PG 19 work and older bugs are done. The owner needs to know pg_constraint and the partition foreign-key code.
- Priority: proceed; audience a sizeable minority; review cost low. Message #1 shows a production regression since PG 18 that affects schemas with many partitions, and the fix copies the existing contypid index. No patch text was read; review cost is judged from the brief's description of one index and four call sites.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
