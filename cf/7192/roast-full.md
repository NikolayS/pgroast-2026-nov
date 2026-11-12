# Design pre-review: GiST multirange consistent fix

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7192](https://commitfest.postgresql.org/patch/7192/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [GiST multirange index scans can fail to return rows](https://www.postgresql.org/message-id/flat/CAH2-Wzm1GpQ9qixP%3DXs2TYdfnLM88OZ%3DyUFHNuajS_W7U90-tQ%40mail.gmail.com), started 2026-08-01
- Messages read: 8 of 8
- Patch prose read: 0001-Add-regression-test-for-multirange-GiST-containment-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible case in message #1 (501 rows from the sequential scan, 500 from the index scan), and the fix changes one scan-time test in an existing opclass with no new user-visible surface. Confidence is medium because only 3 of 8 messages and no fix code were visible.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The fix code and messages #4 to #8 were not visible, so the correctness of the v4 predicate for empty and unbounded multiranges and on internal pages could not be checked. The claim that stored keys do not change also rests on the description only.

## The proposal, restated

**Problem.** A GiST index scan with the contained-by operator on a multirange column can miss rows that a sequential scan returns. The leaf key is the union range, and the leaf test treats it as the real value, so a gapped query rejects matching rows without a recheck. Peter Geoghegan says this affects Postgres 14 and up.

**Proposed design.** The patch changes the contained-by case in multirange_gist_consistent() to use a new test, so a union range key is no longer rejected wrongly. Alexander Korotkov's v4 uses the rule that if q contains r, then union(q) contains union(r). Andrey Borodin's earlier v2 reused range_gist_consistent_int_multirange() for this strategy instead.

**User-visible surface.**
- Contained-by scans on multirange GiST indexes return rows they used to drop; there is no new syntax, GUC or catalog change.

**What the author claims.**
- The sequential scan returns 501 rows and the index scan returns 500 for the query '{[1,11),[21,31),[41,51)}'.
- The bug needs gaps in the query multirange; contiguous queries agree, so existing tests do not catch it.
- Korotkov says contained-by is the only predicate that gives false negatives on the union range.
- Borodin says using the internal predicate for every strategy would be correct but would add heap rechecks.
- Borodin reviewed v4 and says the new predicate is correct.

**Thread state.** Geoghegan reported the bug, Borodin posted v2, and Korotkov posted v3 and v4 after review comments from Reshke. Borodin's last comments on v4 ask for a comment rewording and a typo fix in the commit message, and no later version is visible.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-01; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | **yes** | Message #1 says the bug affects 14 and up, so the fix is a back-patch candidate for all supported branches. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | The new contained-by test in multirange_gist_consistent() runs on every such GiST scan and may send more tuples to heap recheck. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the fix meant for back-patching to 14 and later, and does the same patch apply to each branch?
2. Can you confirm that the stored keys do not change, so existing multirange GiST indexes need no REINDEX?
3. What argument or test shows that the v4 predicate gives no false negatives for empty multiranges and for unbounded ones?
4. How many more heap rechecks does the v4 test cause compared with the v2 approach on the regression data?
5. Do internal pages use the same contained-by test, and is it correct there as well as on leaf pages?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Per the brief, Alexander Korotkov wrote v3 and v4 and Peter Geoghegan reported the bug; the owner needs to know GiST opclass consistent functions and the range and multirange code.
- **Liability:** The owner signs for the correctness of the new contained-by predicate on all back branches from 14, including empty and unbounded multiranges. A wrong predicate would again give silent wrong results.
- **Author follow-up:** The brief shows v2, v3 and v4 posted after review comments; Borodin's last wording and typo comments on v4 have no visible reply.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug gives silent wrong results for contained-by scans on multirange GiST indexes on 14 and later, and the fix is limited to one consistent function. Few users index multiranges with GiST, but wrong results justify attention now.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus search returned no earlier attempt at this fix, and the thread does not point to any earlier discussion. This is a wrong-results bug fix with agreement from three people who know the area, so no prior rejection reason applies.
- standards-compat: 0 objection(s). This is a wrong-results bug fix with no new syntax, GUC, catalog entry or dump-visible object, so pg_dump and psql are unaffected. The only user-visible change is that contained-by scans on multirange GiST indexes now return the rows a sequential scan returns, which brings the index path in line with documented semantics. The fix is in the consistent function and does not change stored keys, so existing indexes should stay valid without a REINDEX, though this cannot be confirmed without the code. A back-patch to 14 seems appropriate.
- replication-ondisk: 0 objection(s). The fix changes only the contained-by test in multirange_gist_consistent() at scan time. Stored GiST keys (the union range), WAL, catalogs and the wire protocol stay the same, so existing indexes need no REINDEX and a back-patch is not blocked by on-disk or replication concerns.
- security: 0 objection(s). The fix changes only the result of the contained-by test in multirange_gist_consistent(). It adds no privilege, role, GUC, catalog object, log output or code path that an unprivileged user controls. Returning rows that were wrongly dropped does not bypass RLS or column privileges, because those still apply above the index scan. Nothing in this proposal has a security or privilege aspect for this angle to judge.
- maintenance-fit: 0 objection(s). This is a wrong-results bug fix inside an existing opclass, with no new syntax, GUC, catalog or API surface, so it adds no lasting maintenance cost. A committer who knows multirange GiST wrote v4, and two others reviewed it; only back-patch scope and the empty or unbounded cases remain open, and those are questions for the critic.
- already-exists: 0 objection(s). This is a wrong-results bug fix in the existing multirange GiST opclass. No existing feature, extension or setting makes a GiST contained-by scan return the rows it drops. Disabling index scans or using a sequential scan is a workaround and does not fix the opclass. The fix changes the existing consistent function and adds no second mechanism.
- demand: 0 objection(s). This is a wrong-results bug fix, and demand is not in question: a GiST index scan with contained-by on a multirange column drops rows that a sequential scan returns (501 vs 500, message #1). Every user of multirange GiST indexes on 14 and later can get silently incomplete answers, and the fix adds no syntax, GUC or catalog surface.

### Precedents retrieved

- P1: [unnesting multirange data types](https://commitfest.postgresql.org/patch/3178/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P2: [Pass sk_attno to GIST consistent function](https://commitfest.postgresql.org/patch/4902/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P3: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P4: [Rename trim_array() for consistency with the rest of array_* functions](https://commitfest.postgresql.org/patch/5335/) (cf, Withdrawn, 2024-11-04) - judged not relevant; reason summarised from its thread
- P5: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P6: [Make uselocale protection more consistent](https://commitfest.postgresql.org/patch/4407/) (cf, Withdrawn, 2023-08-01) - judged not relevant; reason summarised from its thread
- P7: [Check consistency of GUC defaults between .sample.conf and pg_settings.boot_val](https://commitfest.postgresql.org/patch/3736/) (cf, Returned with feedback, 2024-01-26) - judged not relevant; reason summarised from its thread
- P8: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P9: [Docs: make source file path references consistent](https://commitfest.postgresql.org/patch/6380/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P10: [PATCH: Attempt to make dbsize a bit more consistent](https://commitfest.postgresql.org/patch/2701/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P11: [DOCS: Make the Server Application docs synopses more consistent](https://commitfest.postgresql.org/patch/5647/) (cf, Withdrawn, 2026-03-23) - judged not relevant
- P12: [Add sortsupport for range types and btree_gist](https://commitfest.postgresql.org/patch/3686/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P13: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P14: [Index usage for 'element <@ const range'](https://commitfest.postgresql.org/patch/1038/) (cf, Returned with feedback, 2017-03-28) - judged not relevant
- P15: [Make the am_xxx function Asserts consistent](https://commitfest.postgresql.org/patch/4570/) (cf, Withdrawn, 2024-01-09) - judged not relevant
- P16: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant
- P17: [Protect syscache from bloating with negative cache entries](https://commitfest.postgresql.org/patch/931/) (cf, Returned with feedback, 2021-10-05) - judged not relevant
- P18: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P19: [remove deprecated v8.2 containment operators](https://commitfest.postgresql.org/patch/2798/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P20: [\[PATCH\] Add support function for containment operators](https://commitfest.postgresql.org/patch/4304/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P21: [Contains and is contained by operators of inet datatypes](https://commitfest.postgresql.org/patch/805/) (cf, Rejected, 2016-11-30) - judged not relevant
- P22: [Negative RelfilenumberMap cache entries from pg_filenode_relation()](https://commitfest.postgresql.org/patch/6066/) (cf, Withdrawn, 2025-09-18) - judged not relevant
- P23: [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (cf, Rejected, 2016-03-11) - judged not relevant
- P24: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 67 s; cost reported by the CLI: $0.87
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 44 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:26:59.406Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
