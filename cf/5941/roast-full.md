# Design pre-review: CAST(... ON DEFAULT) - WIP build on top of Error-Safe User Functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5941](https://commitfest.postgresql.org/patch/5941/) (Needs review); authors: Jian He (stayreal)
- Thread: [CAST(... ON DEFAULT) - WIP build on top of Error-Safe User Functions](https://www.postgresql.org/message-id/flat/CADkLM%3Dfv1JfY4Ufa-jcwwNbjQixNViskQ8jZu3Tz_p656i_4hQ%40mail.gmail.com), started 2022-12-19
- Messages read: 101 of 101
- Patch prose read: v35-0005-WIP-Short-circuit-default-of-CAST-DEFAULT-ON-CON.patch, v35-0001-error-safe-for-casting-text-to-other-types-per-p.patch, v35-0003-CAST-expr-AS-newtype-DEFAULT-expr-ON-CONVERSION-.patch, v35-0002-introduce-pg_proc.proerrorsafe.patch, v35-0004-WIP-Reorganize-Safe-Cast-operations-to-avoid-uni.patch, 0001-add-OidInputFunctionCall.patch, 0002-add-stringTypeDatumSafe.patch, 0003-CAST-ON-DEFAULT-work-in-progress.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support: the syntax is in the SQL standard and a committer committed the prerequisite patches. The design is not yet agreed. Reviewers and the author still disagree on which errors fall back to DEFAULT for nested and constant-folded casts (standards-compat-1). No committer has accepted ERROR { SAFE | UNSAFE }, which keeps two of the three concerns raised against CREATE CAST ... WITH SAFE FUNCTION (standards-compat-2).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the cost of the new function-call steps and of soft-error constant folding for plain queries could not be judged. The two LLM-generated WIP patches in v35 could not be assessed either.

## The proposal, restated

**Problem.** A failed type cast aborts the whole query, so users cannot fall back to a value when one input does not convert. The thread points to bulk ETL from all-text tables and to existing TRY_CAST in SQL Server and the Oracle CAST clause. The syntax was accepted into the SQL standard after SQL:2023.

**Proposed design.** The parser builds a new SafeTypeCastExpr that evaluates only the conversion with an ErrorSaveContext and evaluates the DEFAULT expression only when that conversion fails softly. A new pg_proc.proerrorsafe flag marks functions that can report soft errors, and a cast is accepted only if its function is marked. The planner folds constants under a soft-error context, with a WIP patch to avoid evaluating DEFAULT when the cast succeeds.

**User-visible surface.**
- SQL syntax: CAST(expr AS type DEFAULT expr ON CONVERSION ERROR) and CAST(expr AS type ERROR ON CONVERSION ERROR).
- CREATE FUNCTION and ALTER FUNCTION option ERROR { UNSAFE | SAFE }.
- New catalog column pg_proc.proerrorsafe, set true for built-in cast and input functions.
- New error at parse time when the cast function is not error safe, for example in citext casts.
- fmgr README: any function call in an expression compiled with an ErrorSaveContext may receive one.

**What the author claims.**
- Almost all built-in pg_cast functions are now error safe; many type patches were committed in March 2026.
- Making only part of an expression tree error safe is not feasible, so some nested cases behave differently.
- Not all functions are error safe, so CAST ... DEFAULT cannot be fully consistent in every case.
- The DEFAULT expression is evaluated normally and can raise its own error.
- proerrorsafe puts no load on the executor, according to Corey Huinker.

**Thread state.** Peter Eisentraut committed many prerequisite patches, rejected CREATE CAST ... WITH SAFE FUNCTION and suggested a pg_cast_conversion_succeeds() function instead; the patch then moved to PG20 CommitFests. Corey Huinker and Henson Choi still disagree with the author on nested casts, constant folding order and reg* lookup failures, and v35 adds two LLM-generated WIP patches.

## 2. Prior attempts

No precedent found (corpus searched for items before 2022-12-19; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Error-catching rules depend on constant folding, plan shape and nesting

fixable · level 2, rubric item 6 · design, SQL-standard conflict or non-standard syntax, planner behaviour change · `standards-compat-1`

An ETL script tests `cast(cast('a' as int) as text default null on conversion error)` with literals and gets an error, but the same query over a column returns NULL. A prepared statement may fold the cast in a custom plan but not in a generic plan. The user finds out in production, when the same data gives a different result.

Message #75 shows two nested casts with the same value that differ only in whether the literal is typed: one returns NULL, the other errors. Messages #99 and #100 disagree on whether evaluation is top-down or bottom-up, and #99 adds an ARRAY-only flag, `arg_errorsafe`. The proposal does not tie these rules to the standard text. Once released, whatever v35 does becomes the behaviour users depend on.

Thread: [#73 (jian he, 2026-03-26)](https://www.postgresql.org/message-id/CACJufxH_1EtEBMb0JvxaM3Gmnt33HYrS37m5eYjJ_OfBkMVFJg%40mail.gmail.com), [#75 (jian he, 2026-03-28)](https://www.postgresql.org/message-id/CACJufxHGox47X4zNtVeNw7H%3DSZ7ATAAD5SEYpvxa2o5mPV0XCA%40mail.gmail.com), [#87 (jian he, 2026-06-23)](https://www.postgresql.org/message-id/CACJufxGqn5fQCPkUeRC4FQLMSFZ9vBMqq65zXmsRnp5OAMOkaA%40mail.gmail.com), [#99 (jian he, 2026-10-30)](https://www.postgresql.org/message-id/CACJufxG8U4_%2BC-Hsr%3DpDs3wE1rK1mjdzoON8MPMFi0x8XwShYQ%40mail.gmail.com), [#100 (Corey Huinker, 2026-11-05)](https://www.postgresql.org/message-id/CADkLM%3Df3pTngq1R3FeVaMThFTwcYx0e6babYjuGTYHzqvq2m8A%40mail.gmail.com), [#101 (Corey Huinker, 2026-11-06)](https://www.postgresql.org/message-id/CADkLM%3DfUkbJzzFOvZyEKL9mJePU5EJe4%3Dsb9FFD_CkMbeghTCA%40mail.gmail.com)

**Question:** Which errors fall back to DEFAULT, stated once as a rule tied to the standard text, and does that rule give the same result for literals, columns, parameters and nested casts?

Counter-check: held. Message #75 shows two nested casts of the same value with different results, and messages #99 and #100 still disagree on evaluation order. The author's summary in message #94 calls the inconsistency unsolvable and states no single rule.

### 2. ERROR SAFE function option repeats the churn that CREATE CAST SAFE drew

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, catalog changes, pg_dump / restore round-trip, ABI / extension API · `standards-compat-2`

An extension makes its cast and input functions use ereturn(), but it does not ship a new version script with `alter function ... error safe`. `cast(x as citext default ... on conversion error)` then fails at parse time on every row. The user finds out at the first run, and pg_input_is_valid() on the same type still works.

Message #70 rejected CREATE CAST ... WITH SAFE FUNCTION because it forces new extension SQL files, and because error-safe input functions got only a documentation note. Moving the marker to CREATE FUNCTION (#89, #94) answers the first point but not the second or third. #94 now also gates input-function coercion on proerrorsafe, so CAST DEFAULT and pg_input_is_valid() use different rules for the same input function. No committer has accepted the new syntax.

Thread: [#70 (Peter Eisentraut, 2026-03-24)](https://www.postgresql.org/message-id/5ae9578e-f25e-49c5-97ab-ad27bc2050b5%40eisentraut.org), [#89 (Henson Choi, 2026-06-29)](https://www.postgresql.org/message-id/CAAAe_zDuTTbDGy_OPrcw3DEHqcjMdrWoo3954v03r6HogGQ37w%40mail.gmail.com), [#94 (jian he, 2026-07-07)](https://www.postgresql.org/message-id/CACJufxEqoju91FZub%2BFPWJ3HGSTuH8NtAC5Qg3u5Vihcj8yyCw%40mail.gmail.com), [#96 (Corey Huinker, 2026-07-27)](https://www.postgresql.org/message-id/CADkLM%3Dfu2TSd3DjDVC4EvfVXaz9o4f3GsCQh7%2BVrhhYF2F0PWw%40mail.gmail.com), [#98 (Corey Huinker, 2026-10-14)](https://www.postgresql.org/message-id/CADkLM%3De%3DUPR%3DA9VeVRU5yGAsC3M5h9_1XE-G087Mfu3qqz6csw%40mail.gmail.com)

**Question:** Why is a catalog flag that every extension must set needed, when pg_input_is_valid() already relies on the documented ereturn() convention without one?

Counter-check: held. Message #88 argues that an unmarked design surprises users, and #96 calls the default false a protection. No committer has answered the input-function and extension-churn points from message #70, and #94 gates I/O coercion on a flag that pg_input_is_valid() does not use.

### 3. Smaller first step suggested by the committer is not taken

fixable · level 2, rubric item 11 · design, maintenance burden ("who maintains this in 10 years"), SQL-standard conflict or non-standard syntax · `maintenance-fit-4`

The project commits new grammar, a catalog column, new CREATE FUNCTION syntax and executor opcodes in one step. Nested-cast semantics are still disputed, so each future answer to that dispute becomes a user-visible behavior change to standard syntax.

Message #70 proposes pg_cast_conversion_succeeds(srcvalue, desttype) as a first step, mirroring pg_input_is_valid(), so the higher-level syntax can wait. Message #81 proposes limiting the first version to casts that CoerceViaIO can handle. Messages #98 and #100 show the reviewer and the author still disagree on whether inner casts and constant-folded failures fall back. v35 still ships all of it together.

Thread: [#70 (Peter Eisentraut, 2026-03-24)](https://www.postgresql.org/message-id/5ae9578e-f25e-49c5-97ab-ad27bc2050b5%40eisentraut.org), [#81 (Corey Huinker, 2026-03-31)](https://www.postgresql.org/message-id/CADkLM%3DcXsprF2GvcmqOG%2BME_N2iyHNr_--xW-6ZaKd32yF_NzA%40mail.gmail.com), [#98 (Corey Huinker, 2026-10-14)](https://www.postgresql.org/message-id/CADkLM%3De%3DUPR%3DA9VeVRU5yGAsC3M5h9_1XE-G087Mfu3qqz6csw%40mail.gmail.com), [#100 (Corey Huinker, 2026-11-05)](https://www.postgresql.org/message-id/CADkLM%3Df3pTngq1R3FeVaMThFTwcYx0e6babYjuGTYHzqvq2m8A%40mail.gmail.com)

**Question:** Why not commit pg_cast_conversion_succeeds() or a CoerceViaIO-only subset first, and settle the nested-cast rules before fixing them in standard syntax?

Counter-check: held. Message #71 answers the staged step only by calling it IS CASTABLE, and #59 defers that to a separate patch. The committer kept reviewing the full patch (#76), so staging is a choice and no condition, but the nested-cast rules are still open.

### 4. Syntax ships while its standard definition is still being amended

fixable · level 2, rubric item 6 · SQL-standard conflict or non-standard syntax · `standards-compat-3`

Postgres releases DEFAULT clauses that accept any expression, including aggregates, window functions and subqueries (message #99). If the standard amendment then restricts the DEFAULT operand, every query using the broader form becomes a Postgres-only dialect that cannot be narrowed without breaking users.

In messages #2 and #8, the risk named was that the committee would change the spelling, and the agreed plan was to wait for something solid. Message #84, in April 2026, still amends the paper: NULL ON CONVERSION ERROR is removed and the DEFAULT operand becomes an implicitly typed value specification. The patch dropped NULL ON CONVERSION ERROR (#85), but it does not say how the broad DEFAULT form matches the amended text.

Thread: [#2 (Tom Lane, 2023-01-02)](https://www.postgresql.org/message-id/1432468.1672675047%40sss.pgh.pa.us), [#8 (Tom Lane, 2023-01-03)](https://www.postgresql.org/message-id/1911498.1672773346%40sss.pgh.pa.us), [#16 (Vik Fearing, 2025-07-22)](https://www.postgresql.org/message-id/762ae707-7fdc-43d8-a77a-3a10d12ce21d%40postgresfriends.org), [#84 (Vik Fearing, 2026-04-02)](https://www.postgresql.org/message-id/4e00626a-66ca-45af-9866-30d022523b37%40postgresfriends.org), [#85 (jian he, 2026-04-03)](https://www.postgresql.org/message-id/CACJufxFpyVfbUfWWTA4osVSy7fhNC4H%2BLzxRLFS65Z%2B9na4thg%40mail.gmail.com), [#99 (jian he, 2026-10-30)](https://www.postgresql.org/message-id/CACJufxG8U4_%2BC-Hsr%3DpDs3wE1rK1mjdzoON8MPMFi0x8XwShYQ%40mail.gmail.com)

**Question:** Does the amended standard text allow any value expression, including aggregates and subqueries, as the DEFAULT operand, and has that amendment been accepted?

Counter-check: held. In message #21 the standard's author called the aggregate restriction arbitrary, and a Postgres superset of the standard is no failure. But the thread does not say whether the amendment in message #84 was accepted, which was the condition set in message #8.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | New pg_proc.proerrorsafe column needs a catversion bump, and upgraded extension functions stay unmarked until an update script runs. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | Adds pg_proc.proerrorsafe and sets it in pg_proc.dat for built-in cast and input functions. |
| ABI / extension API | **yes** | The fmgr README widens the ErrorSaveContext contract, and parse_coerce.h, optimizer.h and ParseState change. |
| SQL-standard conflict or non-standard syntax | **yes** | Adds standard CAST ... ON CONVERSION ERROR grammar and nonstandard ERROR { SAFE \| UNSAFE } on CREATE and ALTER FUNCTION. |
| new GUC | no |  |
| planner behaviour change | **yes** | clauses.c folds constants under a soft-error context, and a WIP patch changes when DEFAULT is evaluated. |
| pg_dump / restore round-trip | **yes** | pg_dump.c and ruleutils.c must emit the ERROR SAFE attribute and deparse the new cast form in views. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | New expression node, executor opcodes, JIT steps and a per-function flag that every cast function author must keep correct. |
| performance on the hot path | **yes** | Touches execExpr.c, execExprInterp.c, llvmjit_expr.c and eval_const_expressions; the thread gives no measurements for plain queries. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What single rule decides which errors fall back to DEFAULT, and does it give the same result for literals, columns, parameters and nested casts?
2. Why does CAST ... DEFAULT need a pg_proc flag that every extension must set, when pg_input_is_valid() relies on the ereturn() convention without one?
3. Has the standard amendment from message #84 been accepted, and does it allow aggregates and subqueries as the DEFAULT operand?
4. Why not commit pg_cast_conversion_succeeds() or a CoerceViaIO-only subset first, and add the DEFAULT syntax after the nested-cast rules are agreed?
5. Should a missing schema or relation in a reg* cast fall back to DEFAULT or raise an error?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Peter Eisentraut committed the prerequisite patches and removed himself as committer on 2026-04-10. The owner needs to know the parser, constant folding, expression execution and JIT.
- **Liability:** Permanent standard syntax whose nested-cast and folding behaviour becomes a compatibility promise, a pg_proc column and CREATE FUNCTION option carried by pg_dump for good, and new executor and JIT steps.
- **Author follow-up:** The author posted v35, keeps the patch rebased, and moved the entry to PG20-3 himself on 2026-09-02.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** ETL users and users of standard SQL gain, and the prerequisites are committed. The main patch is about 160 KiB across parser, planner, executor and JIT, and the nested-cast rule and the ERROR SAFE marker are not yet agreed.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Error-safe marker keeps the costs of the rejected CREATE CAST SAFE design. Refuted (not specific): Duplicate of standards-compat-2, which makes the same point and adds the mismatch with pg_input_is_valid().
- `prior-attempts-2` Committer's smaller first step was set aside while the patch grew. Refuted (not specific): Duplicate of maintenance-fit-4, which also covers the CoerceViaIO-only subset from message #81.
- `prior-attempts-3` Standard text was still being amended in 2026. Refuted (not specific): Duplicate of standards-compat-3.
- `replication-ondisk-1` proerrorsafe brings back the extension upgrade churn rejected in message #70. Refuted (not specific): Duplicate of standards-compat-2. The pg_upgrade case is the same extension update cost seen from another side.
- `maintenance-fit-1` No committer owns the patch since the prerequisite work landed. Refuted (wrong facts): A committer commented on the main patch in message #70 and said in #76 he would keep reviewing it; he withdrew on the day the entry moved to PG20. Ownership belongs in the committability field and does not show a design flaw.
- `maintenance-fit-2` proerrorsafe is an unchecked promise that future changes can break silently. Refuted (already handled): An unchecked declaration copies IMMUTABLE and PARALLEL SAFE, as message #98 notes, and the documentation says errors not reported softly abort as usual. A missed hard-error path raises an error, which is loud; the extension-script point duplicates standards-compat-2.
- `maintenance-fit-3` Soft-error plumbing spreads into common parser and planner paths. Refuted (no failure scenario): The scenario is a future mistake by another patch, with no concrete regression. The design keeps a separate node and separate opcodes so plain casts are not burdened (messages #23, #55, #97), though measurements would still be a fair request.
- `already-exists-1` Text-source fallback is already possible with pg_input_is_valid(). Refuted (already handled): pg_input_is_valid() cannot test function-based casts, which round differently from I/O conversion (message #24), as the objection itself says. The staged-path question is covered by maintenance-fit-4.
- `demand-1` Demand shown is for text-source ETL, which pg_input_is_valid already covers. Refuted (already handled): The material shows lasting demand: the syntax is in the SQL standard, SQL Server and Oracle have equivalents, message #2 approves of the goal, and a committer committed the prerequisite patches. proerrorsafe defaults to false, so unmarked functions cost only users of the feature.

### Review angles

- prior-attempts: 3 objection(s). No listed precedent matches; [P7] concerns object-address functions and its recorded reason does not apply. The relevant earlier rejections are inside this thread (messages #9 and #70).
- standards-compat: 3 objection(s). Three objections are raised. The pg_dump changes for proerrorsafe follow the existing practice for function attributes, so they are not raised here.
- replication-ondisk: 1 objection(s). Apart from the new pg_proc column, the feature adds no on-disk, WAL or wire-protocol state. Stored views go through ruleutils deparse, so pg_upgrade does not depend on the new node layout. Internal API changes such as evaluate_expr and ParseState affect only major releases.
- security: 0 objection(s). No security objection found. Marking ERROR SAFE grants no new capability: C functions already need superuser, SQL and PL/pgSQL functions ignore the context, and a function that is wrongly marked still raises its errors as usual. For reg* casts, catalog existence is already visible, and message #90 shows the schema USAGE check stays a hard error. Not checked without the code: whether any soft path in the reg* lookups could also turn a permission or lock error into a fallback to DEFAULT.
- maintenance-fit: 4 objection(s). Objections listed; none rest on precedents, since [P7] concerns object address functions and its recorded reason does not apply here.
- already-exists: 1 objection(s). The standard syntax and the function-based casts are not covered by any existing feature I can name, so this angle raises only the convenience question.
- demand: 1 objection(s). Demand for the core syntax looks real: the syntax is accepted into the SQL standard, Oracle has the same clause and SQL Server has TRY_CAST. Several independent reviewers also want it, so the remaining question is only how much surface beyond the text-ETL case is justified.

### Precedents retrieved

- P1: [Allow functions defined as casts to domains to be called during casting](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P2: [Allow values to be cast to domain types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P3: [Precision loss casting float to numeric](https://commitfest.postgresql.org/patch/1541/) (cf, Returned with feedback, 2018-03-17) - judged not relevant; reason summarised from its thread
- P4: [Add internal Numeric to Uint64 conversion function.](https://commitfest.postgresql.org/patch/3550/) (cf, Returned with feedback, 2022-04-05) - judged not relevant; reason summarised from its thread
- P5: [Add Roman numeral conversion to to_number](https://commitfest.postgresql.org/patch/1227/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P6: [Add internal Numeric to Uint64 conversion function.](https://commitfest.postgresql.org/patch/3627/) (cf, Returned with feedback, 2022-12-14) - judged not relevant; reason summarised from its thread
- P7: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - used in the review; reason summarised from its thread
- P8: [Allow XML arrays to be cast to other data types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [Add missing type conversion functions for PL/Python](https://commitfest.postgresql.org/patch/1499/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P10: [Revert error-throwing wrappers for the printf family of functions.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0c071936e94c6859afb2ec8d2c8dddf7bcdab7ee) (revert, Reverted, 2015-05-19) - judged not relevant
- P11: [Support ORDER BY in CREATE FUNCTION for Set Returning Functions](https://commitfest.postgresql.org/patch/88/) (cf, Rejected, 2015-05-03) - judged not relevant; reason summarised from its thread
- P12: [conversion from size in human readable text format to text](https://commitfest.postgresql.org/patch/432/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P13: [Revert "Test conversion of NaN between float4 and float8."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5c4c771dafefa7d0ee80a6bd6955b7a3940452a1) (revert, Reverted, 2018-05-05) - judged not relevant
- P14: [psql: default to ON_ERROR_STOP](https://commitfest.postgresql.org/patch/3446/) (cf, Returned with feedback, 2022-01-17) - judged not relevant; reason summarised from its thread
- P15: [Revert yesterday's change to make pg_cast.h say 'OID = 0' in DATA entries. On reflection, we ought to get rid of that mechanism entirely.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=45c79a3094a246e94ed6725c7ade0a61bd3fe56a) (revert, Reverted, 2005-04-13) - judged not relevant
- P16: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant
- P17: [Role Attribute Bitmask Representation](https://commitfest.postgresql.org/patch/85/) (cf, Rejected, 2015-01-12) - judged not relevant
- P18: [Allow to warn or error on ... = NULL expressions](https://commitfest.postgresql.org/patch/1728/) (cf, Returned with feedback, 2018-12-01) - judged not relevant
- P19: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P20: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant
- P21: [Return ERROR on COMMIT instead of silently ROLLBACK](https://commitfest.postgresql.org/patch/2522/) (cf, Returned with feedback, 2021-03-25) - judged not relevant
- P22: [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://commitfest.postgresql.org/patch/2664/) (cf, Returned with feedback, 2020-11-24) - judged not relevant
- P23: [Move tablespace path re-creation from the makefiles to pg_regress (test for mkdir function in the backend)](https://commitfest.postgresql.org/patch/3067/) (cf, Withdrawn, 2021-04-16) - judged not relevant
- P24: [Revert to 9.6 treatment of ALTER TYPE enumtype ADD VALUE.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=28e07270768518524291d7d7906668eb67f6b8a5) (revert, Reverted, 2017-09-27) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 219 s; cost reported by the CLI: $10.52
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 113 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:37:31.642Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
