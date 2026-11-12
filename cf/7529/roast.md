# Design pre-review: Support casts from numeric to oid8

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7529](https://commitfest.postgresql.org/patch/7529/); [thread](https://www.postgresql.org/message-id/flat/TY5PR01MB18445597DD6EC7AB442CFA7A8FA642%40TY5PR01MB18445.jpnprd01.prod.outlook.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection stops the idea at level 1 or 2. already-exists-1 asks for demand, but the cost is one pg_cast row and one function, and the cast mirrors the existing numeric-to-int8 cast. standards-compat-1 is a fixable design point: the proposal gives no reason for accepting negatives down to -18446744073709551615 in assignment context. Only one message is visible, so the list's view of the wraparound rule is unknown.

Not checked: Without the code, the range check, the soft-error path and the volatility and leakproof marking of the new function could not be checked.

## Prior attempts

None found before 2026-10-23.

## Objections a reviewer is likely to raise

### 1. Quoted literal already covers the full oid8 range

A user who writes 18446744073709551615::oid8 gets an error today. Writing '18446744073709551615'::oid8, or value::text::oid8 for a computed numeric, already gives the result. The patch adds a permanent pg_cast and pg_proc entry so that the quotes can be dropped.

Message #1 states that the quoted form works, so the existing way can do the job with two extra characters. For computed numeric values, a cast through text reaches the existing oid8 input function. Message #1 shows no field report or user request, and oid8 values above INT64_MAX are rare in practice. A committer would likely ask whether this convenience justifies a new catalog cast that must be kept forever.

**Question:** Which users or tools write oid8 values above INT64_MAX as bare literals or numeric expressions, and why does quoting or casting through text not work for them?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/TY5PR01MB18445597DD6EC7AB442CFA7A8FA642%40TY5PR01MB18445.jpnprd01.prod.outlook.com); `already-exists-1`</sub>

### 2. Assignment cast wraps negative numeric values into large oid8 values

An application inserts a computed numeric such as -1.0 or -18446744073709551615 into an oid8 column. The row silently stores 18446744073709551615 or 1 instead of raising an error. The user finds out later, when lookups by that identifier return the wrong object or nothing.

The cast applies in assignment context, so INSERT and UPDATE take this path without any explicit request. Message #1 justifies the wraparound by pointing to oid8 text input, but text input is a compatibility rule for strings, and a cast from an arithmetic type is a different contract. The doc context the patch extends already shows `INSERT INTO mytable VALUES(-1);  -- fails` for the OID types, so -1 would fail while -1.0 succeeds. The bound on the absolute value also accepts negatives that wrap to small positive values, which look valid.

**Question:** Why should an assignment cast from numeric wrap negative values instead of raising "out of range" as the other integer-to-OID conversions do?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/TY5PR01MB18445597DD6EC7AB442CFA7A8FA642%40TY5PR01MB18445.jpnprd01.prod.outlook.com); `standards-compat-1`</sub>

## Questions for the author

1. Why does the cast accept negative numeric values down to -18446744073709551615 in assignment context, where they wrap to small positive oid8 values, instead of raising an out-of-range error?
2. Which users or tools write oid8 values above INT64_MAX as unquoted literals or numeric expressions, and why does quoting or a cast through text not work for them?
3. Does an oid8-to-numeric cast exist, and if not, should this patch add it so the conversion has both directions?
4. Are matching casts for oid, or from other integer types to oid8, in scope for this work?

## Touch points

- **catalog changes:** The patch adds one pg_cast row and one pg_proc row for the numeric-to-oid8 conversion.
- **SQL-standard conflict or non-standard syntax:** The new assignment cast rounds fractional numeric input and wraps negative values modulo 2^64 without an error.
- **maintenance burden ("who maintains this in 10 years"):** The rounding and wraparound rules of the cast become behaviour that applications can depend on after release.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #1 states that the quoted form '18446744073709551615'::oid8 already works.

## Who would commit it, and when

- Owner: silent. A committer who knows the numeric code and the oid8 type would have to own it; nobody has replied to message #1.
- Priority: proceed; audience niche; review cost low. The patch is about 7.5 KiB, touches numeric.c and two catalog data files, and helps only users who write oid8 values above INT64_MAX as numeric. The negative-range rule should be settled on the list before commit.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
