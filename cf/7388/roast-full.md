# Design pre-review: Add ASCII fast path to Unicode normalization functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7388](https://commitfest.postgresql.org/patch/7388/) (Needs review); authors: Andrew Dunstan (adunstan)
- Thread: [Add ASCII fast path to Unicode normalization functions](https://www.postgresql.org/message-id/flat/8cb366d6-a30f-49a4-b5d3-a5dfc0f2fea2%40dunslane.net), started 2026-09-14
- Messages read: 16 of 16
- Patch prose read: v5-0001-Add-ASCII-fast-path-to-Unicode-normalization-func.patch, 0001-Add-ASCII-fast-path-to-Unicode-normalization-functio.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem and the workload are stated (`url IS NORMALIZED` over ClickBench data), the change adds no user-visible surface, and message #3 measured no regression on non-ASCII input. The patch reuses the existing is_valid_ascii() and stays inside varlena.c. Confidence is medium because the code was not read and no benchmark of v5 as a whole is shown.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check that restarting at the last ASCII character gives the same result as a full decode, or that form-name matching and its error behaviour match the old path. Builds without SIMD support and big-endian machines are untested in the material.

## The proposal, restated

**Problem.** unicode_is_normalized(), unicode_assigned() and normalize() decode every input to char32_t code points before doing any work, even when the input is pure ASCII. This makes queries like `url IS NORMALIZED` over mostly-ASCII data slow. The author cites a LinkedIn post reporting a 6x gap to CedarDB at a matched thread count.

**Proposed design.** A SIMD byte scan with the existing is_valid_ascii() finds the leading valid ASCII prefix, and an all-ASCII string returns at once. Otherwise only the rest is decoded, starting with the last ASCII character so it can still compose with a following combining mark. unicode_assigned() walks the rest without counting code points first, and the form argument is matched in the text value without building a C string.

**What the author claims.**
- About 10x faster on pure ASCII and about 4x on an 85/15 ASCII/non-ASCII mix, with no measurable regression on non-ASCII input.
- ASCII has no decomposition, a combining class of zero, and is fully assigned, so results stay the same in all four forms.
- Restarting at the last ASCII character gives the same quick-check answer as starting from the beginning of the string.
- Allocating the C string for the form name is a noticeable part of the cost once the fast path is taken.

**Thread state.** Chao Li, Tristan Partin, David Wheeler and Rui Zhao replied in favour, and Bilal and Rui contributed code that v5 folds in. Tristan's last comment nits remain, and Andrew plans to commit during the November CF and fix them then.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-14; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | varlena.c gains an ASCII-prefix path and hand-written form-name matching that must stay in agreement with the full decode path. |
| performance on the hot path | **yes** | Every call to normalize(), IS NORMALIZED and unicode_assigned() now runs a byte scan first, including on non-ASCII input. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What does v5 as a whole measure against master on all-ASCII, mixed, non-ASCII and late-non-ASCII input?
2. What do the non-ASCII and late-non-ASCII cases measure on a build where is_valid_ascii() has no SIMD support?
3. Does an invalid form name still raise the same error when the input string is all ASCII?
4. Does the form-name matching in the text value accept and reject exactly the same names as the old pg_strcasecmp() path, including non-ASCII bytes in the name?
5. Has the patch been run on a big-endian machine?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andrew Dunstan wrote the patch and, per the thread state, plans to commit it during the November CommitFest.
- **Liability:** The owner signs for identical results in all four normalization forms, because expression indexes and CHECK constraints on normalize() or IS NORMALIZED depend on them. The liability lasts as long as the fast path exists.
- **Author follow-up:** The author folded Bilal's and Rui's code into v5 and plans to fix Tristan's remaining comment nits at commit.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Users who run normalize() or IS NORMALIZED over mostly-ASCII text gain about 10 times on pure ASCII per message #1, confirmed in message #3. The patch is 18.4 KB in one backend file plus regression tests, and adds no user-visible surface.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P9] (SIMD for List functions) shares only the SIMD keyword, and its reason does not apply: it stalled because no hot workload was shown, while this proposal names one (url IS NORMALIZED on ClickBench) and three reviewers measured it. The thread points to no earlier attempt at this idea.
- standards-compat: 0 objection(s). The patch changes no SQL syntax, function signature, GUC, catalog or dump output, and the stated results of normalize(), IS NORMALIZED and unicode_assigned() stay the same in all four forms. The one possible contract change is form-name matching done in the text value, and v5 adds regression tests for invalid and mixed-case form names. Two points cannot be checked without the code: whether an invalid form still raises an error for all-ASCII input, and whether case folding still matches pg_strcasecmp().
- replication-ondisk: 0 objection(s). The patch changes only src/backend/utils/adt/varlena.c and regression files, with no catalog, WAL, on-disk, wire protocol or exported header change. The one persistent link is expression indexes and CHECK constraints built on normalize() or IS NORMALIZED, which would need reindexing after pg_upgrade if results changed, but the proposal claims identical results in all four forms and tests prefix sweeps, so no failure scenario stands.
- security: 0 objection(s). The proposal changes only how three existing SQL functions compute their results, so it has no privilege or exposure aspect. It adds no object, privilege, setting or new input path, and the results for every input stay the same, so non-superusers gain nothing new. Rejecting invalid form names is a correctness question for the code review.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle I find no objection a committer would send. The change stays inside varlena.c and its regression tests, adds no GUC, catalog, ABI or dependency, and reuses the existing is_valid_ascii(). Its correctness rests on ASCII having no decomposition, being all starters and being fully assigned, and Unicode stability rules keep those facts fixed. On the hot path, non-ASCII input pays one byte scan, and message #3 and message #5 measured no regression there and a gain on late non-ASCII input. The author is a committer who plans to commit and own it, and four other people reviewed or contributed. [P9] does not fit: it was returned because no workload used the List functions, while this patch targets `IS NORMALIZED` and normalize() on real text columns. The material does not show runs on builds without SIMD support or on big-endian machines, and checking that the form-name matching agrees with the old parsing needs the code.
- already-exists: 0 objection(s). No existing feature, extension or hook gives the same result. Users could precheck with a regex or an encoding test before calling normalize(), but that only moves the same work into SQL and does not speed up the built-in functions. The patch reuses the existing is_valid_ascii() and adds no new mechanism, so item 12 is not at issue. [P9] does not fit, because this thread shows a concrete workload (IS NORMALIZED over ClickBench URLs).
- demand: 0 objection(s). The thread shows enough demand for an internal speedup that adds no user-visible surface. Two people wrote the same patch independently after one public benchmark (messages #1, #7), four others reviewed or added code, and the target workload is named: `url IS NORMALIZED` over ClickBench data. The cost stays inside varlena.c and falls only on callers of these functions. Message #3 shows no regression on non-ASCII input. [P9] does not apply, because it failed for lack of any workload and this proposal names one.

### Precedents retrieved

- P1: [CREATE DATABASE ... STRICT_UNICODE](https://commitfest.postgresql.org/patch/4876/) (cf, Withdrawn, 2024-12-11) - judged not relevant; reason summarised from its thread
- P2: [Revert COPY OUT to follow the pre-8.3 handling of ASCII control characters, namely that \r, \n, \t, \b, \f, \v are dumped as those two-character...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a63b63ff96293e153d25e8e054a830d70f69938a) (revert, Reverted, 2007-12-03) - judged not relevant
- P3: [Revert "Rename unicode_combining_table to unicode_width_table"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1563ecbc1be8b8e5c57651cf5c87f90dea9aea8f) (revert, Reverted, 2021-08-26) - judged not relevant
- P4: [pg_upgrade check for Unicode update](https://commitfest.postgresql.org/patch/5563/) (cf, Withdrawn, 2025-03-18) - judged not relevant; reason summarised from its thread
- P5: [Fix pg_stat_statements display of normalized FETCH counts](https://commitfest.postgresql.org/patch/6756/) (cf, Withdrawn, 2026-05-12) - judged not relevant; reason summarised from its thread
- P6: [PGSERVICEFILE as part of a normal connection string \[duplicated please delete\]](https://commitfest.postgresql.org/patch/5659/) (cf, Withdrawn, 2025-03-20) - judged not relevant; reason summarised from its thread
- P7: [URL extension with Unicode and setter-functions support](https://commitfest.postgresql.org/patch/5432/) (cf, Rejected, 2024-12-19) - judged not relevant; reason summarised from its thread
- P8: [PgVA aka PostgresVectorAcceleration using SIMD vector instructions starting with hex_encode](https://commitfest.postgresql.org/patch/3498/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P9: [optimize several list functions with SIMD intrinsics](https://commitfest.postgresql.org/patch/4238/) (cf, Returned with feedback, 2023-07-10) - used in the review; reason summarised from its thread
- P10: [Unicode mapping scripts cleanup](https://commitfest.postgresql.org/patch/360/) (cf, Returned with feedback, 2015-10-03) - judged not relevant; reason summarised from its thread
- P11: [\[WIP\] Temporal query processing with range types - Temporal Normalization](https://commitfest.postgresql.org/patch/2045/) (cf, Withdrawn, 2020-01-30) - judged not relevant
- P12: [Allow regex operations in PL/Perl using UTF8 characters in non-UTF8 encoded databases](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Server-Side_Languages) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P13: [LockHasWaiters() crashes on fast-path locks](https://commitfest.postgresql.org/patch/6732/) (cf, Withdrawn, 2026-09-03) - judged not relevant
- P14: [Allow reliable XML operation non-UTF8 server encodings (xpath(), in particular, is known to not work)](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P15: [use SIMD in GetPrivateRefCountEntry()](https://commitfest.postgresql.org/patch/6101/) (cf, Withdrawn, 2025-12-15) - judged not relevant
- P16: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P17: [Code comments: why some text-handling functions are leakproof](https://commitfest.postgresql.org/patch/3510/) (cf, Returned with feedback, 2022-03-28) - judged not relevant
- P18: [SIMD optimization for list_sort and tuple-sort](https://commitfest.postgresql.org/patch/5607/) (cf, Withdrawn, 2025-08-06) - judged not relevant
- P19: [vector search support](https://commitfest.postgresql.org/patch/4292/) (cf, Returned with feedback, 2023-11-07) - judged not relevant
- P20: [Vectored I/O in bulk_write.c](https://commitfest.postgresql.org/patch/4891/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P21: [Skip collecting decoded changes of already-aborted transactions](https://commitfest.postgresql.org/patch/4436/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P22: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P23: [Stores short varlena in an array](https://commitfest.postgresql.org/patch/5461/) (cf, Rejected, 2025-01-25) - judged not relevant
- P24: [Leaking memory in text_overlay function](https://commitfest.postgresql.org/patch/658/) (cf, Rejected, 2016-08-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 109 s; cost reported by the CLI: $1.33
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:49:16.753Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
