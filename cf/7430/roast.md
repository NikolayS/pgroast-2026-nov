# Design pre-review: bufmgr Half state CAS

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7430](https://commitfest.postgresql.org/patch/7430/); [thread](https://www.postgresql.org/message-id/flat/CAE8JnxN3SPJxvAQZ7doESAHrfayx%2BAXUVr4c3RadApPONz00zQ%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The proposal fails at level 1b on demand-1. The only evidence of a gain is one test on a one-page table on one Apple M2, and the author reports negligible difference for other queries. Two reviewers doubt the mechanism, and no profile or retry count ties the gain to fewer CAS retries. The surviving design objections were not evaluated, because the benefit has to be agreed first.

Not checked: Without the code, the review could not check how the fallback half CAS locks, how the half is chosen on big-endian builds, or whether static assertions tie the bit layout to each CAS site. The level 2 objections on these points were not evaluated.

## Prior attempts

- [Move PinBuffer and UnpinBuffer to atomics](https://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell) (related; Returned with feedback, 2015-12-16). "Which you made ineffective because PinBuffer() doesn't take a lock anymore. Mutual exclusion through locks only works if all participants take the locks." ([message](https://www.postgresql.org/message-id/20150914154142.GB9666%40alap3.anarazel.de)) Different now: P2 moved pinning to a lock-free atomic state. This proposal narrows the existing state CAS loops to 32-bit halves, so the shared risk is only that one path stops observing bits another path relies on.
- [Revert "For all ppc compilers, implement pg_atomic_fetch_add_ with inline asm."](https://www.postgresql.org/message-id/flat/7517.1568470247@sss.pgh.pa.us) (related; Reverted, 2019-09-15). This reverts commit e7ff59686eacf5021fb84be921116986c3828d8a. It defined pg_atomic_fetch_add_u32_impl() without defining pg_atomic_compare_exchange_u32_impl(), which is incompatible with src/include/port/atomics/fallback.h. Per buildfarm member prairiedog. Different now: P1 added an inline-asm fetch_add for ppc that did not fit fallback.h. This proposal adds new compare-exchange variants that must also fit fallback.h and generic.h.

## Objections a reviewer is likely to raise

### 1. Targeted contention shown only on one synthetic single-page workload

A committer adds new half-width atomics to port/atomics.h and branches in UnlockBufHdrExt for every build. Production workloads rarely mix pin/unpin and lock/unlock on one header within the CAS window, so most users see no gain. They still carry the 1-client slowdown and the TerminateBufferIO branch.

The only demand evidence is the author's own benchmark: a 10-row table on one page, forced index scans, eight clients, one Apple M2 (messages #3, #5). Message #5 says other queries showed negligible difference. Message #4 doubts the CAS window can be contended at this concurrency. The thread has no field report, -general or -bugs thread, or profile from a real system that shows this retry pattern.

**Question:** Which real workload or field report shows CAS retries on buffer header state between pinners and lockers, and does pgbench or an x86 many-core run show the gain?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/CAE8JnxPFpkXJxh7SBCmPxP%2BC%2BM-tf31C2m%3Dw7qs8_TAs0UKFCQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/fnrfw2no7hwa5pg24zfmljgz26ahmlufkq2ue365f7iy4amahw%40n42iohfhyuwi), [#5](https://www.postgresql.org/message-id/CAE8JnxP_Gbrv36r1VoVYJJrf41dKosm%3DnR0z41vXhHXFLfbGnQ%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. What do x86 and other many-core machines show at high client counts, for the one-page test and for a standard read-only pgbench run?
2. Does a profile or a CAS retry count, base against patched, show that the 8-client gain comes from fewer retries on the buffer header state?
3. Which real workload or field report shows pinners and lockers of one buffer causing each other's CAS loops to retry?
4. What do write workloads and single-client runs show, given the extra branch in UnlockBufHdrExt for callers such as TerminateBufferIO?
5. On which platforms is a 32-bit CAS on half of a pg_atomic_uint64 atomic with respect to 64-bit operations, including the spinlock fallback and big-endian layouts?

## Touch points

- **ABI / extension API:** Patch 0001 adds pg_atomic_compare_exchange_u64_lo and _hi to port/atomics.h, and 0002 weakens the UnlockBufHdrExt return value.
- **maintenance burden ("who maintains this in 10 years"):** The half-width atomics need fallback and generic versions, and each CAS site depends on the bit layout in buf_internals.h.
- **performance on the hot path:** Patch 0002 changes the CAS loops in PinBuffer, MarkBufferDirty, BufferLockAttempt and UnlockBufHdrExt, which every backend runs.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Targeted contention shown only on one synthetic single-page workload

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas and Andres Freund replied in the thread, and both doubt the gain. The owner must know both the buffer manager and the port atomics layer.
- Not now, show demand first: Benchmarks on x86 and at high client counts, including a standard pgbench run and write workloads, that show a gain and no single-client loss. A profile or retry count that ties the gain to fewer CAS retries would also be needed.
- Priority: park; audience unclear; review cost medium. The gain is shown on one single-page test on one machine, and other queries showed negligible difference. The patches are small, but they add port-layer atomics and change hot buffer manager paths, so review should wait for broader measurements.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
