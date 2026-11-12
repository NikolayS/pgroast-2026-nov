# Design pre-review: Fix UNIQUE WITHOUT OVERLAPS + NULLS NOT DISTINCT + multiranges

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7465](https://commitfest.postgresql.org/patch/7465/); [thread](https://www.postgresql.org/message-id/flat/CA%2BrenyV93dvyRbjYy320WaWE7Rg-uRDV4JfFBZ2AmmUcODZEgw%40mail.gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). This is a correctness fix for a shipped feature: a declared NULLS NOT DISTINCT constraint accepts duplicates on multirange columns. No objection survived at levels 1a or 1b. The one surviving objection, standards-compat-1, is fixable: the backpatch gives 18 users no guidance for tables that already hold duplicate NULL keys. That point and the C signature change in a back branch are for a committer to judge.

Not checked: Without the code, this review could not check whether index_recheck_constraint is exported, or whether the new parameter leaves ordinary EXCLUDE constraints unchanged as the commit message says.

## Prior attempts

None found before 2026-10-09.

## Objections a reviewer is likely to raise

### 1. Backpatched fix gives no path for 18 tables already holding duplicates

A user on 18 has a multirange table with WITHOUT OVERLAPS and NULLS NOT DISTINCT, and it already holds duplicate NULL keys. After the minor update, a later REINDEX, or a pg_dump restore that rebuilds the constraint, may fail. The user first sees the duplicates during an upgrade or a restore.

The commit message says the bug "wrongly permitted duplicates", so some 18 tables can hold rows that the fixed constraint rejects. The patch touches only execIndexing.c and the regression tests, and it has no documentation hunk. The proposal does not say how users find such rows before or after updating. Backpatched constraint fixes usually come with a release-note query that finds violating rows and advice on whether to REINDEX.

**Question:** What release-note text and detection query will tell 18 users whether their multirange temporal keys already hold duplicate NULLs, and what to do about them?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CA%2BrenyV93dvyRbjYy320WaWE7Rg-uRDV4JfFBZ2AmmUcODZEgw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. What release-note text and detection query will tell 18 users whether their multirange temporal keys already hold duplicate NULLs, and how to repair them?
2. Is index_recheck_constraint visible to extensions, and if so, does the 18 backpatch keep the old signature working?
3. Do the new tests in without_overlaps.sql cover a multirange unique constraint with NULLS NOT DISTINCT for both the insert path and the update path?
4. Will the open question about non-strict operators in EXCLUDE constraints go to a separate thread, so that it does not hold up this fix?

## Touch points

- **ABI / extension API:** The patch adds a parameter to index_recheck_constraint in execIndexing.c and is to be backpatched to 18.
- **pg_dump / restore round-trip:** Restoring a dump of an 18 table that already holds duplicate NULL keys may fail once the constraint is enforced.
- **backpatchability:** The commit message says "Backpatch-through: 18", and the fix changes constraint behaviour in a minor release.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Eisentraut is signed up as reviewer in the CommitFest entry but has not posted in the thread. The owner needs to know the executor's exclusion-constraint recheck code and the temporal constraint feature.
- Priority: proceed; audience niche; review cost low. The bug affects only unique constraints that combine WITHOUT OVERLAPS, NULLS NOT DISTINCT and a multirange column, but it lets a declared constraint accept duplicates. The patch is 8.4 KB and touches one executor file plus regression tests.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
