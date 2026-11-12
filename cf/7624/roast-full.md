# Design pre-review: Match jsonpath variable names exactly when checking mutability

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7624](https://commitfest.postgresql.org/patch/7624/) (Needs review); authors: Tomás Senart (tsenart)
- Thread: [\[PATCH v1\] Match jsonpath variable names exactly when checking mutability](https://www.postgresql.org/message-id/flat/CAEEBa3VS8xnenaeLB_JUu2Nku6w6CLpaRvjf3BbXayfOt%2BeSbg%40mail.gmail.com), started 2026-11-10
- Messages read: 1 of 1
- Patch prose read: v1-0001-Match-jsonpath-variable-names-exactly-when-checki.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection at levels 1a or 1b survived. Message #1 states a wrong-results bug with a reproduction, and the fix makes jspIsMutableWalker() match names as GetJsonPathVar() does. The one surviving objection, standards-compat-1, is fixable: a back-patch to 17 makes some existing index definitions fail at restore or pg_upgrade, and the proposal gives users no way to find them first. A committer who knows the SQL/JSON code should judge the back-patch.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the review cannot confirm that the new comparison equals the one in GetJsonPathVar(). It also cannot confirm that no other prefix comparison remains or that the back-patch applies to 17.

## The proposal, restated

**Problem.** jspIsMutableWalker() compares a jsonpath variable with PASSING argument names only over the variable name's length, so a PASSING argument "aa" can set the datatype of $a. Since commit 0f271e8e8d9, GetJsonPathVar() needs an exact match at execution, so the two can disagree. create index can then accept an expression that depends on TimeZone and return wrong results, or reject an immutable one.

**Proposed design.** jspIsMutableWalker() also requires equal name lengths, as GetJsonPathVar() does. The patch adds two regression tests to sqljson_queryfuncs: one where the PASSING name extends the variable name, and one where it is a prefix of it. The author asks for a back-patch to 17.

**User-visible surface.**
- create index on a JSON_EXISTS or other SQL/JSON expression with PASSING now fails with "functions in index expression must be marked IMMUTABLE" in the reported case.
- Some index expressions that were wrongly rejected as mutable are now accepted.
- Restoring a dump with an index that was wrongly accepted now fails at create index.

**What the author claims.**
- On master, a wrongly accepted index gives 1 row with an index scan and 0 rows with a seq scan after a TimeZone change.
- The cost is one strlen() for each PASSING argument the check examines, and evaluation does not change.
- make check and make installcheck pass.
- The code came with SQL/JSON in 17, so the fix should be back-patched there.
- REINDEX does not fix affected indexes, so users must drop them or replace them with truly immutable expressions.

**Thread state.** Only the first message is visible. No one has replied, and the CommitFest entry is in Needs review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-10; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-patch makes existing dumps and pg_upgrade fail without a way to find them

fixable · level 2, rubric item 13 · backpatchability, pg_dump / restore round-trip, on-disk format / pg_upgrade · `standards-compat-1`

A user on 17.x has a JSON_EXISTS index with PASSING names like aa and a. After a minor update, pg_restore of an older dump fails at create index. A later pg_upgrade to a new major version fails at the same index. The user finds out only when the restore or upgrade stops.

Message #1 and the commit message say restoring such definitions "will now fail at CREATE INDEX" and that REINDEX does not help. pg_upgrade restores index definitions through pg_dump, so it fails in the same way. Neither text offers a catalog query or release-note wording that lets a user find affected indexes before the update. A committer would likely accept the wrong-results fix but ask for this guidance before a minor-release back-patch.

Thread: [#1 (Tomás Senart, 2026-11-10)](https://www.postgresql.org/message-id/CAEEBa3VS8xnenaeLB_JUu2Nku6w6CLpaRvjf3BbXayfOt%2BeSbg%40mail.gmail.com)

**Question:** What query or release-note text will let 17.x users find affected index expressions before a minor update or a pg_upgrade?

Counter-check: held. Message #1 warns that restoring such a dump fails, but gives no query or release-note text for finding affected indexes. pg_upgrade restores the schema through the same create index, so the gap is real and a new patch version can close it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade restores index definitions with create index, which now rejects indexes the old check accepted. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Message #1 says restoring a wrongly accepted index definition from a dump now fails at create index. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The author asks for a back-patch to 17, which changes what create index accepts in a minor release. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What query or release-note text will let 17.x users find affected index expressions before a minor update or a pg_upgrade?
2. Does any code besides jspIsMutableWalker() still compare PASSING names by prefix?
3. Does the patch apply to the 17 and 18 branches as it is, or do those branches need a different version?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the SQL/JSON query functions and jsonpath code would have to own it; no one has replied to message #1.
- **Liability:** The owner signs for a behaviour change in minor releases of 17 and later: some create index statements that worked now fail, including during restore and pg_upgrade. The owner also signs for the release-note guidance.
- **Author follow-up:** The author names himself as contact and invites rework of the patch; the thread has no later message that shows follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug gives wrong results from an index, but only for SQL/JSON index expressions where one PASSING name is a prefix of another. The patch is 5.0 KB, touches one source file and one regression test, and adds no user-visible surface.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` pg_upgrade from 17 can fail on indexes the old check accepted. Refuted (not specific): Duplicate of standards-compat-1, which covers both the minor-release restore and the pg_upgrade failure and asks for the same guidance.
- `maintenance-fit-1` Two separate name lookups can drift apart again. Refuted (no failure scenario): The scenario needs a future patch that changes one lookup and misses the other; this patch does not add the duplication, it removes the present divergence. The two lookups walk different structures at plan time and at execution, and the new tests cover both mismatch directions in the walker.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no proposal with the same or a similar idea, and the thread does not point to an earlier discussion. The patch is a bug fix that makes jspIsMutableWalker() match GetJsonPathVar() exactly, so no earlier rejection reason applies.
- standards-compat: 1 objection(s). The fix aligns the mutability check with the exact match GetJsonPathVar() already uses at execution. It adds no syntax, GUC, or catalog change, so the standards side is fine. The only point is how the back-patch handles existing indexes.
- replication-ondisk: 1 objection(s). The patch has no on-disk format, WAL, catalog, wire protocol or ABI change. The only persistent effect is on indexes that already exist with definitions the fixed check would reject, and the objection above covers that.
- security: 0 objection(s). The patch has no privilege or exposure aspect. It makes the immutability check at create index stricter and grants nothing, and a wrongly accepted index only gives wrong results on a table the creator already controls.
- maintenance-fit: 1 objection(s). This is a narrow wrong-results bugfix with no hot-path cost, no new API and no extension fit question. The single objection is about preventing the same drift in the future, and a new version could address it.
- already-exists: 0 objection(s). This is a correctness bugfix in jspIsMutableWalker(). No existing feature, extension or setting can stop create index from accepting a TimeZone-dependent expression, so nothing from the already-exists angle applies. The fix extends the existing check and adds no second mechanism.
- demand: 0 objection(s). This is a correctness bugfix with a reproduction in message #1: an index can return 1 row while a seq scan returns 0 after a TimeZone change. Wrong results from an index need no demand evidence, and the cost of one strlen() falls only on PASSING checks.

### Precedents retrieved

- P1: [Revert "Add gitignore entries for jsonpath_gram.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7b084b38310cfe9c8b58cc615a81df625c771f5d) (revert, Reverted, 2019-03-22) - judged not relevant
- P2: [Add jsonpath string method $.replace(from, to)](https://commitfest.postgresql.org/patch/5261/) (cf, Withdrawn, 2024-09-25) - judged not relevant; reason summarised from its thread
- P3: [Revert "Reorganise jsonpath operators and methods"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d) (revert, Reverted, 2024-01-03) - judged not relevant
- P4: [Add IMMUTABLE column attribute](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Data_Types) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P5: [Make selectivity example match wording](https://commitfest.postgresql.org/patch/3695/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P6: [Add checks on search_path for IMMUTABLE and SECURITY DEFINER functions](https://commitfest.postgresql.org/patch/3684/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P7: [Permute underscore separated components of columns before fuzzy matching](https://commitfest.postgresql.org/patch/4282/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P8: [SQL/JSON: JSON_TABLE](https://commitfest.postgresql.org/patch/1473/) (cf, Returned with feedback, 2020-08-03) - judged not relevant; reason summarised from its thread
- P9: [\[PATCH v1\] psql: Do not complete HEADER MATCH for COPY TO](https://commitfest.postgresql.org/patch/7585/) (cf, Withdrawn, 2026-11-04) - judged not relevant; reason summarised from its thread
- P10: [Support multiple password-based authentication tokens concurrently](https://commitfest.postgresql.org/patch/3748/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P11: [pg_hba_lookup function to get all matching pg_hba.conf entries](https://commitfest.postgresql.org/patch/356/) (cf, Returned with feedback, 2016-04-08) - judged not relevant; reason summarised from its thread
- P12: [(SQL/PGQ) cache lookup failed for label](https://commitfest.postgresql.org/patch/6776/) (cf, Withdrawn, 2026-05-15) - judged not relevant
- P13: [pg_regress.c: Fix "make check" on Mac OS X: Pass DYLD_LIBRARY_PATH](https://commitfest.postgresql.org/patch/4340/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P14: [Two pass check for deadlock](https://commitfest.postgresql.org/patch/1602/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P15: [json_table function PLAN clause](https://commitfest.postgresql.org/patch/6332/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P16: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P17: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P18: [Allow direct lookups of SpecialJoinInfo by ojrelid](https://commitfest.postgresql.org/patch/4313/) (cf, Withdrawn, 2023-09-06) - judged not relevant
- P19: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant
- P20: [SQL/JSON json_table() function PLAN clause](https://commitfest.postgresql.org/patch/6590/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P21: [Reorganize collation lookup time and place](https://commitfest.postgresql.org/patch/1956/) (cf, Returned with feedback, 2019-02-01) - judged not relevant
- P22: [Issues with 2PC at recovery: CLOG lookups and GlobalTransactionData](https://commitfest.postgresql.org/patch/5532/) (cf, Returned with feedback, 2025-07-07) - judged not relevant
- P23: [TimestampTz->Text->TimestampTz casting fails with DateStyle 'Postgres'](https://commitfest.postgresql.org/patch/5371/) (cf, Returned with feedback, 2025-01-07) - judged not relevant
- P24: [Add new wait event to XactLockTableWait](https://commitfest.postgresql.org/patch/5804/) (cf, Withdrawn, 2026-03-31) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 99 s; cost reported by the CLI: $0.78
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 59 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:09:36.675Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
