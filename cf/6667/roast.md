# Design pre-review: DISTINCT in plain aggregate window functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6667](https://commitfest.postgresql.org/patch/6667/); [thread](https://www.postgresql.org/message-id/flat/CABXr29H2X%2BHtaPw-R3EheZUgv9fM7nSBjQCCaWCRv62mDYdM3w%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has lasting demand, and no surviving objection questions it. Two design points would likely stop a committer before code review. The grow-only hash table ignores work_mem with no plan to bound it (maintenance-fit-1), and patch 1 lets create view store queries that no release can run (replication-ondisk-1). The frame-dependent type support and value order (standards-compat-2) and the undocumented partial subset (standards-compat-3) also need agreement on the list.

Not checked: Without the code, the per-row cost for window aggregates that do not use DISTINCT could not be checked. Both paths are also unchecked for strict aggregates and pass-by-reference types.

## Prior attempts

- [\[PoC\] Implementation of distinct in Window Aggregates: take two](https://www.postgresql.org/message-id/flat/CALbMxBzgfgxt3tVrEZ+MD0gxZSMLTyoV9kBLCPogLMQbxfYpUw@mail.gmail.com) (same idea; Returned with feedback, 2024-02-01). "The patch which you submitted has been awaiting your attention for quite some time now." ([message](https://www.postgresql.org/message-id/CALDaNm0z1znX0VRM5p9YeRLgsZSQhe0MCm3OQ81baMXC9v2rAw%40mail.gmail.com)) Different now: Patch 2 uses the same sort and dedup per partition. This series adds a grow-only hash path and a roadmap, and the author replied to review; [P2] ended over test failures and no reply.
- [Add support for Distinct in Window aggregates](https://commitfest.postgresql.org/patch/4243/) (same idea; Returned with feedback, 2023-08-03). reason not recorded Different now: No reason is recorded for [P1], so the precedent does not show what this series would need to change.
- [window function improvements WIP](https://www.postgresql.org/message-id/flat/CAN1Pwonf4waD+PWkEFK8ANLua8fPjZ4DmV+hixO62+LiR8gwaA@mail.gmail.com) (similar idea; Returned with feedback, 2020-09-17). "I think that Vik Fearing's patch about "Implement <null treatment> for window functions" is much clear, better and has a chance to be committed." ([message](https://www.postgresql.org/message-id/CAN1PwomQ8JxaTdWkVUSt4XK_irct__78f0YFiBCMT6HF91e0Jw%40mail.gmail.com)) Different now: This series uses generic sort and hash support for the argument type, with no code tied to specific data types. [P4] moved on to a different feature after review.

## Objections a reviewer is likely to raise

### 1. Grow-only hash path ignores work_mem with no plan to bound it

A user runs count(DISTINCT user_id) OVER (PARTITION BY tenant ORDER BY ts) on a large, high-cardinality partition. The hash table holds every distinct value, so the backend grows past work_mem without spilling. The operator finds out when the backend or the whole server hits out-of-memory.

Message #3 shows that every comparable path spills at work_mem, including HashAgg and the patch's own sort path, and this one does not. Message #4 and the v3-0003 commit message call it a known limitation that "would require additional design", but they give no design and no target patch. A committer who pushes patch 3 owns a memory-unbounded executor path in a released version.

**Question:** How will the grow-only path stay within work_mem, and will that land before or together with patch 3?

<sub>fixable, level 2, performance on the hot path; messages [#3](https://www.postgresql.org/message-id/CAAAe_zCfVbNrOUF5FaE38EcwYbtXacThR9xdVe2UJJPeDSL2KA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CABXr29EwORuhHY9KhzWW1HsRG98j2rJ6AbOO_TKnkz91imvGiw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Stored views can hold DISTINCT window aggregates that never execute

A user runs create view with count(distinct x) over (partition by p order by o rows between 3 preceding and current row). The view is stored in pg_rewrite, dumped and restored without complaint. Every select on it then fails at executor init with FEATURE_NOT_SUPPORTED, and the user finds out only when the application queries the view.

Patch 1 moves the rejection from parse analysis to executor init for all frame shapes. The frame clause is fully known at parse analysis, so unsupported shapes could still be rejected there. Only the supported shapes need to reach stored rules. The author's reason, that parse-time rejection blocks view round-tripping, applies to supported shapes only, and for the others it lets the catalog keep rules that no release can run until patches 4-6 land.

**Question:** Why not keep the parse-time error for frame shapes the executor cannot run, so pg_rewrite never stores a view that fails on every execution?

<sub>fixable, level 2, catalog changes; messages [#1](https://www.postgresql.org/message-id/CABXr29H2X%2BHtaPw-R3EheZUgv9fM7nSBjQCCaWCRv62mDYdM3w%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CABXr29EwORuhHY9KhzWW1HsRG98j2rJ6AbOO_TKnkz91imvGiw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 3. Accepted types and value order depend on the frame shape

A report uses count(DISTINCT amount::money) OVER (PARTITION BY p) and works. A later edit adds ORDER BY o for a running total, and the same aggregate fails at execution because money is not hashable. array_agg(DISTINCT x) returns values in a different order after the same edit.

Message #3 shows that GROUP BY agg(DISTINCT x) accepts sortable but non-hashable types, while patch 3 rejects them on grow-only frames. Message #4 defers this as an executor-level decision, but it is part of the user contract once committed. Non-window DISTINCT feeds sorted input in practice, and users depend on that order even though the standard does not require it. A prior attempt drew a question on whether DISTINCT window aggregates are well defined under order sensitivity [P4].

Precedent: [window function improvements WIP](https://www.postgresql.org/message-id/flat/CAN1Pwonf4waD+PWkEFK8ANLua8fPjZ4DmV+hixO62+LiR8gwaA@mail.gmail.com) (Returned with feedback, 2020-09-17)

**Question:** Can the grow-only path fall back to sort for non-hashable types, and what array_agg(DISTINCT) order will the documentation promise across frame shapes?

<sub>fixable, level 2, design; messages [#3](https://www.postgresql.org/message-id/CAAAe_zCfVbNrOUF5FaE38EcwYbtXacThR9xdVe2UJJPeDSL2KA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CABXr29EwORuhHY9KhzWW1HsRG98j2rJ6AbOO_TKnkz91imvGiw%40mail.gmail.com); `standards-compat-2`</sub>

### 4. Partial support by frame shape ships with no documented contract

A release ships patches 1-3 only. Users find by trial that OVER (PARTITION BY p) works and OVER (ROWS 3 PRECEDING) does not, with no SGML entry listing the supported frames. If the sliding-frame patches stall, that frame-dependent rule stays for years.

Message #4 confirms there are no SGML changes and that docs are deferred to the full series. Patches 4-8 have no code posted, and message #4 says CURRENT ROW-start frames have not been studied. The project would commit a syntax whose valid forms depend on frame class, with no plan to finish if the later work stops.

**Question:** Is the patches 1-3 subset a complete, documentable feature on its own if patches 4-8 never land?

<sub>fixable, level 1b, desirability; messages [#3](https://www.postgresql.org/message-id/CAAAe_zCfVbNrOUF5FaE38EcwYbtXacThR9xdVe2UJJPeDSL2KA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CABXr29EwORuhHY9KhzWW1HsRG98j2rJ6AbOO_TKnkz91imvGiw%40mail.gmail.com); `standards-compat-3`</sub>

## Questions for the author

1. How will the grow-only hash table stay within work_mem, and will that land before or together with patch 3?
2. Why not keep the parse-time error for frame shapes the executor cannot run, so that create view rejects them?
3. Can the grow-only path fall back to sort for types that are sortable but not hashable, such as money?
4. What order of values will the documentation promise for array_agg(DISTINCT x) across whole-partition and grow-only frames?
5. Are patches 1-3 a complete feature that can be documented on its own if patches 4-8 never land?

## Touch points

- **catalog changes:** The new windistinct field in WindowFunc changes stored rules in pg_rewrite and needs a catversion bump.
- **ABI / extension API:** Extensions that read or build WindowFunc nodes see the new windistinct field in primnodes.h.
- **SQL-standard conflict or non-standard syntax:** The series accepts agg(DISTINCT x) OVER (...), but only for some frame shapes and argument types.
- **planner behaviour change:** Patch 1 touches clauses.c, and the executor chooses sort or hash where message #3 suggests the planner should.
- **pg_dump / restore round-trip:** ruleutils.c now prints DISTINCT on window aggregates, so views with unsupported frames dump and restore but fail when queried.
- **backpatchability:** see objection standards-compat-3: Partial support by frame shape ships with no documented contract
- **maintenance burden ("who maintains this in 10 years"):** Patches 2 and 3 add two DISTINCT paths to nodeWindowAgg.c, and five roadmap patches have no posted code.
- **performance on the hot path:** The grow-only hash table is not bounded by work_mem, and the series changes the WindowAgg aggregate code.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows window frame execution in nodeWindowAgg.c and the parse and ruleutils handling of WindowFunc. No committer has spoken in the visible thread.
- Priority: fix design first; audience a sizeable minority; review cost medium. Three earlier attempts in [P1], [P2] and [P4] and Oracle support noted in message #3 show lasting demand. Three patches of about 60 KiB, mostly in nodeWindowAgg.c, need a memory bound and a decision on where unsupported frames fail before detailed review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
