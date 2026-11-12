# Design pre-review: Quickselect instead of a heap for ORDER BY ... LIMIT N large bounded sorts

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7565](https://commitfest.postgresql.org/patch/7565/); [thread](https://www.postgresql.org/message-id/flat/attQRRU-Yh-e6rKj%40floki), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the basic design, and no precedent argues against it. Three fixable level 2 points remain open: the reported LIMIT 1 and 2 slowdown on reversed input (maintenance-fit-1), cost_sort() still charging the heap (maintenance-fit-3), and the EXPLAIN label (standards-compat-1). Someone who knows tuplesort has to decide whether the small-LIMIT slowdown is acceptable. Confidence is medium because v2 and v3 are unreviewed and the numbers come from one machine.

Not checked: The code was not read, so the quickselect's correctness, its memory accounting against work_mem, and the switch to the heap fallback are unchecked. Behaviour under parallel sort, incremental sort and rescans with a changing bound also could not be judged.

## Prior attempts

None found before 2026-10-23.

## Objections a reviewer is likely to raise

### 1. Regression on ORDER BY ... DESC LIMIT 1 over append-ordered tables

An application runs `select ... order by created_at desc limit 1` on a table with no usable index, and the heap is physically in insert order. Every tuple is admitted, which is the patch's reversed-input case. After upgrade the query is up to 1.6x slower with en_US text keys, and nothing in EXPLAIN shows why.

Message #4 reports v2 at 1.13x for int8 and 1.63x for en_US at LIMIT 1 on reversed input. Reversed input is not an exotic order: it is what a DESC sort over append-only data sees. LIMIT 1 and 2 are among the most common bounded sorts, so the users who pay are not the large-LIMIT users who gain. Message #4 offers a running-minimum special case but does not commit to it.

**Question:** Will the patch keep master's speed at LIMIT 1 and 2 on reversed input, and with what mechanism?

<sub>fixable, level 2, performance on the hot path; messages [#2](https://www.postgresql.org/message-id/CANWCAZa1eLTUZ7aM36u%3DM5zKuCR1Z%3D-M4k%2BQzjXpCzTsjJtb6g%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/auNiWcC4iIAyjxEo%40floki); `maintenance-fit-1`</sub>

### 2. Cost model still charges the heap the executor no longer runs

A DBA sees the faster Sort+Limit in benchmarks, but the planner still costs it at tuples * log2(2 * N) and keeps choosing a different plan. If cost_sort() is changed later, plan choices change for every bounded sort, including the LIMIT 1 cases the patch makes slower.

Message #1 asks whether cost_sort() should change and gets no answer; message #2 defers it with EXPLAIN as polish. Executor and cost model would then describe different algorithms, and fixing that later moves plans in a release with no executor change to explain it. EXPLAIN still printing "top-N heapsort" adds to the confusion during support.

**Question:** What cost_sort() formula matches quickselect, and which plan shapes does it change on the benchmark queries?

<sub>fixable, level 2, planner behaviour change; messages [#1](https://www.postgresql.org/message-id/attQRRU-Yh-e6rKj%40floki), [#2](https://www.postgresql.org/message-id/CANWCAZa1eLTUZ7aM36u%3DM5zKuCR1Z%3D-M4k%2BQzjXpCzTsjJtb6g%40mail.gmail.com); `maintenance-fit-3`</sub>

### 3. EXPLAIN keeps reporting "top-N heapsort" when quickselect ran

After upgrade, a DBA reads "Sort Method: top-N heapsort" in EXPLAIN ANALYZE for a LIMIT 1000000 query and tunes work_mem or the query assuming heap behaviour, though quickselect ran. If the label is instead renamed, plan viewers and scripts that match the string misclassify every bounded sort.

v1 to v3 always print the old label, and message #1 asks whether to add a method or reuse the name. Message #2 defers this as polish, but it is a user-visible contract. EXPLAIN text and JSON output feed external tools, and the memory reported for quickselect, with its 1000-tuple floor and heap fallback, means something different from a heap. Either way the change needs a decision that the patch does not yet make.

**Question:** Will EXPLAIN name the method that actually ran, and if it does, which tools or expected outputs that match "top-N heapsort" would need to change?

<sub>fixable, level 2, psql UX conventions; messages [#1](https://www.postgresql.org/message-id/attQRRU-Yh-e6rKj%40floki), [#2](https://www.postgresql.org/message-id/CANWCAZa1eLTUZ7aM36u%3DM5zKuCR1Z%3D-M4k%2BQzjXpCzTsjJtb6g%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/auTXamBHfNQVDsoN%40floki); `standards-compat-1`</sub>

## Questions for the author

1. Will the patch keep master's speed at LIMIT 1 and 2 on reversed input, and with what mechanism?
2. What cost_sort() formula matches quickselect, and which plan shapes does it change on the benchmark queries?
3. Will EXPLAIN name the method that ran, and which tools or expected outputs that match "top-N heapsort" would then need to change?
4. Will abbreviated keys stay on for bounded sorts, given that tuplesort_set_bound() decides before any tuple arrives?
5. Do the gains and the 1000-tuple floor hold on aarch64, with parallel and incremental sort, and with very wide tuples?

## Touch points

- **ABI / extension API:** The experimental 0003 adds a tuplelen callback to the tuplesort variant interface in tuplesort.h.
- **new GUC:** 0002 adds debug_topn_quickselect (not for commit) and 0003 adds debug_topn_context_swap (experimental).
- **planner behaviour change:** cost_sort() still charges tuples * log2(2 * N) for a bounded sort that now runs quickselect.
- **psql UX conventions:** EXPLAIN reports "top-N heapsort" whichever method ran.
- **maintenance burden ("who maintains this in 10 years"):** tuplesort.c gains a quickselect path with a 1000-tuple floor and keeps the heap as a work_mem fallback.
- **performance on the hot path:** Every ORDER BY ... LIMIT sort changes method; v2 is slower at LIMIT 1 and 2 on reversed input.

## Who would commit it, and when

- Owner: qualified committer engaged. John Naylor replied once in message #2 with design direction; nobody has reviewed v2 or v3. The owner must know tuplesort.c and its memory accounting.
- Priority: proceed; audience a sizeable minority; review cost medium. The gain is about 0.35x to 0.44x of master's time at large LIMIT and on reversed input from LIMIT 8 up, and the change stays inside tuplesort.c with no new user-facing surface in 0001. The 17 KiB main patch touches a core sort path, so review needs care.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
