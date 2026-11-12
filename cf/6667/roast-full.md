# Design pre-review: DISTINCT in plain aggregate window functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6667](https://commitfest.postgresql.org/patch/6667/) (Needs review); authors: Haibo Yan (billyean)
- Thread: [\[PATCH\] DISTINCT in plain aggregate window functions](https://www.postgresql.org/message-id/flat/CABXr29H2X%2BHtaPw-R3EheZUgv9fM7nSBjQCCaWCRv62mDYdM3w%40mail.gmail.com), started 2026-04-08
- Messages read: 4 of 4
- Patch prose read: v3-0001-Parse-and-deparse-DISTINCT-in-window-aggregate-ca.patch, v3-0002-Support-DISTINCT-in-whole-partition-window-aggreg.patch, v3-0003-Extend-DISTINCT-window-aggregates-to-grow-only-fr.patch, v1-0001-Support-parsing-and-deparsing-DISTINCT-in-window-.patch, v1-0002-Implement-executor-support-for-DISTINCT-in-whole-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has lasting demand, and no surviving objection questions it. Two design points would likely stop a committer before code review. The grow-only hash table ignores work_mem with no plan to bound it (maintenance-fit-1), and patch 1 lets create view store queries that no release can run (replication-ondisk-1). The frame-dependent type support and value order (standards-compat-2) and the undocumented partial subset (standards-compat-3) also need agreement on the list.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the per-row cost for window aggregates that do not use DISTINCT could not be checked. Both paths are also unchecked for strict aggregates and pass-by-reference types.

## The proposal, restated

**Problem.** Postgres rejects queries such as count(DISTINCT x) OVER (PARTITION BY p) at parse time with "DISTINCT is not implemented for window functions". The thread does not name users who asked for this. The author also says the parse-time rejection blocks view round-tripping and any later executor work.

**Proposed design.** An eight-patch roadmap adds support step by step, by frame class and then by feature. Posted patches 1-3 carry DISTINCT through parse and deparse with a new WindowFunc field. They run whole-partition frames once per partition through sort and dedup, and grow-only frames (UNBOUNDED PRECEDING, no EXCLUDE) through an incremental hash table of values already seen.

**User-visible surface.**
- SQL: agg(DISTINCT x) OVER (...) is accepted for plain aggregates with a single argument and supported frame shapes.
- Error moves from parse time to executor init: "DISTINCT is not yet implemented for window aggregates" (FEATURE_NOT_SUPPORTED) for frames still out of scope.
- pg_get_viewdef() and stored rules now print DISTINCT on window aggregates.
- Node change: new boolean windistinct field in WindowFunc, visible to extensions that read parse trees.
- Grow-only frames reject argument types that are not hashable, such as money.

**What the author claims.**
- Splitting the work by frame shape matters more than splitting by aggregate kind, because moving frames are the hard part.
- Whole-partition results are constant per partition, so the executor finalizes once and reuses the result.
- Mixed DISTINCT and non-DISTINCT window aggregates work in one query.
- Different array_agg(DISTINCT ...) value order between the two paths is not a correctness issue, because the standard does not fix that order.
- The unbounded memory of the grow-only hash path is a known limitation, not the intended end state.

**Thread state.** Henson Choi reviewed v2 and found results matching equivalent rewrites, but raised hashability, value order, unbounded hash memory, coverage gaps and missing docs. The author posted v3 with more tests and documented limits, and kept the patch split; no one has replied to v3.

## 2. Prior attempts

### [PoC] Implementation of distinct in Window Aggregates: take two (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/4519/](https://commitfest.postgresql.org/patch/4519/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CALbMxBzgfgxt3tVrEZ+MD0gxZSMLTyoV9kBLCPogLMQbxfYpUw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALbMxBzgfgxt3tVrEZ+MD0gxZSMLTyoV9kBLCPogLMQbxfYpUw@mail.gmail.com)
- **Date:** decided 2024-02-01; started 2023-08-27
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The patch crashed in cfbot and failed the regress, postgres_fdw, recovery and pg_upgrade tests. The author did not reply to that report, so the CommitFest manager moved the entry to Returned with Feedback. No one objected to the design.
- **Reason (quote):** "The patch which you submitted has been awaiting your attention for quite some time now." ([message](https://www.postgresql.org/message-id/CALDaNm0z1znX0VRM5p9YeRLgsZSQhe0MCm3OQ81baMXC9v2rAw%40mail.gmail.com))
- **What is different this time:** Patch 2 uses the same sort and dedup per partition. This series adds a grow-only hash path and a roadmap, and the author replied to review; [P2] ended over test failures and no reply.

### Add support for Distinct in Window aggregates (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/4243/](https://commitfest.postgresql.org/patch/4243/)
- **Date:** decided 2023-08-03; started 2023-03-12
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** No reason is recorded for [P1], so the precedent does not show what this series would need to change.

### window function improvements WIP (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2419/](https://commitfest.postgresql.org/patch/2419/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAN1Pwonf4waD+PWkEFK8ANLua8fPjZ4DmV+hixO62+LiR8gwaA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAN1Pwonf4waD+PWkEFK8ANLua8fPjZ4DmV+hixO62+LiR8gwaA@mail.gmail.com)
- **Date:** decided 2020-09-17; started 2020-01-13
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane asked whether DISTINCT on a window function is well defined, given one output row per input row and order sensitivity. He also objected to datatype-specific code, so the author suspended the DISTINCT work. Surafel Temesgen's review of the null treatment patch found white space errors, uncommented functions, oversized regression tests and an unexplained change to makeBoolAConst. The author then judged Vik Fearing's competing <null treatment> patch the better design and offered to withdraw his own, and Michael Paquier closed the entry.
- **Reason (quote):** "I think that Vik Fearing's patch about "Implement <null treatment> for window functions" is much clear, better and has a chance to be committed." ([message](https://www.postgresql.org/message-id/CAN1PwomQ8JxaTdWkVUSt4XK_irct__78f0YFiBCMT6HF91e0Jw%40mail.gmail.com))
- **What is different this time:** This series uses generic sort and hash support for the argument type, with no code tied to specific data types. [P4] moved on to a different feature after review.


## 3. Objections a reviewer is likely to raise

### 1. Grow-only hash path ignores work_mem with no plan to bound it

fixable · level 2, rubric item 8 · performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A user runs count(DISTINCT user_id) OVER (PARTITION BY tenant ORDER BY ts) on a large, high-cardinality partition. The hash table holds every distinct value, so the backend grows past work_mem without spilling. The operator finds out when the backend or the whole server hits out-of-memory.

Message #3 shows that every comparable path spills at work_mem, including HashAgg and the patch's own sort path, and this one does not. Message #4 and the v3-0003 commit message call it a known limitation that "would require additional design", but they give no design and no target patch. A committer who pushes patch 3 owns a memory-unbounded executor path in a released version.

Thread: [#3 (Henson Choi, 2026-06-26)](https://www.postgresql.org/message-id/CAAAe_zCfVbNrOUF5FaE38EcwYbtXacThR9xdVe2UJJPeDSL2KA%40mail.gmail.com), [#4 (Haibo Yan, 2026-06-28)](https://www.postgresql.org/message-id/CABXr29EwORuhHY9KhzWW1HsRG98j2rJ6AbOO_TKnkz91imvGiw%40mail.gmail.com)

**Question:** How will the grow-only path stay within work_mem, and will that land before or together with patch 3?

Counter-check: held. Message #4 and the v3-0003 commit message confirm that the hash table ignores work_mem, and they give no design or target patch. Naming the limitation in comments does not bound the memory.

### 2. Stored views can hold DISTINCT window aggregates that never execute

fixable · level 2, rubric item 12 · catalog changes, pg_dump / restore round-trip · `replication-ondisk-1`

A user runs create view with count(distinct x) over (partition by p order by o rows between 3 preceding and current row). The view is stored in pg_rewrite, dumped and restored without complaint. Every select on it then fails at executor init with FEATURE_NOT_SUPPORTED, and the user finds out only when the application queries the view.

Patch 1 moves the rejection from parse analysis to executor init for all frame shapes. The frame clause is fully known at parse analysis, so unsupported shapes could still be rejected there. Only the supported shapes need to reach stored rules. The author's reason, that parse-time rejection blocks view round-tripping, applies to supported shapes only, and for the others it lets the catalog keep rules that no release can run until patches 4-6 land.

Thread: [#1 (Haibo Yan, 2026-04-08)](https://www.postgresql.org/message-id/CABXr29H2X%2BHtaPw-R3EheZUgv9fM7nSBjQCCaWCRv62mDYdM3w%40mail.gmail.com), [#4 (Haibo Yan, 2026-06-28)](https://www.postgresql.org/message-id/CABXr29EwORuhHY9KhzWW1HsRG98j2rJ6AbOO_TKnkz91imvGiw%40mail.gmail.com)

**Question:** Why not keep the parse-time error for frame shapes the executor cannot run, so pg_rewrite never stores a view that fails on every execution?

Counter-check: held. The frame options are known at parse analysis, so unsupported shapes could still be rejected there. The commit message's round-trip reason covers only shapes the executor can run.

### 3. Accepted types and value order depend on the frame shape

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, planner behaviour change · `standards-compat-2`

A report uses count(DISTINCT amount::money) OVER (PARTITION BY p) and works. A later edit adds ORDER BY o for a running total, and the same aggregate fails at execution because money is not hashable. array_agg(DISTINCT x) returns values in a different order after the same edit.

Message #3 shows that GROUP BY agg(DISTINCT x) accepts sortable but non-hashable types, while patch 3 rejects them on grow-only frames. Message #4 defers this as an executor-level decision, but it is part of the user contract once committed. Non-window DISTINCT feeds sorted input in practice, and users depend on that order even though the standard does not require it. A prior attempt drew a question on whether DISTINCT window aggregates are well defined under order sensitivity [P4].

**Precedents.**
- [window function improvements WIP](https://commitfest.postgresql.org/patch/2419/) (Returned with feedback, 2020-09-17); thread: [link](https://www.postgresql.org/message-id/flat/CAN1Pwonf4waD+PWkEFK8ANLua8fPjZ4DmV+hixO62+LiR8gwaA@mail.gmail.com)

Thread: [#3 (Henson Choi, 2026-06-26)](https://www.postgresql.org/message-id/CAAAe_zCfVbNrOUF5FaE38EcwYbtXacThR9xdVe2UJJPeDSL2KA%40mail.gmail.com), [#4 (Haibo Yan, 2026-06-28)](https://www.postgresql.org/message-id/CABXr29EwORuhHY9KhzWW1HsRG98j2rJ6AbOO_TKnkz91imvGiw%40mail.gmail.com)

**Question:** Can the grow-only path fall back to sort for non-hashable types, and what array_agg(DISTINCT) order will the documentation promise across frame shapes?

Counter-check: held. Message #4 concedes both differences and defers them, so the thread does not answer the objection. It stands on message #3 alone; [P4] records a question, so it should not be cited as support.

### 4. Partial support by frame shape ships with no documented contract

fixable · level 1b, rubric item 6 · desirability, SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-3`

A release ships patches 1-3 only. Users find by trial that OVER (PARTITION BY p) works and OVER (ROWS 3 PRECEDING) does not, with no SGML entry listing the supported frames. If the sliding-frame patches stall, that frame-dependent rule stays for years.

Message #4 confirms there are no SGML changes and that docs are deferred to the full series. Patches 4-8 have no code posted, and message #4 says CURRENT ROW-start frames have not been studied. The project would commit a syntax whose valid forms depend on frame class, with no plan to finish if the later work stops.

Thread: [#3 (Henson Choi, 2026-06-26)](https://www.postgresql.org/message-id/CAAAe_zCfVbNrOUF5FaE38EcwYbtXacThR9xdVe2UJJPeDSL2KA%40mail.gmail.com), [#4 (Haibo Yan, 2026-06-28)](https://www.postgresql.org/message-id/CABXr29EwORuhHY9KhzWW1HsRG98j2rJ6AbOO_TKnkz91imvGiw%40mail.gmail.com)

**Question:** Is the patches 1-3 subset a complete, documentable feature on its own if patches 4-8 never land?

Counter-check: held. Message #4 defers the SGML changes to the full series, and patches 4-8 have no posted code. The thread does not say whether patches 1-3 are a complete feature if the rest stalls.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The new windistinct field in WindowFunc changes stored rules in pg_rewrite and needs a catversion bump. |
| ABI / extension API | **yes** | Extensions that read or build WindowFunc nodes see the new windistinct field in primnodes.h. |
| SQL-standard conflict or non-standard syntax | **yes** | The series accepts agg(DISTINCT x) OVER (...), but only for some frame shapes and argument types. |
| new GUC | no |  |
| planner behaviour change | **yes** | Patch 1 touches clauses.c, and the executor chooses sort or hash where message #3 suggests the planner should. |
| pg_dump / restore round-trip | **yes** | ruleutils.c now prints DISTINCT on window aggregates, so views with unsupported frames dump and restore but fail when queried. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-3: Partial support by frame shape ships with no documented contract |
| maintenance burden ("who maintains this in 10 years") | **yes** | Patches 2 and 3 add two DISTINCT paths to nodeWindowAgg.c, and five roadmap patches have no posted code. |
| performance on the hot path | **yes** | The grow-only hash table is not bounded by work_mem, and the series changes the WindowAgg aggregate code. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How will the grow-only hash table stay within work_mem, and will that land before or together with patch 3?
2. Why not keep the parse-time error for frame shapes the executor cannot run, so that create view rejects them?
3. Can the grow-only path fall back to sort for types that are sortable but not hashable, such as money?
4. What order of values will the documentation promise for array_agg(DISTINCT x) across whole-partition and grow-only frames?
5. Are patches 1-3 a complete feature that can be documented on its own if patches 4-8 never land?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows window frame execution in nodeWindowAgg.c and the parse and ruleutils handling of WindowFunc. No committer has spoken in the visible thread.
- **Liability:** The owner signs for SQL syntax whose valid forms depend on frame shape, a new field in stored rules, and a hash path with no memory bound. This lasts until the later roadmap patches land.
- **Author follow-up:** The author posted v2 and v3, answered the review of v2, and moved the entry to the next CommitFest in September 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Three earlier attempts in [P1], [P2] and [P4] and Oracle support noted in message #3 show lasting demand. Three patches of about 60 KiB, mostly in nodeWindowAgg.c, need a memory bound and a decision on where unsupported frames fail before detailed review.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Plan-dependent value order revives the earlier well-definedness concern. Refuted (not specific): Duplicate of standards-compat-2, which covers the same value-order difference together with the type restriction.
- `standards-compat-1` Rejection moves to executor init, so stored objects accept queries that cannot run. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same executor-init concern and adds that only supported shapes need to reach stored rules.
- `security-1` Grow-only hash path lets any user exceed work_mem without limit. Refuted (not specific): Duplicate of maintenance-fit-1. The privilege framing adds little, because the objection itself notes that array_agg already lets any user grow memory without limit.
- `maintenance-fit-2` Unsupported frame shapes now fail at executor start. Refuted (not specific): Duplicate of replication-ondisk-1, which describes the same stored-view failure.
- `maintenance-fit-3` Partial series fixes executor strategy choice with no committer engaged. Refuted (not specific): Duplicate of standards-compat-3 on the stalled-series risk, and message #4 answers the redundancy point for patch 3. The lack of a committer belongs in the committability field.
- `already-exists-1` Patches 2-3 cover frames that existing rewrites already handle. Refuted (already handled): The material shows demand that lasted: [P1], [P2] and [P4] are earlier attempts by two other authors, and message #3 says Oracle runs the whole-partition forms. [P2] records that no one objected to the design.

### Review angles

- prior-attempts: 1 objection(s). The earlier attempts ended over process problems or for reasons not recorded. Only the order-sensitivity point from [P4] may still apply, and it is raised above.
- standards-compat: 3 objection(s). Three objections raised. Value order and hashability are combined into one.
- replication-ondisk: 1 objection(s). The only persistent change is the new windistinct field in WindowFunc, stored in pg_rewrite and needing a catversion bump. There is no WAL, on-disk relation format, wire protocol or replication change.
- security: 1 objection(s). The proposal adds no new privilege, role or object type. Aggregates run with the caller's rights as they do today, so the only security issue is the unbounded memory use described above.
- maintenance-fit: 3 objection(s). Objections were found from the maintenance and fit angle. Per-tuple cost for non-DISTINCT window queries could not be checked without the code.
- already-exists: 1 objection(s). No extension or contrib module provides DISTINCT window aggregates, so only the query-rewrite alternative applies from this angle.
- demand: 0 objection(s). The thread names no users, but [P1], [P2] and [P4] show three earlier attempts by two authors between 2020 and 2024, and message #3 confirms Oracle supports the whole-partition forms. The cost falls mostly on queries that use DISTINCT, and a committer would likely accept the long-standing parse-time error as evidence of a real gap.

### Precedents retrieved

- P1: [Add support for Distinct in Window aggregates](https://commitfest.postgresql.org/patch/4243/) (cf, Returned with feedback, 2023-08-03) - used in the review
- P2: [\[PoC\] Implementation of distinct in Window Aggregates: take two](https://commitfest.postgresql.org/patch/4519/) (cf, Returned with feedback, 2024-02-01) - used in the review; reason summarised from its thread
- P3: [parallel distinct union and aggregate support patch](https://commitfest.postgresql.org/patch/2769/) (cf, Returned with feedback, 2021-11-08) - judged not relevant; reason summarised from its thread
- P4: [window function improvements WIP](https://commitfest.postgresql.org/patch/2419/) (cf, Returned with feedback, 2020-09-17) - used in the review; reason summarised from its thread
- P5: [Removing useless DISTINCT clauses](https://commitfest.postgresql.org/patch/1369/) (cf, Returned with feedback, 2018-04-07) - judged not relevant; reason summarised from its thread
- P6: [Remove useless distinct clauses](https://commitfest.postgresql.org/patch/2663/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P7: [Implement <null treatment> for window functions](https://commitfest.postgresql.org/patch/2628/) (cf, Returned with feedback, 2021-03-25) - judged not relevant; reason summarised from its thread
- P8: [possibility to write window function in PL languages (like PLpgSQL)](https://commitfest.postgresql.org/patch/2697/) (cf, Withdrawn, 2021-01-27) - judged not relevant; reason summarised from its thread
- P9: [\[WIP\] Show a human-readable n_distinct in pg_stats view](https://commitfest.postgresql.org/patch/2061/) (cf, Withdrawn, 2019-08-28) - judged not relevant; reason summarised from its thread
- P10: [Allow DISTINCT to work in multiple-argument aggregate calls](https://wiki.postgresql.org/wiki/Not_Worth_Doing#SQL_Commands) (do_not_want, Not worth doing, date unknown) - used in the review
- P11: [quick and dirty fix: Quadratic planning time for ordered paths over partitioned tables](https://commitfest.postgresql.org/patch/5516/) (cf, Rejected, 2025-02-03) - judged not relevant
- P12: [loop over unbound cursor](https://commitfest.postgresql.org/patch/2558/) (cf, Withdrawn, 2020-08-10) - judged not relevant
- P13: [Enable fine-grained control over what gets logged on connection attempt (reduces log size)](https://commitfest.postgresql.org/patch/4005/) (cf, Returned with feedback, 2023-08-03) - judged not relevant
- P14: [iteration over record in plpgsql](https://commitfest.postgresql.org/patch/124/) (cf, Returned with feedback, 2015-07-07) - judged not relevant
- P15: [Revert: Avoid looping over all type cache entries in TypeCacheRelCallback()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8daa62a10c911c851f7e9ec5ef7b90cfd4b73212) (revert, Reverted, 2024-08-25) - judged not relevant
- P16: [Implement type input functions for extended statistics types](https://commitfest.postgresql.org/patch/4114/) (cf, Withdrawn, 2023-03-01) - judged not relevant
- P17: [Revert "Optimize various aggregate deserialization functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f3b56eea23554e1756a26080db273156f23f4f2) (revert, Reverted, 2023-10-10) - judged not relevant
- P18: [Revert "Add a test for transition table usage in FOR EACH ROW trigger."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a19ea9c6601bfb06dfd9f4c1060550dbc3f7bde1) (revert, Reverted, 2017-05-16) - judged not relevant
- P19: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P20: [Support ORDER BY in CREATE FUNCTION for Set Returning Functions](https://commitfest.postgresql.org/patch/88/) (cf, Rejected, 2015-05-03) - judged not relevant
- P21: [Erase the distinctClause if the result is unique by definition](https://commitfest.postgresql.org/patch/2433/) (cf, Returned with feedback, 2022-04-05) - judged not relevant
- P22: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant
- P23: [Reduce memory usage of aggregates in set returning functions](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Functions) (do_not_want, Not worth doing, date unknown) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 156 s; cost reported by the CLI: $2.40
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 4 of 23 kept
- Tone check: 103 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:59:09.274Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
