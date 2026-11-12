# Design pre-review: Report buffer access strategies in EXPLAIN

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7530](https://commitfest.postgresql.org/patch/7530/); [thread](https://www.postgresql.org/message-id/flat/auCYYbDyrQEwIsQq%40bdtpg), 19 of 19 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection is at level 1a or 1b, and the two that survived at level 2 are fixable. standards-compat-1 asks for the JSON, XML and YAML shape of the property before planned counters and ring size are added. maintenance-fit-1 asks whether 0008 is worth nine index AM call sites when only Bloom reports a non-normal strategy. Both can be settled in the thread without changing the goal.

Not checked: The code and v5 were not read, so the VERBOSE gating, the non-text output shape and the cost of the recording calls in heap and index AM paths are unchecked. The parallel worker handling in 0006 is also unjudged.

## Prior attempts

- [In bitmap scan explain node show how many relevant heap pages were not examined](https://www.postgresql.org/message-id/flat/CAMkU=1xQcMJgMeNxatR60R+-Bv4TeeO1ZcWmrJS4McLvbBmoXg@mail.gmail.com) (related; Returned with feedback, 2020-03-31). "Some of both: it seems like these ought to be consistent, and the lack of complaints so far about regular index-only scans suggests that people don't need the info." ([message](https://www.postgresql.org/message-id/25659.1585542569%40sss.pgh.pa.us)) Different now: P9 added a counter to Bitmap Heap Scan only, and the objection was that index-only scans had nothing comparable. This series covers table and index scan nodes alike, and the author agreed to show the property only with VERBOSE.

## Objections a reviewer is likely to raise

### 1. Machine-readable EXPLAIN form of the new property is not defined

A plan visualizer or auto_explain log parser reads EXPLAIN (FORMAT JSON) output. It finds "Buffer Access Strategies" as either a comma-joined string or an array, and per-worker values in a form nobody agreed on. If the shape changes later, for example when ring size or per-strategy counters are added, these tools break after an upgrade.

The text sample "normal, bulkread" is the only form shown, and the documentation hunks in 0007 and 0008 describe only text output. Message #6 and message #8 already plan per-strategy buffer counters as the next step, and message #5 plans ring size, so the key and value shape chosen now must allow those additions. The VERBOSE gating agreed in message #19 is not posted yet, so the visible contract is still the v4 default output.

**Question:** What key name and value type does the property have in JSON, XML and YAML, for the leader and for each worker, and how will later counters fit without changing that shape?

<sub>fixable, level 2, psql UX conventions; messages [#5](https://www.postgresql.org/message-id/auMCcWJ%2FK9CE%2BVpP%40bdtpg), [#6](https://www.postgresql.org/message-id/CAN12%2BYLQ0KGs7r1YvtddHu7Ypjhc%3D0f4QiOo0S7PrQV80Uw4WA%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAN12%2BY%2BC9B_6GNjY5t_dS9BpDe%2BA%3DFhyQzKysm_8rqiEyRotbQ%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAP53Pkxfk62X_BpFX_3z8zsRKgeOQBGNGMZk7Tt8NKHPjvgtgA%40mail.gmail.com), [#19](https://www.postgresql.org/message-id/au2o6TU%2Bqc61264l%40bdtpg); `standards-compat-1`</sub>

### 2. Index AM strategy hooks add nine call sites for one contrib case

A later patch makes nbtree or GIN use a bulkread ring on some new path but does not call RecordIndexScanStrategy() there. EXPLAIN keeps printing "normal" for that scan. No test fails, and a developer chasing cache pollution trusts the wrong value until someone reads the AM code.

0008 places strategy recording in every in-core index AM, at nine call sites (message #19). Today only Bloom reports anything other than normal. Lukas questioned whether 0008 adds value (message #19 quotes him), and the author agreed that Bloom alone may not justify it. Each future strategy change in any index AM, in core or out of core, must now remember this call, and nothing checks that it does.

**Question:** Which index AM, other than Bloom, is expected to select a non-normal strategy, and how would a missed RecordIndexScanStrategy() call be detected?

<sub>fixable, level 2, ABI / extension API; messages [#13](https://www.postgresql.org/message-id/CAP53Pkxfk62X_BpFX_3z8zsRKgeOQBGNGMZk7Tt8NKHPjvgtgA%40mail.gmail.com), [#19](https://www.postgresql.org/message-id/au2o6TU%2Bqc61264l%40bdtpg); `maintenance-fit-1`</sub>

## Questions for the author

1. What key name and value type does the property have in JSON, XML and YAML output, for the leader and for each worker?
2. How will the planned ring size and per-strategy counters fit into that output without changing its shape?
3. Which index AM other than Bloom is expected to select a non-normal strategy, in core or as an extension?
4. How would a missed RecordIndexScanStrategy() call in a new index AM code path be detected?
5. What does EXPLAIN (ANALYZE, BUFFERS) cost with the added instrumentation, compared with master, on a plan with many scan nodes?

## Touch points

- **ABI / extension API:** Table and index scan instrumentation gain a strategy mask, and BufferAccessStrategyType moves from bufmgr.h to buf.h.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Machine-readable EXPLAIN form of the new property is not defined
- **psql UX conventions:** EXPLAIN (ANALYZE, BUFFERS) prints a new "Buffer Access Strategies" property on table and index scan nodes.
- **backpatchability:** The author proposes to backpatch the README and pg_stat_io documentation fixes in 0001 and 0003.
- **maintenance burden ("who maintains this in 10 years"):** 0008 adds record calls in every in-core index AM, and 0006 adds per-worker DSM code to Sample and TID scans.
- **performance on the hot path:** 0005 and 0008 touch heapam.c and the index AM search paths; recording is requested only when buffer instrumentation is on.

## Who would commit it, and when

- Owner: silent. Michael Paquier reviewed only the documentation patches. 0004 to 0008 need someone who knows the buffer manager and the executor's parallel instrumentation code; Andres, Tomas and Melanie were added to the thread but have not replied.
- Priority: proceed; audience niche; review cost medium. The author states that the feature mostly targets developers and advanced users. The series is eight patches across explain.c, heapam.c, the executor scan nodes and all in-core index AMs, but 0001 to 0003 are separate documentation fixes and the remaining open points are small.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 5.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
