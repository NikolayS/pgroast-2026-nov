# Design pre-review: Improve hash aggregate spilling by writing only the needed columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7310](https://commitfest.postgresql.org/patch/7310/); [thread](https://www.postgresql.org/message-id/flat/7247cfafe659f49340ebd5d9e31a5ced45381879.camel%40icloud.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem and the design are stated in message #1, the cost falls only on executions that spill, and two reviewers reproduced the gains. The earlier planner-side attempt [P3] was reverted for a regression that this executor-side design avoids. Confidence is medium because the code was not read and no committer has spoken in the visible thread.

Not checked: The code was not read, so the handling of input slot types, Datum ownership between the compact slot and the input slot, and memory context use on rescan could not be checked. Test coverage in v2 is unknown.

## Prior attempts

- [Revert "Use CP_SMALL_TLIST for hash aggregate"](https://www.postgresql.org/message-id/flat/20200614181418.mx4bvljmfkkhoqzl@alap3.anarazel.de) (similar idea; Reverted, 2020-07-13). "This reverts commit 4cad2534da6d17067d98cf04be2dfc1bda8f2cd0 due to a performance regression." Different now: P3 narrowed the hash aggregate input in the planner with CP_SMALL_TLIST, so every hash aggregate paid for the projection. This patch leaves the planner and the scan unchanged and narrows tuples only inside nodeAgg, on the first spill.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does v2 add regression tests for sparse-column reconstruction, respill, grouping sets with spilling, rescans and the JIT case from message #3?
2. How does the write path handle an input slot that is neither a heap tuple slot nor a minimal tuple slot?
3. How long do the Datums that the reconstructed input slot borrows from the compact spill slot stay valid, and where does the code document that?
4. Will the commit message say that queries grouping on expressions see no gain because the scan already projects?

## Touch points

- **ABI / extension API:** The patch adds fields to AggState in execnodes.h, which shifts struct offsets and the JIT field number FIELDNO_AGGSTATE_ALL_PERGROUPS.
- **maintenance burden ("who maintains this in 10 years"):** nodeAgg.c gains a second spill tuple layout, a column map and a direct respill path, with no regression tests so far.
- **performance on the hot path:** The patch changes spill code reached from lookup_hash_entries(), which runs for every input tuple; hashagg_spill_tuple() is marked pg_noinline for that reason.

## Who would commit it, and when

- Owner: silent. Someone who knows the executor's hash aggregate spill code in nodeAgg.c and the JIT type synchronization rule for AggState.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is about 10 KiB in two files, adds no user-visible surface, and shows about 24% on TPC-H Q18 and a reviewer-reported -38% geometric mean on spilling ClickBench queries. It helps analytic queries on wide tables that spill, and it needs regression tests and a committer who knows nodeAgg.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
