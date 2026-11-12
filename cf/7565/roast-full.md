# Design pre-review: Quickselect instead of a heap for ORDER BY ... LIMIT N large bounded sorts

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7565](https://commitfest.postgresql.org/patch/7565/) (Needs review); authors: Greg Burd (gregburd)
- Thread: [Quickselect instead of a heap for ORDER BY ... LIMIT N large bounded sorts](https://www.postgresql.org/message-id/flat/attQRRU-Yh-e6rKj%40floki), started 2026-10-23
- Messages read: 5 of 5
- Patch prose read: v3-0002-Add-debug_topn_quickselect-GUC-for-testing-not-fo.patch, v3-0001-Use-quickselect-instead-of-a-heap-for-bounded-sor.patch, v3-0003-Experimental-copy-bounded-sort-survivors-into-a-f.patch, v1-0002-Add-debug_topn_quickselect-GUC-for-testing-not-fo.patch, v1-0001-Use-quickselect-instead-of-a-heap-for-large-bound.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the basic design, and no precedent argues against it. Three fixable level 2 points remain open: the reported LIMIT 1 and 2 slowdown on reversed input (maintenance-fit-1), cost_sort() still charging the heap (maintenance-fit-3), and the EXPLAIN label (standards-compat-1). Someone who knows tuplesort has to decide whether the small-LIMIT slowdown is acceptable. Confidence is medium because v2 and v3 are unreviewed and the numbers come from one machine.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the quickselect's correctness, its memory accounting against work_mem, and the switch to the heap fallback are unchecked. Behaviour under parallel sort, incremental sort and rescans with a changing bound also could not be judged.

## The proposal, restated

**Problem.** A bounded sort keeps a heap of N tuples and pays O(log N) comparisons per admitted tuple, so at large LIMIT, or on input that admits most tuples, it can be slower than a full sort. Bounded sorts also turn off abbreviated keys. Users running ORDER BY ... LIMIT with a large N, or on unfavourable input order, are affected.

**Proposed design.** Tuplesort keeps a buffer of up to Max(2 * N, 1000) tuples and, when it fills, runs quickselect to keep the N smallest and free the rest; incoming tuples are still rejected with one comparison against the largest survivor. At the end the survivors go through tuplesort_sort_memtuples() like an unbounded sort. The heap remains only as a fallback when N tuples alone no longer fit in work_mem.

**User-visible surface.**
- Ties on all sort keys can come out in a different order than before; contrib/cube's test needed a tiebreaker.
- debug_topn_quickselect GUC to switch back to the heap (0002, marked not for commit).
- debug_topn_context_swap GUC to turn off the experimental context swap (0003, experimental).
- New tuplelen callback in the tuplesort variant interface (0003, experimental).
- EXPLAIN still reports "top-N heapsort" whichever method ran.

**What the author claims.**
- About 0.35x to 0.44x of master's time at LIMIT 1M to 2.4M on 5M rows, client-side, including a ~360 ms scan.
- On reversed input, gains start small and reach about 0.2x to 0.4x by LIMIT 10000; random input with a small LIMIT matches master.
- v2 is still slower at LIMIT 1 and 2 on reversed input (1.13x int8, 1.63x en_US at LIMIT 1).
- Peak memory is unchanged; the 1000-tuple floor fits in the initial memtuples array.
- Worst case is O(n log n) per prune, via fallback to sorting after 2 * log2(n) rounds.

**Thread state.** John Naylor replied once: use one method or none, try a buffer floor, and consider Andres' survivor-copy idea; Greg did both in v2 and fixed a contrib/cube tie-order failure in v3. Nobody has reviewed v2 or v3 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-23; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Regression on ORDER BY ... DESC LIMIT 1 over append-ordered tables

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-1`

An application runs `select ... order by created_at desc limit 1` on a table with no usable index, and the heap is physically in insert order. Every tuple is admitted, which is the patch's reversed-input case. After upgrade the query is up to 1.6x slower with en_US text keys, and nothing in EXPLAIN shows why.

Message #4 reports v2 at 1.13x for int8 and 1.63x for en_US at LIMIT 1 on reversed input. Reversed input is not an exotic order: it is what a DESC sort over append-only data sees. LIMIT 1 and 2 are among the most common bounded sorts, so the users who pay are not the large-LIMIT users who gain. Message #4 offers a running-minimum special case but does not commit to it.

Thread: [#2 (John Naylor, 2026-10-28)](https://www.postgresql.org/message-id/CANWCAZa1eLTUZ7aM36u%3DM5zKuCR1Z%3D-M4k%2BQzjXpCzTsjJtb6g%40mail.gmail.com), [#4 (Greg Burd, 2026-10-29)](https://www.postgresql.org/message-id/auNiWcC4iIAyjxEo%40floki)

**Question:** Will the patch keep master's speed at LIMIT 1 and 2 on reversed input, and with what mechanism?

Counter-check: held. Message #4 reports 1.13x (int8) and 1.63x (en_US) at LIMIT 1 on reversed input, and v3 does not include the running-minimum case. A descending sort over insert-ordered rows admits every tuple, so the scenario is concrete.

### 2. Cost model still charges the heap the executor no longer runs

fixable · level 2, rubric item 9 · planner behaviour change · `maintenance-fit-3`

A DBA sees the faster Sort+Limit in benchmarks, but the planner still costs it at tuples * log2(2 * N) and keeps choosing a different plan. If cost_sort() is changed later, plan choices change for every bounded sort, including the LIMIT 1 cases the patch makes slower.

Message #1 asks whether cost_sort() should change and gets no answer; message #2 defers it with EXPLAIN as polish. Executor and cost model would then describe different algorithms, and fixing that later moves plans in a release with no executor change to explain it. EXPLAIN still printing "top-N heapsort" adds to the confusion during support.

Thread: [#1 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/attQRRU-Yh-e6rKj%40floki), [#2 (John Naylor, 2026-10-28)](https://www.postgresql.org/message-id/CANWCAZa1eLTUZ7aM36u%3DM5zKuCR1Z%3D-M4k%2BQzjXpCzTsjJtb6g%40mail.gmail.com)

**Question:** What cost_sort() formula matches quickselect, and which plan shapes does it change on the benchmark queries?

Counter-check: held. Message #1 asks whether cost_sort() should change and nobody answers; message #2's "polish" reply was about EXPLAIN only. The gap is a missed plan choice and no regression against master, so it stays fixable.

### 3. EXPLAIN keeps reporting "top-N heapsort" when quickselect ran

fixable · level 2, rubric item 12 · psql UX conventions, planner behaviour change · `standards-compat-1`

After upgrade, a DBA reads "Sort Method: top-N heapsort" in EXPLAIN ANALYZE for a LIMIT 1000000 query and tunes work_mem or the query assuming heap behaviour, though quickselect ran. If the label is instead renamed, plan viewers and scripts that match the string misclassify every bounded sort.

v1 to v3 always print the old label, and message #1 asks whether to add a method or reuse the name. Message #2 defers this as polish, but it is a user-visible contract. EXPLAIN text and JSON output feed external tools, and the memory reported for quickselect, with its 1000-tuple floor and heap fallback, means something different from a heap. Either way the change needs a decision that the patch does not yet make.

Thread: [#1 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/attQRRU-Yh-e6rKj%40floki), [#2 (John Naylor, 2026-10-28)](https://www.postgresql.org/message-id/CANWCAZa1eLTUZ7aM36u%3DM5zKuCR1Z%3D-M4k%2BQzjXpCzTsjJtb6g%40mail.gmail.com), [#5 (Greg Burd, 2026-10-30)](https://www.postgresql.org/message-id/auTXamBHfNQVDsoN%40floki)

**Question:** Will EXPLAIN name the method that actually ran, and if it does, which tools or expected outputs that match "top-N heapsort" would need to change?

Counter-check: held. Message #1 asks this question and message #2 defers it, so no version decides what EXPLAIN reports. The tuning scenario is weak, but a label that names a heap when no heap ran is a real consistency gap.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The experimental 0003 adds a tuplelen callback to the tuplesort variant interface in tuplesort.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | 0002 adds debug_topn_quickselect (not for commit) and 0003 adds debug_topn_context_swap (experimental). |
| planner behaviour change | **yes** | cost_sort() still charges tuples * log2(2 * N) for a bounded sort that now runs quickselect. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | EXPLAIN reports "top-N heapsort" whichever method ran. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | tuplesort.c gains a quickselect path with a 1000-tuple floor and keeps the heap as a work_mem fallback. |
| performance on the hot path | **yes** | Every ORDER BY ... LIMIT sort changes method; v2 is slower at LIMIT 1 and 2 on reversed input. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the patch keep master's speed at LIMIT 1 and 2 on reversed input, and with what mechanism?
2. What cost_sort() formula matches quickselect, and which plan shapes does it change on the benchmark queries?
3. Will EXPLAIN name the method that ran, and which tools or expected outputs that match "top-N heapsort" would then need to change?
4. Will abbreviated keys stay on for bounded sorts, given that tuplesort_set_bound() decides before any tuple arrives?
5. Do the gains and the 1000-tuple floor hold on aarch64, with parallel and incremental sort, and with very wide tuples?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. John Naylor replied once in message #2 with design direction; nobody has reviewed v2 or v3. The owner must know tuplesort.c and its memory accounting.
- **Liability:** Correct results and no slowdown for every ORDER BY ... LIMIT sort, including the work_mem fallback to the heap, for as long as the code stays in tuplesort.c.
- **Author follow-up:** Good sign: the author posted v2 and v3 within the thread, acted on the review in message #2, and offered to run any requested workload.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The gain is about 0.35x to 0.44x of master's time at large LIMIT and on reversed input from LIMIT 8 up, and the change stays inside tuplesort.c with no new user-facing surface in 0001. The 17 KiB main patch touches a core sort path, so review needs care.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Patch now carries several strategies after asking for one. Refuted (already handled): The fallback reuses master's heap code, and 0001 has a regression test that forces that path and compares it with a full sort. The running-minimum path is not in any patch version, and the 2 * log2(n) cutoff comes from introselect, not from one machine.
- `demand-1` LIMIT 1 and 2 on descending-arriving input get slower. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same LIMIT 1 and 2 regression on reversed input with the same numbers and scenario.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no row about this idea or a close relative. The thread's earlier discussions do not argue against it. Message #1 cites a 2010 thread on quickselect for median and percentile, which is a different use. Message #1 also cites a July thread where the top-N heap lost to a full sort at a large LIMIT, which supports the problem. Message #1 reports a 2014 point that tuplesort cannot allocate a new structure when it runs out of work_mem; the proposal reuses the existing memtuples array, so that point does not apply. Message #2 points to an earlier idea to copy survivors into a new context, and the experimental 0003 builds on it.
- standards-compat: 1 objection(s). Other points do not hold up from this angle. Tie order on equal keys is unspecified in SQL, and the patch documents it. The debug GUCs are marked not for commit, there is no new syntax, and pg_dump is not involved. The cost_sort() change belongs to the planner angle.
- replication-ondisk: 0 objection(s). The proposal changes only in-memory tuplesort work for bounded sorts, so it has no persistent or cross-node state. It adds no WAL records, on-disk format change, catalog column or protocol message, and does not affect pg_upgrade or standbys. The tuplelen callback in tuplesort.h is in the experimental 0003, targets a major release, and has no ABI cost outside extensions that define their own tuplesort variants. The tie-order change is a query-result question for other angles.
- security: 0 objection(s). The proposal changes only how tuplesort picks the N smallest tuples, so it adds no privilege, role, exposure or authentication surface. Worst-case cost from data chosen by an unprivileged user is held to O(n log n) per prune by the fallback to sorting, and peak memory stays within work_mem, so no new denial-of-service path follows from the design.
- maintenance-fit: 3 objection(s). Fit is good: the change stays inside tuplesort, adds no dependency, and John Naylor, who knows the sort code, has engaged once. The objections concern cost to common small-LIMIT queries and the number of code paths left behind.
- already-exists: 0 objection(s). No existing feature or extension meets this need. The only user-side workaround is a LIMIT above INT_MAX / 2, which forces a full sort, and message #1 shows quickselect beating it in all but one case. A change to tuplesort's internal top-N algorithm cannot be done from an extension or a hook. The patch replaces the heap and keeps it only as a work_mem fallback, so it adds no second user-facing way to do the same job.
- demand: 1 objection(s). Demand is otherwise shown: ORDER BY ... LIMIT is a core path, message #1 cites an earlier hackers thread where large LIMITs hit this, and reversed input, which is common for insert-ordered tables, gains at LIMIT 8 and above. The objection is limited to who pays at very small N.

### Precedents retrieved

- P1: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant; reason summarised from its thread
- P2: [Fix out-of-bounds with function PQescapeInternal](https://commitfest.postgresql.org/patch/4910/) (cf, Rejected, 2024-04-04) - judged not relevant; reason summarised from its thread
- P3: [Switch logtape and tuplesort to use int64 instead of long](https://commitfest.postgresql.org/patch/4579/) (cf, Withdrawn, 2023-09-26) - judged not relevant; reason summarised from its thread
- P4: [Memory-bounded HashAgg](https://commitfest.postgresql.org/patch/63/) (cf, Rejected, 2015-01-19) - judged not relevant; reason summarised from its thread
- P5: [Memory accounting towards memory-bounded HashAgg](https://commitfest.postgresql.org/patch/59/) (cf, Rejected, 2015-01-16) - judged not relevant; reason summarised from its thread
- P6: [Memory Accounting (towards memory-bounded HashAgg)](https://commitfest.postgresql.org/patch/269/) (cf, Rejected, 2015-07-26) - judged not relevant; reason summarised from its thread
- P7: [Bug: Reading from single byte character column type may cause out of bounds memory reads](https://commitfest.postgresql.org/patch/3767/) (cf, Returned with feedback, 2022-09-08) - judged not relevant; reason summarised from its thread
- P8: [Avoid obscure DES key buffer loop bounds in pgcrypto](https://commitfest.postgresql.org/patch/6953/) (cf, Withdrawn, 2026-09-25) - judged not relevant; reason summarised from its thread
- P9: [Partial sort](https://commitfest.postgresql.org/patch/409/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P10: [Incremental sort](https://commitfest.postgresql.org/patch/1011/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P11: [Pass down Limit bounds \[take-2\]](https://commitfest.postgresql.org/patch/866/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P12: [Revert "Add support for parsing of large XML data (>= 10MB)"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f2743a7d70e7b2891277632121bb51e739743a47) (revert, Reverted, 2024-01-26) - judged not relevant
- P13: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant
- P14: [Improved crc32c performance with AVX-512 algorithm.](https://commitfest.postgresql.org/patch/5100/) (cf, Returned with feedback, 2025-04-02) - judged not relevant
- P15: [An implementation of multi-key quick sort](https://commitfest.postgresql.org/patch/5013/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P16: [Revert indexed and enlargable binary heap implementation.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=810f64a01567610af7b92b0de930f16f3e20064e) (revert, Reverted, 2024-04-11) - judged not relevant
- P17: [Compress prune/freeze records with Delta Frame of Reference algorithm](https://commitfest.postgresql.org/patch/6574/) (cf, Returned with feedback, 2026-07-07) - judged not relevant
- P18: [SIMD optimization for list_sort and tuple-sort](https://commitfest.postgresql.org/patch/5607/) (cf, Withdrawn, 2025-08-06) - judged not relevant
- P19: [Automatic notification of top transaction IDs](https://commitfest.postgresql.org/patch/3232/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P20: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P21: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P22: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P23: [Automatic notification for top transaction IDs](https://commitfest.postgresql.org/patch/3198/) (cf, Withdrawn, 2021-07-01) - judged not relevant
- P24: [logical_work_mem limit for reorderbuffer](https://commitfest.postgresql.org/patch/1428/) (cf, Returned with feedback, 2018-03-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 119 s; cost reported by the CLI: $1.85
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 73 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:11:06.300Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
