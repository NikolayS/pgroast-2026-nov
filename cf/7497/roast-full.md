# Design pre-review: pg_stat_statements: fix SQL/JSON behavior keywords and JSON_TABLE query jumbling

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7497](https://commitfest.postgresql.org/patch/7497/) (Needs review); authors: Ilya Staroverov (ilyastar)
- Thread: [Re: pg_stat_statements replaces SQL/JSON ON EMPTY / ON ERROR keywords with parameters](https://www.postgresql.org/message-id/flat/457969A9-5D24-422F-B09E-F388053DE028%40localhost.localdomain), started 2026-10-19
- Messages read: 9 of 9
- Patch prose read: v6-0001-pg_stat_statements-Add-tests-for-JSON_TABLE-colum.patch, v6-0002-Include-JSON_TABLE-column-definitions-in-query-ju.patch, v3-0001-Don-t-give-the-SQL-JSON-behavior-constants-a-loca.patch, v3-0002-Include-JSON_TABLE-column-definitions-in-query-ju.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with examples, was reported on pgsql-bugs, and the remaining patch reuses the existing query_jumble_ignore node attributes. Michael Paquier committed the tests and the behavior keyword fix and limited the query ID change to master. Confidence is medium because only 3 of 9 messages are visible and no reply to v6 is shown.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it could not be checked that every jumbled field of the JSON_TABLE plan nodes affects the result, or that the new NESTED PATH Const locations match the literals. The discussion in messages #4 to #9 was not visible.

## The proposal, restated

**Problem.** pg_stat_statements merges JSON_TABLE queries that differ in column types, ON EMPTY or ON ERROR behaviors, NESTED PATH structure or PLAN join types, because TableFunc.colvalexprs and TableFunc.plan are not jumbled. It also replaced behavior keywords such as TRUE or EMPTY with parameter symbols, so separate entries showed the same text. Users who monitor SQL/JSON queries get wrong or confusing entries.

**Proposed design.** The behavior keyword fix leaves the behavior Const without a location; Michael Paquier committed it. The remaining patch includes TableFunc.colvalexprs and TableFunc.plan in query jumbling and gives NESTED PATH Consts their literal location so they show as parameter symbols. JsonExpr.column_name and JsonTablePath.name are marked query_jumble_ignore, so column names and path names stay out of the query ID.

**User-visible surface.**
- Query IDs of JSON_TABLE queries change on master.
- Query IDs of other SQL/JSON query functions change, since their NULL column_name is no longer jumbled.
- pg_stat_statements normalized text shows NESTED PATH and column path literals as parameter symbols.
- pg_stat_statements normalized text keeps ON EMPTY and ON ERROR keywords (committed part).

**What the author claims.**
- Everything left in the jumbled plan nodes affects the query result.
- Equivalent PLAN spellings, such as no PLAN and PLAN (p0 OUTER p1), still share a query ID.
- Queries that differ only in column names, path names, or PATH after NESTED stay merged.
- Ignoring column names matches how column definition lists of functions in FROM are handled.
- The series applies to master and REL_19_STABLE and passes tests on both.

**Thread state.** Michael Paquier committed the new sqljson.sql tests and the behavior keyword fix, and ruled out any back-patch. v6 adopts Sami Imseih's request to ignore column names; no reply to v6 is visible.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-19; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes jumble attributes of TableFunc, JsonExpr and JsonTablePath in primnodes.h, so query IDs seen by extensions change. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Query IDs of JSON_TABLE and other SQL/JSON query functions change, so Michael Paquier ruled out a back-patch in message #2. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Future fields added to JsonTablePathScan or JsonTableSiblingJoin are jumbled by default and need a decision each time. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do reviewers accept jumbling TableFunc.plan with the fields listed in message #1, which Sami Imseih earlier found hard to get comfortable with?
2. Is it accepted that COLUMNS (a int) and COLUMNS (b int) now share one entry, although they read different keys?
3. Should "a jsonb PATH" and "a jsonb FORMAT JSON PATH" keep separate query IDs, and which reviewer has agreed to that?
4. Were tests for WITHOUT UNIQUE and JSON_ARRAYAGG ORDER BY, which Michael Paquier listed, added to sqljson.sql?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier, who knows query jumbling and pg_stat_statements, reviewed the series and committed the tests and the behavior keyword fix.
- **Liability:** The owner signs for the query ID change of all SQL/JSON query functions in the next major version, and for the jumble attributes on the JSON_TABLE plan nodes from then on.
- **Author follow-up:** The author answered each review within days and posted v3 through v6, adopting the reviewers' requests each time.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Only users who monitor JSON_TABLE queries with pg_stat_statements benefit, but their entries are wrong today. The remaining patch is about 20 KiB, mostly expected test output, touches two source files, and a committer who knows the area is engaged.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P7] dealt with FETCH count normalization, and its reason (both forms jumble to the same node, no monitoring gain) does not apply, because the JSON_TABLE fields jumbled here change query results. [P2] proposed hashing relation names instead of OIDs, a different idea. The thread points to no earlier attempt at JSON_TABLE jumbling.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch adds no syntax, keyword, GUC, catalog object or psql change. Query ID changes are limited to master and stated in the commit message, and Michael Paquier ruled out a back-patch in messages #2 and #4. Ignoring column names follows the column definition lists of functions in FROM (message #7). The author first argued that columns without PATH read different keys (message #1), then accepted ignoring the names in message #9, so the point is settled in the thread. The FORMAT JSON split matches existing JSON_QUERY behavior. [P2] and [P7] do not apply: neither recorded reason concerns jumbling fields that affect query results.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state. It changes query jumbling and Const locations, which feed only in-memory query IDs and pg_stat_statements text; no catalog, WAL, on-disk or wire change, and master-only, so query ID consistency between a primary and a standby on the same major version holds.
- security: 0 objection(s). The proposal changes only which JSON_TABLE parse-tree fields go into the query ID and which Consts get a text location. It adds no privilege, role, function or setting, and it does not change who can read pg_stat_statements or what entries show beyond normalized constants. Literals still become parameter symbols, so query texts expose no new values.
- maintenance-fit: 0 objection(s). No maintenance or fit objection holds. The change reuses the existing node-attribute jumbling, touches no hot path for non-users, and adds jumbling cost only to JSON_TABLE queries. Michael Paquier, who knows this area, committed the tests and the keyword fix, and the query ID change is limited to master, which Michael Paquier required in messages #2 and #4. Open design questions (TableFunc.plan, the FORMAT JSON split) belong to other angles.
- already-exists: 0 objection(s). The proposal fixes query jumbling inside pg_stat_statements' own query ID computation. No existing feature, setting or extension can make JSON_TABLE column definitions part of the query ID, and compute_query_id = off with an external jumbler would replace the mechanism, not reuse it. The patch extends the existing node-attribute jumbling, so it adds no second mechanism.
- demand: 0 objection(s). No demand objection. pg_stat_statements merges JSON_TABLE queries that differ in column types, behaviors or PLAN join types, so entries are wrong; this was reported on pgsql-bugs and two reviewers and a committer engaged. Costs fall on master only, where query IDs already change across major versions, and neither [P7] nor [P2] has a recorded reason that applies.

### Precedents retrieved

- P1: [Query Jumbling for CALL and SET utility statements](https://commitfest.postgresql.org/patch/3864/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P2: [Fix pg_stat_statements query jumbling to use relation name instead of relation oid](https://commitfest.postgresql.org/patch/365/) (cf, Rejected, 2015-11-15) - used in the review; reason summarised from its thread
- P3: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant; reason summarised from its thread
- P4: [Sampling of queries in pg_stat_statements](https://commitfest.postgresql.org/patch/5389/) (cf, Withdrawn, 2024-11-19) - judged not relevant; reason summarised from its thread
- P5: [Extracting only the columns needed for a query](https://commitfest.postgresql.org/patch/2331/) (cf, Withdrawn, 2020-07-01) - judged not relevant; reason summarised from its thread
- P6: [Revert current_query() change to use debug_query_string again; add comment.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1c175cd76a9db7d6114139a28f838697e88306ba) (revert, Reverted, 2009-01-08) - judged not relevant
- P7: [Fix pg_stat_statements display of normalized FETCH counts](https://commitfest.postgresql.org/patch/6756/) (cf, Withdrawn, 2026-05-12) - used in the review; reason summarised from its thread
- P8: [Optionally record Plan IDs to track plan changes for a query](https://commitfest.postgresql.org/patch/5486/) (cf, Returned with feedback, 2025-03-24) - judged not relevant; reason summarised from its thread
- P9: [libpq compression](https://commitfest.postgresql.org/patch/1635/) (cf, Returned with feedback, 2019-04-09) - judged not relevant; reason summarised from its thread
- P10: [Cancel problems of query to pg_stat_statements](https://commitfest.postgresql.org/patch/5825/) (cf, Rejected, 2026-03-25) - judged not relevant
- P11: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P12: [\[WIP\] Temporal query processing with range types - Temporal Normalization](https://commitfest.postgresql.org/patch/2045/) (cf, Withdrawn, 2020-01-30) - judged not relevant
- P13: [Revert cascading of JSON_TABLE's ON ERROR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=af6fad879fbf9b542cfde05c52695c9dffa6bb47) (revert, Reverted, 2026-07-14) - judged not relevant
- P14: [pg_stat_statements with plans (v02)](https://commitfest.postgresql.org/patch/1470/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P15: [Sampling in pg_stat_statements](https://commitfest.postgresql.org/patch/5390/) (cf, Returned with feedback, 2025-04-09) - judged not relevant
- P16: [Autoprepare: implicitly replace literals with parameters and store generalized plan](https://commitfest.postgresql.org/patch/1747/) (cf, Returned with feedback, 2020-07-01) - judged not relevant
- P17: [Hstore error code cleanup](https://commitfest.postgresql.org/patch/626/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P18: [pg_stat_statements and extended query protocol](https://commitfest.postgresql.org/patch/4146/) (cf, Withdrawn, 2023-01-30) - judged not relevant
- P19: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P20: [Revert some careless search-and-replace: "ADD" in comment text should not be replaced with "ADD_P".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=12119188fe04cd20a6f1317f62886b685ae71c59) (revert, Reverted, 2005-12-29) - judged not relevant
- P21: [Logging plan of the currently running query](https://commitfest.postgresql.org/patch/3142/) (cf, Withdrawn, 2023-06-06) - judged not relevant
- P22: [pg_stat_statements: Add gc_count and query_file_size to pgss_info](https://commitfest.postgresql.org/patch/6554/) (cf, Returned with feedback, 2026-03-25) - judged not relevant
- P23: [pg_stat_statements with plans](https://commitfest.postgresql.org/patch/969/) (cf, Returned with feedback, 2017-03-04) - judged not relevant
- P24: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 86 s; cost reported by the CLI: $1.41
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 46 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:01:14.635Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
