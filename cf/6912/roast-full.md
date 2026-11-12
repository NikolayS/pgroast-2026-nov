# Design pre-review: SQL/JSON: JSON_TRANSFORM (SQL standard, subclause 6.44)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6912](https://commitfest.postgresql.org/patch/6912/) (Needs review); authors: Srinath Reddy (srinath-reddy-sadipiralla)
- Thread: [SQL/JSON: JSON_TRANSFORM (SQL standard, subclause 6.44)](https://www.postgresql.org/message-id/flat/CAFC%2Bb6pQxA0-YV8B-8KFub_bix_wdhA%2BoHrv7S1rYb2tQjjPzg%40mail.gmail.com), started 2026-06-18
- Messages read: 8 of 8
- Patch prose read: v4-0002-SQL-JSON-rework-JSON_TRANSFORM-execution-add-.-an.patch, v4-0003-SQL-JSON-support-per-action-behavior-clauses-in-J.patch, v4-0001-SQL-JSON-Add-initial-JSON_TRANSFORM-implementatio.patch, v4-0005-SQL-JSON-support-PATH-jsonpath-source-in-JSON_TRA.patch, v4-0004-JSON_TRANSFORM-fix-crash-add-deparse-support-lax-.patch, v1-0002-SQL-JSON-rework-JSON_TRANSFORM-execution-add-.-and-R.patch, v1-0001-SQL-JSON-Add-initial-JSON_TRANSFORM-implementation.patch, v1-0003-SQL-JSON-support-per-action-behavior-clauses-in-JSON.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection is fatal, and none is at level 1: the problem is stated and the demand objection was refuted. The four surviving objections are fixable level 2 points. standards-compat-2 and maintenance-fit-2 need someone who has the standard text and knows the SQL/JSON node design. maintenance-fit-1 and standards-compat-1 are design points the author can answer on the list.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 8 messages and no code, documentation or tests were read. The grammar behavior for IGNORE after a function call, the json input and RETURNING support, and the handling of unsupported path steps could not be checked.

## The proposal, restated

**Problem.** Postgres has only its own jsonb_set, jsonb_insert and jsonb_delete_path for changing a JSON value, with no SQL-standard spelling. Those functions cannot act on every member at one level in one call, have no "replace, else error" or "insert unless present" rules, and return NULL when the new value is NULL. The author writes for users who want portable, declarative JSON mutation.

**Proposed design.** The patch adds JSON_TRANSFORM (SQL/JSON T883, subclause 6.44) as a JsonExpr op that applies one INSERT, REPLACE, REMOVE or RENAME at a jsonpath target. The executor rebuilds the input jsonb in one streaming pass instead of calling jsonb_set and its siblings. Per-operation ON EXISTING, ON MISSING, ON NULL, ON EMPTY and ON ERROR clauses, lax/strict target paths and a "= PATH <jsonpath>" source are also supported.

**User-visible surface.**
- New SQL syntax: JSON_TRANSFORM(doc, INSERT | REPLACE | REMOVE | RENAME path [= value | = PATH jsonpath] [behaviors]), returning jsonb.
- New behavior clauses per operation: ON EXISTING, ON MISSING, ON NULL, ON EMPTY, ON ERROR, with standard implicit defaults.
- Two new unreserved keywords, EXISTING and MISSING; IGNORE gets a grammar precedence to share use with IGNORE NULLS.
- Wildcard '.*' allowed in target paths; REMOVE and REPLACE may end in it, INSERT and RENAME may not.
- Deparse output for JSON_TRANSFORM in EXPLAIN VERBOSE and stored expressions such as index definitions.

**What the author claims.**
- It is the SQL-standard, portable form of JSON mutation, where the jsonb_* functions are Postgres-specific.
- One call can do what the jsonb_* functions cannot, such as REMOVE '$.*.password' or conditional insert and replace.
- The streaming pass is O(size) however many members an action touches, against O(N * size) by delegation.
- INSERT, RENAME and empty-wildcard behavior match both the standard and Oracle; lax/strict handling matches jsonb_path_query().
- Peter Eisentraut suggested the feature and guided the author off-list.

**Thread state.** Zsolt Parragi reviewed v1 and v2 against Oracle docs, found crashes, missing deparse and lax-mode gaps, all addressed by v4 (five patches). No one has answered whether to stay with the standard's single operation per call or allow Oracle-style multiple operations, and the listed reviewer has not posted.

## 2. Prior attempts

### Add jsonb_translate(jsonb, from, to) (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/6086/](https://commitfest.postgresql.org/patch/6086/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CA+v5N41RzrpHVSkeN9EOey0JZbhfS3LB4OsRUD7bzTxrbBAszA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+v5N41RzrpHVSkeN9EOey0JZbhfS3LB4OsRUD7bzTxrbBAszA@mail.gmail.com)
- **Date:** decided 2026-01-25; started 2025-09-27
- **Outcome:** Withdrawn (cf)
- **Who objected:** Andrew Dunstan, Pavel Stehule, Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Andrew Dunstan judged blind replacement of every instance of a string, regardless of context, to be a niche use and asked why it could not be an extension. Pavel Stehule suggested using JsonPath to say what to replace, and Peter Eisentraut pointed to json_transform in Oracle and the SQL standard draft, noting that json_transform and json_translate together might confuse users. The thread ends there, and the author later withdrew the entry.
- **Reason (quote):** "Blindly replacing every instance of a single string regardless of context seems at best a niche use. Is there any reason not to make this an extension?" ([message](https://www.postgresql.org/message-id/ea9fa483-db55-48c8-a566-8a6d83690f78%40dunslane.net))
- **What is different this time:** P1 replaced every value that equals a given string, with no context. This proposal targets values by jsonpath with SQL-standard syntax, which replies in P1 pointed to as the alternative.


## 3. Objections a reviewer is likely to raise

### 1. Claimed portability rests on a subset of T883 nobody checked against the text

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, pg_dump / restore round-trip · `standards-compat-2`

A user ports a query that uses an array position target such as '$.arr[0]', a filter, a RETURNING clause or a json input. It fails or returns jsonb with a changed key order. If Postgres later corrects a default to match the standard text, views and indexes created earlier behave differently after upgrade.

Portability is the main argument in message #1. The proposal does not cover array positions, filters, RETURNING or json input, although the existing JSON_QUERY and JSON_VALUE support RETURNING. The only reviewer used Oracle documentation without the standard (messages #2, #5), so defaults such as ERROR ON MISSING for an empty wildcard rest only on the author's reading (message #3). Deparse writes the defaults explicitly, which helps dumps, but a stored behavior that was wrong would still need a compatibility break to fix.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-06-18)](https://www.postgresql.org/message-id/CAFC%2Bb6pQxA0-YV8B-8KFub_bix_wdhA%2BoHrv7S1rYb2tQjjPzg%40mail.gmail.com), [#2 (Zsolt Parragi, 2026-06-20)](https://www.postgresql.org/message-id/CAN4CZFPN%3D887T0njqw%2BbOHUiO6ion6rR%3Db1%2BEcx%2BgAaCuL89Wg%40mail.gmail.com), [#3 (Srinath Reddy Sadipiralla, 2026-06-21)](https://www.postgresql.org/message-id/CAFC%2Bb6rX08OYaQSGNWmGfjhBCE-Qf7tAtbXzv%3DD9K%2B%2B4W%3DLDBg%40mail.gmail.com), [#5 (Zsolt Parragi, 2026-07-02)](https://www.postgresql.org/message-id/CAN4CZFOfvdPy4J2oLL-fMph_A3YO7prHKv3bFQYDgD_DYj2VQA%40mail.gmail.com)

**Question:** Which parts of T883 does the patch cover, which parts are left out, and has someone with the standard text confirmed the chosen defaults?

Counter-check: held. The 0002 walker handles only key and wildcard steps, and the only reviewer says in messages #2 and #5 that he cannot check the standard. RETURNING and json input are unstated, not shown absent, so that part should be put as a question.

### 2. Reusing JsonExpr with a NULL path_spec breaks existing walkers

fixable · level 2, rubric item 9 · design, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

Code in core or an extension that handles JsonExpr assumes path_spec is set, as contain_mutable_functions_walker did. A JSON_TRANSFORM expression reaches it and the backend crashes. This happens at run time, in a path nobody tested, such as CREATE INDEX in message #5.

JSON_TRANSFORM is added as a JsonExpr op whose jsonpath lives in a new action field, while path_spec stays NULL. Review already found three crashes from this, in the expression walker, the mutator and clauses.c (messages #4 and #6), and each fix was a local special case. Every future consumer of JsonExpr has to remember the same special case, and nothing enforces it.

Thread: [#4 (Srinath Reddy Sadipiralla, 2026-07-02)](https://www.postgresql.org/message-id/CAFC%2Bb6pz%2BQh889V14x8wm1usjNtS419JuaJQ1g5xeuYtUsz5OQ%40mail.gmail.com), [#5 (Zsolt Parragi, 2026-07-02)](https://www.postgresql.org/message-id/CAN4CZFOfvdPy4J2oLL-fMph_A3YO7prHKv3bFQYDgD_DYj2VQA%40mail.gmail.com), [#6 (Srinath Reddy Sadipiralla, 2026-07-08)](https://www.postgresql.org/message-id/CAFC%2Bb6oAK75SV_1TRz9RDA7D-LeR9VDXSoLS1NSXdUSy4nYAYw%40mail.gmail.com)

**Question:** Why is JSON_TRANSFORM a JsonExpr op and not its own node, and how are all JsonExpr consumers in core shown to handle a NULL path_spec?

Counter-check: held. One correction: only the clauses.c crash came from the NULL path_spec; message #4 traces the walker and mutator crashes to the untraversed action field. The concern stands, because the thread does not say why JSON_TRANSFORM reuses JsonExpr or how other consumers were checked.

### 3. Second jsonpath walker duplicates jsonpath_exec semantics in the executor

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later release extends jsonpath, for example with a new accessor or changed lax unwrapping rules, and updates jsonpath_exec.c. The JSON_TRANSFORM walker in execExprInterp.c keeps the old behaviour without any error. Users then see JSON_TRANSFORM and jsonb_path_query() disagree on the same path, and find out from a bug report.

Patch 0002 adds its own path-to-steps converter and recursive rebuilder (jtSetPath, jtSetPathObject). Patch 0004 then adds lax/strict handling "similar to jsonb_path_query" as separate code. The walker accepts only key and wildcard steps, so array positions and filters, listed as open in the brief, would need a third copy of path logic. Nothing shown makes the two evaluators share code or tests that would fail when they diverge.

Thread: [#6 (Srinath Reddy Sadipiralla, 2026-07-08)](https://www.postgresql.org/message-id/CAFC%2Bb6oAK75SV_1TRz9RDA7D-LeR9VDXSoLS1NSXdUSy4nYAYw%40mail.gmail.com), [#7 (Srinath Reddy Sadipiralla, 2026-07-27)](https://www.postgresql.org/message-id/CAFC%2Bb6oJR028bP-vVLo3OhWM_oFgg324z%2BNr3MZoJWTvLYwzXQ%40mail.gmail.com)

**Question:** Can the target-path matching reuse jsonpath_exec.c, or which shared tests fail when JSON_TRANSFORM and jsonb_path_query() disagree on a path?

Counter-check: held. The 0002 commit message describes a self-contained path-to-steps converter and walker, and 0004 adds lax/strict handling to it separately. The material names no shared code or test that fails when it and jsonb_path_query() disagree.

### 4. IGNORE ON clause may not parse after a function-call value

fixable · level 2, rubric item 10 · SQL-standard conflict or non-standard syntax · `standards-compat-1`

A user writes `json_transform(j, replace '$.a' = to_jsonb(x) ignore on missing)`. The value ends in a function call, so the parser may treat IGNORE as the start of IGNORE NULLS and then fail on ON. The user gets a syntax error for a form the standard allows.

Patch 0003 resolves the IGNORE NULLS / IGNORE ON conflict by giving IGNORE_P a precedence that always shifts toward IGNORE NULLS after a function application. That keeps window syntax working, but it seems to send exactly the case where a value or PATH source ends in a function call down the wrong branch. The commit message does not say whether that case parses, and nothing in the thread tests it. Silencing a grammar conflict with precedence hides it from future gram.y changes.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-06-18)](https://www.postgresql.org/message-id/CAFC%2Bb6pQxA0-YV8B-8KFub_bix_wdhA%2BoHrv7S1rYb2tQjjPzg%40mail.gmail.com)

**Question:** Does `replace '$.a' = some_func(x) ignore on missing` parse, and if so, which grammar rule makes it work despite the IGNORE_P precedence?

Counter-check: held. The 0003 commit message says the parser shifts IGNORE toward NULLS after a function application, and no example in the thread has a function-call value followed by IGNORE ON. The grammar is not shown, so the material cannot rule out the syntax error.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | JSON_TRANSFORM is accepted in expression indexes, so a later change to its semantics affects index contents kept by pg_upgrade. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | Stored expressions in views, indexes and defaults carry the new JsonExpr op and its action fields. |
| ABI / extension API | **yes** | The patch changes JsonExpr in primnodes.h and adds parse nodes, which extensions that walk expression trees see. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds JSON_TRANSFORM syntax, the keywords EXISTING and MISSING, and a precedence for IGNORE. |
| new GUC | no |  |
| planner behaviour change | **yes** | Patches 0004 and 0005 change clauses.c, including the mutability check for the new op. |
| pg_dump / restore round-trip | **yes** | The new ruleutils deparse must reproduce every operation and behavior clause for stored expressions. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a separate jsonpath target walker in execExprInterp.c, JIT support, grammar and deparse code. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which parts of T883 does the patch cover, which does it leave out, and has someone with the standard text confirmed the chosen defaults?
2. Does JSON_TRANSFORM accept json input or a RETURNING clause, and are array position targets such as '$.arr[0]' and filter expressions rejected with an error?
3. Why is JSON_TRANSFORM a JsonExpr op with a NULL path_spec instead of its own node, and how were the other JsonExpr consumers in core checked?
4. Can target-path matching reuse jsonpath_exec.c, or which shared tests fail when JSON_TRANSFORM and jsonb_path_query() disagree on a path?
5. Does `replace '$.a' = some_func(x) ignore on missing` parse, and which grammar rule makes it work despite the IGNORE_P precedence?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the SQL/JSON grammar and JsonExpr execution and who has the SQL standard text. Message #1 reports off-list guidance from Peter Eisentraut, but the visible messages show no committer posting.
- **Liability:** The owner signs for the semantics of each operation and default once views and expression indexes store them, and for a second jsonpath walker that must follow jsonpath_exec.c in every later release.
- **Author follow-up:** The author answered review within a day in message #3 and posted four versions that address the reported crashes and gaps.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Five patches touch the grammar, parse analysis, the executor, JIT, clauses.c and ruleutils, and the semantics need a check against the standard that the only reviewer could not do. The goal has a standard feature and an on-list suggestion behind it, but no committer has posted in the visible thread.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Expression indexes freeze semantics that no one has checked against the standard. Refuted (not specific): Duplicate of standards-compat-2, which already covers the unchecked semantics and their effect on stored indexes. The pg_upgrade consequence applies to a behavior fix in any new immutable function, and Postgres handles such fixes with a release-note REINDEX instruction.
- `maintenance-fit-3` No committer has engaged on a large new SQL/JSON surface. Refuted (not specific): The failure scenario is a duplicate of standards-compat-2. The rest is a committability note, which the rubric keeps apart from the verdict, and message #1 gives a path from one operation to a list that keeps single-operation deparse output valid.
- `demand-1` Portability benefit not shown for the single-operation form. Refuted (already handled): The material shows demand evidence the rubric accepts: a SQL-standard feature (T883), an Oracle implementation, and a committer's on-list suggestion recorded in P1. Message #1 also names cases jsonb_set cannot do in one call; the Oracle list form is an open scope question.

### Review angles

- prior-attempts: 0 objection(s). No precedent rejection applies. P1 died as a niche, context-free operation that could live in an extension, and this proposal answers that with jsonpath targets and standard syntax. P4 and P6 share only the jsonb or SQL/JSON area and record no reason that applies here.
- standards-compat: 2 objection(s). The single-operation form can grow into a comma-separated list later without breaking existing calls, so the open direction question is not a compatibility objection. The keywords are unreserved, and deparse writes explicit behaviors, which helps the pg_dump round-trip.
- replication-ondisk: 1 objection(s). There is no new WAL, on-disk or wire-protocol state. The only persistent state is stored expressions such as views, indexes and defaults, which pg_dump handles through the new ruleutils deparse.
- security: 0 objection(s). JSON_TRANSFORM is a pure function of its jsonb input and needs no new privilege, file access or cross-session state, so this angle finds no privilege or exposure issue. Without the code, two things cannot be checked: that a JSON_TRANSFORM node is treated as leaky in security barrier views and RLS quals, and that the RENAME duplicate-key error reveals no more than existing jsonb errors do.
- maintenance-fit: 3 objection(s). There is no hot-path cost for non-users. The cost falls on users of the feature and on future maintainers.
- already-exists: 0 objection(s). Existing tools cover only part of this. jsonb_set, jsonb_insert, the #- operator and jsonb_set_lax (for NULL handling) cover single-path edits. No core function or extension removes or replaces every member matched by '.*' in one call, and standard grammar such as JSON_TRANSFORM can only live in core. Postgres already added JSON_QUERY and JSON_VALUE next to the jsonb_path_* functions, so adding a standard spelling beside a native one follows existing practice.
- demand: 1 objection(s). Standard conformance and a committer's suggestion are real but modest signals. The only objection asks for evidence of demand and for an answer to the single-operation versus Oracle-style question.

### Precedents retrieved

- P1: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - used in the review; reason summarised from its thread
- P2: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P3: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant; reason summarised from its thread
- P4: [jsonb array-style subscription](https://commitfest.postgresql.org/patch/485/) (cf, Returned with feedback, 2016-03-21) - used in the review; reason summarised from its thread
- P5: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P6: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - used in the review
- P7: [SQL/JSON json_table() function PLAN clause](https://commitfest.postgresql.org/patch/6590/) (cf, Withdrawn, 2026-05-19) - judged not relevant; reason summarised from its thread
- P8: [json_table function PLAN clause](https://commitfest.postgresql.org/patch/6332/) (cf, Withdrawn, 2026-05-19) - judged not relevant; reason summarised from its thread
- P9: [Support % wildcard in extension upgrade scripts](https://commitfest.postgresql.org/patch/3654/) (cf, Returned with feedback, 2023-08-01) - judged not relevant; reason summarised from its thread
- P10: [Support % wildcard in extension upgrade scripts](https://commitfest.postgresql.org/patch/3892/) (cf, Withdrawn, 2022-09-14) - judged not relevant
- P11: [Fix bug of COPY (on_error set_null)](https://commitfest.postgresql.org/patch/6774/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P12: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P13: [bytea PL/Perl transform](https://commitfest.postgresql.org/patch/4252/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P14: [Add jsonpath string method $.replace(from, to)](https://commitfest.postgresql.org/patch/5261/) (cf, Withdrawn, 2024-09-25) - judged not relevant
- P15: [Revert "Add gitignore entries for jsonpath_gram.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7b084b38310cfe9c8b58cc615a81df625c771f5d) (revert, Reverted, 2019-03-22) - judged not relevant
- P16: [remove C++ keywords](https://commitfest.postgresql.org/patch/1075/) (cf, Returned with feedback, 2017-04-06) - judged not relevant
- P17: [Revert "Reorganise jsonpath operators and methods"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d) (revert, Reverted, 2024-01-03) - judged not relevant
- P18: [Documentation (mostly) of implementing TRANSFORM FOR TYPE in a PL](https://commitfest.postgresql.org/patch/3554/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P19: [revert: Transform OR clauses to ANY expression](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ff9f72c68f678ded340b431c3e280fe56644a3e7) (revert, Reverted, 2024-04-09) - judged not relevant
- P20: [Detect the null type correctly in setop case by peeking the later exprs.](https://commitfest.postgresql.org/patch/2653/) (cf, Withdrawn, 2020-07-23) - judged not relevant
- P21: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant
- P22: [Table AM modifications to accept column projection lists](https://commitfest.postgresql.org/patch/2922/) (cf, Withdrawn, 2021-06-30) - judged not relevant
- P23: [SQL/JSON: JSON_TABLE](https://commitfest.postgresql.org/patch/1473/) (cf, Returned with feedback, 2020-08-03) - judged not relevant
- P24: [Storage layer hooks to enable data transformation in extensions](https://commitfest.postgresql.org/patch/6351/) (cf, Withdrawn, 2026-02-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 154 s; cost reported by the CLI: $2.40
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 85 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:08:04.033Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
