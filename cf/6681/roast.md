# Design pre-review: docs: Include database collation check on SQL from alter_collation.sgml

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6681](https://commitfest.postgresql.org/patch/6681/); [thread](https://www.postgresql.org/message-id/flat/DHOTJNEQ82FK.2VR8AIQNLO6UP%40gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is sound and message #3 reproduced the gap: the current query returns no rows for an index on the database default collation. The remaining concerns are fixable design points at level 2. v2 drops named ICU collations that the current query reports (standards-compat-1) and lists indexes only while the lead-in still promises all dependent objects (standards-compat-2, prior-attempts-1). No level fails outright.

Not checked: The query was tested only with libc, in message #3. The ICU cases, non-index dependents and the rendered docs build were not checked by anyone in the thread.

## Prior attempts

- [Revert per-index collation version tracking feature.](https://www.postgresql.org/message-id/flat/CA+hUKGLhj5t1fcjqAu8iD9B3ixJtsTNqyCCD4V0aTO9kAKAjjA@mail.gmail.com) (related; Reverted, 2021-05-07). Design problems were discovered in the handling of composite types and record types that would cause some relevant versions not to be recorded. Misgivings were also expressed about the use of the pg_depend catalog for this purpose. We're out of time for this release so we'll revert and try again. Commits reverted: 1bf946bd: Doc: Document known problem with Windows collation versions. cf002008: Remove no-longer-relevant test case. ef387bed: Fix bogus collation-version-recording logic. 0fb0a050: Hide internal error for pg_collation_actual_version(<bad OID>). ff942057: Suppress "warning: variable 'collcollate' set but not used". d50e3b1f: Fix assertion in collation version lookup. f24b1569: Rethink extraction of collation dependencies. 257836a7: Track collation versions for indexes. cd6f479e: Add pg_depend.refobjversion. 7d1297df: Remove pg_collation.collversion. Different now: P1 tracked collation versions per index in pg_depend and was a feature, reverted for design problems. This proposal only changes a documentation query; P1 supports only the technical point that collations inside composite types are easy to miss.

## Objections a reviewer is likely to raise

### 1. v2 stops reporting ICU collation mismatches the current query catches

An administrator with ICU-based indexes upgrades libicu and runs the documented query. It returns no rows, so they run ALTER COLLATION ... REFRESH VERSION or ALTER DATABASE ... REFRESH COLLATION VERSION without rebuilding. The indexes stay corrupt until queries return wrong results.

The current query compares collversion with pg_collation_actual_version() for every provider, so named ICU collations are covered today. The v2 WHERE clause accepts only collprovider 'c', and the default branch only datlocprovider 'c', which drops named ICU collations and ICU database defaults. Message #3 reported this and the thread has no reply. A change to the reference query for the default-collation gap should not reduce coverage for other providers.

**Question:** Can the query cover the default collation without dropping named ICU collations and ICU database defaults, which the current query reports?

<sub>fixable, level 2, design; messages [#2](https://www.postgresql.org/message-id/DHSC1UP35NS0.VACTPNXNR7FS%40gmail.com), [#3](https://www.postgresql.org/message-id/CAOh5eDWPy5%3DHQgYMAQLShUFDNmny%3DYbDRsoQT_o1K%2B94iTvK2Q%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Query narrows to indexes while the text still promises all dependents

A user has a check constraint, materialized view or partition bound that depends on a libc collation with a version change. The v2 query lists only indexes, so the user rebuilds those, refreshes the version, and the warning goes away. The other objects stay inconsistent and nothing reports them again.

The old query read pg_depend and listed every dependent object. v2 reads only pg_index.indcollation. The surrounding sentence is unchanged and still says the query finds "the objects that depend on them", and the server HINT says to rebuild "all objects affected". Message #2 explains the move to an index-only query only by the wiki source, not by a reason to drop the other objects.

**Question:** Why does the documented query drop non-index dependents, and if that is intended, how will the page tell readers which objects it no longer covers?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/DHOTJNEQ82FK.2VR8AIQNLO6UP%40gmail.com), [#2](https://www.postgresql.org/message-id/DHSC1UP35NS0.VACTPNXNR7FS%40gmail.com); `standards-compat-2`</sub>

### 3. Index-only query misses composite-type and non-index dependencies

After a glibc upgrade, an administrator runs the v2 query and sees no row for an index on a composite-typed column with a text field, or for a partitioned table keyed on text. The administrator refreshes the version without rebuilding. Wrong index scans or misrouted rows show up later.

v2 reads only pg_index.indcollation, which records one collation per index column. It does not record collations nested inside composite types, index predicates, constraints or partition keys. [P1] was reverted partly because composite and record types caused relevant versions not to be recorded, and the same gap applies to this query. The unchanged lead-in still promises "the objects that depend on them", but v2 returns indexes only.

Precedent: [Revert per-index collation version tracking feature.](https://www.postgresql.org/message-id/flat/CA+hUKGLhj5t1fcjqAu8iD9B3ixJtsTNqyCCD4V0aTO9kAKAjjA@mail.gmail.com) (Reverted, 2021-05-07)

**Question:** Which object kinds does the v2 query knowingly skip, and will the page say so or point to the wiki for those cases?

<sub>fixable, level 2, prior rejection; messages [#2](https://www.postgresql.org/message-id/DHSC1UP35NS0.VACTPNXNR7FS%40gmail.com), [#3](https://www.postgresql.org/message-id/CAOh5eDWPy5%3DHQgYMAQLShUFDNmny%3DYbDRsoQT_o1K%2B94iTvK2Q%40mail.gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. Can the query cover the database default collation without dropping the named ICU collations that the current query reports?
2. Should the default-collation branch also handle a database whose datlocprovider is ICU, which neither the old query nor v2 reports?
3. Why does v2 list indexes only, and could it keep the pg_depend branch for named collations and add a default-collation branch?
4. Which object kinds does v2 knowingly skip, such as indexes on composite-typed columns and partition keys, and will the page say so?
5. Should the lead-in sentence change, since it still says the query finds "the objects that depend on them"?

## Touch points

- **catalog changes:** The example query reads pg_index, pg_collation and pg_database, but the patch changes no catalog.
- **backpatchability:** The current query misses default-collation indexes in released branches too, so a committer must decide whether to backpatch the docs fix.
- **maintenance burden ("who maintains this in 10 years"):** The v2 query names provider codes 'c' and 'd', so it needs an edit whenever provider coverage changes.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The brief asks whether the longer query belongs in the reference page or on the Locale_data_changes wiki page.

## Who would commit it, and when

- Owner: silent. Someone who knows collation version tracking and the locale providers (libc, ICU) would have to check the query. No committer has posted in the thread.
- Priority: fix design first; audience a sizeable minority; review cost low. The fix helps any administrator who checks for indexes to rebuild after a C library upgrade, and the patch is one docs hunk. The ICU coverage and the index-only scope need an answer before a committer can take it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
