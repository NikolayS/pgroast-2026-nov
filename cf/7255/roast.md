# Design pre-review: Fix races in Windows pthread emulation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7255](https://commitfest.postgresql.org/patch/7255/); [thread](https://www.postgresql.org/message-id/flat/CAAH-eSXFXSEtXpb4kUAGOKKdm3o3y-yWfjsJ7R7zUL6g-swuzw%40mail.gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b. The hang is confirmed by Nazir's CI, by hamerkop and by an independent reproducer. The one surviving objection, maintenance-fit-1, is a fixable design point: v2 keeps separate MSVC and MinGW load paths in both the ECPG and libpq copies, and Heikki's request in message #8 for one primitive is not settled. The backpatch decision also needs a committer.

Not checked: The code was not read, so this review cannot confirm that the acquire load is sufficient on ARM64 or that no changed logic in ecpg-pthread-win32.h is compiled into applications. The back branches were not built.

## Prior attempts

- [Fix MINGW build on ARM64 on Windows](https://www.postgresql.org/message-id/flat/CAHXCYb2OjNHtoGVKyXtXmw4B3bUXwJX6M-Lcp1KcMCRUMLOocA@mail.gmail.com) (related; Returned with feedback, 2025-04-07). "fairywren has been failing rather horribly since this went in [1]." ([message](https://www.postgresql.org/message-id/3312149.1744001936%40sss.pgh.pa.us)) Different now: P16 changed setjmp handling for MinGW toolchains and was reverted after an msys2 buildfarm animal broke. This patch fixes a mutex race; the only link is its MinGW-only branch, tested only on emulated x64 MinGW.

## Objections a reviewer is likely to raise

### 1. Compiler-specific atomic loads added to two duplicated mutex copies

A later fix to the Windows mutex state machine changes the MSVC ReadAcquire branch in libpq and misses the MinGW __atomic_load_n branch, or the ECPG copy. Nothing fails at build time. Users of the build that was missed see the hang again, and it shows up only as a rare timeout on CI or the buildfarm.

v2 adds an #ifdef _MSC_VER split in both src/interfaces/ecpg and src/interfaces/libpq, so the same ordering logic now exists in four places. Postgres already has an atomics layer in port/atomics.h, and the proposal does not say why it cannot be used here. Heikki asked for one primitive for both compilers in message #8. Message #9 keeps the split to avoid a compare-exchange on the fast path, but no benchmark exists, and message #4 calls ecpg performance on Windows a minor concern.

**Question:** Can the fast-path load use one existing primitive, such as InterlockedCompareExchange or a port/atomics.h read, in one place shared by ECPG and libpq?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#4](https://www.postgresql.org/message-id/e39c8c80-5554-4bc1-a232-aaa552e556d9%40iki.fi), [#8](https://www.postgresql.org/message-id/0f62028a-54bf-427b-9bed-65ee89b34b7c%40iki.fi), [#9](https://www.postgresql.org/message-id/CAAH-eSVLTosOtXkPOnnkwHNRanZ%2BGTioYBoN01LUQv3PyNeKSQ%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Can the lock fast path use one primitive for both MSVC and MinGW, such as InterlockedCompareExchange, and what does that cost compared with the split in v2?
2. Which back branches does the patch target, and has v2 been built and tested on REL_17_STABLE and REL_18_STABLE?
3. When will the separate pthread_once fix be posted, and does native ARM64 stay exposed to the ordering problem from message #1 until then?
4. Will the deterministic reproducer be added as a regression test?
5. Should this patch or CommitFest entry 6314 land first, and does the fix then need to exist in only one copy on master?

## Touch points

- **ABI / extension API:** The patch touches ecpg-pthread-win32.h; v2 leaves the mutex and pthread_once_t layouts unchanged.
- **backpatchability:** v2 applies to REL_17_STABLE and REL_18_STABLE but was not built there, and the backpatch decision is open.
- **maintenance burden ("who maintains this in 10 years"):** v2 adds an MSVC and a MinGW load path to each of the two mutex copies in ECPG and libpq.
- **performance on the hot path:** The mutex lock fast path in libpq and ECPG on Windows gains an acquire load; no variant was benchmarked.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki reviewed the approach in messages #4 and #8; the owner needs to know Windows interlocked primitives and the libpq and ECPG thread code.
- Priority: proceed; audience a sizeable minority; review cost low. The patch fixes a confirmed deadlock in threaded libpq and ECPG clients on Windows, seen in CI and on hamerkop. It is about 4 KiB across three files, adds no user-visible surface, and a committer is already reviewing it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
