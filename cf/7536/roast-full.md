# Design pre-review: JSON_SERIALIZE for JSONB returns parts of the internal JSONB representation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7536](https://commitfest.postgresql.org/patch/7536/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [JSON_SERIALIZE for JSONB returns parts of the internal JSONB representation](https://www.postgresql.org/message-id/flat/03583FE7-A5EE-407C-907D-67865F3F8BFC%40gmail.com), started 2026-03-03
- Messages read: 3 of 3
- Patch prose read: v1-0002-Add-tests-for-JSON_SERIALIZE-with-a-jsonb-argumen.patch, v1-0001-Fix-JSON_SERIALIZE-with-a-jsonb-argument.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is reproduced on master, the docs allow a jsonb argument, and no objection to fixing it survived. The two surviving objections, standards-compat-1 and replication-ondisk-1, are fixable and concern expressions already stored when the parser-only fix is backpatched through 17. A committer who knows the SQL/JSON parser code has to choose between the parser fix and an executor-side fix for back branches. No one but the author has replied yet.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the review could not confirm that the new coercion is right for every RETURNING type. It also could not check whether other callers of makeJsonConstructorExpr() change behaviour.

## The proposal, restated

**Problem.** JSON_SERIALIZE() on a jsonb value returns a few bytes of the internal jsonb header, such as \x03 for '[1,2,4]'::jsonb, instead of the JSON text. The reporter saw it on 18.3; the patch author reproduced it on master. The output can also be text that is not valid in the server encoding.

**Proposed design.** makeJsonConstructorExpr() built the output coercion for a json input because it took the input type from the RETURNING format, which is always JSON. The patch builds the coercion from the type of the argument instead, so json_out() no longer runs on jsonb data. The fix is in the parser, so stored expressions keep the old behaviour until recreated.

**User-visible surface.**
- JSON_SERIALIZE() with a jsonb argument returns the JSON text instead of header bytes.
- Views and other stored expressions that call JSON_SERIALIZE() on jsonb stay wrong until recreated.

**What the author claims.**
- The docs allow any JSON type as the JSON_SERIALIZE() argument, so jsonb should work.
- Converting in the executor would cover old views but adds a step and keeps the wrong coercion in the tree.
- Casting jsonb to json in transformJsonSerializeExpr() works but converts twice.
- The fix should be backpatched through 17.

**Thread state.** The reporter posted in March; the patch author replied in September with v1 and rebased to v2 in November. No committer or other reviewer has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-03-03; 23 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Parser-only fix leaves existing views wrong after a minor update

fixable · level 2, rubric item 6 · backpatchability, pg_dump / restore round-trip · `standards-compat-1`

A user on 17 or 18 has a view that calls JSON_SERIALIZE() on a jsonb column. After the minor update the view still returns header bytes. A copy restored from pg_dump re-parses the view and returns JSON text, so the two servers give different results for the same view.

Message #2 and the 0001 commit message say stored expressions keep the old coercion until recreated, and the patch targets back branches through 17. A minor release is expected to fix bugs without user action, but a minor update does not re-parse stored rules. Message #2 rejects the executor-side fix because it adds a step and keeps the wrong coercion in the tree, but it is the only option that also fixes existing views. The proposal does not give release-note text or a query that finds affected views.

Thread: [#2 (shihao zhong, 2026-09-19)](https://www.postgresql.org/message-id/CAGRkXqQXwh_gJmQE3PZj0v13omMPgFGBbb8TJoBFH8ECsAFCUg%40mail.gmail.com)

**Question:** How will users on 17 and 18 find and fix existing views after the minor update, or why is an executor-side fix for old trees not worth its cost?

Counter-check: held. Message #2 names the limitation and two alternatives, but it gives no way to find affected views and does not weigh the executor-side fix for back branches. A release note or a catalog query can close this, so fixable is the right severity.

### 2. Expression indexes and stored generated columns keep old values after upgrade

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, backpatchability, catalog changes · `replication-ondisk-1`

A user has an expression index or a stored generated column on JSON_SERIALIZE(jsonb_col). pg_upgrade or dump and restore re-parses the expression with the fixed parser, but pg_upgrade copies the index files and table data unchanged. Index lookups with the new expression then miss rows, and generated column values no longer match their expression, with no error.

Message #2 and the commit message say the fix lives in the parser and that stored expressions must be recreated. Re-parsing happens silently whenever an expression is deparsed and restored, so the stored expression and the stored data can disagree after pg_upgrade. The proposal does not say whether JSON_SERIALIZE() on jsonb is accepted in index and generated column expressions. It gives no reindex or rewrite instruction.

Thread: [#2 (shihao zhong, 2026-09-19)](https://www.postgresql.org/message-id/CAGRkXqQXwh_gJmQE3PZj0v13omMPgFGBbb8TJoBFH8ECsAFCUg%40mail.gmail.com)

**Question:** Can JSON_SERIALIZE() on jsonb appear in an index or stored generated column, and if so, what must users run after pg_upgrade or a minor update?

Counter-check: held. The material does not rule out JSON_SERIALIZE() on jsonb in an index or stored generated column, and pg_upgrade re-parses expressions while copying data files unchanged. A reindex or rewrite note would answer it, so it stays fixable.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | An expression index or stored generated column on JSON_SERIALIZE(jsonb) could hold old values after pg_upgrade re-parses the expression. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | see objection replication-ondisk-1: Expression indexes and stored generated columns keep old values after upgrade |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | A view restored from pg_dump is re-parsed and returns JSON text, while the original view still returns header bytes. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message asks for a backpatch through 17, and stored expressions keep the old coercion after a minor update. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How do users on 17 and 18 find the views, rules and defaults that call JSON_SERIALIZE() on jsonb and must be recreated after the minor update?
2. For back branches, why is the parser-only fix better than an executor-side conversion that also corrects stored expressions?
3. Can JSON_SERIALIZE() on jsonb appear in an expression index or a stored generated column, and if so, what must users run after pg_upgrade?
4. Do other SQL/JSON constructors that go through makeJsonConstructorExpr() build the same wrong coercion for a jsonb input?
5. What release-note text do you propose for the minor releases?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the SQL/JSON constructor code in the parser and executor, and the rules for backpatching fixes that affect stored expressions.
- **Liability:** The owner signs for a behaviour change in 17 and 18 minor releases, for the release-note text, and for any reports about stored views, indexes or generated columns that stay wrong.
- **Author follow-up:** The author posted v1 in message #2 and a rebased v2 in message #3 about two months later, so the author is still following the patch.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** A documented function returns jsonb header bytes, sometimes invalid in the server encoding, and the fix touches one parser file and adds tests. Only callers of JSON_SERIALIZE() with jsonb are affected, and the open point is the back-branch handling of stored expressions.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-2` Backpatched parser fix leaves existing views wrong with no way to find them. Refuted (not specific): Duplicate of standards-compat-1. It also understates message #2, which rejects the executor-side fix for two reasons, the added step and the wrong coercion staying in the tree.
- `maintenance-fit-1` Parser-only fix leaves existing views returning header bytes after minor upgrade. Refuted (not specific): Duplicate of standards-compat-1, which states the same stored-view scenario and adds the pg_dump divergence case.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P6] reverted the whole SQL/JSON feature set in 2022, and its record gives only the list of reverted commits, not a reason that applies to this parser fix for JSON_SERIALIZE() with a jsonb argument. The thread does not point to any earlier discussion of this bug or fix.
- standards-compat: 1 objection(s). The fix matches the documented contract that any JSON type is accepted. The current output is header bytes, so no user should rely on it.
- replication-ondisk: 2 objection(s). The fix adds no new WAL records, catalogs, wire messages or on-disk formats. The objections above are only about expressions already stored in catalogs and data built from them.
- security: 0 objection(s). The patch adds no privilege, role or default, and gives no user access to anything they could not reach before. The bug only exposed header bytes of the caller's own jsonb value, and the fix removes that. One point a committer may still check: after a backpatched parser-only fix, views that already exist can keep returning text that is not valid in the server encoding. That is a correctness and encoding issue, not a privilege boundary, and other angles cover it.
- maintenance-fit: 1 objection(s). Beyond the back-branch view issue, the fix is small, is confined to one parser file, and adds no hot-path cost, API or new surface to maintain.
- already-exists: 0 objection(s). The proposal fixes a bug, so no existing feature or extension meets the need. Casting the argument first (`json_serialize(x::json)`) or using `x::text` avoids the wrong output, but users who hit the bug cannot know to do that, and the docs say any JSON type is accepted. The precedent [P6] is the 2022 SQL/JSON revert and gives no reason that applies to this fix.
- demand: 0 objection(s). This is a bug fix for a documented function that returns jsonb header bytes, possibly invalid in the server encoding, instead of JSON text; a user reported it on 18.3. Correct output from a documented function needs no further proof of demand, and the cost falls only on callers of JSON_SERIALIZE() with jsonb.

### Precedents retrieved

- P1: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - judged not relevant; reason summarised from its thread
- P2: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P3: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant; reason summarised from its thread
- P4: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P5: [jsonb array-style subscription](https://commitfest.postgresql.org/patch/485/) (cf, Returned with feedback, 2016-03-21) - judged not relevant; reason summarised from its thread
- P6: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - used in the review
- P7: [SERIALIZABLE on standby servers](https://commitfest.postgresql.org/patch/1799/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P8: [WIP: Pgbench Errors and serialization/deadlock retries](https://commitfest.postgresql.org/patch/1419/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P9: [WIP: Pgbench Errors and serialization/deadlock retries](https://commitfest.postgresql.org/patch/1645/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P10: [SQL/JSON: JSON_TABLE](https://commitfest.postgresql.org/patch/1473/) (cf, Returned with feedback, 2020-08-03) - judged not relevant
- P11: [WIP: Pgbench Serialization and deadlock errors](https://commitfest.postgresql.org/patch/1170/) (cf, Returned with feedback, 2017-09-28) - judged not relevant
- P12: [bytea PL/Perl transform](https://commitfest.postgresql.org/patch/4252/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P13: [Refactor bytea_sortsupport()](https://commitfest.postgresql.org/patch/5288/) (cf, Returned with feedback, 2025-02-13) - judged not relevant
- P14: [BUG: cannot read big bytea (~600MB)](https://commitfest.postgresql.org/patch/1521/) (cf, Rejected, 2018-03-01) - judged not relevant
- P15: [format_datum debugging function](https://commitfest.postgresql.org/patch/5165/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P16: [Reducing serialized size of Nodes from nodeToString output](https://commitfest.postgresql.org/patch/4685/) (cf, Returned with feedback, 2024-04-04) - judged not relevant
- P17: [psql export import bytea data](https://commitfest.postgresql.org/patch/913/) (cf, Rejected, 2017-09-07) - judged not relevant
- P18: [Free list same_input_transnos in preprocess_aggref](https://commitfest.postgresql.org/patch/3920/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P19: [Revert "Convert *GetDatum() and DatumGet*() macros to inline functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e8d78581bb864369035c736a82ecfb57482a1a09) (revert, Reverted, 2022-09-12) - judged not relevant
- P20: [psql JSON output format](https://commitfest.postgresql.org/patch/4707/) (cf, Returned with feedback, 2024-05-15) - judged not relevant
- P21: [invalid bit/varbit input causes the log file to be unreadable](https://commitfest.postgresql.org/patch/2618/) (cf, Withdrawn, 2020-06-30) - judged not relevant
- P22: [Fix for PL/Python slow input arrays traversal issue](https://commitfest.postgresql.org/patch/687/) (cf, Rejected, 2016-09-22) - judged not relevant
- P23: [Revert the default toast compression strategy to the former behavior where there's no limit on the size of datum we'll try to compress.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=229bffedbc52f12aadb95ca2ee24d5bef36b7c7d) (revert, Reverted, 2009-01-06) - judged not relevant

### Warnings

- precedent search failed for "SQL/JSON constructor jsonb output wrong": Spread syntax requires ...iterable[Symbol.iterator] to be a function

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 102 s; cost reported by the CLI: $1.01
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 23 kept
- Tone check: 64 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:06:39.440Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
