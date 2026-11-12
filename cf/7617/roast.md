# Design pre-review: Keep TIDBitmap pagetable memory across bitmap scans

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7617](https://commitfest.postgresql.org/patch/7617/); [thread](https://www.postgresql.org/message-id/flat/tidbitmap-cache-v1.20261108.b957201f%40burd.me), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The contention is measured and the problem is stated well, but the case for a tidbitmap-only cache is open at level 1b. Message #3 shows that setting both glibc thresholds removes the faults in every tested node and leaves the patch a 5.5% to 11.8% gain (already-exists-1). The author agrees there that a process-wide fix is the better target. All numbers are at 96 clients on one synthetic workload (demand-2).

Not checked: The patch text was not readable, so the reset callback, the handling of reused arrays and any change to tidbitmap.h could not be checked. Three level 2 objections on the work_mem cap, the per-module placement and the design constants survived but were not evaluated.

## Prior attempts

- [Add GUC to tune glibc's malloc behaviour](https://www.postgresql.org/message-id/flat/3424675.QJadu78ljV@aivenlaptop) (related; Returned with feedback, 2023-08-01). "I think your approach is fundamentally overfitting our code to a specific malloc implementation, in a way that's not tunable by mere mortals. It just seems like a dead end to me." ([message](https://www.postgresql.org/message-id/20230627181746.qz4373pbobkto3un%40awork3.anarazel.de)) Different now: P1 tuned glibc through mallopt with a GUC. This patch keeps memory inside tidbitmap.c and works on any libc, but message #3 suggests a backend-start mallopt fix that is close to P1.

## Objections a reviewer is likely to raise

### 1. Most of the gain is available today through glibc malloc environment variables

An operator on glibc sets MALLOC_TRIM_THRESHOLD_ and MALLOC_MMAP_THRESHOLD_ in the postmaster environment and gets 4,040 TPS with no faults. tidbitmap.c then carries a permanent per-backend cache that adds about 6% on top of that. Hash join and sort also stop faulting under the same setting, and the patch does nothing for them.

Message #3 shows that setting both glibc thresholds removes the faults in every tested node, and that the patch adds only 5.5% to 11.8% on top. glibc reads these thresholds from environment variables, so no code change is needed, and [P1] records the suggestion to document those variables instead of tuning them in core. The author's strongest answer is musl, which has no such knobs (+63.8%). That is a real gap, but it is narrower than the headline numbers suggest.

Precedent: [Add GUC to tune glibc's malloc behaviour](https://www.postgresql.org/message-id/flat/3424675.QJadu78ljV@aivenlaptop) (Returned with feedback, 2023-08-01)

**Question:** With both glibc thresholds set, or under jemalloc, which workloads still gain enough to justify a permanent cache in tidbitmap.c, and how many deployments use musl?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/tidbitmap-cache-v1.20261108.b957201f%40burd.me), [#3](https://www.postgresql.org/message-id/avPCm7p7ktH_e2Kj%40werk); `already-exists-1`</sub>

### 2. Demand shown only by one synthetic high-concurrency benchmark

The cache is committed for a 96-client, single-socket, all-cached pgbench case. Most installations run far fewer concurrent large bitmap scans and see little change. They still carry the retained memory and the extra code path.

The only evidence in messages #1 and #3 is the author's benchmark: 16M rows, about 17k pages per bitmap, 96 clients. The material has no field reports, no -general or -bugs threads, and no numbers at low or moderate concurrency. David Geier's earlier presizing measurement in message #1 comes from the same development work, so it does not show user demand.

**Question:** What gain does the patch give at 4, 16 and 32 clients, and is there any production report of this kernel lock contention from bitmap scans?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/tidbitmap-cache-v1.20261108.b957201f%40burd.me), [#3](https://www.postgresql.org/message-id/avPCm7p7ktH_e2Kj%40werk); `demand-2`</sub>

## Questions for the author

1. With both glibc thresholds set, or under jemalloc, which workloads still gain enough from the 5.5% to 11.8% rehash saving to justify a cache in tidbitmap.c?
2. Will you continue this patch, or switch to a process-wide fix that also covers hash join and sort, and how would that fix differ from P1?
3. What gain does the patch give at 4, 16 and 32 clients, and is there a production report of this lock contention from bitmap scans?
4. How much memory do many backends keep after a session raised work_mem far above the default, and why is work_mem a better cap than a small fixed one?
5. How does the cache relate to the radix tree replacement of the pagetable, and do the 2048-slot threshold and two entries still apply after it?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** 0002 adds a per-backend cache in tidbitmap.c with constants tied to glibc's 128 kB mmap threshold and simplehash growth.
- **performance on the hot path:** 0002 changes how the TIDBitmap pagetable grows during every large bitmap scan, and keeps up to work_mem per backend afterwards.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Messages #2 and #3 agree that memory retention belongs in a process-wide or allocator-level fix, which also covers hash join and sort.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the memory-context layer and tidbitmap.c. No committer has posted in the visible thread; the only reviewer so far objects to the placement.
- Priority: needs design rethink; audience unclear; review cost medium. The goal is valid and well measured, but the thread agrees that a process-wide fix is the better target, and the numbers cover only 96 clients. Review cost is a guess because the patch was not readable; the change is confined to tidbitmap.c and a test module.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
