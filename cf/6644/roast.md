# Design pre-review: Extract numeric filed in JSONB more effectively

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6644](https://commitfest.postgresql.org/patch/6644/); [thread](https://www.postgresql.org/message-id/flat/CAKU4AWoqAVya6PBhn%2BBCbFaBMt3z-2%3Di5fKO3bW%3D6HPhbid2Dw%40mail.gmail.com), 114 of 114 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at level 1, and the support-function design follows the direction given in message #17. Three fixable level 2 points remain. The v4 to v6 mails do not show that rewritten casts return the same values and errors as master (prior-attempts-1). Nothing keeps the typed extractors and the cast functions in step later (maintenance-fit-2). The code-sharing question about JSON_VALUE from message #85 is still open (already-exists-1).

Not checked: Without the code, nobody can confirm that the typed extractors match the casts on every input. That also decides whether expression indexes over `(j->'a')::numeric` stay valid. The leakproof marking and the safety of direct SQL calls to the new functions are also unchecked.

## Prior attempts

- [Add missing type conversion functions for PL/Python](https://www.postgresql.org/message-id/flat/CAL_NLpKXMwv2ewZbtvUzujZxMgUq++6TgNTFRnY4+Tcsx7+1GQ@mail.gmail.com) (related; Returned with feedback, 2018-10-02). "So this patch is making some subtle changes to behavior. I don't think we want that." ([message](https://www.postgresql.org/message-id/68a01d38-441e-482f-82a7-f0d2f375fa90%40iki.fi)) Different now: P9 changed PL/Python output conversion, not jsonb casts. It is an analogy only: a faster direct conversion path was returned because it changed results.

## Objections a reviewer is likely to raise

### 1. Rewritten casts must behave exactly like the casts they replace

A user runs `(j->'a')::int2` on a value such as 1.5, 70000 or a string. The query that gets rewritten returns a different value or a different error than the same cast that is not rewritten, for example a direct call to `int2(jsonb)`. The user notices only when results or error handling differ between two forms of the same query.

In [P9], direct type conversions that bypassed the normal conversion path were returned because they changed results in subtle ways. Here, v4 onward drops the numeric-intermediate shape and adds direct typed extractors for int2, int4, int8, float4 and float8 (messages #109, #111). Message #67 already found one divergence in error text, and v12 restored the old text only with an extra argument. The material does not say whether v6 keeps rounding, overflow errors and error texts identical to master for every target type.

Precedent: [Add missing type conversion functions for PL/Python](https://www.postgresql.org/message-id/flat/CAL_NLpKXMwv2ewZbtvUzujZxMgUq++6TgNTFRnY4+Tcsx7+1GQ@mail.gmail.com) (Returned with feedback, 2018-10-02)

**Question:** Do tests show that every rewritten form returns the same values and the same errors as the unrewritten cast, for each target type, including rounding, overflow and type mismatch?

<sub>fixable, level 2, prior rejection; messages [#67](https://www.postgresql.org/message-id/CAKU4AWpM%3DC5NxumoHQB1%3Did0OBc63kUgMUWyRNO2%3DOrHj6ROyQ%40mail.gmail.com), [#68](https://www.postgresql.org/message-id/CAKU4AWrYOYSXMmotwhqiErci7aMz--8LxfpLiSg%2B0%2B7XNJBcuA%40mail.gmail.com), [#109](https://www.postgresql.org/message-id/CABXr29HN04CqHBGZ_k6_jQmCL5FO59tDE4PUL9U56Rpu%3Dd4%3D0A%40mail.gmail.com), [#111](https://www.postgresql.org/message-id/CABXr29G35c6Qo-Ab4EpH24gHS%2BejjJGfhmW3nQ1eEK%2BRgt_9Wg%40mail.gmail.com), [#113](https://www.postgresql.org/message-id/CABXr29EZNEgjJVNRX3gLOjRZVjXT6tda2_m8DdxQTfr8s8mkEQ%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Typed extractors duplicate cast semantics that must stay identical

A later patch changes how int4(jsonb) rounds, range-checks or words its error. The 35 typed extractors keep the old behaviour. `(j->'a')::int4` then returns a different result or error depending on whether the planner rewrote it, and users first see this as a report that the same query behaves differently in a PL/pgSQL expression and in a plain query.

The rewrite is correct only while each typed extractor matches the existing cast exactly, and nothing makes them fail together when one changes. Messages #67 and #68 show that the error text already diverged once. The brief says it is not known whether v6 keeps errors such as "cannot cast jsonb string to type integer". In [P9], direct conversion paths were returned because they changed behaviour.

Precedent: [Add missing type conversion functions for PL/Python](https://www.postgresql.org/message-id/flat/CAL_NLpKXMwv2ewZbtvUzujZxMgUq++6TgNTFRnY4+Tcsx7+1GQ@mail.gmail.com) (Returned with feedback, 2018-10-02)

**Question:** Which mechanism, test or shared code path ensures that every typed extractor gives the same result and error as the cast it replaces, for every target type?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#67](https://www.postgresql.org/message-id/CAKU4AWpM%3DC5NxumoHQB1%3Did0OBc63kUgMUWyRNO2%3DOrHj6ROyQ%40mail.gmail.com), [#68](https://www.postgresql.org/message-id/CAKU4AWrYOYSXMmotwhqiErci7aMz--8LxfpLiSg%2B0%2B7XNJBcuA%40mail.gmail.com), [#74](https://www.postgresql.org/message-id/43a988594ac91a63dc4bb49a94303a42%40anastigmatix.net), [#113](https://www.postgresql.org/message-id/CABXr29EZNEgjJVNRX3gLOjRZVjXT6tda2_m8DdxQTfr8s8mkEQ%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Second fast path overlaps JSON_VALUE ... RETURNING without shared code

A later patch changes how jsonb scalars become numeric or bool, for example error codes or int2 overflow text. It updates the JSON_VALUE RETURNING path but misses the many typed extractors, or the reverse. Users then get different errors for `(j->'a')::int2` and `json_value(j, '$.a' returning int2)`, and only bug reports show it.

Message #85 shows JSON_VALUE ... RETURNING numeric already extracts a typed scalar without the jsonb round trip, and asks whether this patch should reuse or share code with it. Message #86 answers with one benchmark where the patch is faster (1.75 ms against 2.59 ms) and says code sharing is unclear. The v6 series, with five extractor families and seven target types, does not answer that question. A committer would likely ask why the support function cannot rewrite to the existing JSON_VALUE machinery, or why that machinery cannot be made faster and reused, before many single-purpose pg_proc entries are added.

**Question:** Why can the support function not rewrite into the existing JSON_VALUE RETURNING machinery, or share its scalar conversion code, and what is the measured cost of doing so?

<sub>fixable, level 2, design; messages [#85](https://www.postgresql.org/message-id/8102ff5b-b156-409e-a48f-e53e63a39b36%40eisentraut.org), [#86](https://www.postgresql.org/message-id/8734t6c5rh.fsf%40163.com), [#113](https://www.postgresql.org/message-id/CABXr29EZNEgjJVNRX3gLOjRZVjXT6tda2_m8DdxQTfr8s8mkEQ%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Do tests show that every rewritten form returns the same values and errors as the unrewritten cast, for each target type, on rounding, overflow, JSON null and type mismatch?
2. Do the typed extractors call the same conversion and error code as the jsonb cast functions, or do they carry their own copy?
3. Why can the support function not rewrite into the JSON_VALUE ... RETURNING machinery or share its scalar conversion code, and what would that cost in speed?
4. Does the support function rewrite a SubscriptingRef whose subscript is not a constant?
5. What did v6 change to fix the v5 CI failure in the jsonb_path_query_first path?

## Touch points

- **catalog changes:** The series adds typed extractor rows to pg_proc.dat, sets prosupport on existing jsonb cast functions and bumps catversion.
- **ABI / extension API:** The patch changes src/include/utils/jsonb.h, which extensions include.
- **planner behaviour change:** A SupportRequestSimplify function rewrites cast(extract(...)) into a typed extractor call at plan time, visible in EXPLAIN VERBOSE.
- **maintenance burden ("who maintains this in 10 years"):** One typed extractor per extractor family and target type must stay identical to the matching jsonb cast function.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** JSON_VALUE ... RETURNING already extracts typed scalars, and the series does not share code with it.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the jsonb cast and jsonpath execution code and planner support functions. The brief says no reviewer has commented on v4 to v6.
- Priority: needs champion; audience a sizeable minority; review cost medium. Casts of jsonb fields to numbers and bool are common, and the author measured about 2 times faster on small documents. The five-patch series touches jsonb.c, jsonfuncs.c, jsonpath_exec.c and pg_proc.dat, needs a rebase, and has had no review since v4.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
