# Design pre-review: CAST(... ON DEFAULT) - WIP build on top of Error-Safe User Functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5941](https://commitfest.postgresql.org/patch/5941/); [thread](https://www.postgresql.org/message-id/flat/CADkLM%3Dfv1JfY4Ufa-jcwwNbjQixNViskQ8jZu3Tz_p656i_4hQ%40mail.gmail.com), 101 of 101 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support: the syntax is in the SQL standard and a committer committed the prerequisite patches. The design is not yet agreed. Reviewers and the author still disagree on which errors fall back to DEFAULT for nested and constant-folded casts (standards-compat-1). No committer has accepted ERROR { SAFE | UNSAFE }, which keeps two of the three concerns raised against CREATE CAST ... WITH SAFE FUNCTION (standards-compat-2).

Not checked: Without the code, the cost of the new function-call steps and of soft-error constant folding for plain queries could not be judged. The two LLM-generated WIP patches in v35 could not be assessed either.

## Prior attempts

None found before 2022-12-19.

## Objections a reviewer is likely to raise

### 1. Error-catching rules depend on constant folding, plan shape and nesting

An ETL script tests `cast(cast('a' as int) as text default null on conversion error)` with literals and gets an error, but the same query over a column returns NULL. A prepared statement may fold the cast in a custom plan but not in a generic plan. The user finds out in production, when the same data gives a different result.

Message #75 shows two nested casts with the same value that differ only in whether the literal is typed: one returns NULL, the other errors. Messages #99 and #100 disagree on whether evaluation is top-down or bottom-up, and #99 adds an ARRAY-only flag, `arg_errorsafe`. The proposal does not tie these rules to the standard text. Once released, whatever v35 does becomes the behaviour users depend on.

**Question:** Which errors fall back to DEFAULT, stated once as a rule tied to the standard text, and does that rule give the same result for literals, columns, parameters and nested casts?

<sub>fixable, level 2, design; messages [#73](https://www.postgresql.org/message-id/CACJufxH_1EtEBMb0JvxaM3Gmnt33HYrS37m5eYjJ_OfBkMVFJg%40mail.gmail.com), [#75](https://www.postgresql.org/message-id/CACJufxHGox47X4zNtVeNw7H%3DSZ7ATAAD5SEYpvxa2o5mPV0XCA%40mail.gmail.com), [#87](https://www.postgresql.org/message-id/CACJufxGqn5fQCPkUeRC4FQLMSFZ9vBMqq65zXmsRnp5OAMOkaA%40mail.gmail.com), [#99](https://www.postgresql.org/message-id/CACJufxG8U4_%2BC-Hsr%3DpDs3wE1rK1mjdzoON8MPMFi0x8XwShYQ%40mail.gmail.com), [#100](https://www.postgresql.org/message-id/CADkLM%3Df3pTngq1R3FeVaMThFTwcYx0e6babYjuGTYHzqvq2m8A%40mail.gmail.com), [#101](https://www.postgresql.org/message-id/CADkLM%3DfUkbJzzFOvZyEKL9mJePU5EJe4%3Dsb9FFD_CkMbeghTCA%40mail.gmail.com); `standards-compat-1`</sub>

### 2. ERROR SAFE function option repeats the churn that CREATE CAST SAFE drew

An extension makes its cast and input functions use ereturn(), but it does not ship a new version script with `alter function ... error safe`. `cast(x as citext default ... on conversion error)` then fails at parse time on every row. The user finds out at the first run, and pg_input_is_valid() on the same type still works.

Message #70 rejected CREATE CAST ... WITH SAFE FUNCTION because it forces new extension SQL files, and because error-safe input functions got only a documentation note. Moving the marker to CREATE FUNCTION (#89, #94) answers the first point but not the second or third. #94 now also gates input-function coercion on proerrorsafe, so CAST DEFAULT and pg_input_is_valid() use different rules for the same input function. No committer has accepted the new syntax.

**Question:** Why is a catalog flag that every extension must set needed, when pg_input_is_valid() already relies on the documented ereturn() convention without one?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#70](https://www.postgresql.org/message-id/5ae9578e-f25e-49c5-97ab-ad27bc2050b5%40eisentraut.org), [#89](https://www.postgresql.org/message-id/CAAAe_zDuTTbDGy_OPrcw3DEHqcjMdrWoo3954v03r6HogGQ37w%40mail.gmail.com), [#94](https://www.postgresql.org/message-id/CACJufxEqoju91FZub%2BFPWJ3HGSTuH8NtAC5Qg3u5Vihcj8yyCw%40mail.gmail.com), [#96](https://www.postgresql.org/message-id/CADkLM%3Dfu2TSd3DjDVC4EvfVXaz9o4f3GsCQh7%2BVrhhYF2F0PWw%40mail.gmail.com), [#98](https://www.postgresql.org/message-id/CADkLM%3De%3DUPR%3DA9VeVRU5yGAsC3M5h9_1XE-G087Mfu3qqz6csw%40mail.gmail.com); `standards-compat-2`</sub>

### 3. Smaller first step suggested by the committer is not taken

The project commits new grammar, a catalog column, new CREATE FUNCTION syntax and executor opcodes in one step. Nested-cast semantics are still disputed, so each future answer to that dispute becomes a user-visible behavior change to standard syntax.

Message #70 proposes pg_cast_conversion_succeeds(srcvalue, desttype) as a first step, mirroring pg_input_is_valid(), so the higher-level syntax can wait. Message #81 proposes limiting the first version to casts that CoerceViaIO can handle. Messages #98 and #100 show the reviewer and the author still disagree on whether inner casts and constant-folded failures fall back. v35 still ships all of it together.

**Question:** Why not commit pg_cast_conversion_succeeds() or a CoerceViaIO-only subset first, and settle the nested-cast rules before fixing them in standard syntax?

<sub>fixable, level 2, design; messages [#70](https://www.postgresql.org/message-id/5ae9578e-f25e-49c5-97ab-ad27bc2050b5%40eisentraut.org), [#81](https://www.postgresql.org/message-id/CADkLM%3DcXsprF2GvcmqOG%2BME_N2iyHNr_--xW-6ZaKd32yF_NzA%40mail.gmail.com), [#98](https://www.postgresql.org/message-id/CADkLM%3De%3DUPR%3DA9VeVRU5yGAsC3M5h9_1XE-G087Mfu3qqz6csw%40mail.gmail.com), [#100](https://www.postgresql.org/message-id/CADkLM%3Df3pTngq1R3FeVaMThFTwcYx0e6babYjuGTYHzqvq2m8A%40mail.gmail.com); `maintenance-fit-4`</sub>

### 4. Syntax ships while its standard definition is still being amended

Postgres releases DEFAULT clauses that accept any expression, including aggregates, window functions and subqueries (message #99). If the standard amendment then restricts the DEFAULT operand, every query using the broader form becomes a Postgres-only dialect that cannot be narrowed without breaking users.

In messages #2 and #8, the risk named was that the committee would change the spelling, and the agreed plan was to wait for something solid. Message #84, in April 2026, still amends the paper: NULL ON CONVERSION ERROR is removed and the DEFAULT operand becomes an implicitly typed value specification. The patch dropped NULL ON CONVERSION ERROR (#85), but it does not say how the broad DEFAULT form matches the amended text.

**Question:** Does the amended standard text allow any value expression, including aggregates and subqueries, as the DEFAULT operand, and has that amendment been accepted?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#2](https://www.postgresql.org/message-id/1432468.1672675047%40sss.pgh.pa.us), [#8](https://www.postgresql.org/message-id/1911498.1672773346%40sss.pgh.pa.us), [#16](https://www.postgresql.org/message-id/762ae707-7fdc-43d8-a77a-3a10d12ce21d%40postgresfriends.org), [#84](https://www.postgresql.org/message-id/4e00626a-66ca-45af-9866-30d022523b37%40postgresfriends.org), [#85](https://www.postgresql.org/message-id/CACJufxFpyVfbUfWWTA4osVSy7fhNC4H%2BLzxRLFS65Z%2B9na4thg%40mail.gmail.com), [#99](https://www.postgresql.org/message-id/CACJufxG8U4_%2BC-Hsr%3DpDs3wE1rK1mjdzoON8MPMFi0x8XwShYQ%40mail.gmail.com); `standards-compat-3`</sub>

## Questions for the author

1. What single rule decides which errors fall back to DEFAULT, and does it give the same result for literals, columns, parameters and nested casts?
2. Why does CAST ... DEFAULT need a pg_proc flag that every extension must set, when pg_input_is_valid() relies on the ereturn() convention without one?
3. Has the standard amendment from message #84 been accepted, and does it allow aggregates and subqueries as the DEFAULT operand?
4. Why not commit pg_cast_conversion_succeeds() or a CoerceViaIO-only subset first, and add the DEFAULT syntax after the nested-cast rules are agreed?
5. Should a missing schema or relation in a reg* cast fall back to DEFAULT or raise an error?

## Touch points

- **on-disk format / pg_upgrade:** New pg_proc.proerrorsafe column needs a catversion bump, and upgraded extension functions stay unmarked until an update script runs.
- **catalog changes:** Adds pg_proc.proerrorsafe and sets it in pg_proc.dat for built-in cast and input functions.
- **ABI / extension API:** The fmgr README widens the ErrorSaveContext contract, and parse_coerce.h, optimizer.h and ParseState change.
- **SQL-standard conflict or non-standard syntax:** Adds standard CAST ... ON CONVERSION ERROR grammar and nonstandard ERROR { SAFE | UNSAFE } on CREATE and ALTER FUNCTION.
- **planner behaviour change:** clauses.c folds constants under a soft-error context, and a WIP patch changes when DEFAULT is evaluated.
- **pg_dump / restore round-trip:** pg_dump.c and ruleutils.c must emit the ERROR SAFE attribute and deparse the new cast form in views.
- **maintenance burden ("who maintains this in 10 years"):** New expression node, executor opcodes, JIT steps and a per-function flag that every cast function author must keep correct.
- **performance on the hot path:** Touches execExpr.c, execExprInterp.c, llvmjit_expr.c and eval_const_expressions; the thread gives no measurements for plain queries.

## Who would commit it, and when

- Owner: silent. Peter Eisentraut committed the prerequisite patches and removed himself as committer on 2026-04-10. The owner needs to know the parser, constant folding, expression execution and JIT.
- Priority: fix design first; audience a sizeable minority; review cost high. ETL users and users of standard SQL gain, and the prerequisites are committed. The main patch is about 160 KiB across parser, planner, executor and JIT, and the nested-cast rule and the ERROR SAFE marker are not yet agreed.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 9 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
