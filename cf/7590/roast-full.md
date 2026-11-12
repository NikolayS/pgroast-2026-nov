# Design pre-review: BUG #19689: MERGE INSERT accepts a set-returning function during PREPARE but fails at EXECUTE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7590](https://commitfest.postgresql.org/patch/7590/) (Needs review); authors: Tender Wang (tndrwang)
- Thread: [BUG #19689: MERGE INSERT accepts a set-returning function during PREPARE but fails at EXECUTE](https://www.postgresql.org/message-id/flat/19689-0a1b106035cd8a5b%40postgresql.org), started 2026-09-15
- Messages read: 2 of 2
- Patch prose read: 0001-Reject-set-returning-functions-in-MERGE-INSERT-actio.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is not in question: the statement fails on every execution today, and the patch moves the error to parse analysis. The only surviving objection, maintenance-fit-1, is a fixable design point about where the check lives. A committer who knows parse analysis should decide between the p_hasTargetSRFs check and an expression kind that check_srf_call_placement() rejects, and should decide on back-patching.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the error text, the SQLSTATE and the exact placement of the check are unverified. The material also does not show whether a stored SQL-function body that holds such a MERGE can exist and would fail to restore.

## The proposal, restated

**Problem.** A set-returning function in the VALUES of a MERGE WHEN NOT MATCHED INSERT action passes parse analysis, so PREPARE succeeds. Every EXECUTE then fails with "set-valued function called in context that cannot accept a set". The reporter saw this on 20devel, 18.6 and 17.11, and wants the error raised when the statement is analyzed.

**Proposed design.** MERGE INSERT values are transformed with EXPR_KIND_VALUES_SINGLE, which allows SRFs and sets p_hasTargetSRFs, but transformMergeStmt() later clears hasTargetSRFs and no ProjectSet node expands the action target list. The patch checks pstate->p_hasTargetSRFs right after the INSERT values are transformed and raises an error. The check sits only in the CMD_INSERT case, so UPDATE actions and SRFs in the MERGE source do not change.

**User-visible surface.**
- merge ... when not matched then insert values (...) containing a set-returning function now fails at parse analysis, and so at PREPARE, instead of at execution.

**What the author claims.**
- The executor can never run an SRF in a MERGE INSERT action, so the statement always fails today.
- The check does not affect other MERGE actions or SRFs used in the MERGE data source.
- A MERGE INSERT action is meant to insert one row per candidate change row.

**Thread state.** The bug report and one reply from the patch author with a first patch are visible. The author asks whether rejecting SRFs here is the intended behavior and has not added a regression test yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-15; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Ad hoc flag check instead of the expression-kind mechanism for SRF placement

fixable · level 2, rubric item 12 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a MERGE action or moves MERGE UPDATE SET to a new transform path. It does not copy the p_hasTargetSRFs check, and nothing fails at build or test time. Users find out when PREPARE succeeds again and every EXECUTE fails.

Postgres already decides where SRFs are allowed through ParseExprKind and check_srf_call_placement(), which gives one error text per context. The patch keeps EXPR_KIND_VALUES_SINGLE, which allows SRFs, and adds a separate test in the CMD_INSERT case only (message #2). The proposal does not say whether MERGE UPDATE SET has the same gap, so the fix may cover one of two places. A committer would likely ask for a MERGE-specific expression kind, or a reason why the existing mechanism does not fit.

Thread: [#2 (Tender Wang, 2026-09-16)](https://www.postgresql.org/message-id/CAHewXN%3Dt2OW_gehDsso17CLgqRfMfnhzDsby7pXWpzSy_-e9Tg%40mail.gmail.com)

**Question:** Why is a p_hasTargetSRFs check in CMD_INSERT better than giving MERGE action expressions their own expression kind, and does MERGE UPDATE SET need the same fix?

Counter-check: held. check_srf_call_placement() is the existing place for this rule, and the proposal gives no reason for a separate flag test. The MERGE UPDATE SET part should be dropped: EXPR_KIND_UPDATE_SOURCE already rejects SRFs at parse analysis, as far as I recall.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch changes which expressions parse analysis accepts in the values of a MERGE insert action. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #1 reproduces the bug on 17.11 and 18.6, and the proposal does not say whether to back-patch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The check is a separate p_hasTargetSRFs test in the CMD_INSERT case of parse_merge.c, outside check_srf_call_placement(). |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why is a p_hasTargetSRFs check in the CMD_INSERT case better than giving MERGE insert values an expression kind that check_srf_call_placement() rejects?
2. Should the fix be back-patched to 17 and 18, where message #1 also reproduces the failure?
3. What are the new error message text and SQLSTATE, and do they match what check_srf_call_placement() reports for other contexts?
4. Does a non-prepared MERGE with such a function now report a different error message than before?
5. Do MERGE UPDATE SET actions already reject set-returning functions at parse analysis?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows parse analysis and the MERGE transform code would have to own it. No committer has spoken in the two visible messages.
- **Liability:** One parse-analysis check in parse_merge.c and its error message, plus the back-patch decision for 17 and 18. If the check stays a separate flag test, later MERGE action paths must remember to copy it.
- **Author follow-up:** The author offers in message #2 to add a regression test in the next version once the intended behavior is confirmed.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 1.9 KB in one parser file and fixes a reported bug that makes a prepared MERGE fail on every execution. Few users write a set-returning function in MERGE insert values, and the remaining design point is small.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The only precedent, [P4], is about reimplementing target-list SRFs in the planner, a different idea. Its recorded reason (planner complexity, rescan correctness) does not apply to a parse-time check in MERGE INSERT. The thread cites no earlier discussion of this idea, so no precedent was found.
- standards-compat: 0 objection(s). From the standards and compatibility angle, the change only moves an error that always happens at executor start to parse analysis. No working query, script or dump changes behaviour, and the SQL standard has no set-returning functions in MERGE insert values. Not checked without the code: the new error text and SQLSTATE, whether a stored SQL-function body holding such a MERGE could fail restore, and whether a back-patch is intended.
- replication-ondisk: 0 objection(s). The patch adds one parse-analysis check in parse_merge.c, with no change to on-disk format, WAL, catalogs, wire protocol or extension ABI. One edge was not checked: if a stored SQL function body holding such a MERGE can be created, its dump would fail to restore during pg_upgrade.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It moves an existing error for SRFs in MERGE INSERT values from execution to parse analysis, so it adds no new capability, default, privilege check, information path or resource use for any role.
- maintenance-fit: 1 objection(s). The fix is small, touches only parse_merge.c, and costs nothing for users who do not use the feature. The only objection from this angle is about where the check lives. Back-patch scope and the error text could not be checked from the material.
- already-exists: 0 objection(s). This is a bug fix that moves an existing, always-failing case to an earlier error at parse analysis. No existing feature, extension or hook does this, and the fix adds no new mechanism. [P4] is about how target-list SRFs are run in general, so its reasons do not apply here.
- demand: 0 objection(s). This is a bug fix. The statement already fails on every execution, so moving the error to parse analysis takes nothing from working users and adds no feature surface. A field report in message #1 reproduces it on 17.11, 18.6 and 20devel. Demand is not the question here; whether to back-patch it is.

### Precedents retrieved

- P1: [Revert MERGE patch](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=08ea7a2291db21a618d19d612c8060cda68f1892) (revert, Reverted, 2018-04-12) - judged not relevant
- P2: [Add tests for PL/pgSQL SRFs](https://commitfest.postgresql.org/patch/5223/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P3: [copy_srf (was COPY as a set returning function)](https://commitfest.postgresql.org/patch/869/) (cf, Returned with feedback, 2017-01-31) - judged not relevant; reason summarised from its thread
- P4: [Reimplement targetlist SRFs using ROWS FROM ()](https://commitfest.postgresql.org/patch/759/) (cf, Returned with feedback, 2016-10-03) - used in the review; reason summarised from its thread
- P5: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant
- P6: [Allow pageinspect's bt_page_stats function to return a set of rows instead of a single row](https://commitfest.postgresql.org/patch/3713/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P7: [Reduce memory usage of aggregates in set returning functions](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Functions) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P9: [Replace polyphase merge with a simple k-way balanced merge](https://commitfest.postgresql.org/patch/959/) (cf, Returned with feedback, 2017-02-27) - judged not relevant
- P10: [Range Merge Join](https://commitfest.postgresql.org/patch/3160/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P11: [MERGE](https://commitfest.postgresql.org/patch/1446/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P12: [bytea PL/Perl transform](https://commitfest.postgresql.org/patch/4252/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P13: [Permute underscore separated components of columns before fuzzy matching](https://commitfest.postgresql.org/patch/4282/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P14: [Make it possible to use RETURNING together with conditional DO INSTEAD rules, such as for partitioning setups](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Views_and_Rules) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P15: [Revert "Reject degenerate SPLIT PARTITION with DEFAULT partition"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0392fb900eb89f52988cccd33046443c39c70d1c) (revert, Reverted, 2026-05-20) - judged not relevant
- P16: [Upper limit arguments of pg_logical_slot_xxx_changes functions accept invalid values](https://commitfest.postgresql.org/patch/1725/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P17: [Add function - pg_accept_connections_start_time()](https://commitfest.postgresql.org/patch/5576/) (cf, Returned with feedback, 2025-07-25) - judged not relevant
- P18: [function pg_setting_value_split() to parse shared_preload_libraries etc.](https://commitfest.postgresql.org/patch/2782/) (cf, Withdrawn, 2020-11-29) - judged not relevant
- P19: [Make transformAExprIn() return a flattened bool expression directly](https://commitfest.postgresql.org/patch/6723/) (cf, Withdrawn, 2026-07-11) - judged not relevant
- P20: [Make selectivity example match wording](https://commitfest.postgresql.org/patch/3695/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P21: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P22: [revert: Transform OR clauses to ANY expression](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ff9f72c68f678ded340b431c3e280fe56644a3e7) (revert, Reverted, 2024-04-09) - judged not relevant
- P23: [NOT IN to ANTI JOIN transformation](https://commitfest.postgresql.org/patch/2023/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P24: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 105 s; cost reported by the CLI: $1.07
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:03:16.561Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
