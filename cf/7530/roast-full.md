# Design pre-review: Report buffer access strategies in EXPLAIN

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7530](https://commitfest.postgresql.org/patch/7530/) (Needs review); authors: Bertrand Drouvot (bertranddrouvot.pg)
- Thread: [Report buffer access strategies in EXPLAIN](https://www.postgresql.org/message-id/flat/auCYYbDyrQEwIsQq%40bdtpg), started 2026-10-27
- Messages read: 19 of 19
- Patch prose read: v4-0001-Update-buffer-strategy-descriptions-in-the-buffer.patch, v4-0002-Describe-buffer-access-strategy-types-in-the-glos.patch, v4-0003-Clarify-buffer-access-strategies-in-pg_stat_io-do.patch, v4-0004-Refactor-table-scan-instrumentation-lookup.patch, v4-0005-Add-buffer-access-strategies-to-table-scan-instru.patch, v4-0006-Add-table-scan-instrumentation-for-sample-and-TID.patch, v4-0007-Report-buffer-access-strategies-for-table-scans-i.patch, v4-0008-Report-buffer-access-strategies-for-index-scans-i.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection is at level 1a or 1b, and the two that survived at level 2 are fixable. standards-compat-1 asks for the JSON, XML and YAML shape of the property before planned counters and ring size are added. maintenance-fit-1 asks whether 0008 is worth nine index AM call sites when only Bloom reports a non-normal strategy. Both can be settled in the thread without changing the goal.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code and v5 were not read, so the VERBOSE gating, the non-text output shape and the cost of the recording calls in heap and index AM paths are unchecked. The parallel worker handling in 0006 is also unjudged.

## The proposal, restated

**Problem.** Users and developers cannot see in EXPLAIN whether a scan used normal buffer replacement or a strategy ring meant to limit cache pollution. pg_stat_io shows strategies only per backend and context, with no per plan node attribution. The series also fixes stale buffer README and pg_stat_io text that says strategy I/O happens outside shared buffers.

**Proposed design.** Table and index scan instrumentation gain an optional bitmask of strategies, which the heap AM and the in-core index AMs fill in; an empty mask means the AM does not report one. EXPLAIN (ANALYZE, BUFFERS) prints every strategy seen across rescans, per leader and worker that ran the scan. Sample and TID scans get per worker instrumentation, and patches 0001 to 0003 are documentation fixes that can go in separately.

**User-visible surface.**
- New EXPLAIN property "Buffer Access Strategies:" (for example "normal, bulkread") on Seq, Sample, Bitmap Heap, TID and TID Range Scan nodes.
- Same property on Index Scan, Index Only Scan and Bitmap Index Scan nodes; core AMs report normal and Bloom reports bulkread.
- Table AM and index AM C API: optional strategy mask in scan instrumentation, with record helpers being renamed to RecordTableScanStrategy() and RecordIndexScanStrategy().
- BufferAccessStrategyType moves from bufmgr.h to buf.h.
- New glossary entries for the normal, bulkread, bulkwrite and vacuum strategies, plus reworded pg_stat_io context descriptions.

**What the author claims.**
- Tomas proposed the idea earlier with support from Andres, and Andres agreed to keep the first version to the strategy type only.
- Patches 0001 to 0003 are documentation bug fixes, and the README and pg_stat_io fixes could be backpatched.
- The work is largely orthogonal to Lukas Fittl's instrumentation work and needs only rebasing.
- The feature mostly targets developers and advanced users.

**Thread state.** Sami Imseih reviewed every version and supports it; Michael Paquier reviewed the documentation patches and Lukas Fittl reviewed v4. Open points: show the property only with VERBOSE (the author agreed), whether 0008 is worth it for Bloom alone, and L2 cache wording awaiting Andres.

## 2. Prior attempts

### In bitmap scan explain node show how many relevant heap pages were not examined (related)

- **Link:** [https://commitfest.postgresql.org/patch/2443/](https://commitfest.postgresql.org/patch/2443/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAMkU=1xQcMJgMeNxatR60R+-Bv4TeeO1ZcWmrJS4McLvbBmoXg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAMkU=1xQcMJgMeNxatR60R+-Bv4TeeO1ZcWmrJS4McLvbBmoXg@mail.gmail.com)
- **Date:** decided 2020-03-31; started 2019-05-18
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane questioned the need for the counter, because index-only scans report only "Heap Fetches: n" and get no comparable data. He held that the two should be consistent and that the lack of complaints about index-only scans suggests users do not need the information. Amit Kapila marked the entry Returned with Feedback at the end of the commitfest, without further opinions.
- **Reason (quote):** "Some of both: it seems like these ought to be consistent, and the lack of complaints so far about regular index-only scans suggests that people don't need the info." ([message](https://www.postgresql.org/message-id/25659.1585542569%40sss.pgh.pa.us))
- **What is different this time:** P9 added a counter to Bitmap Heap Scan only, and the objection was that index-only scans had nothing comparable. This series covers table and index scan nodes alike, and the author agreed to show the property only with VERBOSE.


## 3. Objections a reviewer is likely to raise

### 1. Machine-readable EXPLAIN form of the new property is not defined

fixable · level 2, rubric item 10 · psql UX conventions, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A plan visualizer or auto_explain log parser reads EXPLAIN (FORMAT JSON) output. It finds "Buffer Access Strategies" as either a comma-joined string or an array, and per-worker values in a form nobody agreed on. If the shape changes later, for example when ring size or per-strategy counters are added, these tools break after an upgrade.

The text sample "normal, bulkread" is the only form shown, and the documentation hunks in 0007 and 0008 describe only text output. Message #6 and message #8 already plan per-strategy buffer counters as the next step, and message #5 plans ring size, so the key and value shape chosen now must allow those additions. The VERBOSE gating agreed in message #19 is not posted yet, so the visible contract is still the v4 default output.

Thread: [#5 (Bertrand Drouvot, 2026-10-29)](https://www.postgresql.org/message-id/auMCcWJ%2FK9CE%2BVpP%40bdtpg), [#6 (Sami Imseih, 2026-10-29)](https://www.postgresql.org/message-id/CAN12%2BYLQ0KGs7r1YvtddHu7Ypjhc%3D0f4QiOo0S7PrQV80Uw4WA%40mail.gmail.com), [#8 (Sami Imseih, 2026-11-02)](https://www.postgresql.org/message-id/CAN12%2BY%2BC9B_6GNjY5t_dS9BpDe%2BA%3DFhyQzKysm_8rqiEyRotbQ%40mail.gmail.com), [#13 (Lukas Fittl, 2026-11-06)](https://www.postgresql.org/message-id/CAP53Pkxfk62X_BpFX_3z8zsRKgeOQBGNGMZk7Tt8NKHPjvgtgA%40mail.gmail.com), [#19 (Bertrand Drouvot, 2026-11-06)](https://www.postgresql.org/message-id/au2o6TU%2Bqc61264l%40bdtpg)

**Question:** What key name and value type does the property have in JSON, XML and YAML, for the leader and for each worker, and how will later counters fit without changing that shape?

Counter-check: held. The thread and the documentation hunks in 0007 and 0008 show only the text form. Messages #5, #6 and #8 plan ring size and per-strategy counters, so the JSON key and value type are a fair thing to settle now.

### 2. Index AM strategy hooks add nine call sites for one contrib case

fixable · level 2, rubric item 9 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch makes nbtree or GIN use a bulkread ring on some new path but does not call RecordIndexScanStrategy() there. EXPLAIN keeps printing "normal" for that scan. No test fails, and a developer chasing cache pollution trusts the wrong value until someone reads the AM code.

0008 places strategy recording in every in-core index AM, at nine call sites (message #19). Today only Bloom reports anything other than normal. Lukas questioned whether 0008 adds value (message #19 quotes him), and the author agreed that Bloom alone may not justify it. Each future strategy change in any index AM, in core or out of core, must now remember this call, and nothing checks that it does.

Thread: [#13 (Lukas Fittl, 2026-11-06)](https://www.postgresql.org/message-id/CAP53Pkxfk62X_BpFX_3z8zsRKgeOQBGNGMZk7Tt8NKHPjvgtgA%40mail.gmail.com), [#19 (Bertrand Drouvot, 2026-11-06)](https://www.postgresql.org/message-id/au2o6TU%2Bqc61264l%40bdtpg)

**Question:** Which index AM, other than Bloom, is expected to select a non-normal strategy, and how would a missed RecordIndexScanStrategy() call be detected?

Counter-check: held. Message #19 confirms nine call sites and that Bloom is the only in-tree index AM reporting a non-normal strategy. The proposal names no check that catches a missed record call, and the author concedes Bloom alone may not justify 0008.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Table and index scan instrumentation gain a strategy mask, and BufferAccessStrategyType moves from bufmgr.h to buf.h. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Machine-readable EXPLAIN form of the new property is not defined |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | EXPLAIN (ANALYZE, BUFFERS) prints a new "Buffer Access Strategies" property on table and index scan nodes. |
| security / privileges | no |  |
| backpatchability | **yes** | The author proposes to backpatch the README and pg_stat_io documentation fixes in 0001 and 0003. |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0008 adds record calls in every in-core index AM, and 0006 adds per-worker DSM code to Sample and TID scans. |
| performance on the hot path | **yes** | 0005 and 0008 touch heapam.c and the index AM search paths; recording is requested only when buffer instrumentation is on. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What key name and value type does the property have in JSON, XML and YAML output, for the leader and for each worker?
2. How will the planned ring size and per-strategy counters fit into that output without changing its shape?
3. Which index AM other than Bloom is expected to select a non-normal strategy, in core or as an extension?
4. How would a missed RecordIndexScanStrategy() call in a new index AM code path be detected?
5. What does EXPLAIN (ANALYZE, BUFFERS) cost with the added instrumentation, compared with master, on a plan with many scan nodes?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Michael Paquier reviewed only the documentation patches. 0004 to 0008 need someone who knows the buffer manager and the executor's parallel instrumentation code; Andres, Tomas and Melanie were added to the thread but have not replied.
- **Liability:** The owner signs for a new EXPLAIN property in all output formats, an optional strategy mask in the table AM and index AM instrumentation API, and per-worker DSM code for Sample and TID scans. External EXPLAIN parsers depend on the key and value shape from the first release.
- **Author follow-up:** The author answered each review within days, posted four versions, and announced a v5 with the VERBOSE change and the renames.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The author states that the feature mostly targets developers and advanced users. The series is eight patches across explain.c, heapam.c, the executor scan nodes and all in-core index AMs, but 0001 to 0003 are separate documentation fixes and the remaining open points are small.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` More parallel worker instrumentation code ahead of a planned rework. Refuted (already handled): Lukas, who owns the unposted rework, wrote in message #12 that he sees no immediate problems, and in message #13 that 0006 seems fine. 0006 copies the pattern of three existing nodes, and [P24] records a test failure in another feature.
- `demand-1` No user report shows a need for per-node strategy in EXPLAIN. Refuted (no failure scenario): A DBA seeing a property with no setting to change it is not a failure, and pg_stat_get_backend_io cannot attribute per plan node. Several developers back the feature and the author agreed to VERBOSE only; [P9] was about consistency with index-only scans.
- `demand-2` Index scan reporting (0008) adds AM-wide hooks for one Bloom case. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about 0008 and Bloom and adds a concrete scenario of a missed record call.

### Review angles

- prior-attempts: 0 objection(s). No precedent records rejecting this idea. P9's demand argument is weakened here: support from Tomas and Andres is reported (message #1), and output moves to VERBOSE (message #19). P24's revert reason is a test-level risk, outside levels 1-2.
- standards-compat: 1 objection(s). Default-output noise was raised in message #13, and the author agreed to VERBOSE gating in message #19. The strategy names reuse the existing pg_stat_io context values. Only the non-text format contract remains open from this angle.
- replication-ondisk: 0 objection(s). The series adds no persistent or cross-node state: no on-disk format, WAL record, catalog, wire-protocol or replication change. The strategy mask lives in per-query executor instrumentation and parallel DSM, so it exists only inside one server version. The struct changes and the move of BufferAccessStrategyType to buf.h target master (v20) only, and only the doc patches 0001 to 0003 are proposed for backpatch. The nbtree VACUUM bookkeeping invariant does not apply, because nbtsearch.c only records the scan strategy and leaves no state for a later VACUUM pass.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. EXPLAIN ANALYZE already requires the user to run the query, and the new property shows only which buffer replacement policy that user's own scan picked; this follows from table size versus shared_buffers, which pg_class and pg_settings already show.
- maintenance-fit: 2 objection(s). The nbtree VACUUM bookkeeping invariant does not apply, because 0008 only records a scan strategy in nbtsearch.c. Patches 0001 to 0003 are documentation fixes and raise no maintenance concern.
- already-exists: 0 objection(s). No existing feature gives per plan node strategy attribution. pg_stat_io and pg_stat_get_backend_io() count I/O per backend and context, and the author answered this in message #7. The heap rule for choosing bulkread is not documented for users, and it changes across rescans (message #4). No extension or contrib module that reports this is known to me, and a hook would not be cheaper than extending the existing scan instrumentation. Whether the property earns its place, given that most nodes print normal, is a desirability question for another angle (message #13).
- demand: 2 objection(s). Patches 0001 to 0003 fix stale documentation and need no demand case. The objections above apply only to the EXPLAIN reporting in 0004 to 0008.

### Precedents retrieved

- P1: [Eagerly evict bulkwrite strategy ring](https://commitfest.postgresql.org/patch/5919/) (cf, Withdrawn, 2026-02-19) - judged not relevant; reason summarised from its thread
- P2: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P3: [explain_regress, explain(MACHINE), and default to explain(BUFFERS)](https://commitfest.postgresql.org/patch/3409/) (cf, Withdrawn, 2023-01-15) - judged not relevant; reason summarised from its thread
- P4: [revert to showing buffer counts in explain (buffers)](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=27cb626f7a28abd18c0e8a34fc8d6674b0f6cdf3) (revert, Reverted, 2010-02-16) - judged not relevant
- P5: [Implement TODOs for index operator strategy number validation](https://commitfest.postgresql.org/patch/7152/) (cf, Withdrawn, 2026-09-09) - judged not relevant; reason summarised from its thread
- P6: [Revert "Include information on buffer usage during planning phase, in EXPLAIN output."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19db23bcbda99e93321cb0636677ec9c6e121a2a) (revert, Reverted, 2020-04-03) - judged not relevant
- P7: [explain: show JIT details in non-text format, even if zero](https://commitfest.postgresql.org/patch/2766/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P8: [New strategies for freezing, advancing relfrozenxid early](https://commitfest.postgresql.org/patch/3843/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P9: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - used in the review; reason summarised from its thread
- P10: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - judged not relevant; reason summarised from its thread
- P11: [Revert the default toast compression strategy to the former behavior where there's no limit on the size of datum we'll try to compress.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=229bffedbc52f12aadb95ca2ee24d5bef36b7c7d) (revert, Reverted, 2009-01-06) - judged not relevant
- P12: [Revert function to get memory context stats for processes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb844b9f06568aa29303122ac0bb7b4c3cce2542) (revert, Reverted, 2025-05-23) - judged not relevant
- P13: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant; reason summarised from its thread
- P14: [vacuum verbose is unclear](https://commitfest.postgresql.org/patch/2425/) (cf, Returned with feedback, 2020-07-05) - judged not relevant
- P15: [\[BUG\] Assert fails for local buffers in BufferIsExclusiveLocked](https://commitfest.postgresql.org/patch/5427/) (cf, Withdrawn, 2024-12-09) - judged not relevant
- P16: [Showing applied extended statistics in explain Part2](https://commitfest.postgresql.org/patch/5077/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P17: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - used in the review
- P18: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P19: [Generate Doxygen documentation](https://commitfest.postgresql.org/patch/4641/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P20: [pg_buffercache: Add per-relation summary stats](https://commitfest.postgresql.org/patch/6555/) (cf, Returned with feedback, 2026-05-04) - judged not relevant
- P21: [Revert "Add eager and lazy freezing strategies to VACUUM."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6c6b4972664491de408670354fdbccc7c83a1c89) (revert, Reverted, 2023-01-26) - judged not relevant
- P22: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - judged not relevant
- P23: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [Revert "Show index search count in EXPLAIN ANALYZE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d00107cd63e780753aa25563fa37603369997d0c) (revert, Reverted, 2025-03-05) - used in the review

### Warnings

- attachment limit (8) reached; v1-0001-Update-bulk-read-buffer-ring-description-in-the-b.patch not read
- attachment limit (8) reached; v1-0002-Describe-buffer-access-strategy-types-in-the-glos.patch not read
- attachment limit (8) reached; v1-0003-Refactor-table-scan-instrumentation-lookup.patch not read
- attachment limit (8) reached; v1-0004-Report-buffer-access-strategy-for-table-scans-in-.patch not read
- attachment limit (8) reached; v1-0005-Report-buffer-access-strategy-for-index-scans-in-.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 109 s; cost reported by the CLI: $3.28
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 74 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:05:20.727Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
