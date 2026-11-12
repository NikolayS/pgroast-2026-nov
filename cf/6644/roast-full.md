# Design pre-review: Extract numeric filed in JSONB more effectively

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6644](https://commitfest.postgresql.org/patch/6644/) (Needs review); authors: Andy Fan (andyf), Haibo Yan (billyean)
- Thread: [Extract numeric filed in JSONB more effectively](https://www.postgresql.org/message-id/flat/CAKU4AWoqAVya6PBhn%2BBCbFaBMt3z-2%3Di5fKO3bW%3D6HPhbid2Dw%40mail.gmail.com), started 2023-08-01
- Messages read: 114 of 114
- Patch prose read: v14-0002-try-to-apply-jsonb-extraction-optimization-to-_t.patch, v14-0001-optimize-casting-jsonb-to-a-given-type.patch, v1-0001-Add-jsonb-operator-to-return-a-numeric-directly.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at level 1, and the support-function design follows the direction given in message #17. Three fixable level 2 points remain. The v4 to v6 mails do not show that rewritten casts return the same values and errors as master (prior-attempts-1). Nothing keeps the typed extractors and the cast functions in step later (maintenance-fit-2). The code-sharing question about JSON_VALUE from message #85 is still open (already-exists-1).

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm that the typed extractors match the casts on every input. That also decides whether expression indexes over `(j->'a')::numeric` stay valid. The leakproof marking and the safety of direct SQL calls to the new functions are also unchecked.

## The proposal, restated

**Problem.** A cast such as `(j->'a')::numeric` builds an intermediate jsonb from the extracted JsonbValue, then the cast function unpacks it again. The author measured this at about half the possible speed. The target users run casts of jsonb fields to numeric, bool or other number types in queries.

**Proposed design.** The latest series (v6, five patches) attaches a SupportRequestSimplify function to the jsonb cast functions. It rewrites `cast(extract(...))` into a call to an explicit typed extractor, so SQL stays unchanged. It covers object field, array element, extract path, multi-subscript and jsonb_path_query_first(_tz), each to numeric, bool, int2, int4, int8, float4 and float8.

**User-visible surface.**
- New SQL-callable typed extractor functions in pg_proc, such as jsonb_object_field_numeric or jsonb_array_element_int4, one per extractor family and target type.
- Planner support functions set as prosupport on existing jsonb cast functions such as "numeric"(jsonb) and bool(jsonb).
- EXPLAIN VERBOSE output shows the rewritten typed extractor call instead of the cast.

**What the author claims.**
- Object-field casts to numeric and bool run about 2.1 to 2.2 times faster in a microbenchmark.
- Users get the speedup without changing queries and without new operators.
- Shared helpers and macros keep the code duplication from the many typed extractors under control.
- jsonb_path_query is set-returning, so this prosupport model cannot optimize it.
- Text, date and uuid targets need different semantics, so they are out of scope.

**Thread state.** Pavel Stehule gave +1 and Dmitry Dolgov approved v1 and asked for SubscriptingRef support, which was added. Andy Fan questioned the duplication in the explicit-function approach; CI failed on v5 in the jsonpath-first path, v6 was posted, and CFbot now reports the patch needs a rebase.

## 2. Prior attempts

### Add missing type conversion functions for PL/Python (related)

- **Link:** [https://commitfest.postgresql.org/patch/1499/](https://commitfest.postgresql.org/patch/1499/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAL_NLpKXMwv2ewZbtvUzujZxMgUq++6TgNTFRnY4+Tcsx7+1GQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAL_NLpKXMwv2ewZbtvUzujZxMgUq++6TgNTFRnY4+Tcsx7+1GQ@mail.gmail.com)
- **Date:** decided 2018-10-02; started 2018-01-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Heikki Linnakangas, Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Heikki Linnakangas found that the new tests showed changed behavior: on unpatched master, returning 100::float8 from a function declared to return int2 fails with "invalid input syntax for integer", but with the patch it succeeds. He said such behavior changes are not wanted, and the author agreed. No revised patch followed, and Michael Paquier returned the entry citing the behavior changes.
- **Reason (quote):** "So this patch is making some subtle changes to behavior. I don't think we want that." ([message](https://www.postgresql.org/message-id/68a01d38-441e-482f-82a7-f0d2f375fa90%40iki.fi))
- **What is different this time:** P9 changed PL/Python output conversion, not jsonb casts. It is an analogy only: a faster direct conversion path was returned because it changed results.


## 3. Objections a reviewer is likely to raise

### 1. Rewritten casts must behave exactly like the casts they replace

fixable · level 2, rubric item 7 · prior rejection, planner behaviour change, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A user runs `(j->'a')::int2` on a value such as 1.5, 70000 or a string. The query that gets rewritten returns a different value or a different error than the same cast that is not rewritten, for example a direct call to `int2(jsonb)`. The user notices only when results or error handling differ between two forms of the same query.

In [P9], direct type conversions that bypassed the normal conversion path were returned because they changed results in subtle ways. Here, v4 onward drops the numeric-intermediate shape and adds direct typed extractors for int2, int4, int8, float4 and float8 (messages #109, #111). Message #67 already found one divergence in error text, and v12 restored the old text only with an extra argument. The material does not say whether v6 keeps rounding, overflow errors and error texts identical to master for every target type.

**Precedents.**
- [Add missing type conversion functions for PL/Python](https://commitfest.postgresql.org/patch/1499/) (Returned with feedback, 2018-10-02); thread: [link](https://www.postgresql.org/message-id/flat/CAL_NLpKXMwv2ewZbtvUzujZxMgUq++6TgNTFRnY4+Tcsx7+1GQ@mail.gmail.com)

Thread: [#67 (Andy Fan, 2023-08-31)](https://www.postgresql.org/message-id/CAKU4AWpM%3DC5NxumoHQB1%3Did0OBc63kUgMUWyRNO2%3DOrHj6ROyQ%40mail.gmail.com), [#68 (Andy Fan, 2023-09-01)](https://www.postgresql.org/message-id/CAKU4AWrYOYSXMmotwhqiErci7aMz--8LxfpLiSg%2B0%2B7XNJBcuA%40mail.gmail.com), [#109 (Haibo Yan, 2026-04-08)](https://www.postgresql.org/message-id/CABXr29HN04CqHBGZ_k6_jQmCL5FO59tDE4PUL9U56Rpu%3Dd4%3D0A%40mail.gmail.com), [#111 (Haibo Yan, 2026-04-10)](https://www.postgresql.org/message-id/CABXr29G35c6Qo-Ab4EpH24gHS%2BejjJGfhmW3nQ1eEK%2BRgt_9Wg%40mail.gmail.com), [#113 (Haibo Yan, 2026-04-27)](https://www.postgresql.org/message-id/CABXr29EZNEgjJVNRX3gLOjRZVjXT6tda2_m8DdxQTfr8s8mkEQ%40mail.gmail.com)

**Question:** Do tests show that every rewritten form returns the same values and the same errors as the unrewritten cast, for each target type, including rounding, overflow and type mismatch?

Counter-check: held. Message #109 drops the numeric-intermediate shape, and nothing in the v4 to v6 mails says that values, overflow errors and error text match master for each target type. [P9] concerns PL/Python and serves only as an analogy; the objection stands without it.

### 2. Typed extractors duplicate cast semantics that must stay identical

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A later patch changes how int4(jsonb) rounds, range-checks or words its error. The 35 typed extractors keep the old behaviour. `(j->'a')::int4` then returns a different result or error depending on whether the planner rewrote it, and users first see this as a report that the same query behaves differently in a PL/pgSQL expression and in a plain query.

The rewrite is correct only while each typed extractor matches the existing cast exactly, and nothing makes them fail together when one changes. Messages #67 and #68 show that the error text already diverged once. The brief says it is not known whether v6 keeps errors such as "cannot cast jsonb string to type integer". In [P9], direct conversion paths were returned because they changed behaviour.

**Precedents.**
- [Add missing type conversion functions for PL/Python](https://commitfest.postgresql.org/patch/1499/) (Returned with feedback, 2018-10-02); thread: [link](https://www.postgresql.org/message-id/flat/CAL_NLpKXMwv2ewZbtvUzujZxMgUq++6TgNTFRnY4+Tcsx7+1GQ@mail.gmail.com)

Thread: [#67 (Andy Fan, 2023-08-31)](https://www.postgresql.org/message-id/CAKU4AWpM%3DC5NxumoHQB1%3Did0OBc63kUgMUWyRNO2%3DOrHj6ROyQ%40mail.gmail.com), [#68 (Andy Fan, 2023-09-01)](https://www.postgresql.org/message-id/CAKU4AWrYOYSXMmotwhqiErci7aMz--8LxfpLiSg%2B0%2B7XNJBcuA%40mail.gmail.com), [#74 (Chapman Flack, 2023-09-13)](https://www.postgresql.org/message-id/43a988594ac91a63dc4bb49a94303a42%40anastigmatix.net), [#113 (Haibo Yan, 2026-04-27)](https://www.postgresql.org/message-id/CABXr29EZNEgjJVNRX3gLOjRZVjXT6tda2_m8DdxQTfr8s8mkEQ%40mail.gmail.com)

**Question:** Which mechanism, test or shared code path ensures that every typed extractor gives the same result and error as the cast it replaces, for every target type?

Counter-check: held. The series does not say whether the typed extractors share conversion and error code with the jsonb cast functions, so a later change to one can miss the other. The PL/pgSQL detail is weak, since message #45 concedes those expressions are planned too.

### 3. Second fast path overlaps JSON_VALUE ... RETURNING without shared code

fixable · level 2, rubric item 12 · design, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), catalog changes · `already-exists-1`

A later patch changes how jsonb scalars become numeric or bool, for example error codes or int2 overflow text. It updates the JSON_VALUE RETURNING path but misses the many typed extractors, or the reverse. Users then get different errors for `(j->'a')::int2` and `json_value(j, '$.a' returning int2)`, and only bug reports show it.

Message #85 shows JSON_VALUE ... RETURNING numeric already extracts a typed scalar without the jsonb round trip, and asks whether this patch should reuse or share code with it. Message #86 answers with one benchmark where the patch is faster (1.75 ms against 2.59 ms) and says code sharing is unclear. The v6 series, with five extractor families and seven target types, does not answer that question. A committer would likely ask why the support function cannot rewrite to the existing JSON_VALUE machinery, or why that machinery cannot be made faster and reused, before many single-purpose pg_proc entries are added.

Thread: [#85 (Peter Eisentraut, 2024-03-04)](https://www.postgresql.org/message-id/8102ff5b-b156-409e-a48f-e53e63a39b36%40eisentraut.org), [#86 (Andy Fan, 2024-03-04)](https://www.postgresql.org/message-id/8734t6c5rh.fsf%40163.com), [#113 (Haibo Yan, 2026-04-27)](https://www.postgresql.org/message-id/CABXr29EZNEgjJVNRX3gLOjRZVjXT6tda2_m8DdxQTfr8s8mkEQ%40mail.gmail.com)

**Question:** Why can the support function not rewrite into the existing JSON_VALUE RETURNING machinery, or share its scalar conversion code, and what is the measured cost of doing so?

Counter-check: held. Message #86 answers question 3a of message #85 only with "not sure if we could share some code", and the v4 to v6 mails add jsonb_path_query_first without returning to it. The error-drift scenario is weak, because the two spellings already differ on master.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The series adds typed extractor rows to pg_proc.dat, sets prosupport on existing jsonb cast functions and bumps catversion. |
| ABI / extension API | **yes** | The patch changes src/include/utils/jsonb.h, which extensions include. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | A SupportRequestSimplify function rewrites cast(extract(...)) into a typed extractor call at plan time, visible in EXPLAIN VERBOSE. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | One typed extractor per extractor family and target type must stay identical to the matching jsonb cast function. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | JSON_VALUE ... RETURNING already extracts typed scalars, and the series does not share code with it. |

## 5. Questions the author must answer before anyone reviews code

1. Do tests show that every rewritten form returns the same values and errors as the unrewritten cast, for each target type, on rounding, overflow, JSON null and type mismatch?
2. Do the typed extractors call the same conversion and error code as the jsonb cast functions, or do they carry their own copy?
3. Why can the support function not rewrite into the JSON_VALUE ... RETURNING machinery or share its scalar conversion code, and what would that cost in speed?
4. Does the support function rewrite a SubscriptingRef whose subscript is not a constant?
5. What did v6 change to fix the v5 CI failure in the jsonb_path_query_first path?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the jsonb cast and jsonpath execution code and planner support functions. The brief says no reviewer has commented on v4 to v6.
- **Liability:** The owner signs for each typed extractor matching its jsonb cast function in values and errors, for as long as both exist. The owner also signs for the new pg_proc rows.
- **Author follow-up:** The CommitFest entry lists a second author who posted v4 to v6 and a fix after the v5 CI failure. The patch has needed a rebase since 2026-11-09.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Casts of jsonb fields to numbers and bool are common, and the author measured about 2 times faster on small documents. The five-patch series touches jsonb.c, jsonfuncs.c, jsonpath_exec.c and pg_proc.dat, needs a rebase, and has had no review since v4.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Typed extractors may change cast results and errors for existing queries. Refuted (not specific): Duplicate of prior-attempts-1. This version also misplaces the CI failure, which was on v5; v6 was posted as the fix.
- `standards-compat-2` Dozens of new SQL-callable functions become a public contract. Refuted (already handled): Message #17 accepts SQL-callable helpers "under the hood", and messages #90 and #94 ask for explicit pg_proc functions. The rewrite runs at plan time, so stored views and pg_dump keep the cast unless a user writes the undocumented function by hand.
- `standards-compat-3` Overlap with the standard JSON_VALUE ... RETURNING is not addressed. Refuted (already handled): Messages #86 and #87 answer the desirability part: JSON_VALUE requires users to rewrite queries, which message #17 rejects, and it measured slower. Casts and JSON_VALUE already differ in error behavior on master through ON ERROR, so the patch does not create that split.
- `maintenance-fit-1` About 35 new SQL-callable functions become permanent catalog API. Refuted (not specific): Duplicate of standards-compat-2.
- `maintenance-fit-3` Overlap with JSON_VALUE ... RETURNING not answered for the new series. Refuted (not specific): Duplicate of already-exists-1.
- `demand-1` Full type and extractor matrix rests only on microbenchmarks. Refuted (already handled): Reviewers asked for the breadth: message #17 names array indexing and nested fields, #29 names array element and extract path, #100 names subscripting, and #9 lists the existing casts. A smaller gain on toasted documents is not a failure, and non-users pay no run-time cost.

### Review angles

- prior-attempts: 1 objection(s). Within the thread, the first design (a new operator, message #1) was dropped after objections in messages #7 and #17. The current support-function design is the one Tom Lane suggested in message #17, so that earlier rejection does not apply.
- standards-compat: 3 objection(s)
- replication-ondisk: 0 objection(s). The proposal changes no on-disk format, WAL record, replication stream, wire message or extension-visible struct. Its only persistent footprint is new built-in pg_proc rows and prosupport on existing jsonb cast functions, which pg_upgrade and pg_dump handle like any built-in function. One point needs the code to confirm: expression indexes and generated columns over `(j->'a')::numeric` must yield identical values on both paths, or indexes built before the upgrade would no longer match what new queries compute.
- security: 0 objection(s). The latest design does not touch privileges. It rewrites existing casts into new typed extractor functions that read the same jsonb value and return the same scalar, so the caller sees nothing new. The internal-return type-safety issue (messages #36, #37) no longer applies, because v6 uses typed functions. Two points need the code to confirm: that the new functions are not marked leakproof, which would change pushdown past security barrier views and RLS quals, and that direct SQL calls cannot crash, as they did in message #48.
- maintenance-fit: 3 objection(s). Hot-path cost for non-users looks small: the support function runs once at plan time and only on jsonb casts.
- already-exists: 1 objection(s). The base need is not met by `(j->'a')::numeric` at the same speed, and JSON_VALUE requires query changes, so the only objection from this angle is overlap with JSON_VALUE.
- demand: 1 objection(s). Demand for the basic idea is reasonable: several reviewers, including Tom Lane in message #17, want the common cast syntax to be fast without query changes. The objection covers only the breadth of the series.

### Precedents retrieved

- P1: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P2: [jsonb array-style subscription](https://commitfest.postgresql.org/patch/485/) (cf, Returned with feedback, 2016-03-21) - judged not relevant; reason summarised from its thread
- P3: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P4: [Precision loss casting float to numeric](https://commitfest.postgresql.org/patch/1541/) (cf, Returned with feedback, 2018-03-17) - judged not relevant; reason summarised from its thread
- P5: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P6: [CAST( ... ON DEFAULT)](https://commitfest.postgresql.org/patch/4074/) (cf, Rejected, 2023-03-28) - judged not relevant; reason summarised from its thread
- P7: [Add numeric_trim(numeric)](https://commitfest.postgresql.org/patch/429/) (cf, Returned with feedback, 2016-03-15) - judged not relevant; reason summarised from its thread
- P8: [Allow functions defined as casts to domains to be called during casting](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [Add missing type conversion functions for PL/Python](https://commitfest.postgresql.org/patch/1499/) (cf, Returned with feedback, 2018-10-02) - used in the review; reason summarised from its thread
- P10: [Direct converting numeric types to bool](https://commitfest.postgresql.org/patch/1577/) (cf, Rejected, 2018-03-01) - judged not relevant; reason summarised from its thread
- P11: [Implement type input functions for extended statistics types](https://commitfest.postgresql.org/patch/4114/) (cf, Withdrawn, 2023-03-01) - judged not relevant
- P12: [Adds a `to_typemod(type_name)` function](https://commitfest.postgresql.org/patch/1408/) (cf, Returned with feedback, 2018-03-01) - judged not relevant
- P13: [Extracting only the columns needed for a query](https://commitfest.postgresql.org/patch/2331/) (cf, Withdrawn, 2020-07-01) - judged not relevant
- P14: [optimize several list functions with SIMD intrinsics](https://commitfest.postgresql.org/patch/4238/) (cf, Returned with feedback, 2023-07-10) - judged not relevant
- P15: [Optimize memory allocation in function "bringetbitmap"](https://commitfest.postgresql.org/patch/390/) (cf, Returned with feedback, 2015-12-23) - judged not relevant
- P16: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P17: [JIT: Optimize generated functions earlier to lower memory usage](https://commitfest.postgresql.org/patch/2298/) (cf, Withdrawn, 2019-11-12) - judged not relevant
- P18: [Add internal Numeric to Uint64 conversion function.](https://commitfest.postgresql.org/patch/3550/) (cf, Returned with feedback, 2022-04-05) - judged not relevant
- P19: [Add Roman numeral conversion to to_number](https://commitfest.postgresql.org/patch/1227/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P20: [Add internal Numeric to Uint64 conversion function.](https://commitfest.postgresql.org/patch/3627/) (cf, Returned with feedback, 2022-12-14) - judged not relevant
- P21: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P22: [SQL/JSON support in PostgreSQL](https://commitfest.postgresql.org/patch/1063/) (cf, Returned with feedback, 2018-03-02) - judged not relevant
- P23: [Allow XML arrays to be cast to other data types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P24: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant

### Warnings

- 2 long messages in the middle of the thread were left out of the prompts

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 168 s; cost reported by the CLI: $7.55
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 93 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:57:35.077Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
