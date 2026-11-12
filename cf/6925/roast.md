# Design pre-review: Add PRODUCT() aggregate function

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6925](https://commitfest.postgresql.org/patch/6925/); [thread](https://www.postgresql.org/message-id/flat/CAM2%2B6%3DWG8gpOti%2B1-N_ra0mNcNrdhMjSJcDvcA7wzjLD-qKH3g%40mail.gmail.com), 29 of 29 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No level fails. The only surviving objection, standards-compat-1, is a fixable design point: v3 documents product(real) as returning double precision, while sum(real) returns real, and the thread says the two match. Demand is supported by the report in message #10 that the next SQL standard requires PRODUCT. A committer is already listed on the CommitFest entry.

Not checked: The code was not read, so the accumulator functions' null handling, float overflow checks and parallel combine behaviour are unverified. Only 3 of 29 messages were visible, so later agreement or disagreement in the thread may be missing.

## Prior attempts

None found before 2026-06-23.

## Objections a reviewer is likely to raise

### 1. product(real) returns double precision while sum(real) returns real

A user writes a query or view with sum(r) and product(r) over a real column, expecting both results to be real like the input. The product column comes back as double precision. This changes the result type of views, the types that drivers bind, and the output of CREATE TABLE AS. The user finds out only when inspecting column types or when client code checks them.

Message #26 asked for the float variants to match sum(float4)/sum(float8), and messages #27 and #28 state that v3 matches them. But sum(float4) returns real, and v3 documents product(real) as returning double precision. A float8 transition state may be justified because products overflow real quickly. Even so, the commit message and docs do not say that the result type departs from sum(), and the claim of consistency in the thread is not accurate for real.

**Question:** Should product(real) return real like sum(real), and if not, can the docs and commit message state the difference and the reason for it?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#26](https://www.postgresql.org/message-id/CA%2BvB%3DAEcvBbkkFUh%2BDCm9jMqr-T-_T0v%3DM1uGL1xD1tGSnCv%2BA%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/CAM2%2B6%3DXp%3DFaW6rOCwNzWoxH7SjjEyw%2BncyuoLr_%2BLuNEdEDLFA%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAM2%2B6%3DUOZTGe9x7HAdNQAuQPVANo8_KuTBwPmqwT%2BWJQ-uyQ6A%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Should product(real) return real like sum(real), and if not, can the docs and commit message state the difference and the reason?
2. Should product() be committed before the standard's ON EMPTY syntax, or wait for the other thread?
3. Does Dean Rasheed accept the per-type integer variants after the reply about exact integer results?
4. How does product() compare in speed with sum() and with exp(sum(ln(x))) on the same data?

## Touch points

- **catalog changes:** The patch adds pg_proc and pg_aggregate rows and needs a CATALOG_VERSION_NO bump.
- **SQL-standard conflict or non-standard syntax:** Message #10 says the next SQL standard defines PRODUCT with an ON EMPTY clause, which this patch leaves to another thread.
- **pg_dump / restore round-trip:** An existing user aggregate named product with the same argument types is shadowed by the pg_catalog one after upgrade.
- **maintenance burden ("who maintains this in 10 years"):** Six aggregate variants and four new accumulator functions in numeric.c stay in core for good.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A user can already create the same aggregate with numeric_mul as transition and combine function.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Eisentraut proposed the idea and is listed as committer on the CommitFest entry; the work needs someone who knows the numeric and float aggregate code.
- Priority: proceed; audience a sizeable minority; review cost low. The patch adds catalog rows and small accumulator functions that reuse numeric_mul and float8mul, and costs nothing for users who do not call product(). A committer is listed and a reviewer has no more comments on v3; the open point is the result type for real.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
