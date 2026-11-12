# Design pre-review: Call CalculateShmemSize() only once

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7561](https://commitfest.postgresql.org/patch/7561/); [thread](https://www.postgresql.org/message-id/flat/CAExHW5ttQauXiYCM4OKL8xKHZrndTN2jMN4WQjm1PkPNuqX_bg%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived against the change on master, where it is a small refactoring that the listed committer supports. The one surviving objection, maintenance-fit-1, concerns the proposed PG 19 backpatch: the patch changes an exported signature in ipc.h and adds a value in the middle of an enum, and the thread names no future backpatch that needs it. Targeting master only resolves it, and that choice belongs to the committer.

Not checked: Without the code, the state transitions on crash restart, the EXEC_BACKEND path and the postgres -C path could not be checked. Only 3 of the 7 messages were visible, so later review points may already answer some questions.

## Prior attempts

None found before 2026-10-30.

## Objections a reviewer is likely to raise

### 1. Backpatching to PG 19 changes exported API and startup behaviour in a stable branch

A packager ships a PG 19 minor release with this patch. An extension built against an earlier 19.x minor that calls CreateSharedMemoryAndSemaphores() now passes the wrong arguments. A library that registers a shared memory request between sizing and initialization now fails at server start. The user finds out only after the minor update.

The patch changes the signature of an exported function and adds an enum value in the middle of shmem_request_state. It also makes a previously accepted request timing fail at runtime. The only reason given for PG 19 is fewer conflicts in later backpatches (message #1, message #3), and the author says it is not a bug fix. A committer would likely ask for master only, since stable branches do not usually take refactorings that change exported symbols.

**Question:** Which concrete future backpatch would conflict without this change, and can the PG 19 version keep the old function signature and enum order?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAExHW5ttQauXiYCM4OKL8xKHZrndTN2jMN4WQjm1PkPNuqX_bg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAFC%2Bb6pDgRvied%3D1ERdX3GcOAZGV%2B%2BcThZkzeUdgrktzmvc2Pg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAExHW5uc4LodxiiY6S5uOaUNJk9wK-fXt93Ak-g4LJvLEzJ5jg%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Which concrete future backpatch would conflict without this change, and can a PG 19 version keep the old function signature and enum order?
2. Does any core path or known extension register a shared memory request between sizing and initialization today, so that SRS_SIZED would now reject it?
3. How do the EXEC_BACKEND child attachment and late allocation paths avoid entering SRS_SIZED?
4. Does postgres -C still report runtime-computed GUCs and exit before shared memory creation under the new call order?
5. Was restart_after_crash tested in a non-assert build, where the Assert against a second sizing call does not run?

## Touch points

- **ABI / extension API:** CreateSharedMemoryAndSemaphores() gains a Size argument in ipc.h, and SRS_SIZED rejects shared memory requests registered after sizing.
- **backpatchability:** Message #1 proposes PG 19 for a refactoring that changes an exported signature and tightens the request rules.

## Who would commit it, and when

- Owner: qualified committer engaged. Nathan Bossart is listed as committer on the CommitFest entry, reviewed the patch in message #2, and says there that the double call came from his own earlier work.
- Priority: proceed; audience niche; review cost low. The patch is about 9 KB across six startup and shared memory files, benefits developers who maintain this code, and has a committer engaged. The PG 19 backpatch is the only open point, and it can be dropped without changing the master patch.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
