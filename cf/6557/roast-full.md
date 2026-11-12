# Design pre-review: POC: PLpgSQL FOREACH IN JSON ARRAY

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6557](https://commitfest.postgresql.org/patch/6557/) (Needs review); authors: Pavel Stehule (okbobcz)
- Thread: [POC: PLpgSQL FOREACH IN JSON ARRAY](https://www.postgresql.org/message-id/flat/CAFj8pRD9Jjv%2Bm%3D6DP6vWWn5NnZeTPH9eoPFCnA1JE5hRKRDMxQ%40mail.gmail.com), started 2026-02-28
- Messages read: 28 of 28
- Patch prose read: v20260803-0001-FOREACH-scalar-IN-ARRAY-jsonb_expr.patch, v20260228-1-0001-FOREACH-scalar-IN-JSON-ARRAY.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Two committers asked for a generic mechanism and did not question the goal, so the demand objections do not stop review at level 1b. The design is not yet agreed. Tom Lane's concern in message #20 that the iterator should not know the target type has no reply after the options struct (maintenance-fit-1). The author also leaves the casting default open (maintenance-fit-2), and nobody discussed putting by-name binding under IN ARRAY (standards-compat-3).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the supported iterator modes, the handling of plain json input, and the real cost of moving coercion into PL/pgSQL could not be checked. Only 3 of 28 messages were shown in full; later messages are known through the objections.

## The proposal, restated

**Problem.** PL/pgSQL code that loops over JSON array elements today uses FOR IN SELECT over jsonb_array_elements, which runs the SQL executor for each loop. The author wants a faster and more readable loop for PL/pgSQL authors who process JSON arrays.

**Proposed design.** The first version added new syntax FOREACH target IN JSON ARRAY expr. After Peter Eisentraut and Tom Lane asked for a generic mechanism, the patch now extends SubscriptRoutines with an optional iterator constructor, implements it for arrays and jsonb, and lets the existing FOREACH ... IN ARRAY accept a jsonb expression. JSON object fields go to a composite variable or a list of variables by name, and the iterator takes an optional target type and coercion mode.

**User-visible surface.**
- FOREACH target IN ARRAY now accepts a jsonb expression; the earlier IN JSON ARRAY syntax is gone from the latest patch.
- Object elements are assigned to a composite target or a variable list by field name, not by position.
- SLICE greater than zero is rejected for jsonb arrays; a RECORD target needs a known structure first.
- C API: SubscriptRoutines gets an optional iterator constructor, with options for result type and an ForeachAIteratorMode enum.
- Error messages from existing FOREACH IN ARRAY cases may change, as Tom Lane said was acceptable.

**What the author claims.**
- The loop is about 3 to 4 times faster than FOR IN SELECT over jsonb_array_elements, best case on a 1000-element array.
- Jim Jones reported about 3 times faster in his own test.
- The feature is new and has no compatibility issues.
- The iterator must know the target type to do jsonb-to-composite conversion and to avoid I/O casts.
- Splitting iterate and coerce into separate callbacks makes scalar loops about 2 times slower, slower than FOR IN SELECT.

**Thread state.** Tom Lane questioned whether the iterator should know the target type and benchmarked a separate coerce() callback; Pavel answered with an options struct, now passed to the constructor. Nobody has replied to the last two versions, and the entry is in Needs review after a Ready for Committer status was reverted.

## 2. Prior attempts

### iteration over record in plpgsql (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/124/](https://commitfest.postgresql.org/patch/124/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAFj8pRDCUgKQ57viHtcciybjyzqJn9=oA+0-reFHYF4LJSxjJg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAFj8pRDCUgKQ57viHtcciybjyzqJn9=oA+0-reFHYF4LJSxjJg@mail.gmail.com)
- **Date:** decided 2015-07-07; started 2015-01-16
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Jim Nasby (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane objected that row_to_array casts all fields to text and drops SQL datatypes, and said row_to_json already covers the case. He also asked that the two features be split and that FOREACH get its own discussion. Jim Nasby worried that the FOREACH pairing rule would block iteration that follows array dimensions. The later messages move to typed iteration over records and broader dynamic typing, and no revised design was agreed.
- **Reason (quote):** "While I don't have a problem with hstore_to_array, I don't think that row_to_array is a very good idea; it's basically encouraging people to throw away SQL datatypes altogether and imagine that everything is text." ([message](https://www.postgresql.org/message-id/25419.1427653646%40sss.pgh.pa.us))
- **What is different this time:** P1 extended FOREACH with a key/value variable list over a row turned into a text array. This proposal assigns jsonb object fields by name to typed targets with coercion, so it does not reduce values to text.


## 3. Objections a reviewer is likely to raise

### 1. Iterator API in SubscriptRoutines is still shaped for PL/pgSQL

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

An extension author adds iteration to a subscriptable type such as hstore. The constructor takes a target type and a five-value ForeachAIteratorMode, so the author must write coercion and composite conversion logic that only PL/pgSQL defines. Each type's iterator then casts slightly differently, and users find out when the same FOREACH behaves differently per type.

In message #20, Tom Lane said the point of using SubscriptRoutines is a general facility, and that iterators probably should not know the target type. The v20260803 patch still passes the result type and a coercion mode to the constructor (messages #26, #28), and PL/pgSQL remains the only caller. Message #25 says a separate coerce step is slower than FOR IN SELECT, but its own table shows the int case at about 48 ms against about 61 ms for FOR IN SELECT. A cleaner API would still be faster than today's FOR IN SELECT loop.

Thread: [#20 (Tom Lane, 2026-07-01)](https://www.postgresql.org/message-id/3648848.1782942004%40sss.pgh.pa.us), [#22 (Pavel Stehule, 2026-07-07)](https://www.postgresql.org/message-id/CAFj8pRC8uULe7Zk%3Dj4DSUf0Xu2b%3D6_mEn%3D4PfKOYmP7rvES%3D1Q%40mail.gmail.com), [#24 (Tom Lane, 2026-07-23)](https://www.postgresql.org/message-id/1474498.1784830027%40sss.pgh.pa.us), [#25 (Pavel Stehule, 2026-07-24)](https://www.postgresql.org/message-id/CAFj8pRDEXOarQ3Ym%3DndLtUm6P%2BL9htRv7kPdk6nnb4T12D-nFw%40mail.gmail.com), [#26 (Pavel Stehule, 2026-07-27)](https://www.postgresql.org/message-id/CAFj8pRAw4sBhS%2BRH2%2BABoaTqTBKRe_zqyy2dxmpPKdyF7uvxgg%40mail.gmail.com), [#28 (Pavel Stehule, 2026-08-03)](https://www.postgresql.org/message-id/CAFj8pRCC1zCPeYfVJLFPKqtth5xHHf7vVTCLTd4H%3D5MSW2H5JA%40mail.gmail.com)

**Question:** Which ForeachAIteratorMode values must every iterator support, and why can coercion not stay in PL/pgSQL, given that the coerce-callback numbers still beat FOR IN SELECT?

Counter-check: held. Tom Lane's concern in message #20 has no reply after the options struct in messages #26 to #28. The speed claim holds only for int, since the numeric case in the same mode takes about 72 ms against 61 ms for FOR IN SELECT.

### 2. Casting defaults fixed under old syntax before strict or lax is decided

fixable · level 2, rubric item 6 · design, maintenance burden ("who maintains this in 10 years"), SQL-standard conflict or non-standard syntax · `maintenance-fit-2`

A release ships FOREACH x IN ARRAY jsonb_expr with the current mixed rule, where 3.14 casts to int but "3.14" fails. Users write functions that rely on this. A later release that adds lax mode or ERROR ON ERROR clauses cannot change the default without silently changing how those functions behave.

Messages #1 and #28 and the latest commit message leave the strict-versus-lax question open and invite discussion, but nobody in the thread has answered it. Message #21 shows that plain assignment from jsonb and FOREACH conversion already follow different rules. Reusing the existing IN ARRAY keyword means the default casting behaviour is part of a long-lived language contract, so it should be settled before commit and not after.

Thread: [#1 (Pavel Stehule, 2026-02-28)](https://www.postgresql.org/message-id/CAFj8pRD9Jjv%2Bm%3D6DP6vWWn5NnZeTPH9eoPFCnA1JE5hRKRDMxQ%40mail.gmail.com), [#21 (Pavel Stehule, 2026-07-05)](https://www.postgresql.org/message-id/CAFj8pRA%2BtuLvb4kZVEeLjNOK7-oqKwdgTuBu3ofNGkVU4RO2RQ%40mail.gmail.com), [#25 (Pavel Stehule, 2026-07-24)](https://www.postgresql.org/message-id/CAFj8pRDEXOarQ3Ym%3DndLtUm6P%2BL9htRv7kPdk6nnb4T12D-nFw%40mail.gmail.com), [#28 (Pavel Stehule, 2026-08-03)](https://www.postgresql.org/message-id/CAFj8pRCC1zCPeYfVJLFPKqtth5xHHf7vVTCLTd4H%3D5MSW2H5JA%40mail.gmail.com)

**Question:** Which casting default is proposed for jsonb elements, and how would later ERROR ON ERROR clauses be added without changing what existing functions do?

Counter-check: held. The latest commit message states a preference for the old strict behaviour but still presents the choice as an open question, and no reviewer answered it. Optional clauses could be added later over a strict default, so this stays fixable.

### 3. IN ARRAY becomes the spelling for any subscriptable type

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, ABI / extension API · `standards-compat-3`

An extension type such as hstore adds the new iterator, and FOREACH k IN ARRAY h starts to loop over a value that is not an array, with its own target-binding and slicing rules. Readers of the function cannot tell from the statement whether the loop binds by position or by name.

The docs already say other FOREACH variants for non-array composites "may be added in future", which suggests a separate keyword. The latest patch drops IN JSON ARRAY and puts jsonb under IN ARRAY, though jsonb loops bind by name and reject SLICE > 0. Message #13 backs a generic iterator, but the thread has not settled the user-facing spelling.

Thread: [#13 (Tom Lane, 2026-03-12)](https://www.postgresql.org/message-id/4166.1773289811%40sss.pgh.pa.us), [#16 (Pavel Stehule, 2026-03-17)](https://www.postgresql.org/message-id/CAFj8pRDHdO9bLM4%3D8%3Dp0MF8rNpSKdw-F_%3DzG%3DYxWui8S4vn5RQ%40mail.gmail.com), [#28 (Pavel Stehule, 2026-08-03)](https://www.postgresql.org/message-id/CAFj8pRCC1zCPeYfVJLFPKqtth5xHHf7vVTCLTd4H%3D5MSW2H5JA%40mail.gmail.com)

**Question:** Why should non-array subscriptable types share the IN ARRAY spelling instead of a distinct FOREACH variant, given that their binding rules differ?

Counter-check: held. Message #13 supports a generic iterator but says nothing about the user-facing keyword. The hstore case is hypothetical, but for jsonb alone the latest patch already puts by-name binding and a SLICE restriction under IN ARRAY without discussion.

### 4. JSON keys bind to local variable names, and misses become NULLs

fixable · level 2, rubric item 10 · design, SQL-standard conflict or non-standard syntax · `standards-compat-2`

A developer renames local variables x, y to v_x, v_y in a function that uses FOREACH x, y IN ARRAY. The loop still runs, but no key matches, so every iteration assigns NULL to both variables. Nobody sees a problem until wrong data shows up downstream.

The documentation hunk shows that missing keys give NULL with no error, as in the '{}' row. Local variable names are internal to PL/pgSQL today. This rule makes them part of the data contract, which no other PL/pgSQL target list does, since arrays and composites bind by position. Whether a missing key should raise an error belongs to the undecided strict and lax question.

Thread: [#1 (Pavel Stehule, 2026-02-28)](https://www.postgresql.org/message-id/CAFj8pRD9Jjv%2Bm%3D6DP6vWWn5NnZeTPH9eoPFCnA1JE5hRKRDMxQ%40mail.gmail.com), [#15 (Pavel Stehule, 2026-03-12)](https://www.postgresql.org/message-id/CAFj8pRBUd1Ojs8O%2BcJY3qmruuM%3DQPby_XTJtL9d6YtUX15OvTQ%40mail.gmail.com)

**Question:** What stops a variable rename from turning every value into NULL, and why is no error raised when no key matches any target variable?

Counter-check: held. Message #1 explains why binding is by name, and jsonb_populate_record also gives NULL for missing keys. But the thread does not discuss a variable list where no key matches, and the rename scenario is concrete.

### 5. Benchmarks skip the existing idiom of building an array once, then FOREACH

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A PL/pgSQL author who needs a fast loop over a jsonb array can already write `foreach x in array array(select jsonb_array_elements_text($1)::int)`, or array_agg over jsonb_populate_recordset for objects. That runs the executor once, not per row. If this idiom is close in speed, core carries a new iterator API in SubscriptRoutines and new FOREACH semantics for a small gain.

Every speed figure in the thread compares the patch only with FOR IN SELECT over jsonb_array_elements (message #1, #4, #25). The materialize-then-FOREACH idiom uses only existing features, and message #25 shows FOREACH over a ready array at about 11 ms against 25 ms for the patch. Name-based mapping of objects to a composite also exists already through jsonb_populate_recordset and jsonb_to_recordset. So the measured gain over the best existing approach is unknown, and so is the readability gain.

Thread: [#1 (Pavel Stehule, 2026-02-28)](https://www.postgresql.org/message-id/CAFj8pRD9Jjv%2Bm%3D6DP6vWWn5NnZeTPH9eoPFCnA1JE5hRKRDMxQ%40mail.gmail.com), [#4 (Jim Jones, 2026-03-03)](https://www.postgresql.org/message-id/d6ac54b9-79f4-485b-a540-96c56cd0b991%40uni-muenster.de), [#25 (Pavel Stehule, 2026-07-24)](https://www.postgresql.org/message-id/CAFj8pRDEXOarQ3Ym%3DndLtUm6P%2BL9htRv7kPdk6nnb4T12D-nFw%40mail.gmail.com)

**Question:** How does the patch compare with FOREACH over an array built once by array(select jsonb_array_elements...) or array_agg over jsonb_populate_recordset, for scalars and composites?

Counter-check: held. Every figure in the thread compares only with FOR IN SELECT, and the array-built-once idiom can do the job for scalars and composites. The 11 ms control excludes the cost of building the array, so the real gap is unknown in both directions.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | SubscriptRoutines in subscripting.h gets an optional iterator constructor with a target type and a ForeachAIteratorMode option. |
| SQL-standard conflict or non-standard syntax | **yes** | FOREACH ... IN ARRAY accepts jsonb with by-name binding and a casting rule that differs from the SQL/JSON lax default. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new iterator contract for every subscriptable type, and casting semantics in PL/pgSQL that cannot change after release. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | FOR IN SELECT over jsonb_array_elements and FOREACH over an array built once already cover the use case. |

## 5. Questions the author must answer before anyone reviews code

1. Why can coercion not stay in PL/pgSQL, given that message #25 shows the separate coerce callback at about 48 ms against 61 ms for FOR IN SELECT in the int case?
2. Which ForeachAIteratorMode values does PL/pgSQL use, and which must every type's iterator support?
3. Which casting default will FOREACH over jsonb keep, and how would a later ERROR ON ERROR clause fit without changing existing functions?
4. Why should jsonb share the IN ARRAY spelling when it binds object fields by name and rejects SLICE greater than zero?
5. How does the patch compare with FOREACH over an array built once by array(select jsonb_array_elements...), for scalar and composite targets?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane is a listed reviewer and commented on the iterator design in message #20, but has not replied to the last two versions. The owner must know PL/pgSQL execution and the subscripting support API.
- **Liability:** The owner signs for the iterator contract in SubscriptRoutines that extension types may implement, and for the jsonb casting and by-name binding rules in FOREACH. Both are permanent once released.
- **Author follow-up:** The author posted new versions after each review round and moved the entry to the next CommitFest in October 2026, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The thread names no workload where jsonb loop overhead is the bottleneck, and the gain applies mainly to loops with a pure PL/pgSQL body. The patch is about 40 KiB in pl_exec.c and the subscripting code, and it adds an extension-facing API whose shape a committer questioned without a visible answer.

## Appendix

### Held up in the counter-check, not listed above

- `demand-1` (fixable, level 1b) Speedup shown only for empty loop bodies, with no production workload. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-1` Default cast behaviour for jsonb elements is undecided but fixed once shipped. Refuted (not specific): Duplicate of maintenance-fit-2, which describes the current rule more exactly as mixed (3.14 casts to int, "3.14" fails) and cites message #21.

### Review angles

- prior-attempts: 0 objection(s). No precedent gives a rejection reason that still applies. P1's objection was about dropping SQL datatypes through text, and this design keeps target types. P3's objection to a PL/pgSQL-only mechanism was addressed when the patch moved to SubscriptRoutines after messages #11 and #13. P10 records no reason. The open question about target-type knowledge in the iterator (messages #20, #24) belongs to the design angle.
- standards-compat: 3 objection(s). Three objections found within the standards and compatibility angle.
- replication-ondisk: 0 objection(s). The proposal changes no on-disk format, WAL record, catalog, or wire protocol. Its only cross-boundary surface is an optional field appended to SubscriptRoutines; extensions are rebuilt for each major release, and a NULL field means no iterator, so existing subscript handlers keep working. Whether the iterator API itself is general enough is a design question for another angle.
- security: 0 objection(s). The proposal has no privilege or exposure aspect worth raising. FOREACH iterates over a jsonb value the function already holds, under the caller's existing privileges, and adds no catalog object, role, GUC or SQL-visible function. The iterator runs inside PL/pgSQL, away from query quals, so leakproofness and RLS are not involved. Per-element memory is reset after each call, and large jsonb arrays can already be expanded with jsonb_array_elements. One point could not be checked without the code: whether a third-party subscripting handler's iterator constructor runs with any extra trust.
- maintenance-fit: 2 objection(s). Hot-path cost is not a concern: the change touches only FOREACH and an optional trailing SubscriptRoutines field. Ownership is a committability issue, not an objection. Tom Lane is engaged but has not replied to the last two versions.
- already-exists: 1 objection(s). One objection raised. No extension is known to provide this loop, and the subscripting hook needs core placement.
- demand: 1 objection(s). Peter Eisentraut and Tom Lane engaged with the design and did not question its value, which lowers rejection confidence. The objection asks for evidence of demand and is not a rejection.

### Precedents retrieved

- P1: [iteration over record in plpgsql](https://commitfest.postgresql.org/patch/124/) (cf, Returned with feedback, 2015-07-07) - used in the review; reason summarised from its thread
- P2: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - judged not relevant; reason summarised from its thread
- P3: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - used in the review; reason summarised from its thread
- P4: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P5: [jsonb array-style subscription](https://commitfest.postgresql.org/patch/485/) (cf, Returned with feedback, 2016-03-21) - judged not relevant; reason summarised from its thread
- P6: [de-TOAST'ing using a iterator](https://commitfest.postgresql.org/patch/2194/) (cf, Returned with feedback, 2020-11-27) - judged not relevant; reason summarised from its thread
- P7: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P8: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5974/) (cf, Withdrawn, 2025-08-19) - judged not relevant; reason summarised from its thread
- P9: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5975/) (cf, Withdrawn, 2025-12-31) - judged not relevant; reason summarised from its thread
- P10: [Generic type subscription](https://commitfest.postgresql.org/patch/793/) (cf, Returned with feedback, 2017-01-31) - used in the review
- P11: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P12: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P13: [loop over unbound cursor](https://commitfest.postgresql.org/patch/2558/) (cf, Withdrawn, 2020-08-10) - judged not relevant
- P14: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant
- P15: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P16: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P17: [Revert "Stabilize output of new regression test case".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cbadba8dd632fc0d4162f7d686fec631bce7dfd0) (revert, Reverted, 2018-02-14) - judged not relevant
- P18: [Make the am_xxx function Asserts consistent](https://commitfest.postgresql.org/patch/4570/) (cf, Withdrawn, 2024-01-09) - judged not relevant
- P19: [Rename trim_array() for consistency with the rest of array_* functions](https://commitfest.postgresql.org/patch/5335/) (cf, Withdrawn, 2024-11-04) - judged not relevant
- P20: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P21: [array_random](https://commitfest.postgresql.org/patch/5881/) (cf, Withdrawn, 2026-01-06) - judged not relevant
- P22: [num_sa_scans in genericcostestimate](https://commitfest.postgresql.org/patch/3871/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P23: [Revert badly-broken patch to dump comments on composite-type columns.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=18661f282bae5599abb68f53bc8b451b2d67bd75) (revert, Reverted, 2004-08-06) - judged not relevant
- P24: [Generic WAL compression](https://commitfest.postgresql.org/patch/1363/) (cf, Returned with feedback, 2018-03-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 170 s; cost reported by the CLI: $3.32
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 86 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:54:21.642Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
