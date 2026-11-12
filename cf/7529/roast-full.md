# Design pre-review: Support casts from numeric to oid8

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7529](https://commitfest.postgresql.org/patch/7529/) (Needs review); authors: Koshi Shibagaki (k-zshiba)
- Thread: [\[PATCH\] Support casts from numeric to oid8](https://www.postgresql.org/message-id/flat/TY5PR01MB18445597DD6EC7AB442CFA7A8FA642%40TY5PR01MB18445.jpnprd01.prod.outlook.com), started 2026-10-23
- Messages read: 1 of 1
- Patch prose read: v1-0001-Add-an-assignment-cast-from-numeric-to-oid8.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection stops the idea at level 1 or 2. already-exists-1 asks for demand, but the cost is one pg_cast row and one function, and the cast mirrors the existing numeric-to-int8 cast. standards-compat-1 is a fixable design point: the proposal gives no reason for accepting negatives down to -18446744073709551615 in assignment context. Only one message is visible, so the list's view of the wraparound rule is unknown.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the range check, the soft-error path and the volatility and leakproof marking of the new function could not be checked.

## The proposal, restated

**Problem.** Integer literals above INT64_MAX parse as numeric, and there is no numeric-to-oid8 cast, so `SELECT 18446744073709551615::oid8` fails although the value fits in oid8. The quoted form '18446744073709551615'::oid8 works. Users who write large oid8 values as plain literals hit this error.

**Proposed design.** The patch adds a numeric-to-oid8 cast usable in assignment and explicit contexts only. It rounds to the nearest integer with ties away from zero, requires the rounded magnitude to fit in uint64, and wraps negative results modulo 2^64 like oid8 text input. NaN and infinity are rejected, with soft-error support.

**User-visible surface.**
- New pg_cast entry: numeric to oid8, assignment context.
- New conversion function in pg_proc backing the cast; the material does not give its name.
- Documentation paragraph in datatype.sgml on numeric-to-oid8 rounding, range and wraparound.
- Negative numeric values convert by wraparound, for example -1 becomes 18446744073709551615.

**What the author claims.**
- Numeric can represent every oid8 value exactly, while signed integer types cannot represent the upper half of the range.
- An implicit cast would round fractional values in comparisons, so `2::oid8 = 1.5` would return true.
- Wrapping negative numeric input matches earlier discussion that negative oid8 text input wraps on purpose.

**Thread state.** Only the first message is visible. No one has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-23; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Quoted literal already covers the full oid8 range

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), catalog changes · `already-exists-1`

A user who writes 18446744073709551615::oid8 gets an error today. Writing '18446744073709551615'::oid8, or value::text::oid8 for a computed numeric, already gives the result. The patch adds a permanent pg_cast and pg_proc entry so that the quotes can be dropped.

Message #1 states that the quoted form works, so the existing way can do the job with two extra characters. For computed numeric values, a cast through text reaches the existing oid8 input function. Message #1 shows no field report or user request, and oid8 values above INT64_MAX are rare in practice. A committer would likely ask whether this convenience justifies a new catalog cast that must be kept forever.

Thread: [#1 ("Koshi Shibagaki (Fujitsu)", 2026-10-23)](https://www.postgresql.org/message-id/TY5PR01MB18445597DD6EC7AB442CFA7A8FA642%40TY5PR01MB18445.jpnprd01.prod.outlook.com)

**Question:** Which users or tools write oid8 values above INT64_MAX as bare literals or numeric expressions, and why does quoting or casting through text not work for them?

Counter-check: held. Message #1 itself shows the quoted form works, and the thread shows no user report or reply. The cost is small and falls on few people, so this is a request to show demand and does not count against the design.

### 2. Assignment cast wraps negative numeric values into large oid8 values

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax · `standards-compat-1`

An application inserts a computed numeric such as -1.0 or -18446744073709551615 into an oid8 column. The row silently stores 18446744073709551615 or 1 instead of raising an error. The user finds out later, when lookups by that identifier return the wrong object or nothing.

The cast applies in assignment context, so INSERT and UPDATE take this path without any explicit request. Message #1 justifies the wraparound by pointing to oid8 text input, but text input is a compatibility rule for strings, and a cast from an arithmetic type is a different contract. The doc context the patch extends already shows `INSERT INTO mytable VALUES(-1);  -- fails` for the OID types, so -1 would fail while -1.0 succeeds. The bound on the absolute value also accepts negatives that wrap to small positive values, which look valid.

Thread: [#1 ("Koshi Shibagaki (Fujitsu)", 2026-10-23)](https://www.postgresql.org/message-id/TY5PR01MB18445597DD6EC7AB442CFA7A8FA642%40TY5PR01MB18445.jpnprd01.prod.outlook.com)

**Question:** Why should an assignment cast from numeric wrap negative values instead of raising "out of range" as the other integer-to-OID conversions do?

Counter-check: held. The proposal gives no reason for accepting values down to -18446744073709551615, which wrap to small positive values in assignment. The cited `VALUES(-1); -- fails` line is hunk context that looks like the domain example, and int4-to-oid already wraps, so neither supports the objection.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds one pg_cast row and one pg_proc row for the numeric-to-oid8 conversion. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The new assignment cast rounds fractional numeric input and wraps negative values modulo 2^64 without an error. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The rounding and wraparound rules of the cast become behaviour that applications can depend on after release. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #1 states that the quoted form '18446744073709551615'::oid8 already works. |

## 5. Questions the author must answer before anyone reviews code

1. Why does the cast accept negative numeric values down to -18446744073709551615 in assignment context, where they wrap to small positive oid8 values, instead of raising an out-of-range error?
2. Which users or tools write oid8 values above INT64_MAX as unquoted literals or numeric expressions, and why does quoting or a cast through text not work for them?
3. Does an oid8-to-numeric cast exist, and if not, should this patch add it so the conversion has both directions?
4. Are matching casts for oid, or from other integer types to oid8, in scope for this work?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the numeric code and the oid8 type would have to own it; nobody has replied to message #1.
- **Liability:** The owner signs for the cast's rounding, range and wraparound rules for as long as oid8 exists, because applications can come to depend on them after release.
- **Author follow-up:** No sign either way; message #1 ends with "Comments welcome".

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 7.5 KiB, touches numeric.c and two catalog data files, and helps only users who write oid8 values above INT64_MAX as numeric. The negative-range rule should be settled on the list before commit.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Assignment cast commits oid8 to silent rounding and wraparound. Refuted (not specific): Duplicate of standards-compat-1 for the wraparound part. The rounding part copies existing practice: numeric-to-int8 is also an assignment cast that rounds ties away from zero, while int8 text input rejects fractions.
- `already-exists-2` New cast path has different rules from oid8 text input. Refuted (no failure scenario): The same difference exists today for int8: `1.5::int8` rounds to 2 in assignment, and `'1.5'::int8` is rejected. So the patch follows existing numeric-to-integer practice. The large-negative part is a duplicate of standards-compat-1.
- `demand-1` No user report for unquoted oid8 literals above INT64_MAX. Refuted (not specific): Duplicate of already-exists-1, which states the same demand question and also covers computed numeric values through text. The claim that message #1 was asked out of curiosity is not supported by the material.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] concerns changing the results of the existing float-to-numeric casts, and its reason (complaints about changed, odd-looking results) does not apply to adding a new numeric-to-oid8 cast. Message #1 cites an earlier discussion where negative oid8 text input wraps on purpose, and uses it as support for the design, not as a prior rejection.
- standards-compat: 1 objection(s). Rounding of fractional values follows the existing numeric-to-integer casts, so I raise no objection to it. Keeping the cast out of implicit context avoids changing existing comparisons.
- replication-ondisk: 0 objection(s). The patch adds one pg_cast row and one pg_proc row, which means a catversion bump and nothing else persistent. It has no on-disk format, WAL, replication, wire protocol or ABI change. The cast is immutable and has no locale or GUC dependence, so standbys and subscribers compute the same result. One edge case is a user-defined numeric-to-oid8 cast in an older cluster, which would conflict during pg_upgrade, but every new built-in cast carries that cost.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It adds a built-in pg_cast entry and conversion function in assignment and explicit contexts only, so function and operator resolution does not change. Users gain no new data, files or sessions, and the conversion is bounded, so it adds no denial-of-service path. Leakproof marking of the new function cannot be checked without the code, but a function that raises errors would not normally be leakproof.
- maintenance-fit: 1 objection(s). Apart from the semantics above, the maintenance cost is small: one pg_cast row, one function, no hot-path or extension API impact.
- already-exists: 2 objection(s). Two objections raised; the listed precedent [P3] is about float-to-numeric precision and does not apply here.
- demand: 1 objection(s). The cost is small and falls mostly on catalog and docs, so the objection is "show demand", not a rejection.

### Precedents retrieved

- P1: [Allow XML arrays to be cast to other data types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P2: [Replace literal 0 values with the appropriate Invalid* constants](https://commitfest.postgresql.org/patch/6480/) (cf, Withdrawn, 2026-02-13) - judged not relevant; reason summarised from its thread
- P3: [Precision loss casting float to numeric](https://commitfest.postgresql.org/patch/1541/) (cf, Returned with feedback, 2018-03-17) - used in the review; reason summarised from its thread
- P4: [Add numeric_trim(numeric)](https://commitfest.postgresql.org/patch/429/) (cf, Returned with feedback, 2016-03-15) - judged not relevant; reason summarised from its thread
- P5: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant; reason summarised from its thread
- P6: [Allow functions defined as casts to domains to be called during casting](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Allow values to be cast to domain types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [Autoprepare: implicitly replace literals with parameters and store generalized plan](https://commitfest.postgresql.org/patch/1747/) (cf, Returned with feedback, 2020-07-01) - judged not relevant; reason summarised from its thread
- P9: [Implicit prepare of statements: replace literals with parameters and store cached plans](https://commitfest.postgresql.org/patch/1150/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P10: [64-bit XIDs](https://commitfest.postgresql.org/patch/3594/) (cf, Returned with feedback, 2023-12-15) - judged not relevant; reason summarised from its thread
- P11: [64-bit XIDs](https://commitfest.postgresql.org/patch/4703/) (cf, Returned with feedback, 2024-09-16) - judged not relevant; reason summarised from its thread
- P12: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P13: [Revert "Add support for parsing of large XML data (>= 10MB)"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f2743a7d70e7b2891277632121bb51e739743a47) (revert, Reverted, 2024-01-26) - judged not relevant
- P14: [standardize integer parsing in front-end utilities](https://commitfest.postgresql.org/patch/2197/) (cf, Withdrawn, 2020-03-05) - judged not relevant
- P15: [unnesting multirange data types](https://commitfest.postgresql.org/patch/3178/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P16: [Inconsistency in owner assignment between INDEX and STATISTICS](https://commitfest.postgresql.org/patch/6571/) (cf, Withdrawn, 2026-03-11) - judged not relevant
- P17: [Protect syscache from bloating with negative cache entries](https://commitfest.postgresql.org/patch/931/) (cf, Returned with feedback, 2021-10-05) - judged not relevant
- P18: [Negative RelfilenumberMap cache entries from pg_filenode_relation()](https://commitfest.postgresql.org/patch/6066/) (cf, Withdrawn, 2025-09-18) - judged not relevant
- P19: [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (cf, Rejected, 2016-03-11) - judged not relevant
- P20: [64-bit transaction identifiers](https://commitfest.postgresql.org/patch/1178/) (cf, Returned with feedback, 2018-03-01) - judged not relevant
- P21: [Fix data types where equality comparison is not intuitive, e.g. box](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Data_Types) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P22: [Check integers in command line parameters](https://commitfest.postgresql.org/patch/1121/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P23: [64-bit TOAST value ID](https://commitfest.postgresql.org/patch/4296/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P24: [doc: PQexecParams binary handling example for REAL data type](https://commitfest.postgresql.org/patch/3568/) (cf, Returned with feedback, 2023-07-26) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 115 s; cost reported by the CLI: $1.02
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 69 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:05:11.075Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
