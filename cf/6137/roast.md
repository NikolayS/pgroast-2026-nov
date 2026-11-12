# Design pre-review: postgres_fdw: Use COPY to speed up batch inserts

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6137](https://commitfest.postgresql.org/patch/6137/); [thread](https://www.postgresql.org/message-id/flat/DDIZJ217OUDK.2R5WE4OGL5PTY%40gmail.com), 37 of 37 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support from several reviewers, but demand-1 survives at level 1b. The only whole-file benchmark (message #36) shows v14 and v16 about twice as slow as HEAD, and only a variant that keeps COPY IN open was faster. Message #35 rejected that variant for connection sharing, and no reply reconciles the two. The benefit of the design as proposed is not yet shown, so design objections were not ranked.

Not checked: Only 3 of 37 messages and no v16 patch text were read, so the benchmark and cancel reports rest on the brief's summary of message #36. The four surviving level 2 objections were not evaluated because level 1b fails first.

## Prior attempts

None found before 2025-10-15.

## Objections a reviewer is likely to raise

### 1. Only independent benchmark shows v16 slower than HEAD for COPY FROM

A user loading a large file with `COPY ft FROM` into a postgres_fdw table with batch_size set upgrades and expects a faster load. In Kirill's two-VM setup the load of 1M rows takes about twice as long as on HEAD. The user finds out on the first production load.

The author's speedup numbers (message #31) come from pgbench running COPY of 100 rows per transaction. Kirill measured COPY of whole files in message #36: v14 and v16 were slower than HEAD on local and cross-AZ VMs, and only his draft connection-level COPY caching made it faster. That caching keeps COPY IN open across batches, which message #35 rejected because of connection sharing. No reply to message #36 is visible, so the user benefit of the design as proposed is not shown.

**Question:** For a bulk COPY FROM of a large file across a real network, is v16 faster than HEAD, and if only with persistent COPY IN, how does that work with shared connections?

<sub>fixable, level 1b, desirability; messages [#31](https://www.postgresql.org/message-id/DHN84NSGBYYO.123WDP7K8D21D%40gmail.com), [#35](https://www.postgresql.org/message-id/CAFY6G8ekRx2U9Bno7ft9fTkth2HX1rA9GKNVnJtqn%3D2X76Xcyw%40mail.gmail.com), [#36](https://www.postgresql.org/message-id/CALdSSPho7cJBAtNnHJLpCz_zOAqA%3DhQz%3DK125zpSORKaW%3DJA5g%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. For COPY FROM of a large file between two hosts, is v16 faster than HEAD, and at which batch_size values?
2. If only a COPY IN kept open across batches beats HEAD, how does that work with foreign partitions that share one cached connection?
3. How does v16 end an open COPY IN on the remote connection when the user cancels or a row fails, and which test covers it?
4. What happens when the remote target is a view, has rules, or has row-level security for the mapped role, and how can users keep remote INSERT?
5. What can ExecForeignBatchCopy do that ExecForeignBatchInsert cannot, given a flag that says the caller is COPY FROM?

## Touch points

- **ABI / extension API:** v16 adds the optional FdwRoutine callback ExecForeignBatchCopy and exports CopyEscapeText() in commands/copy.h.
- **security / privileges:** Remote COPY FROM treats row-level security, views and rules differently from the remote INSERT used today.
- **maintenance burden ("who maintains this in 10 years"):** A new public FDW callback and a COPY IN state that postgres_fdw abort and cancel code must handle.
- **performance on the hot path:** The patch changes how CopyFrom() flushes each batch to foreign tables, and message #36 measured it slower than HEAD.

## Who would commit it, and when

- Owner: qualified committer engaged. Masahiko Sawada reviewed several versions and agreed off-list to the callback design. The owner needs to know the FDW API, CopyFrom() batching and postgres_fdw connection cleanup.
- Priority: needs design rethink; audience a sizeable minority; review cost medium. Users who bulk load into postgres_fdw tables or foreign partitions would gain, and a committer is engaged. But the per-batch COPY design measured slower than HEAD in message #36, and the callback set is not settled, so more code review should wait.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
