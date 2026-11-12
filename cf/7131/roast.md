# Design pre-review: Fix races in Windows pthread emulation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7131](https://commitfest.postgresql.org/patch/7131/); [thread](https://www.postgresql.org/message-id/flat/CAAH-eSXFXSEtXpb4kUAGOKKdm3o3y-yWfjsJ7R7zUL6g-swuzw%40mail.gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the compare-and-exchange fix. Several people reproduced the hang, and the cost falls only on Windows builds of libpq and ECPG. The one surviving objection, maintenance-fit-1, is a fixable design point: v2 adds a per-compiler atomic load to two copies of the mutex code, and Heikki Linnakangas asked for one primitive in message #8. A committer who knows the Windows port must decide that point and the backpatch range.

Not checked: The code and the v2 commit message were not read, so the memory ordering of the new loads on ARM64 and the layout claim for ecpg-pthread-win32.h are unchecked. Messages #4 to #10 are known only from the brief and the reviewer notes.

## Prior attempts

None found before 2026-07-23.

## Objections a reviewer is likely to raise

### 1. Ad hoc compiler-specific atomic loads added to two duplicated copies

A later fix or a new Windows toolchain changes the ordering code in the libpq copy but not in the ECPG copy, or only in the MSVC branch. The MinGW branch has less coverage on the buildfarm. The drift shows up months later as a rare hang in client threads.

v2 adds an `#ifdef _MSC_VER` split, with ReadAcquire on MSVC and __atomic_load_n on other compilers, to two copies of the same mutex code. Heikki asked for one primitive for both compilers in message #8, and message #9 leaves that open. Postgres already has its own atomics and barrier abstraction, and the proposal does not say why that is unusable here. CF entry 6314 would merge the libpq copy into src/port (message #1), so the order of the two patches decides how many copies need this code.

**Question:** Why can the fast path not use the existing Postgres atomics or barrier primitives, or one shared helper, instead of a per-compiler branch in each copy?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAAH-eSXFXSEtXpb4kUAGOKKdm3o3y-yWfjsJ7R7zUL6g-swuzw%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/0f62028a-54bf-427b-9bed-65ee89b34b7c%40iki.fi), [#9](https://www.postgresql.org/message-id/CAAH-eSVLTosOtXkPOnnkwHNRanZ%2BGTioYBoN01LUQv3PyNeKSQ%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Why can the lock fast path not use one atomic load primitive, or one shared helper, for both MSVC and MinGW in both copies?
2. Which back branches should get this fix, and do REL_17_STABLE and REL_18_STABLE build and pass the ECPG and libpq tests with v2?
3. Should this patch or CommitFest entry 6314 go in first, given that 6314 moves the libpq copy into src/port?
4. Is the pthread_once ordering problem on ARM64 still real after v2, and will a separate patch for it follow?
5. Is CommitFest entry 7255 a duplicate of 7131, and which one should be closed?

## Touch points

- **ABI / extension API:** The patch touches the ECPG header ecpg-pthread-win32.h; v2 says it keeps type layouts unchanged, after v1 changed pthread_once_t.
- **backpatchability:** The bug is in released client libraries; v2 applies to REL_17_STABLE and REL_18_STABLE but neither branch was built.
- **maintenance burden ("who maintains this in 10 years"):** v2 adds an MSVC and non-MSVC atomic load branch to two copies of the mutex code, in ECPG and libpq.
- **performance on the hot path:** The lock and unlock fast paths of the Windows client mutex gain an atomic load, and the cost is not benchmarked.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas reviewed v2 in the thread and accepted the approach. The owner needs to know the Windows port and the thread-safety code of libpq and ECPG.
- Priority: proceed; audience niche; review cost low. The patch fixes a reproduced hang for threaded Windows clients of libpq and ECPG, and it also removes a test hang seen in CI and on the buildfarm animal hamerkop. It is about 4 KiB in three client-side files, adds no user-visible surface, and has one open point about the atomic load primitive.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
