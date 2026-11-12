# Design pre-review: Speed up lpad() and rpad()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7340](https://commitfest.postgresql.org/patch/7340/); [thread](https://www.postgresql.org/message-id/flat/CAH7T-apj%2BpFg9bRkXVGfGejS2Uu4WzdMd7KoLKTW11mdsrt1Ew%40mail.gmail.com), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and no level fails. A committer co-wrote the series and Heikki asked for the shared design. Three fixable points are open: demand-2 and maintenance-fit-2 rest on the regressions measured in message #12, which have no reply, and demand-1 on the lack of a production workload with large N. The committer who signs must weigh these.

Not checked: Without the code, this review could not check how cheaply append_padding() and repeat_bytes() could add a small-N or large-source path. It also could not check pad validation for the partial tail.

## Prior attempts

- [optimize several list functions with SIMD intrinsics](https://www.postgresql.org/message-id/flat/20230308002502.GA3378449@nathanxps13) (related; Returned with feedback, 2023-07-10). "Grepping for "tlist_member" and "list_delete_ptr", I don't see any callers in hot codepaths where this could make a noticeable difference." ([message](https://www.postgresql.org/message-id/1184a66b-efa4-6fc8-ba8c-72db6f3b1352%40iki.fi)) Different now: P4 changed List searches and showed no hot callers. This proposal shows large measured gains, but only at N far above what message #7 calls typical, so the question of a real workload is shared.

## Objections a reviewer is likely to raise

### 1. Short-pad callers pay so that large-pad callers gain

A reporting query formats millions of rows with lpad(code, 7, '0') on ARM. Each call now runs more instructions than on master, so the common formatting case gets slower, and nobody notices until a release-to-release comparison.

Message #12 measures lpad(s, 7, '0') at 772 instructions per call against 694 on master, and reports that padding of up to three characters costs more. That size is the usual case John Naylor describes in message #7. Message #12 also finds repeat() with a 1000-byte source 24% slower at 10M. The thread has no reply to these numbers, so the trade between common small calls and rare large calls has not been weighed.

**Question:** Can the patch keep short pads, up to about 16 characters, at master's cost on both x86 and ARM, with timings for a typical row-formatting workload?

<sub>fixable, level 1b, desirability; messages [#7](https://www.postgresql.org/message-id/CANWCAZaB8NsQJrKc-pxRTj_%3DC8CBOTDEWkPW0Eur%3DymUSbUwZQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAGRkXqQ%2BFM6OMnZo%3Da-p2xqXSGGwC6VD4C7hmsJEASiyFBecrQ%40mail.gmail.com); `demand-2`</sub>

### 2. Speedup targets large N, which the thread shows mainly in test data

A committer pushes the shared doubling helper for lpad(), rpad() and repeat(). Typical callers, such as lpad(id, 7, '0'), see no change, so Postgres takes on a new copy strategy and a tuned 16 kB block for a gain few production queries see.

In message #7, John Naylor says real N rarely exceeds 100, and the tables in message #1 show no difference up to N = 100. Message #8 answers with join_hash.sql, benchmarks and reproducers, all of which build test data once and do not run in production loops. No field report, -general thread or user workload is cited, and message #8 got no reply. The cancellation gain from message #11 is real, but adding CHECK_FOR_INTERRUPTS() to the existing loop would also give it.

Precedent: [optimize several list functions with SIMD intrinsics](https://www.postgresql.org/message-id/flat/20230308002502.GA3378449@nathanxps13) (Returned with feedback, 2023-07-10)

**Question:** Which production workloads call lpad(), rpad() or repeat() with results large enough to matter, and how much time do they spend there today?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAH7T-apj%2BpFg9bRkXVGfGejS2Uu4WzdMd7KoLKTW11mdsrt1Ew%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CANWCAZaB8NsQJrKc-pxRTj_%3DC8CBOTDEWkPW0Eur%3DymUSbUwZQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAH7T-aoGV1tTdYy0OrDxkgkue%3Du7wTRxztTLPxUs0OZ%3DL2hF%3Dg%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/179029949414.497877.6537870082909473028%40gmail.com); `demand-1`</sub>

### 3. Fixed 16 kB block makes some repeat() calls slower

A user on Apple Silicon builds 10M test values with repeat() from a 1000-byte source and gets results 24% slower after upgrading. Later reports from other CPUs or glibc versions ask for a different block size. The project then has to tune a constant that depends on memcpy() behaviour it does not control.

On master, repeat() with a large source was already at memcpy() speed, and message #12 shows v5 is 24% slower there, which message #12 attributes to memory cost and says block size does not fix. The 16 kB value rests on one machine (message #10) plus Zen 5 (message #11). The repeat() thread found a different best size on Intel, tied to glibc's non-temporal threshold. Nothing in the tree would detect a later regression on another platform.

**Question:** Can repeat_bytes() keep master's per-copy loop when the source is already large, so that no measured platform regresses, and what policy decides the block size on future hardware?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#10](https://www.postgresql.org/message-id/CAH7T-aq%2BahEAKamY1JU2jN6X_N3OLD63kJ7jfn9YyRVNd1GkBA%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/179029949414.497877.6537870082909473028%40gmail.com), [#12](https://www.postgresql.org/message-id/CAGRkXqQ%2BFM6OMnZo%3Da-p2xqXSGGwC6VD4C7hmsJEASiyFBecrQ%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Can v5 keep pads of a few characters at master's cost on both x86 and ARM, shown with timings for a row-formatting query such as lpad(s, 7, '0')?
2. Which production workloads call lpad(), rpad() or repeat() with results large enough to matter, and how much time do they spend there today?
3. Can repeat_bytes() keep master's per-copy loop for large sources, so that repeat() with a 1000-byte source at 10M is not slower on M2 Max?
4. How does v5 with the 16 kB block perform on an Intel machine, where the repeat() thread found a different best size?
5. How will this series be ordered or merged with the Ready for Committer repeat() patch from the other thread?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** repeat_bytes() carries a fixed 16 kB block size tuned on three machines.
- **performance on the hot path:** The series rewrites the copy loops of lpad(), rpad() and repeat(), which run once per row in formatting queries.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Speedup targets large N, which the thread shows mainly in test data

## Who would commit it, and when

- Owner: qualified committer engaged. Nathan Bossart co-wrote the series and is listed as committer on the CommitFest entry.
- Priority: proceed; audience niche; review cost low. The series is three small patches in oracle_compat.c with a committer engaged, and it makes long lpad() and rpad() calls cancellable. The speed gain reaches only large N, and the regressions from message #12 need an answer before commit.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
