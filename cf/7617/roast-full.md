# Design pre-review: Keep TIDBitmap pagetable memory across bitmap scans

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7617](https://commitfest.postgresql.org/patch/7617/) (Needs review); authors: Greg Burd (gregburd)
- Thread: [Keep TIDBitmap pagetable memory across bitmap scans](https://www.postgresql.org/message-id/flat/tidbitmap-cache-v1.20261108.b957201f%40burd.me), started 2026-11-09
- Messages read: 3 of 3
- Patch prose read: none (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The contention is measured and the problem is stated well, but the case for a tidbitmap-only cache is open at level 1b. Message #3 shows that setting both glibc thresholds removes the faults in every tested node and leaves the patch a 5.5% to 11.8% gain (already-exists-1). The author agrees there that a process-wide fix is the better target. All numbers are at 96 clients on one synthetic workload (demand-2).

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The patch text was not readable, so the reset callback, the handling of reused arrays and any change to tidbitmap.h could not be checked. Three level 2 objections on the work_mem cap, the per-module placement and the design constants survived but were not evaluated.

## The proposal, restated

**Problem.** Large bitmap heap scans regrow the TIDBitmap simplehash pagetable from 256 slots each query, and glibc trims the freed heap at ExecutorEnd, so the next query page-faults the same memory again (about 730 faults per query). With many concurrent backends, these faults and trims contend on a shared kernel page LRU lock, and the author measured a third of CPU in system time at 96 clients. Planner-based presizing fails when estimates are wrong, as with GIN array containment.

**Proposed design.** 0002 keeps the pagetable array and sorted page list of up to two large private bitmaps per backend in a "TIDBitmap cache" memory context. A bitmap that reaches 2048 slots takes a free entry and switches to its array, and a reset callback on the bitmap's context returns the entry on error. Total retained memory is capped at work_mem, with no idle release; 0001 adds a test module.

**User-visible surface.**
- New memory context "TIDBitmap cache" visible in pg_backend_memory_contexts.
- Per-backend retained memory up to work_mem after a large bitmap scan, until a smaller-work_mem bitmap scan or backend exit.
- New test module src/test/modules/test_tidbitmap.

**What the author claims.**
- At 96 clients with glibc defaults, TPS rises 34.8% to 64.9% and page faults per query drop to 0.
- Under musl, gains are +63.8% (btree) and +88.9% (BitmapAnd); under jemalloc +4.7% to +11.9%.
- Contention is on a per-NUMA-node page queue lock, seen on Linux with and without memcg and on FreeBSD.
- Setting both glibc M_TRIM_THRESHOLD and M_MMAP_THRESHOLD at backend start removes the faults process-wide; the patch still adds 5.5% to 11.8% on top.
- Hash join and sort show the same fault pattern; the patch does not help them.

**Thread state.** David Geier agrees memory should be kept but objects to doing it in tidbitmap.c, citing memory use with many connections and other doubling allocations. Greg agreed a process-wide allocator fix is the better target; whether this patch should proceed is open, and the last message is truncated.

## 2. Prior attempts

### Add GUC to tune glibc's malloc behaviour (related)

- **Link:** [https://commitfest.postgresql.org/patch/4387/](https://commitfest.postgresql.org/patch/4387/)
- **Thread:** [https://www.postgresql.org/message-id/flat/3424675.QJadu78ljV@aivenlaptop](https://www.postgresql.org/message-id/flat/3424675.QJadu78ljV@aivenlaptop)
- **Date:** decided 2023-08-01; started 2023-06-22
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Andres Freund (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane objected that such a GUC means nothing on non-glibc platforms, and he suggested documenting the glibc environment variables instead. Andres Freund held that the patch overfits Postgres to one malloc implementation in a way users cannot tune. He argued that Postgres should control allocation itself in aset.c and the other memory contexts, because the cost comes from page faults and zeroing new pages. The CF manager returned the entry because no update addressed these objections.
- **Reason (quote):** "I think your approach is fundamentally overfitting our code to a specific malloc implementation, in a way that's not tunable by mere mortals. It just seems like a dead end to me." ([message](https://www.postgresql.org/message-id/20230627181746.qz4373pbobkto3un%40awork3.anarazel.de))
- **What is different this time:** P1 tuned glibc through mallopt with a GUC. This patch keeps memory inside tidbitmap.c and works on any libc, but message #3 suggests a backend-start mallopt fix that is close to P1.


## 3. Objections a reviewer is likely to raise

### 1. Most of the gain is available today through glibc malloc environment variables

fixable · level 1b, rubric item 2 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An operator on glibc sets MALLOC_TRIM_THRESHOLD_ and MALLOC_MMAP_THRESHOLD_ in the postmaster environment and gets 4,040 TPS with no faults. tidbitmap.c then carries a permanent per-backend cache that adds about 6% on top of that. Hash join and sort also stop faulting under the same setting, and the patch does nothing for them.

Message #3 shows that setting both glibc thresholds removes the faults in every tested node, and that the patch adds only 5.5% to 11.8% on top. glibc reads these thresholds from environment variables, so no code change is needed, and [P1] records the suggestion to document those variables instead of tuning them in core. The author's strongest answer is musl, which has no such knobs (+63.8%). That is a real gap, but it is narrower than the headline numbers suggest.

**Precedents.**
- [Add GUC to tune glibc's malloc behaviour](https://commitfest.postgresql.org/patch/4387/) (Returned with feedback, 2023-08-01); thread: [link](https://www.postgresql.org/message-id/flat/3424675.QJadu78ljV@aivenlaptop)

Thread: [#1 (Greg Burd, 2026-11-09)](https://www.postgresql.org/message-id/tidbitmap-cache-v1.20261108.b957201f%40burd.me), [#3 (Greg Burd, 2026-11-11)](https://www.postgresql.org/message-id/avPCm7p7ktH_e2Kj%40werk)

**Question:** With both glibc thresholds set, or under jemalloc, which workloads still gain enough to justify a permanent cache in tidbitmap.c, and how many deployments use musl?

Counter-check: held. Message #3 shows both glibc thresholds give 4,040 TPS with no faults, and the author calls the remaining gain a much smaller argument. The objection already allows for musl, where no such setting exists.

### 2. Demand shown only by one synthetic high-concurrency benchmark

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-2`

The cache is committed for a 96-client, single-socket, all-cached pgbench case. Most installations run far fewer concurrent large bitmap scans and see little change. They still carry the retained memory and the extra code path.

The only evidence in messages #1 and #3 is the author's benchmark: 16M rows, about 17k pages per bitmap, 96 clients. The material has no field reports, no -general or -bugs threads, and no numbers at low or moderate concurrency. David Geier's earlier presizing measurement in message #1 comes from the same development work, so it does not show user demand.

Thread: [#1 (Greg Burd, 2026-11-09)](https://www.postgresql.org/message-id/tidbitmap-cache-v1.20261108.b957201f%40burd.me), [#3 (Greg Burd, 2026-11-11)](https://www.postgresql.org/message-id/avPCm7p7ktH_e2Kj%40werk)

**Question:** What gain does the patch give at 4, 16 and 32 clients, and is there any production report of this kernel lock contention from bitmap scans?

Counter-check: held. The measurements are careful and cover two kernels and several allocators, but all are at 96 clients on one synthetic workload. The thread has no numbers at lower concurrency and no field report.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0002 adds a per-backend cache in tidbitmap.c with constants tied to glibc's 128 kB mmap threshold and simplehash growth. |
| performance on the hot path | **yes** | 0002 changes how the TIDBitmap pagetable grows during every large bitmap scan, and keeps up to work_mem per backend afterwards. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Messages #2 and #3 agree that memory retention belongs in a process-wide or allocator-level fix, which also covers hash join and sort. |

## 5. Questions the author must answer before anyone reviews code

1. With both glibc thresholds set, or under jemalloc, which workloads still gain enough from the 5.5% to 11.8% rehash saving to justify a cache in tidbitmap.c?
2. Will you continue this patch, or switch to a process-wide fix that also covers hash join and sort, and how would that fix differ from P1?
3. What gain does the patch give at 4, 16 and 32 clients, and is there a production report of this lock contention from bitmap scans?
4. How much memory do many backends keep after a session raised work_mem far above the default, and why is work_mem a better cap than a small fixed one?
5. How does the cache relate to the radix tree replacement of the pagetable, and do the 2048-slot threshold and two entries still apply after it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the memory-context layer and tidbitmap.c. No committer has posted in the visible thread; the only reviewer so far objects to the placement.
- **Liability:** The owner signs for a per-backend cache with no idle release, capped at work_mem, and for constants that depend on glibc and simplehash behaviour. The cache would also need rework if the radix tree work replaces the pagetable.
- **Author follow-up:** The author answered review within a day with new measurements on two kernels and offered the reproducer. He has not said whether he will continue this patch or switch to an allocator-level one.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The goal is valid and well measured, but the thread agrees that a process-wide fix is the better target, and the numbers cover only 96 clients. Review cost is a guess because the patch was not readable; the change is confined to tidbitmap.c and a test module.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Retained memory changes what work_mem means for idle backends. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) Per-module cache for a process-wide allocator problem. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-3` (fixable, level 2) Design constants tied to glibc defaults and current simplehash. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Per-module cache where a general allocation fix is the stated target. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about a tidbitmap-only cache with more direct quotes from messages #2 and #3.
- `maintenance-fit-2` Retained memory scales with work_mem and never decays. Refuted (not specific): Duplicate of standards-compat-1. Both rest on the work_mem cap with no idle release and on message #3 covering only the default.
- `already-exists-2` A memory cache local to tidbitmap.c duplicates allocator-level retention. Refuted (not specific): Duplicate of maintenance-fit-1. The argument about a second retention mechanism outside the memory-context layer is the same point.
- `demand-1` Narrow gain once a process-wide allocator fix exists. Refuted (not specific): Duplicate of maintenance-fit-1 and already-exists-1, which between them cover the small remaining gain and the per-module cache concern.
- `demand-3` Retained memory falls on all bitmap-scan backends, gain only under contention. Refuted (not specific): Duplicate of standards-compat-1. The retained-memory cost with a large work_mem and many backends is the same concern.

### Review angles

- prior-attempts: 1 objection(s). The P1 reason does not apply to the patch as posted. It would apply if the thread switches to setting glibc thresholds at backend start.
- standards-compat: 1 objection(s). From the standards angle, the patch adds no SQL syntax, GUC, catalog object or pg_dump state. The only user-visible change is the new memory context and the memory it keeps.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state: no WAL, on-disk, catalog or wire protocol change, and shared bitmaps for parallel scans are left as before. Without the code it is not possible to check whether exported tidbitmap.h structs or function signatures change for extensions.
- security: 0 objection(s). The proposal has no privilege or exposure aspect worth objecting to. The cache is private to one backend, so reused arrays never cross a process boundary, and reusing freed memory inside a backend is what aset.c already does. A user can already set work_mem and use that memory during a query, so keeping up to work_mem after the scan gives no new denial-of-service path. The memory cost across many connections belongs to the resource and design angles. Not checked, because the code is not readable: whether a reused array is cleared before it is handed to a new bitmap.
- maintenance-fit: 3 objection(s). Objections were found; code-level questions such as the reset callback's correctness cannot be checked without readable patches.
- already-exists: 2 objection(s). Two objections raised; the musl case is a real gap that existing glibc settings cannot cover.
- demand: 3 objection(s). The kernel contention is real and measured. The open question is demand for a tidbitmap-specific fix rather than a general one.

### Precedents retrieved

- P1: [Add GUC to tune glibc's malloc behaviour](https://commitfest.postgresql.org/patch/4387/) (cf, Returned with feedback, 2023-08-01) - used in the review; reason summarised from its thread
- P2: [Adding TRIM_SPACE option to COPY](https://commitfest.postgresql.org/patch/6289/) (cf, Withdrawn, 2026-04-13) - judged not relevant; reason summarised from its thread
- P3: [Making autovacuum logs indicate if insert-based threshold was the triggering condition](https://commitfest.postgresql.org/patch/3802/) (cf, Withdrawn, 2022-11-27) - judged not relevant
- P4: [Rename trim_array() for consistency with the rest of array_* functions](https://commitfest.postgresql.org/patch/5335/) (cf, Withdrawn, 2024-11-04) - judged not relevant; reason summarised from its thread
- P5: [Add numeric_trim(numeric)](https://commitfest.postgresql.org/patch/429/) (cf, Returned with feedback, 2016-03-15) - judged not relevant; reason summarised from its thread
- P6: [Fix and improve allocation formulas](https://commitfest.postgresql.org/patch/6310/) (cf, Withdrawn, 2026-01-13) - judged not relevant; reason summarised from its thread
- P7: [Revert "Improve accounting for memory used by shared hash tables"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aff1dc8dfa7584ddbc695e1aae0f767db66326f) (revert, Reverted, 2025-04-04) - judged not relevant
- P8: [PGDLLIMPORT-ify old_snapshot_threshold.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=14216649f3dc8bd9839702440dd593e958b0920b) (revert, Partially reverted, 2016-04-21) - judged not relevant
- P9: [Use simplehash.h instead of dynahash in SMgr](https://commitfest.postgresql.org/patch/3092/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P10: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P11: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant; reason summarised from its thread
- P12: [Increasing the default values of from_collapse_limit, join_collapse_limit, and/or geqo_threshold](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P13: [Run-time pruning for ModifyTable](https://commitfest.postgresql.org/patch/2173/) (cf, Returned with feedback, 2020-04-07) - judged not relevant
- P14: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - judged not relevant
- P15: [Teach WaitEventSets to grow automatically](https://commitfest.postgresql.org/patch/3533/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P16: [Allow EXPLAIN to Output Page Fault Information](https://commitfest.postgresql.org/patch/5477/) (cf, Withdrawn, 2025-02-08) - judged not relevant
- P17: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P18: [Variable Block Size](https://commitfest.postgresql.org/patch/4429/) (cf, Withdrawn, 2024-01-19) - judged not relevant
- P19: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P20: [pre-allocating WAL files](https://commitfest.postgresql.org/patch/3158/) (cf, Withdrawn, 2022-07-25) - judged not relevant
- P21: [Autovacuum dynamic cost_limit, cost_delay](https://commitfest.postgresql.org/patch/2983/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P22: [Separate memory contexts for relcache and catcache](https://commitfest.postgresql.org/patch/4554/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P23: [dbsize.c functions and partitioned tables](https://commitfest.postgresql.org/patch/1412/) (cf, Returned with feedback, 2018-01-17) - judged not relevant
- P24: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 20; wall time: 175 s; cost reported by the CLI: $2.00
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 101 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:08:04.744Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
