# Design pre-review: bufmgr Half state CAS

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7430](https://commitfest.postgresql.org/patch/7430/) (Needs review); authors: Alexandre Felipe (afelipe)
- Thread: [\[patch\] bufmgr Half state CAS](https://www.postgresql.org/message-id/flat/CAE8JnxN3SPJxvAQZ7doESAHrfayx%2BAXUVr4c3RadApPONz00zQ%40mail.gmail.com), started 2026-10-08
- Messages read: 5 of 5
- Patch prose read: v1-0002-Using-half-state-CAS.patch, v1-0001-half-compare-exchange.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The proposal fails at level 1b on demand-1. The only evidence of a gain is one test on a one-page table on one Apple M2, and the author reports negligible difference for other queries. Two reviewers doubt the mechanism, and no profile or retry count ties the gain to fewer CAS retries. The surviving design objections were not evaluated, because the benefit has to be agreed first.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check how the fallback half CAS locks, how the half is chosen on big-endian builds, or whether static assertions tie the bit layout to each CAS site. The level 2 objections on these points were not evaluated.

## The proposal, restated

**Problem.** The buffer header state is a 64-bit word holding several bit fields, and its CAS loops retry when any bit changes, even bits the caller does not care about. The author targets contention between backends that pin or unpin a buffer and backends that lock or unlock the same buffer.

**Proposed design.** Patch 0001 adds atomics that compare-and-swap only the low or the high 32-bit half of a pg_atomic_uint64. Patch 0002 moves the buffer manager CAS loops to the half that holds the bits each one changes, with a full-width CAS only in SharedBufferBeginSetHintBits. UnlockBufHdrExt picks fetch_and, a full CAS or a low-half CAS depending on which bits it changes.

**User-visible surface.**
- C API: new pg_atomic_compare_exchange_u64_lo and pg_atomic_compare_exchange_u64_hi in port/atomics.h, with fallback and generic versions.
- C API: the return value of UnlockBufHdrExt no longer reflects updated high bits when only low bits change.

**What the author claims.**
- Changes to unrelated bits cause fewer CAS retries.
- On an Apple M2, a read-only test with a one-page table shows about 60% more throughput at 8 clients and about 0.5% less at 1 client.
- Other queries the author tried showed negligible difference.
- The only slower path is UnlockBufHdrExt when the compiler cannot remove a branch, as in TerminateBufferIO.
- TerminateBufferIO, the only caller using the UnlockBufHdrExt return value, reads only BM_PIN_COUNT_WAITER.

**Thread state.** Heikki doubts any gain and suspects a slowdown, and Andres says such gains cannot appear at this concurrency. The author answered with an index-scan benchmark and script, and neither reviewer has replied to it.

## 2. Prior attempts

### Move PinBuffer and UnpinBuffer to atomics (related)

- **Link:** [https://commitfest.postgresql.org/patch/370/](https://commitfest.postgresql.org/patch/370/)
- **Thread:** [https://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell](https://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell)
- **Date:** decided 2015-12-16; started 2015-09-11
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Andres Freund (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Andres Freund judged the first patch incorrect. PinBuffer no longer took the header lock, but StrategyGetBuffer, SyncOneBuffer and BufferAlloc still relied on that lock for mutual exclusion. Work moved to the single atomic state version, which Alexander Korotkov refactored and others benchmarked, and it was still being cleaned up when the entry closed. No visible message states why the entry was returned.
- **Reason (quote):** "Which you made ineffective because PinBuffer() doesn't take a lock anymore. Mutual exclusion through locks only works if all participants take the locks." ([message](https://www.postgresql.org/message-id/20150914154142.GB9666%40alap3.anarazel.de))
- **What is different this time:** P2 moved pinning to a lock-free atomic state. This proposal narrows the existing state CAS loops to 32-bit halves, so the shared risk is only that one path stops observing bits another path relies on.

### Revert "For all ppc compilers, implement pg_atomic_fetch_add_ with inline asm." (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e9fae0696d9e3ff70a1438775ad9f786b854a5](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e9fae0696d9e3ff70a1438775ad9f786b854a5)
- **Thread:** [https://www.postgresql.org/message-id/flat/7517.1568470247@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/7517.1568470247@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/20190831071157.GA3251746@rfd.leadboat.com](https://www.postgresql.org/message-id/flat/20190831071157.GA3251746@rfd.leadboat.com)
- **Date:** decided 2019-09-15; started 2019-09-14
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit e7ff59686eacf5021fb84be921116986c3828d8a. It defined pg_atomic_fetch_add_u32_impl() without defining pg_atomic_compare_exchange_u32_impl(), which is incompatible with src/include/port/atomics/fallback.h. Per buildfarm member prairiedog.
- **Reason (quote):** quote unavailable
- **What is different this time:** P1 added an inline-asm fetch_add for ppc that did not fit fallback.h. This proposal adds new compare-exchange variants that must also fit fallback.h and generic.h.


## 3. Objections a reviewer is likely to raise

### 1. Targeted contention shown only on one synthetic single-page workload

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer adds new half-width atomics to port/atomics.h and branches in UnlockBufHdrExt for every build. Production workloads rarely mix pin/unpin and lock/unlock on one header within the CAS window, so most users see no gain. They still carry the 1-client slowdown and the TerminateBufferIO branch.

The only demand evidence is the author's own benchmark: a 10-row table on one page, forced index scans, eight clients, one Apple M2 (messages #3, #5). Message #5 says other queries showed negligible difference. Message #4 doubts the CAS window can be contended at this concurrency. The thread has no field report, -general or -bugs thread, or profile from a real system that shows this retry pattern.

Thread: [#3 (Alexandre Felipe, 2026-10-14)](https://www.postgresql.org/message-id/CAE8JnxPFpkXJxh7SBCmPxP%2BC%2BM-tf31C2m%3Dw7qs8_TAs0UKFCQ%40mail.gmail.com), [#4 (Andres Freund, 2026-10-14)](https://www.postgresql.org/message-id/fnrfw2no7hwa5pg24zfmljgz26ahmlufkq2ue365f7iy4amahw%40n42iohfhyuwi), [#5 (Alexandre Felipe, 2026-10-15)](https://www.postgresql.org/message-id/CAE8JnxP_Gbrv36r1VoVYJJrf41dKosm%3DnR0z41vXhHXFLfbGnQ%40mail.gmail.com)

**Question:** Which real workload or field report shows CAS retries on buffer header state between pinners and lockers, and does pgbench or an x86 many-core run show the gain?

Counter-check: held. The only evidence is one single-page test on one Apple M2, and message #5 says other queries showed negligible difference. No profile or retry count links the gain to fewer CAS retries, and two reviewers doubt the mechanism.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Patch 0001 adds pg_atomic_compare_exchange_u64_lo and _hi to port/atomics.h, and 0002 weakens the UnlockBufHdrExt return value. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The half-width atomics need fallback and generic versions, and each CAS site depends on the bit layout in buf_internals.h. |
| performance on the hot path | **yes** | Patch 0002 changes the CAS loops in PinBuffer, MarkBufferDirty, BufferLockAttempt and UnlockBufHdrExt, which every backend runs. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Targeted contention shown only on one synthetic single-page workload |

## 5. Questions the author must answer before anyone reviews code

1. What do x86 and other many-core machines show at high client counts, for the one-page test and for a standard read-only pgbench run?
2. Does a profile or a CAS retry count, base against patched, show that the 8-client gain comes from fewer retries on the buffer header state?
3. Which real workload or field report shows pinners and lockers of one buffer causing each other's CAS loops to retry?
4. What do write workloads and single-client runs show, given the extra branch in UnlockBufHdrExt for callers such as TerminateBufferIO?
5. On which platforms is a 32-bit CAS on half of a pg_atomic_uint64 atomic with respect to 64-bit operations, including the spinlock fallback and big-endian layouts?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas and Andres Freund replied in the thread, and both doubt the gain. The owner must know both the buffer manager and the port atomics layer.
- **Liability:** The owner signs for mixed 32-bit and 64-bit atomic access to the buffer header state on every supported platform, including the fallback. That lasts as long as the state layout does.
- **Author follow-up:** The author answered the request for benchmarks within days, with numbers and a script (message #3).

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: Benchmarks on x86 and at high client counts, including a standard pgbench run and write workloads, that show a gain and no single-client loss. A profile or retry count that ties the gain to fewer CAS retries would also be needed.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** park
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The gain is shown on one single-page test on one machine, and other queries showed negligible difference. The patches are small, but they add port-layer atomics and change hot buffer manager paths, so review should wait for broader measurements.

## Appendix

### Held up in the counter-check, not listed above

- `prior-attempts-1` (fixable, level 2) Half-width CAS cannot enforce rules that span both state halves. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-1` (fixable, level 2) UnlockBufHdrExt return value changes meaning without any signal to callers. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Half-word CAS on one shared 64-bit word needs a per-platform safety argument. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Low/high bit placement becomes a silent correctness rule. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-2` New half-width atomics must fit the fallback and generic atomics layers. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same fallback and byte-order gap as a safety question.
- `replication-ondisk-2` UnlockBufHdrExt return value silently loses the high bits. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-1` Mixed-width atomics on one word add a new port-layer contract. Refuted (not specific): Duplicate of replication-ondisk-1.
- `maintenance-fit-3` UnlockBufHdrExt return value becomes partly stale. Refuted (not specific): Duplicate of standards-compat-1.

### Review angles

- prior-attempts: 2 objection(s). No precedent proposes half-width CAS on the buffer state; the two objections rest on recorded reasons from related attempts.
- standards-compat: 1 objection(s). The proposal has no SQL, GUC, catalog, pg_dump, psql or client-visible surface. My angle sees only the internal API contract change above.
- replication-ondisk: 2 objection(s). The patch changes no on-disk format, WAL record, catalog or wire protocol. Buffer header state lives only in shared memory, so pg_upgrade, standbys and subscribers are unaffected.
- security: 0 objection(s). The proposal changes internal atomics and buffer manager CAS loops only. It adds no SQL surface, privilege, role, GUC, monitoring view or authentication path, and it lets no user do or see anything new. Correctness risks of mixed-width atomics are data-safety questions for the design angle, outside security.
- maintenance-fit: 3 objection(s). Not applicable. The open doubts about the benefit (one Apple M2, no x86, no high client counts) belong to the desirability and performance angles.
- already-exists: 0 objection(s). Postgres has no existing feature, option or extension that lowers CAS retries on the buffer header state, and buffer manager internals cannot be replaced from an extension. Plain pg_atomic_compare_exchange_u32 on a separate field would need a different state layout, which is a design question for another angle.
- demand: 1 objection(s). One objection: demand is shown only by a synthetic hot-page test; a broader benchmark or field evidence would settle it.

### Precedents retrieved

- P1: [Revert "For all ppc compilers, implement pg_atomic_fetch_add_ with inline asm."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e9fae0696d9e3ff70a1438775ad9f786b854a5) (revert, Reverted, 2019-09-15) - used in the review
- P2: [Move PinBuffer and UnpinBuffer to atomics](https://commitfest.postgresql.org/patch/370/) (cf, Returned with feedback, 2015-12-16) - used in the review; reason summarised from its thread
- P3: [Atomic pgrename on Windows](https://commitfest.postgresql.org/patch/1386/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P4: [Fix typo 586/686 in atomics/arch-x86.h](https://commitfest.postgresql.org/patch/6295/) (cf, Returned with feedback, 2026-03-12) - judged not relevant; reason summarised from its thread
- P5: [Atomic rename feature for Windows](https://commitfest.postgresql.org/patch/3347/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P6: [Delaying/avoiding BTreeTupleGetNAtts() call within _bt_compare()](https://commitfest.postgresql.org/patch/2429/) (cf, Withdrawn, 2020-11-09) - judged not relevant; reason summarised from its thread
- P7: [Move SLRU data into the regular buffer pool](https://commitfest.postgresql.org/patch/3514/) (cf, Returned with feedback, 2023-09-04) - judged not relevant; reason summarised from its thread
- P8: [fix spinlock contention in LogwrtResult](https://commitfest.postgresql.org/patch/2716/) (cf, Withdrawn, 2022-09-23) - judged not relevant; reason summarised from its thread
- P9: [Batched clock sweep to reduce cross-socket atomic contention](https://commitfest.postgresql.org/patch/7000/) (cf, Withdrawn, 2026-09-11) - judged not relevant; reason summarised from its thread
- P10: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P11: [Fix lock contention for HASHHDR.mutex](https://commitfest.postgresql.org/patch/442/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P12: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P13: [audo-detect and use -moutline-atomics compilation flag for aarch64](https://commitfest.postgresql.org/patch/2637/) (cf, Rejected, 2020-09-29) - judged not relevant
- P14: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P15: [Floating point comparison inconsistencies of the geometric types](https://commitfest.postgresql.org/patch/679/) (cf, Returned with feedback, 2017-03-16) - judged not relevant
- P16: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P17: [introduce bufmgr hooks](https://commitfest.postgresql.org/patch/3853/) (cf, Withdrawn, 2022-10-12) - judged not relevant
- P18: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P19: [Revert "Do execGrouping.c via expression eval machinery."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2a41507dab0f293ff241fe8ae326065998668af8) (revert, Reverted, 2018-02-16) - judged not relevant
- P20: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - used in the review
- P21: [always use runtime checks for CRC-32C instructions](https://commitfest.postgresql.org/patch/4634/) (cf, Withdrawn, 2023-10-31) - judged not relevant
- P22: [Revert 019_replslot_limit.pl related debugging aids.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3f8148c256e067dc2e8929ed174671ba7dc3339c) (revert, Reverted, 2022-07-05) - judged not relevant
- P23: [Changing WAL Header to reduce contention during ReserveXLogInsertLocation()](https://commitfest.postgresql.org/patch/1455/) (cf, Returned with feedback, 2018-07-09) - judged not relevant
- P24: [Configurable file mode mask](https://commitfest.postgresql.org/patch/1069/) (cf, Returned with feedback, 2017-03-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 137 s; cost reported by the CLI: $1.48
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 97 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:54:52.074Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
