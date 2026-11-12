# Design pre-review: BUG #19737: Empty `JSON_OBJECT` cannot use documented `ON NULL` or unique-key clauses

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7527](https://commitfest.postgresql.org/patch/7527/) (Needs review); authors: Narayanan Venkateswaran (narayananv)
- Thread: [BUG #19737: Empty `JSON_OBJECT` cannot use documented `ON NULL` or unique-key clauses](https://www.postgresql.org/message-id/flat/19737-b44cf2e08708b613%40postgresql.org), started 2026-10-02
- Messages read: 10 of 10
- Patch prose read: draft_json_object_fix.txt (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The doc mismatch is real, but the thread shows no query that needs ON NULL or UNIQUE with an empty list (demand-1). Message #1 itself says the clauses do not change the result, and JSON_OBJECT() already returns {}. The docs-only fix from message #2 matches the current parser and was never weighed against the grammar change. The list should choose between the two fixes before code review.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm that the new gram.y productions add no conflicts, or how a view that uses the new forms is deparsed. The v3 documentation text on the difference from the standard is not shown.

## The proposal, restated

**Problem.** The docs show the key/value list of JSON_OBJECT as optional and independent of the ON NULL, UNIQUE and RETURNING clauses. But JSON_OBJECT(ABSENT ON NULL) and JSON_OBJECT(WITH UNIQUE KEYS) fail with a syntax error, while JSON_OBJECT() and JSON_OBJECT(RETURNING jsonb) work. JSON_ARRAY(NULL ON NULL) fails the same way.

**Proposed design.** The patch adds grammar productions in gram.y so that empty JSON_OBJECT accepts the NULL/ABSENT ON NULL, WITH/WITHOUT UNIQUE KEYS and RETURNING clauses, and empty JSON_ARRAY accepts ON NULL and RETURNING. Each such call returns an empty object or array. v3 also notes in func-json.sgml and the commit message that this differs from the SQL standard grammar.

**User-visible surface.**
- SQL syntax: JSON_OBJECT( { NULL | ABSENT } ON NULL ) and JSON_OBJECT( { WITH | WITHOUT } UNIQUE [ KEYS ] ) with no key/value pairs now parse.
- SQL syntax: JSON_ARRAY( { NULL | ABSENT } ON NULL [ RETURNING ... ] ) with no values now parses.
- Documentation: func-json.sgml gets notes on the difference from the SQL standard.

**What the author claims.**
- The parser contradicts the documented syntax, so this is a bug fix.
- The new gram.y productions add no grammar conflicts.
- Regression tests in sqljson.sql cover all clause combinations for empty objects.
- The SQL:2016 grammar nests the ON NULL and uniqueness clauses inside the required key/value list, so Postgres's documented syntax differs from the standard.
- A reviewer reports that Oracle and other systems accept the empty forms, based on an LLM search.

**Thread state.** Nitin Motiani tested v1 and v2, asked for JSON_ARRAY coverage and for the standard difference to be documented, and v3 does both. No committer has replied, and jian he's earlier idea of fixing only the docs got no follow-up.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-02; 23 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No use case shown for clauses that cannot change the result

fixable · level 1b, rubric item 3 · demand, SQL-standard conflict or non-standard syntax, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

Postgres adds grammar for JSON_OBJECT(ABSENT ON NULL) and similar forms, which always return {} or []. If no user needs them, the project still carries syntax that the SQL:2016 grammar does not allow. A later standard revision or other systems may define these forms differently, and Postgres must then keep its own behavior or break callers.

The only demand in the thread is the doc mismatch in message #1, and message #1 itself says the clauses do not change the result. Anyone who hits the error can drop the clause, since JSON_OBJECT() and JSON_ARRAY() already work (message #3, message #7). Message #2 offers a docs-only fix that matches the standard, and nobody weighed it against the grammar change. Message #9 calls the grammar change "probably" right only because the docs already differ from the standard.

**Precedents.**
- [MAP syntax for arrays](https://commitfest.postgresql.org/patch/1632/) (Rejected, 2018-07-19); thread: [link](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru)

Thread: [#1 (PG Bug reporting form, 2026-10-02)](https://www.postgresql.org/message-id/19737-b44cf2e08708b613%40postgresql.org), [#2 (jian he, 2026-10-04)](https://www.postgresql.org/message-id/CACJufxERBd7_cN9-3FSAg4%2BGEdvXcaNQBoOBr1ZQ7Kgcwx7WOQ%40mail.gmail.com), [#3 (Narayanan Venkateswaran, 2026-10-06)](https://www.postgresql.org/message-id/CAFjuD9ehcFRN4M1QkJDQq0Asr8v%2B8dVWcGPc5Z7ENEpy-iA5fg%40mail.gmail.com), [#7 (Narayanan Venkateswaran, 2026-10-19)](https://www.postgresql.org/message-id/CAFjuD9fKAs3%3DSuywg%3D3xMM1GxLjGbCuZ8nt9GK4PFD4JbmY5ow%40mail.gmail.com), [#9 (Nitin Motiani, 2026-10-21)](https://www.postgresql.org/message-id/CAH5HC97O5fkwYkyZr-YtzugAniNNJ3YxS%3DznSw2oCeQqOB0KXQ%40mail.gmail.com)

**Question:** Which real queries or SQL generators emit ON NULL or UNIQUE clauses with an empty list, and why is dropping the clause or fixing the docs not enough for them?

Counter-check: held. The thread shows one bug report and no query or SQL generator that needs the clauses with an empty list, and message #1 says they do not change the result. P10 is about a new keyword and must not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch accepts empty JSON_OBJECT and JSON_ARRAY forms with ON NULL and UNIQUE clauses that the SQL:2016 grammar does not allow. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | A view written with the new forms may dump as text that an older minor release cannot parse, if the fix is back-patched. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix against 18.6, and the thread does not say whether the grammar change is for master only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | New gram.y productions for the empty forms must stay conflict-free and stay accepted in every later release. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | JSON_OBJECT() and JSON_ARRAY() without the clauses already return the same empty object or array. |

## 5. Questions the author must answer before anyone reviews code

1. Which real queries or SQL generators emit ON NULL or UNIQUE clauses with an empty key/value list, and why is dropping the clause not enough for them?
2. Why is the grammar change better than the docs-only fix from message #2, which matches both the current parser and the SQL:2016 grammar?
3. Is the grammar change meant for master only, or for back-patching to the supported branches?
4. If it is back-patched, what happens to SQL and dumps that use the new forms on an older minor release?
5. Do JSON_OBJECTAGG and JSON_ARRAYAGG have the same gap between the documented synopsis and the parser?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the SQL/JSON constructor grammar in gram.y and the SQL/JSON standard text would have to own it. None has replied in the visible thread.
- **Liability:** The owner signs for syntax beyond SQL:2016 that Postgres must accept in every later release, and for any conflict with a later standard revision.
- **Author follow-up:** The author posted v2 and v3 in response to review, adding JSON_ARRAY coverage and a note on the difference from the standard.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A real query, ORM or SQL generator that emits these clauses with an empty list, or documentation from other database systems showing they accept the empty forms. The LLM search in message #5 does not count as this evidence.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches only gram.y, the regression tests and func-json.sgml, and the new forms return what JSON_OBJECT() and JSON_ARRAY() already return. The thread first needs to choose between this grammar change and the docs-only fix from message #2.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Grammar extended beyond the standard when a docs fix would close the bug. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) New syntax in a back-patched bug fix splits minor releases. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-1` Grammar extension beyond the standard, with the docs-only fix not weighed. Refuted (not specific): Duplicate of standards-compat-1 for the docs-only alternative and of standards-compat-2 for back-patching. It adds no separate point.
- `maintenance-fit-2` Separate empty-form productions must track every future constructor clause. Refuted (no failure scenario): The scenario needs a future constructor clause that nobody has proposed. Message #3 shows the empty form already takes RETURNING separately, and message #6 shows that split comes from the standard grammar, so the patch follows existing practice.
- `already-exists-1` Empty forms already work by dropping clauses that have no effect. Refuted (not specific): Duplicate of demand-1 for the missing use case and of standards-compat-1 for the docs-only alternative. P10 concerns a new MAP keyword and expression type, so it does not support this point.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent rejected this idea. P10's standard-conflict concern applies only weakly, because the patch adds no keyword and no new semantics. Within the thread, message #2 proposed a docs-only fix that nobody pursued, and messages #9 and #10 already weighed the difference from the standard, so no prior reason stands unanswered.
- standards-compat: 2 objection(s). Two objections are raised. The listed precedents concern new proprietary syntax with new meaning, while this change only accepts clauses that have no effect, so they are not cited.
- replication-ondisk: 0 objection(s). The change adds parser productions and doc text only. It adds no on-disk format, WAL record, catalog column, wire message or struct change, and the parse tree it builds is the existing empty constructor node. A view using the new forms stores that node and is deparsed by the existing ruleutils code. Whether that deparsed text parses on older minor releases, if back-patched, cannot be checked without the code.
- security: 0 objection(s). The proposal only adds grammar productions so that empty JSON_OBJECT and JSON_ARRAY accept ON NULL, unique-key and RETURNING clauses, and each such call returns an empty object or array. It adds no privilege, role, data access, information exposure or resource-use path, so there is no security angle to object on.
- maintenance-fit: 2 objection(s). Hot-path cost is nil: this is parse-time grammar only. No external dependencies or catalog changes are involved.
- already-exists: 1 objection(s). One objection only. No extension or other feature is relevant, because JSON_OBJECT() and JSON_ARRAY() without clauses already produce the result the proposal wants.
- demand: 1 objection(s). One fixable demand objection. The doc mismatch is a real bug, but the thread shows no user who needs the empty forms.

### Precedents retrieved

- P1: [Revert cascading of JSON_TABLE's ON ERROR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=af6fad879fbf9b542cfde05c52695c9dffa6bb47) (revert, Reverted, 2026-07-14) - judged not relevant
- P2: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - judged not relevant; reason summarised from its thread
- P3: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P4: [empty](https://commitfest.postgresql.org/patch/5037/) (cf, Rejected, 2024-06-21) - judged not relevant
- P5: [Add SQL syntax check](https://commitfest.postgresql.org/patch/4704/) (cf, Rejected, 2024-05-15) - judged not relevant; reason summarised from its thread
- P6: [Revert 95d737ff to add 'ignore_nulls'](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8a026e4f15e2063cd7fe6cb9ffe44f2a81398bd) (revert, Reverted, 2014-09-29) - judged not relevant
- P7: [Ltree syntax improvement](https://commitfest.postgresql.org/patch/1977/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P8: [skip empty transactions for logical replication](https://commitfest.postgresql.org/patch/2384/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P9: [Boolean partition syntax](https://commitfest.postgresql.org/patch/1410/) (cf, Returned with feedback, 2018-03-21) - judged not relevant; reason summarised from its thread
- P10: [MAP syntax for arrays](https://commitfest.postgresql.org/patch/1632/) (cf, Rejected, 2018-07-19) - used in the review; reason summarised from its thread
- P11: [json_table function PLAN clause](https://commitfest.postgresql.org/patch/6332/) (cf, Withdrawn, 2026-05-19) - judged not relevant; reason summarised from its thread
- P12: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P13: [SQL/JSON json_table() function PLAN clause](https://commitfest.postgresql.org/patch/6590/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P14: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant
- P15: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P16: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P17: [jsonb array-style subscription](https://commitfest.postgresql.org/patch/485/) (cf, Returned with feedback, 2016-03-21) - judged not relevant
- P18: [make psql backslash commands honor "\pset null"](https://commitfest.postgresql.org/patch/4603/) (cf, Withdrawn, 2023-10-06) - judged not relevant
- P19: [Add function to detect if an array is empty](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Arrays) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P20: [Make use of C99 designated initialisers for nulls/values arrays](https://commitfest.postgresql.org/patch/2290/) (cf, Rejected, 2019-11-27) - judged not relevant
- P21: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant
- P22: [Improve handling of NULLs in arrays](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Arrays) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P23: [Indeterminate behavior for the GROUP BY clause](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant

### Warnings

- precedent search failed for "SQL/JSON constructor grammar gram.y": Spread syntax requires ...iterable[Symbol.iterator] to be a function
- precedent search failed for "SQL/JSON standard conformance constructors": Spread syntax requires ...iterable[Symbol.iterator] to be a function

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 119 s; cost reported by the CLI: $1.60
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 23 kept
- Tone check: 78 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:04:30.787Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
