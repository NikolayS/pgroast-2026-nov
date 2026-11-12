# Design pre-review: docs: Include database collation check on SQL from alter_collation.sgml

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6681](https://commitfest.postgresql.org/patch/6681/) (Needs review); authors: Matheus Alcantara (matheus.alcantara)
- Thread: [docs: Include database collation check on SQL from alter_collation.sgml](https://www.postgresql.org/message-id/flat/DHOTJNEQ82FK.2VR8AIQNLO6UP%40gmail.com), started 2026-04-09
- Messages read: 3 of 3
- Patch prose read: v2-0001-docs-Include-database-collation-check-on-SQL-from.patch, v1-0001-docs-Include-database-collation-check-on-SQL-from.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is sound and message #3 reproduced the gap: the current query returns no rows for an index on the database default collation. The remaining concerns are fixable design points at level 2. v2 drops named ICU collations that the current query reports (standards-compat-1) and lists indexes only while the lead-in still promises all dependent objects (standards-compat-2, prior-attempts-1). No level fails outright.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The query was tested only with libc, in message #3. The ICU cases, non-index dependents and the rendered docs build were not checked by anyone in the thread.

## The proposal, restated

**Problem.** The alter collation reference page gives a query that lists collations needing a refresh and their dependent objects. Objects that use the database default collation have no pg_depend entry, so the query misses them when the database collation version changes. This affects administrators checking for indexes to rebuild after an OS or library upgrade.

**Proposed design.** v2 replaces the pg_depend-based query with one that expands pg_index.indcollation and joins pg_collation and pg_database. For the default collation it compares datcollversion with pg_database_collation_actual_version(), and for libc collations it compares collversion with pg_collation_actual_version(). The output lists indexes only, with stored and actual versions and the index definition.

**User-visible surface.**
- Changed example query in doc/src/sgml/ref/alter_collation.sgml; the output columns change from Collation and Object to database, table, index, collname, collprovider, versions and index_definition.

**What the author claims.**
- The current documented query does not report objects that use the database default collation.
- v1 still missed indexes that use the database default collation; v2 handles them.
- The v2 query is based on the query on the Locale_data_changes wiki page, changed to check version mismatch.

**Thread state.** The reviewer confirmed that the old query returns no rows and v2 finds an affected index under a simulated libc mismatch. The reviewer flagged that named ICU collations (collprovider 'i') and an ICU database default are not matched by either branch, and the author has not replied.

## 2. Prior attempts

### Revert per-index collation version tracking feature. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721)
- **Thread:** [https://www.postgresql.org/message-id/flat/CA+hUKGLhj5t1fcjqAu8iD9B3ixJtsTNqyCCD4V0aTO9kAKAjjA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKGLhj5t1fcjqAu8iD9B3ixJtsTNqyCCD4V0aTO9kAKAjjA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/CA+hUKGJ_hk3rU==g2FpAMChb_4i+TJacpjjqFsinY-tRM3FBmA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKGJ_hk3rU==g2FpAMChb_4i+TJacpjjqFsinY-tRM3FBmA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/3044030.1618261159@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/3044030.1618261159@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/HK0PR01MB22744393C474D503E16C8509F4709@HK0PR01MB2274.apcprd01.prod.exchangelabs.com](https://www.postgresql.org/message-id/flat/HK0PR01MB22744393C474D503E16C8509F4709@HK0PR01MB2274.apcprd01.prod.exchangelabs.com), [https://www.postgresql.org/message-id/flat/3564817.1618420687@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/3564817.1618420687@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/20210117215940.GE8560@telsasoft.com](https://www.postgresql.org/message-id/flat/20210117215940.GE8560@telsasoft.com)
- **Date:** decided 2021-05-07; started 2020-11-15
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Design problems were discovered in the handling of composite types and record types that would cause some relevant versions not to be recorded. Misgivings were also expressed about the use of the pg_depend catalog for this purpose. We're out of time for this release so we'll revert and try again. Commits reverted: 1bf946bd: Doc: Document known problem with Windows collation versions. cf002008: Remove no-longer-relevant test case. ef387bed: Fix bogus collation-version-recording logic. 0fb0a050: Hide internal error for pg_collation_actual_version(<bad OID>). ff942057: Suppress "warning: variable 'collcollate' set but not used". d50e3b1f: Fix assertion in collation version lookup. f24b1569: Rethink extraction of collation dependencies. 257836a7: Track collation versions for indexes. cd6f479e: Add pg_depend.refobjversion. 7d1297df: Remove pg_collation.collversion.
- **Reason (quote):** quote unavailable
- **What is different this time:** P1 tracked collation versions per index in pg_depend and was a feature, reverted for design problems. This proposal only changes a documentation query; P1 supports only the technical point that collations inside composite types are easy to miss.


## 3. Objections a reviewer is likely to raise

### 1. v2 stops reporting ICU collation mismatches the current query catches

fixable · level 2, rubric item 12 · design · `standards-compat-1`

An administrator with ICU-based indexes upgrades libicu and runs the documented query. It returns no rows, so they run ALTER COLLATION ... REFRESH VERSION or ALTER DATABASE ... REFRESH COLLATION VERSION without rebuilding. The indexes stay corrupt until queries return wrong results.

The current query compares collversion with pg_collation_actual_version() for every provider, so named ICU collations are covered today. The v2 WHERE clause accepts only collprovider 'c', and the default branch only datlocprovider 'c', which drops named ICU collations and ICU database defaults. Message #3 reported this and the thread has no reply. A change to the reference query for the default-collation gap should not reduce coverage for other providers.

Thread: [#2 ("Matheus Alcantara", 2026-04-13)](https://www.postgresql.org/message-id/DHSC1UP35NS0.VACTPNXNR7FS%40gmail.com), [#3 (Shubhra Jain, 2026-09-30)](https://www.postgresql.org/message-id/CAOh5eDWPy5%3DHQgYMAQLShUFDNmny%3DYbDRsoQT_o1K%2B94iTvK2Q%40mail.gmail.com)

**Question:** Can the query cover the default collation without dropping named ICU collations and ICU database defaults, which the current query reports?

Counter-check: held. The old query compares collversion for every provider, and v2 accepts only 'c', so named ICU collations lose coverage. One correction: the old query did not report ICU database defaults either, so that part is a missed case and not a regression.

### 2. Query narrows to indexes while the text still promises all dependents

fixable · level 2, rubric item 13 · design · `standards-compat-2`

A user has a check constraint, materialized view or partition bound that depends on a libc collation with a version change. The v2 query lists only indexes, so the user rebuilds those, refreshes the version, and the warning goes away. The other objects stay inconsistent and nothing reports them again.

The old query read pg_depend and listed every dependent object. v2 reads only pg_index.indcollation. The surrounding sentence is unchanged and still says the query finds "the objects that depend on them", and the server HINT says to rebuild "all objects affected". Message #2 explains the move to an index-only query only by the wiki source, not by a reason to drop the other objects.

Thread: [#1 ("Matheus Alcantara", 2026-04-09)](https://www.postgresql.org/message-id/DHOTJNEQ82FK.2VR8AIQNLO6UP%40gmail.com), [#2 ("Matheus Alcantara", 2026-04-13)](https://www.postgresql.org/message-id/DHSC1UP35NS0.VACTPNXNR7FS%40gmail.com)

**Question:** Why does the documented query drop non-index dependents, and if that is intended, how will the page tell readers which objects it no longer covers?

Counter-check: held. The hunk leaves the sentence "the objects that depend on them" unchanged while v2 reads only pg_index. The thread gives no reason for dropping the constraints and other dependents that the pg_depend query listed for named collations.

### 3. Index-only query misses composite-type and non-index dependencies

fixable · level 2, rubric item 7 · prior rejection, catalog changes · `prior-attempts-1`

After a glibc upgrade, an administrator runs the v2 query and sees no row for an index on a composite-typed column with a text field, or for a partitioned table keyed on text. The administrator refreshes the version without rebuilding. Wrong index scans or misrouted rows show up later.

v2 reads only pg_index.indcollation, which records one collation per index column. It does not record collations nested inside composite types, index predicates, constraints or partition keys. [P1] was reverted partly because composite and record types caused relevant versions not to be recorded, and the same gap applies to this query. The unchanged lead-in still promises "the objects that depend on them", but v2 returns indexes only.

**Precedents.**
- [Revert per-index collation version tracking feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721) (Reverted, 2021-05-07); thread: [#1](https://www.postgresql.org/message-id/flat/CA+hUKGLhj5t1fcjqAu8iD9B3ixJtsTNqyCCD4V0aTO9kAKAjjA@mail.gmail.com), [#2](https://www.postgresql.org/message-id/flat/CA+hUKGJ_hk3rU==g2FpAMChb_4i+TJacpjjqFsinY-tRM3FBmA@mail.gmail.com), [#3](https://www.postgresql.org/message-id/flat/3044030.1618261159@sss.pgh.pa.us), [#4](https://www.postgresql.org/message-id/flat/HK0PR01MB22744393C474D503E16C8509F4709@HK0PR01MB2274.apcprd01.prod.exchangelabs.com), [#5](https://www.postgresql.org/message-id/flat/3564817.1618420687@sss.pgh.pa.us), [#6](https://www.postgresql.org/message-id/flat/20210117215940.GE8560@telsasoft.com)

Thread: [#2 ("Matheus Alcantara", 2026-04-13)](https://www.postgresql.org/message-id/DHSC1UP35NS0.VACTPNXNR7FS%40gmail.com), [#3 (Shubhra Jain, 2026-09-30)](https://www.postgresql.org/message-id/CAOh5eDWPy5%3DHQgYMAQLShUFDNmny%3DYbDRsoQT_o1K%2B94iTvK2Q%40mail.gmail.com)

**Question:** Which object kinds does the v2 query knowingly skip, and will the page say so or point to the wiki for those cases?

Counter-check: held. An index on a composite-typed column has no collation in pg_index.indcollation, so v2 skips it, and partition keys are not in pg_index. P1 reverted a feature and did not reject a docs query, so it supports only the technical point about composite types.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The example query reads pg_index, pg_collation and pg_database, but the patch changes no catalog. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The current query misses default-collation indexes in released branches too, so a committer must decide whether to backpatch the docs fix. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The v2 query names provider codes 'c' and 'd', so it needs an edit whenever provider coverage changes. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The brief asks whether the longer query belongs in the reference page or on the Locale_data_changes wiki page. |

## 5. Questions the author must answer before anyone reviews code

1. Can the query cover the database default collation without dropping the named ICU collations that the current query reports?
2. Should the default-collation branch also handle a database whose datlocprovider is ICU, which neither the old query nor v2 reports?
3. Why does v2 list indexes only, and could it keep the pg_depend branch for named collations and add a default-collation branch?
4. Which object kinds does v2 knowingly skip, such as indexes on composite-typed columns and partition keys, and will the page say so?
5. Should the lead-in sentence change, since it still says the query finds "the objects that depend on them"?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows collation version tracking and the locale providers (libc, ICU) would have to check the query. No committer has posted in the thread.
- **Liability:** A reference-page query that administrators copy into upgrade checks. If it returns no rows for an affected object, the user may refresh the version without a rebuild, and the docs stay responsible for that until corrected.
- **Author follow-up:** The author posted v2 four days after v1 and moved the entry to the next CommitFest in September 2026. The author has not yet replied to the ICU point in message #3.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix helps any administrator who checks for indexes to rebuild after a C library upgrade, and the patch is one docs hunk. The ICU coverage and the index-only scope need an answer before a committer can take it.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Hardcoded provider filter drops ICU coverage the current query has. Refuted (not specific): Duplicate of standards-compat-1. The added point about future providers needing doc edits has no concrete scenario beyond the ICU gap.
- `maintenance-fit-2` Query narrowed from all dependent objects to indexes only. Refuted (not specific): Duplicate of standards-compat-2. Its suggestion to keep a pg_depend branch for named collations is a useful direction for the author.

### Review angles

- prior-attempts: 1 objection(s). One objection raised. No earlier attempt to change this documentation query was found in the precedent list.
- standards-compat: 2 objection(s). There is no change to syntax, GUCs, pg_dump or the catalog. The two objections are about the documented query, which users copy into their upgrade checks.
- replication-ondisk: 0 objection(s). The patch changes only an example query in doc/src/sgml/ref/alter_collation.sgml. It adds no on-disk format, WAL record, catalog column, protocol message or extension API, so nothing in this angle applies. Whether the query covers ICU collations and non-index objects belongs to other angles.
- security: 0 objection(s). This is a docs-only change to an example query that reads catalogs any user can already read, using functions that already exist. It adds no new privilege, default or exposure, so the security angle finds nothing to object to.
- maintenance-fit: 2 objection(s). A docs-only change to an example query adds no hot-path cost and no API, so the maintenance objections are limited to coverage and to future provider changes.
- already-exists: 0 objection(s). No existing feature lists the indexes that use the database default collation after a version change. The connect-time warning and ALTER DATABASE ... REFRESH COLLATION VERSION report the mismatch but list no objects, and the current documented query misses them, as message #3 confirmed. The Locale_data_changes wiki query is the source of v2, not a replacement for correct reference docs. Gaps for ICU collations and non-index objects belong to other angles.
- demand: 0 objection(s). The proposal fixes a documented query that misses indexes on the database default collation, and the reviewer reproduced the gap in message #3. This hits anyone who upgrades the OS C library, so demand is real. The narrower scope of v2 (no ICU branch, indexes only) is a design concern for other angles.

### Precedents retrieved

- P1: [Revert per-index collation version tracking feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721) (revert, Reverted, 2021-05-07) - used in the review
- P2: [Revert "pg_collation_actual_version() -> pg_collation_current_version()."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8556267b2b1b8e1c26037c4c25cf390ee5afb5d9) (revert, Reverted, 2021-02-26) - judged not relevant
- P3: [Track collation and ctype versions separately](https://commitfest.postgresql.org/patch/5141/) (cf, Withdrawn, 2025-02-07) - judged not relevant; reason summarised from its thread
- P4: [Revert "Use libc version as a collation version on glibc systems."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50518ec296aea3af3e00c43c2ccef74c96cb5762) (revert, Reverted, 2019-10-09) - judged not relevant
- P5: [Preserve versions of initdb-created collations in pg_upgrade](https://commitfest.postgresql.org/patch/2328/) (cf, Returned with feedback, 2020-01-24) - judged not relevant; reason summarised from its thread
- P6: [Collation code refactoring](https://commitfest.postgresql.org/patch/4058/) (cf, Withdrawn, 2023-01-11) - judged not relevant; reason summarised from its thread
- P7: [ICU as default collation provider](https://commitfest.postgresql.org/patch/1543/) (cf, Returned with feedback, 2019-02-04) - judged not relevant; reason summarised from its thread
- P8: [Multi-version ICU](https://commitfest.postgresql.org/patch/3956/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [pgbench - document compatibility with older server versions](https://commitfest.postgresql.org/patch/1744/) (cf, Rejected, 2018-09-07) - judged not relevant; reason summarised from its thread
- P10: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant; reason summarised from its thread
- P11: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P12: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant; reason summarised from its thread
- P13: [CREATE COLLATION default provider](https://commitfest.postgresql.org/patch/4366/) (cf, Withdrawn, 2024-01-21) - judged not relevant
- P14: [initdb: change default --locale-provider back to libc.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2535c74b1a6190cc42e13f6b6b55d94bff4b7dd6) (revert, Partially reverted, 2023-06-21) - judged not relevant
- P15: [Allow shared buffer cache contents to affect index cost computations](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P16: [Introduce the ability to enable/disable indexes using ALTER INDEX](https://commitfest.postgresql.org/patch/5274/) (cf, Withdrawn, 2025-09-08) - judged not relevant
- P17: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P18: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P19: [Parallel CREATE INDEX for GiST indexes](https://commitfest.postgresql.org/patch/5026/) (cf, Returned with feedback, 2025-02-06) - judged not relevant
- P20: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P21: [Revert previous change --- turns out the underlying problem affects more cases than I thought, so ExecTypeFromTL() will have to be fixed anyway.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f261461c1d0bb7ab4d4e2442cea313008c06f12) (revert, Reverted, 2003-05-11) - judged not relevant
- P22: [Output affected rows in EXPLAIN](https://commitfest.postgresql.org/patch/4551/) (cf, Rejected, 2023-11-22) - judged not relevant
- P23: [document that psql's FETCH_COUNT does not work with combined (\;) queries](https://commitfest.postgresql.org/patch/2219/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P24: [On-disk bitmap indexes](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 124 s; cost reported by the CLI: $1.50
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 68 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:59:40.381Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
