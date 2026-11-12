# Design pre-review: Fix the wrong behavior of REFRESH MATERIALIZED VIEW CONCURRENTLY

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6579](https://commitfest.postgresql.org/patch/6579/) (Needs review); authors: Rafia Sabih (rafia.sabih), Surya Poondla (spoondla)
- Thread: [Two issues with REFRESH MATERIALIZED VIEW CONCURRENTLY](https://www.postgresql.org/message-id/flat/40d694df-39fd-4a4a-9459-9d6489165f60%40gogi.tv), started 2026-02-09
- Messages read: 22 of 22
- Patch prose read: 0001-Check-for-duplicate-rows-with-NULLs.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal: the thread agrees that the concurrent refresh can leave a view that differs from its query. The two surviving objections, maintenance-fit-1 and maintenance-fit-2, are fixable design points. The header comment still states the wrong invariant, and the patch has not been compared with the explicit NULL check from message #21. Confidence is medium because only 3 of 22 messages and the first patch's prose were visible.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code and the later patch versions were not read, so the exact pre-check query, its schema qualification of operators and its cost could not be checked. Only 3 of 22 messages were visible in full.

## The proposal, restated

**Problem.** REFRESH MATERIALIZED VIEW CONCURRENTLY misses duplicate rows that contain a NULL outside the unique-index columns, so the view silently keeps one row where the query returns two. A plain REFRESH MATERIALIZED VIEW fails on the same data. Users of matviews with nullable columns get a view that no longer matches its query.

**Proposed design.** The latest bug-1 patch changes the duplicate pre-check in refresh_by_match_merge() to use the per-column equality operators of the unique-index columns plus *=, the same conditions as the FULL OUTER JOIN. Rows that are NULL in an indexed column stay distinct, so they still refresh. A separate patch (CF 6580) for the slow refresh with a unique index on a NULL column is now judged not to work.

**User-visible surface.**
- REFRESH MATERIALIZED VIEW CONCURRENTLY now fails with "new data for materialized view ... contains duplicate rows" for duplicates with NULLs in non-indexed columns.
- An alternative patch by ChangAo Chen adds a built-in function record_image_eq_variant; the main patch does not.

**What the author claims.**
- The fixed pre-check catches (test, NULL) twice with a unique index on a, and still accepts (NULL, NULL) twice.
- A unique index on a column of all NULLs makes the refresh 3 to 5 times slower; the bug-2 patch does not fix it.
- The bug-1 patch passes the regression tests, and Surya says v7-0001 is ready.
- The record_image_eq_variant approach forces a delete and insert for every unchanged row with a NULL in any column.

**Thread state.** Robert Haas and Yugo Nagata agree it is a bug but want the function header comment, which states the wrong invariant, rewritten. Yugo says the commit message wrongly claims the (NULL, NULL) case as a fix, and asks why the patch uses equality operators over explicit NULL checks; no reply since July 31.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-02-09; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Fix leaves the documented invariant of refresh_by_match_merge() wrong

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later developer works on the issue 2 slowdown or another change to the diff join and trusts the header comment saying the join is correct for any row with a NULL. They change which key columns enter the join or the pre-check. Duplicates with NULLs outside key columns get through again, and no test or message catches it until a user reports a stale view.

In message #21 Robert Haas says the header comment is wrong in two ways and that the fix should update it. In message #22 Yugo Nagata asks for the comment to explain why the NULL-in-key-column case works. The only patch prose shown, and v7-0001 as described, leave that comment as it is. The correctness of this code rests on a subtle interaction between *= and per-column equality, so the comment is the main defence against regressions here.

Thread: [#21 (Robert Haas, 2026-07-27)](https://www.postgresql.org/message-id/CA%2BTgmoaryPLAMNWhF8cP4L%2Bf%2BwQKkwS5OGCC4rUCwfpm%2Bu9-fg%40mail.gmail.com), [#22 (Yugo Nagata, 2026-07-31)](https://www.postgresql.org/message-id/20260731164448.f84babf2a951ad20e68058a8%40sraoss.co.jp)

**Question:** Can the next version rewrite the header comment to state the real invariant, NULLs in unique-index columns only, and say why the join is then correct?

Counter-check: held. Robert Haas says in message #21 that the fix does not touch the header comment and that its last sentence states the wrong invariant. Yugo Nagata agrees in message #22, and no later version or reply addresses it.

### 2. Pre-check mechanism chosen without comparing it to explicit NULL checks

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years"), performance on the hot path, backpatchability · `maintenance-fit-2`

After a minor release, a site with a large concurrently refreshed matview whose rows often hold NULLs in non-key columns finds the refresh slower. Those rows were skipped by the old IS NOT NULL filter but now enter the EXISTS self-join on newdata. Nobody measured this before the release.

Robert Haas (message #21) suggested a narrower change that checks only for NULLs in unique-index columns. Yugo Nagata (message #22) asked why the patch uses equality operators instead of explicit NULL checks, and the thread has no answer. The thread shows no timing for the new pre-check on large views, only regression-test passes (message #20). A back-patched bug fix should pick the variant that is easiest to reason about and show what it costs.

Thread: [#20 (surya poondla, 2026-07-21)](https://www.postgresql.org/message-id/CAOVWO5pSxK9TZEEQ7xgNQOUFob_9F-u3YRRogXjDyRktgfaC%2BQ%40mail.gmail.com), [#21 (Robert Haas, 2026-07-27)](https://www.postgresql.org/message-id/CA%2BTgmoaryPLAMNWhF8cP4L%2Bf%2BwQKkwS5OGCC4rUCwfpm%2Bu9-fg%40mail.gmail.com), [#22 (Yugo Nagata, 2026-07-31)](https://www.postgresql.org/message-id/20260731164448.f84babf2a951ad20e68058a8%40sraoss.co.jp)

**Question:** How does the equality-operator pre-check compare with Robert's explicit NULL check in clarity and in refresh time on a large view with many NULLs in non-key columns?

Counter-check: held. Message #17 says the pre-check mirrors the join operators, but Yugo's later question in #22 has no reply and no pre-check timing is shown. Any correct fix must scan rows with NULLs in non-key columns, so that cost does not favour either variant.

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
| backpatchability | **yes** | A back-patched fix makes some concurrent refreshes that succeed today fail with the duplicate-rows error after a minor upgrade. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The header comment of refresh_by_match_merge() states an invariant that the fix shows to be wrong, and the patch leaves it unchanged. |
| performance on the hot path | **yes** | The changed pre-check runs on every concurrent refresh and now includes rows with NULLs; the thread shows no timing. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can the next version rewrite the header comment of refresh_by_match_merge() to state the real invariant and explain why the join is correct for NULLs in unique-index columns?
2. Why does the pre-check use the per-column equality operators instead of the explicit NULL check on unique-index columns suggested in message #21?
3. How long does the new pre-check take on a large view with many NULLs in non-indexed columns, compared with the current code?
4. Is the fix meant for back-patching, and if so, what should the release note tell users whose refreshes will start to fail?
5. Which patch version does CommitFest entry 6579 track now, and does its commit message still claim the (NULL, NULL) case as a fix?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Robert Haas reviewed the fix in message #21 and asked for the header comment to be rewritten; the owner needs to know the matview diff join and row comparison operators.
- **Liability:** The owner signs for the correctness of the duplicate pre-check in all supported branches if back-patched, and for refreshes that start to fail after a minor upgrade.
- **Author follow-up:** The authors posted versions up to v7, but Yugo Nagata's questions in message #22 have had no reply since July 31.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** This is a wrong-result fix in one function of matview.c with no new user-visible surface, and reviewers agree it is a bug. It needs a new version that answers messages #21 and #22 before a committer can take it.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only listed precedent, P3, covers a cosmetic REFRESH MATERIALIZED VIEW error-message change and its back-patch policy, not duplicate detection in the concurrent refresh. The thread cites no earlier attempt at this fix, so the prior-attempts angle has nothing to raise.
- standards-compat: 0 objection(s). The visible contract changes in one way: REFRESH MATERIALIZED VIEW CONCURRENTLY now raises the existing "contains duplicate rows" error for duplicates with NULLs in non-key columns. That matches what plain REFRESH already does on the same data, and the old result was a view that silently differed from its query. The fix adds no syntax, GUC, catalog object or pg_dump change. A back-patch would make some scheduled refreshes start failing after a minor upgrade, but the failure replaces silent wrong data, so it needs a release note rather than an objection. The record_image_eq_variant function is only in the alternative patch.
- replication-ondisk: 0 objection(s). The bug-1 patch only changes the duplicate pre-check query in refresh_by_match_merge(). It adds no on-disk format, WAL record, catalog entry, protocol message or extension API, and the refresh still writes through ordinary WAL-logged DML. ChangAo Chen's alternative adds a built-in function, which needs a catversion bump and cannot be back-patched, but the main patch does not take that route.
- security: 0 objection(s). The proposal has no new privilege or exposure aspect. It changes only the duplicate pre-check query, which runs inside REFRESH MATERIALIZED VIEW CONCURRENTLY for a user who already owns the view or holds MAINTAIN. It uses the same per-column equality operators that the existing FULL OUTER JOIN already uses, and the existing duplicate-rows error with its row DETAIL. Not checked without the code: that the new operator references are schema-qualified like the existing ones.
- maintenance-fit: 2 objection(s). The fix stays inside one function and adds no GUC, catalog or API, so it carries little long-term cost. Two committers agree it is a bug. The objections above concern how the fix is documented and justified, not whether it belongs in core.
- already-exists: 0 objection(s). This is a correctness fix inside refresh_by_match_merge(), where REFRESH MATERIALIZED VIEW CONCURRENTLY can leave a view that no longer matches its query. No existing option or extension makes the concurrent refresh detect these duplicates. Plain REFRESH MATERIALIZED VIEW avoids the bug but takes an exclusive lock, so it does not meet the same need. Precedent [P3] concerned a cosmetic error message and does not apply.
- demand: 0 objection(s). This is a wrong-result bugfix. Two committers in messages #21 and #22 agree the matview can silently diverge from its query, so demand needs no further proof. The open questions are the header comment and the choice of equality operators versus explicit NULL checks, which are design points for other angles.

### Precedents retrieved

- P1: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P2: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant; reason summarised from its thread
- P3: [Fix for REFRESH MATERIALIZED VIEW ownership error message](https://commitfest.postgresql.org/patch/1766/) (cf, Rejected, 2018-08-19) - used in the review; reason summarised from its thread
- P4: [Allow granting of permission to REFRESH MATERIALIZED VIEW, CLUSTER, and REINDEX](https://commitfest.postgresql.org/patch/1605/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P5: [Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet](https://commitfest.postgresql.org/patch/4077/) (cf, Withdrawn, 2022-12-27) - judged not relevant; reason summarised from its thread
- P6: [Triggers on Materialized Views](https://commitfest.postgresql.org/patch/1948/) (cf, Returned with feedback, 2019-04-01) - judged not relevant
- P7: [Temporary materialized views](https://commitfest.postgresql.org/patch/1951/) (cf, Returned with feedback, 2019-03-15) - judged not relevant
- P8: [Triggers on materialized views](https://commitfest.postgresql.org/patch/1953/) (cf, Withdrawn, 2019-01-04) - judged not relevant
- P9: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P10: [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://commitfest.postgresql.org/patch/2664/) (cf, Returned with feedback, 2020-11-24) - judged not relevant; reason summarised from its thread
- P11: [Remove useless GROUP BY columns considering unique index](https://commitfest.postgresql.org/patch/4751/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P12: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P13: [Optimizing NULL check on pkey column](https://commitfest.postgresql.org/patch/2699/) (cf, Returned with feedback, 2021-09-27) - judged not relevant
- P14: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant
- P15: [Indexes with Included Columns (was Covering + unique indexes)](https://commitfest.postgresql.org/patch/961/) (cf, Returned with feedback, 2017-04-08) - judged not relevant
- P16: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant
- P17: [Adding CommandID to heap xlog records](https://commitfest.postgresql.org/patch/3882/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P18: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P19: [Statistics fixes and improvements in face of high fraction of NULLs and/or skewed distribution](https://commitfest.postgresql.org/patch/503/) (cf, Returned with feedback, 2016-04-08) - judged not relevant
- P20: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P21: [Make selectivity example match wording](https://commitfest.postgresql.org/patch/3695/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P22: [Permute underscore separated components of columns before fuzzy matching](https://commitfest.postgresql.org/patch/4282/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P23: [Operation log for major operations](https://commitfest.postgresql.org/patch/4018/) (cf, Withdrawn, 2023-03-02) - judged not relevant
- P24: [Revert "Add some temporary code to record stack usage at server process exit."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96112ee7c60557bb192a9aa07b514db2400fd45e) (revert, Reverted, 2016-07-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 108 s; cost reported by the CLI: $1.90
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 54 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:55:23.308Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
