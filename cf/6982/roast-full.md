# Design pre-review: Proposal: INSERT ... BY NAME

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6982](https://commitfest.postgresql.org/patch/6982/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [Proposal: INSERT ... BY NAME](https://www.postgresql.org/message-id/flat/CAJTYsWVDs7vkEN-eD1NAnZ7XqcfpXu3tP1wMGzJQ12G6eo1oRw%40mail.gmail.com), started 2026-07-03
- Messages read: 13 of 13
- Patch prose read: v2-0001-Add-INSERT-.-BY-NAME-to-match-source-columns-by-n.patch, v1-0001-Add-INSERT-.-BY-NAME-to-match-source-columns-by-n.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the parse-analysis design. The one surviving objection, standards-compat-1, is fixable: the patch accepts BY on either side of OVERRIDING and BY POSITION with DEFAULT VALUES, and nobody has checked these forms against the draft standard text. A committer who knows the grammar and the standard would need to sign off on the released syntax.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot confirm that the Query node is unchanged, how names are compared, or which columns get the insert privilege check when no column list is given.

## The proposal, restated

**Problem.** Positional insert ... select breaks or puts data in the wrong column when the source query lists columns in a different order than the target. The author cites the forthcoming SQL standard, which accepted BY NAME for INSERT, and similar syntax in DuckDB and Oracle 23ai.

**Proposed design.** A new optional clause `BY { NAME | POSITION }` sits before OVERRIDING and the source, and BY POSITION is the default. Parse analysis resolves BY NAME by reordering the target column and attribute-number lists to match the source columns, so the query tree is an ordinary positional insert and rule and view deparsing need no change.

**User-visible surface.**
- SQL syntax: `insert into t [ (cols) ] by { name | position } [ overriding ... ] query`, accepted on either side of OVERRIDING.
- New errors for unknown source names, duplicate matches, bare VALUES or DEFAULT VALUES with BY NAME, and subfield or array-element targets.
- psql tab completion for BY NAME and BY POSITION (added in v2).

**What the author claims.**
- The SQL draft standard accepted the feature, and DuckDB and Oracle 23ai implement the same core behavior.
- Target columns that the source does not name get their default values, as if left out of the column list.
- Resolving BY NAME in parse analysis means no rule or view deparsing changes are needed.
- `select *` and named VALUES wrapped in a subquery work with BY NAME, and v2 adds tests for them.

**Thread state.** Four reviewers tested the patch and report that it works, and Andreas Karlsson confirms the standard's semantics for an explicit column list. v2 addressed Vaibhav Dalvi's review, and he left two small items: tab completion after BY NAME, and a regression test for `by name table other_table`.

## 2. Prior attempts

### Corresponding by clause for set operation (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1016/](https://commitfest.postgresql.org/patch/1016/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CALAY4q8XqRqAaNsdC2Om+r8dNEgZbaBHTQYT+8DL0BCRdo7H0w@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALAY4q8XqRqAaNsdC2Om+r8dNEgZbaBHTQYT+8DL0BCRdo7H0w@mail.gmail.com)
- **Date:** decided 2017-04-04; started 2017-01-17
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** After Pavel Stehule marked it ready for committer, Tom Lane found that much code assumes set operation output columns map one-for-one to input columns, and the patch had not changed that code. As a result, qual pushdown into a view fails with "wrong number of tlist entries", UNION ALL appendrel conversion returns columns in the wrong order, and WITH RECURSIVE fails. ruleutils.c also reverse-lists views wrongly and does not track column renames. David Steele returned the entry because no committable patch was likely before the commitfest ended.
- **Reason (quote):** "The main thing that's scaring me off is a realization that there are a *lot* of places that assume that the output columns of a set operation are one-for-one with the columns of the inputs." ([message](https://www.postgresql.org/message-id/30942.1491069285%40sss.pgh.pa.us))
- **What is different this time:** P4 matched set-operation inputs by name and broke code that assumes set-operation outputs map one-for-one to inputs. This proposal resolves BY NAME in parse analysis into an ordinary positional insert with a reordered target column list, a form INSERT already supports.


## 3. Objections a reviewer is likely to raise

### 1. Grammar commitments go beyond the unverified draft standard text

fixable · level 2, rubric item 6 · SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-1`

The published standard fixes BY NAME to one position relative to OVERRIDING, or does not allow BY POSITION with DEFAULT VALUES. Postgres has already released both orders and the no-op form. It must then keep the non-standard spellings for compatibility, or break user scripts that rely on them.

No one has checked the patch against the draft text. Jim Jones said he would try to get it (message #7), and Andreas Karlsson confirmed only the explicit column list case (messages #4 and #5). The patch accepts the clause on either side of OVERRIDING and accepts BY POSITION with DEFAULT VALUES. The thread does not say whether the draft allows either form. Syntax extensions are hard to remove once released, so a committer would likely want each accepted form tied to the draft grammar before commit.

Thread: [#4 (Andreas Karlsson, 2026-07-10)](https://www.postgresql.org/message-id/52c9a352-0e4a-4eaa-a9e1-06fd33cf562f%40proxel.se), [#5 (Andreas Karlsson, 2026-07-10)](https://www.postgresql.org/message-id/c5090d60-09b3-4ca3-97cc-ddb99d87a301%40proxel.se), [#7 (Jim Jones, 2026-07-13)](https://www.postgresql.org/message-id/349ee7d0-ed22-41a3-8d1f-698cc0170e99%40uni-muenster.de), [#12 (Ayush Tiwari, 2026-09-12)](https://www.postgresql.org/message-id/CAJTYsWWrRNhAYQvt-p9puKU5CGj2bc5HU9Jajm4gKzcsKz4BBg%40mail.gmail.com)

**Question:** Which of the accepted forms, both OVERRIDING orders and BY POSITION with DEFAULT VALUES, does the draft standard grammar allow?

Counter-check: held. The thread confirms only the explicit column list case against the accepted change proposal. Nobody has checked the two OVERRIDING orders or BY POSITION with DEFAULT VALUES against the draft, and a released spelling is hard to remove.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes the raw InsertStmt node in parsenodes.h, which extensions that inspect parse trees can see. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds `BY { NAME \| POSITION }` to INSERT from a draft standard that nobody in the thread has read. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Rules and SQL function bodies that use BY NAME are stored and dumped in resolved positional form. |
| psql UX conventions | **yes** | v2 adds tab completion for BY NAME and BY POSITION in tab-complete.in.c. |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: Grammar commitments go beyond the unverified draft standard text |
| maintenance burden ("who maintains this in 10 years") | **yes** | New INSERT grammar forms in gram.y become permanent syntax once released. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which of the accepted forms, both OVERRIDING orders and BY POSITION with DEFAULT VALUES, does the draft standard grammar allow?
2. Does name matching follow identifier case rules, for example a quoted source alias `"C1"` against an unquoted target column c1?
3. How does BY NAME behave with generated columns, identity columns, ON CONFLICT and RETURNING?
4. Without a target column list, does the patch check insert privilege only on the columns that the source query names?
5. Does the Query node gain a new field, or is the change limited to the raw InsertStmt?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows gram.y and parse analysis of INSERT, and who can read the draft SQL standard text. The visible thread and the CommitFest entry show no such committer taking part.
- **Liability:** The owner signs for every accepted spelling of the clause for as long as Postgres keeps INSERT syntax compatible, including any form the published standard later does not allow.
- **Author follow-up:** The author posted v2 in message #12 in response to Vaibhav Dalvi's review, which shows follow-up during review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 29 KiB, touches only the parser, docs, psql completion and tests, and resolves to an ordinary positional insert. The open point is a check of the accepted forms against the draft standard, which does not need a redesign.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Syntax ships before the standard text is final or checked. Refuted (not specific): Duplicate of standards-compat-1, which names the specific forms that go beyond what the thread confirmed. The parts unique to this objection, case folding and unmatched source columns, have no shown conflict with the draft.

### Review angles

- prior-attempts: 0 objection(s). The only related precedent is P4, and its recorded reason does not apply. It failed because set-operation code assumes outputs map one-for-one to inputs, and this patch leaves set operations unchanged and yields an ordinary insert with a reordered column list. The thread cites no earlier attempt at INSERT BY NAME, and message #1 says the author found none.
- standards-compat: 1 objection(s). Other compatibility points hold. BY NAME is opt-in, so existing statements keep their meaning. Resolution in parse analysis means deparsed rules and dumps show a plain positional insert that older servers accept. The INSERT alias needs AS, so no grammar ambiguity is visible.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state for this angle to object to. BY NAME is resolved in parse analysis into an ordinary positional column mapping, so the stored Query tree, pg_rewrite contents, ruleutils deparsing, WAL, on-disk format, replication and the wire protocol are unchanged. The parsenodes.h change touches only the raw InsertStmt, which is never stored. A rule or SQL function body that uses BY NAME is stored and dumped in its resolved positional form, the same way a view stores an expanded select *. Without the code, this angle cannot confirm that the Query node itself gains no new field, which would need a catversion bump.
- security: 0 objection(s). The proposal has no security or privilege aspect worth objecting to. BY NAME is resolved in parse analysis into an ordinary positional insert, so column insert privileges, RLS, triggers and rules apply as they do for an explicit column list. Error texts name only target columns, which the catalogs already expose. One point needs the code to confirm: without a column list, insert privilege should be checked only on matched columns.
- maintenance-fit: 1 objection(s). Other maintenance concerns do not apply. BY NAME is resolved in parse analysis, adds no executor, planner, catalog or ABI cost, and avoids the column-mapping problem that stopped CORRESPONDING [P4]. No committer has engaged yet, but that is a committability point and not a flaw in the design.
- already-exists: 0 objection(s). The existing alternative is an explicit target column list, `insert into t (b, a) select ...`, which already reorders columns, so BY NAME adds convenience and no new capability. I would not send that objection, because the clause is accepted for the next SQL standard, DuckDB and Oracle 23ai already have it, and it resolves to an ordinary positional insert. No extension or hook could give the same grammar. [P4] is about set operations, and its reason, that output columns stop mapping one-for-one to input columns, does not apply here.
- demand: 0 objection(s). The material shows enough demand for a syntax feature whose cost falls only on its users. The SQL committee accepted BY NAME for INSERT (message #4), DuckDB and Oracle 23ai ship it (message #1), and four independent reviewers tested it. BY POSITION stays the default, and BY NAME resolves in parse analysis to an ordinary positional insert, so non-users pay nothing on the hot path. The workaround is an explicit column list in the right order, which helps a little but not when the source is `select *` from a table with a different column order. Questions about the standard text and identifier case rules belong to other angles.

### Precedents retrieved

- P1: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant; reason summarised from its thread
- P2: [Permute underscore separated components of columns before fuzzy matching](https://commitfest.postgresql.org/patch/4282/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P3: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - judged not relevant; reason summarised from its thread
- P4: [Corresponding by clause for set operation](https://commitfest.postgresql.org/patch/1016/) (cf, Returned with feedback, 2017-04-04) - used in the review; reason summarised from its thread
- P5: [Parallel INSERT (INTO ... SELECT ...)](https://commitfest.postgresql.org/patch/2844/) (cf, Returned with feedback, 2021-04-01) - judged not relevant; reason summarised from its thread
- P6: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant; reason summarised from its thread
- P7: [Mention column name in error messages](https://commitfest.postgresql.org/patch/324/) (cf, Returned with feedback, 2015-12-22) - judged not relevant; reason summarised from its thread
- P8: [Mention column name in error messages](https://commitfest.postgresql.org/patch/818/) (cf, Returned with feedback, 2016-11-29) - judged not relevant; reason summarised from its thread
- P9: [INSERT .. ON CONFLICT DO SELECT](https://commitfest.postgresql.org/patch/1241/) (cf, Returned with feedback, 2018-01-22) - judged not relevant
- P10: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P11: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P12: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P13: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P14: [Revert: Let table AM insertion methods control index insertion](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da841aa4dc279bb0053de56121c927ec943edff3) (revert, Reverted, 2024-04-11) - judged not relevant
- P15: [json_table function PLAN clause](https://commitfest.postgresql.org/patch/6332/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P16: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P17: [Making autovacuum logs indicate if insert-based threshold was the triggering condition](https://commitfest.postgresql.org/patch/3802/) (cf, Withdrawn, 2022-11-27) - judged not relevant
- P18: [Implement ASSERTIONs per the SQL standard.](https://commitfest.postgresql.org/patch/1604/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P19: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P20: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P21: [OR-clauses for index](https://commitfest.postgresql.org/patch/454/) (cf, Returned with feedback, 2016-03-29) - judged not relevant
- P22: [INSERT ... ON CONFLICT DO SELECT \[FOR ...\]](https://commitfest.postgresql.org/patch/5413/) (cf, Returned with feedback, 2025-08-05) - judged not relevant
- P23: [parallel distinct union and aggregate support patch](https://commitfest.postgresql.org/patch/2769/) (cf, Returned with feedback, 2021-11-08) - judged not relevant
- P24: [Union Replacement of OR logic](https://commitfest.postgresql.org/patch/4715/) (cf, Rejected, 2024-07-24) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 111 s; cost reported by the CLI: $1.72
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 57 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:12:26.014Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
