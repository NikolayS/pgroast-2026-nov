# Design pre-review: Add PRODUCT() aggregate function

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6925](https://commitfest.postgresql.org/patch/6925/) (Needs review); authors: Jeevan Chalke (jeevan.chalke)
- Thread: [Add PRODUCT() aggregate function](https://www.postgresql.org/message-id/flat/CAM2%2B6%3DWG8gpOti%2B1-N_ra0mNcNrdhMjSJcDvcA7wzjLD-qKH3g%40mail.gmail.com), started 2026-06-23
- Messages read: 29 of 29
- Patch prose read: v3-0001-Add-PRODUCT-aggregate-function.patch, v1-0001-Add-PRODUCT-aggregate-function.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No level fails. The only surviving objection, standards-compat-1, is a fixable design point: v3 documents product(real) as returning double precision, while sum(real) returns real, and the thread says the two match. Demand is supported by the report in message #10 that the next SQL standard requires PRODUCT. A committer is already listed on the CommitFest entry.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the accumulator functions' null handling, float overflow checks and parallel combine behaviour are unverified. Only 3 of 29 messages were visible, so later agreement or disagreement in the thread may be missing.

## The proposal, restated

**Problem.** Postgres has sum() but no aggregate that multiplies values, for example to compound growth factors or combine probabilities. Users today write a custom aggregate or use exp(sum(ln(x))), which fails for zero or negative inputs and loses precision.

**Proposed design.** The patch adds product() for int2, int4, int8, float4, float8 and numeric, and it ignores nulls like sum(). Integer and numeric inputs accumulate in numeric and return numeric. Since v3, float4 and float8 inputs accumulate in float8 and return double precision; the aggregate supports parallel combine but has no inverse transition function for moving windows.

**User-visible surface.**
- New aggregate product(smallint | integer | bigint | numeric) returning numeric.
- New aggregate product(real | double precision) returning double precision (v3; v1 returned numeric).
- New support functions in pg_proc: int2/int4/int8/float4_product_accum.
- product() returns null for an empty input or an all-null group.
- A finite float product that overflows double precision raises an error; Infinity appears only if it is in the input.

**What the author claims.**
- Common request; exp(sum(ln())) breaks on zero and negative inputs and is imprecise.
- Per-type variants give exact integer results, where implicit casts would pick float8 and lose precision.
- A native int64/int128 fast path helps little, because products overflow bigint within a few rows.
- Float return type now matches sum(float4)/sum(float8).
- Proposed by Peter Eisentraut; regression tests cover NULLs, DISTINCT, FILTER, NaN/Infinity, overflow, parallel and window use.

**Thread state.** Vaibhav Dalvi said v3 looks good with no more comments, and Peter Eisentraut is listed as committer. Vik Fearing says the next SQL standard requires PRODUCT(col, 1 ON EMPTY), and Tom Lane asked to treat that as a separate feature in Jeevan's other ON EMPTY thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-06-23; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. product(real) returns double precision while sum(real) returns real

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax · `standards-compat-1`

A user writes a query or view with sum(r) and product(r) over a real column, expecting both results to be real like the input. The product column comes back as double precision. This changes the result type of views, the types that drivers bind, and the output of CREATE TABLE AS. The user finds out only when inspecting column types or when client code checks them.

Message #26 asked for the float variants to match sum(float4)/sum(float8), and messages #27 and #28 state that v3 matches them. But sum(float4) returns real, and v3 documents product(real) as returning double precision. A float8 transition state may be justified because products overflow real quickly. Even so, the commit message and docs do not say that the result type departs from sum(), and the claim of consistency in the thread is not accurate for real.

Thread: [#26 (Vaibhav Dalvi, 2026-09-13)](https://www.postgresql.org/message-id/CA%2BvB%3DAEcvBbkkFUh%2BDCm9jMqr-T-_T0v%3DM1uGL1xD1tGSnCv%2BA%40mail.gmail.com), [#27 (Jeevan Chalke, 2026-09-22)](https://www.postgresql.org/message-id/CAM2%2B6%3DXp%3DFaW6rOCwNzWoxH7SjjEyw%2BncyuoLr_%2BLuNEdEDLFA%40mail.gmail.com), [#28 (Jeevan Chalke, 2026-09-24)](https://www.postgresql.org/message-id/CAM2%2B6%3DUOZTGe9x7HAdNQAuQPVANo8_KuTBwPmqwT%2BWJQ-uyQ6A%40mail.gmail.com)

**Question:** Should product(real) return real like sum(real), and if not, can the docs and commit message state the difference and the reason for it?

Counter-check: held. sum(real) returns real, and v3 documents product(real) as returning double precision, so the statement in messages #27 and #28 that v3 matches sum() is inaccurate for real. The signature table shows the type, but no text gives the reason for the difference.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds pg_proc and pg_aggregate rows and needs a CATALOG_VERSION_NO bump. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | Message #10 says the next SQL standard defines PRODUCT with an ON EMPTY clause, which this patch leaves to another thread. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | An existing user aggregate named product with the same argument types is shadowed by the pg_catalog one after upgrade. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Six aggregate variants and four new accumulator functions in numeric.c stay in core for good. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A user can already create the same aggregate with numeric_mul as transition and combine function. |

## 5. Questions the author must answer before anyone reviews code

1. Should product(real) return real like sum(real), and if not, can the docs and commit message state the difference and the reason?
2. Should product() be committed before the standard's ON EMPTY syntax, or wait for the other thread?
3. Does Dean Rasheed accept the per-type integer variants after the reply about exact integer results?
4. How does product() compare in speed with sum() and with exp(sum(ln(x))) on the same data?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Eisentraut proposed the idea and is listed as committer on the CommitFest entry; the work needs someone who knows the numeric and float aggregate code.
- **Liability:** The name product, its result types and its null result on empty input become permanent SQL surface. They must stay compatible with the standard's PRODUCT once ON EMPTY is added.
- **Author follow-up:** The author posted three versions and answered each review, per messages #5, #9 and #28.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch adds catalog rows and small accumulator functions that reuse numeric_mul and float8mul, and costs nothing for users who do not call product(). A committer is listed and a reviewer has no more comments on v3; the open point is the result type for real.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` New built-in name can take over existing user product() aggregates. Refuted (not specific): Every new built-in function can shadow a user function of the same name, and Postgres accepts this routinely, as the objection itself concedes. Release notes are written at release time from commit messages, so a patch is not expected to carry a compatibility item.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only listed precedent [P1] is a different aggregate, onlyvalue, returned for lack of author follow-up on review bugs. That reason does not apply here: the author answered each review in message #5, #9 and #28. The thread points to the author's separate ON EMPTY patch (message #11, #17), not to an earlier product() attempt.
- standards-compat: 2 objection(s). The standard's PRODUCT(x, 1 ON EMPTY) is defined as COALESCE(PRODUCT(x), 1), per message #22. Plain product() returning null on empty input fits that definition, so shipping it before ON EMPTY creates no conflict.
- replication-ondisk: 0 objection(s). The patch changes no on-disk format, WAL record or protocol message, and adds no ABI-visible struct; its persistent state is new pg_proc and pg_aggregate rows plus a catversion bump. A user aggregate named product with the same signature would be shadowed after pg_upgrade by the pg_catalog version, as with any new built-in, and the proposal does not say whether this was checked.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. product() is a plain built-in aggregate over numeric and float types, executable by anyone like sum(), and it reads only rows the caller can already see. Its cost is bounded by the numeric size limit, and an existing custom aggregate built on numeric_mul gives the same cost to any user today.
- maintenance-fit: 0 objection(s). From the maintenance angle the proposal adds catalog rows that reuse numeric_mul and float8mul, with no hot-path cost for non-users and no new API or dependency; Peter Eisentraut is the listed committer. Shadowing of existing user-defined product() aggregates is a risk shared by every new built-in function, and the ON EMPTY syntax was split off by message #12.
- already-exists: 0 objection(s). A user can already write `create aggregate product(numeric) (sfunc = numeric_mul, stype = numeric, combinefunc = numeric_mul)`, so core adds convenience only. Message #10 says the next SQL standard requires PRODUCT, and a committer proposed the feature, so this objection would likely not be sent. Not checked: how upgraded databases with their own public.product() behave once pg_catalog.product() is found first.
- demand: 0 objection(s). The thread shows enough demand: message #10 reports that the next SQL standard edition requires PRODUCT, and a committer proposed the idea. The cost is a few pg_proc and pg_aggregate rows that only callers of product() use, and the exp(sum(ln())) workaround fails on zero and negative inputs.

### Precedents retrieved

- P1: [onlyvalue aggregate](https://commitfest.postgresql.org/patch/407/) (cf, Returned with feedback, 2015-12-24) - used in the review; reason summarised from its thread
- P2: [Partial aggregates push down](https://commitfest.postgresql.org/patch/3369/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P3: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4019/) (cf, Returned with feedback, 2024-01-27) - judged not relevant; reason summarised from its thread
- P4: [Optimize numeric comparison and aggregation via packed-datum extraction](https://commitfest.postgresql.org/patch/6651/) (cf, Rejected, 2026-05-10) - judged not relevant; reason summarised from its thread
- P5: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4857/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P6: [Push down aggregates below joins.](https://commitfest.postgresql.org/patch/1685/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P7: [Add numeric_trim(numeric)](https://commitfest.postgresql.org/patch/429/) (cf, Returned with feedback, 2016-03-15) - judged not relevant; reason summarised from its thread
- P8: [Precision loss casting float to numeric](https://commitfest.postgresql.org/patch/1541/) (cf, Returned with feedback, 2018-03-17) - judged not relevant; reason summarised from its thread
- P9: [Allow DISTINCT to work in multiple-argument aggregate calls](https://wiki.postgresql.org/wiki/Not_Worth_Doing#SQL_Commands) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P10: [Expand applicability of aggregate's sortop optimization](https://commitfest.postgresql.org/patch/4980/) (cf, Returned with feedback, 2025-07-08) - judged not relevant
- P11: [Direct converting numeric types to bool](https://commitfest.postgresql.org/patch/1577/) (cf, Rejected, 2018-03-01) - judged not relevant
- P12: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P13: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P14: [Indeterminate behavior for the GROUP BY clause](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P15: [Revert "Optimize various aggregate deserialization functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f3b56eea23554e1756a26080db273156f23f4f2) (revert, Reverted, 2023-10-10) - judged not relevant
- P16: [Implement ASSERTIONs per the SQL standard.](https://commitfest.postgresql.org/patch/1604/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P17: [Support for grabbing multiple consecutive values with nextval()](https://commitfest.postgresql.org/patch/3577/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [pg_buffercache: Add per-relation summary stats](https://commitfest.postgresql.org/patch/6555/) (cf, Returned with feedback, 2026-05-04) - judged not relevant
- P19: [Revert "Enable fast default for domains with non-volatile constraints"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0354e29c41a9fb7491b3c7c23f079b1923c045a) (revert, Reverted, 2026-06-08) - judged not relevant
- P20: [window function improvements WIP](https://commitfest.postgresql.org/patch/2419/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P21: [Revert "Test conversion of NaN between float4 and float8."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5c4c771dafefa7d0ee80a6bd6955b7a3940452a1) (revert, Reverted, 2018-05-05) - judged not relevant
- P22: [Check consistency of GUC defaults between .sample.conf and pg_settings.boot_val](https://commitfest.postgresql.org/patch/3736/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P23: [Reduce memory usage of aggregates in set returning functions](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Functions) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P24: [explain_regress, explain(MACHINE), and default to explain(BUFFERS)](https://commitfest.postgresql.org/patch/3409/) (cf, Withdrawn, 2023-01-15) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 119 s; cost reported by the CLI: $2.34
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 57 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:09:22.120Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
