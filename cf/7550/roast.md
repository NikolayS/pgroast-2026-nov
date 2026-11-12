# Design pre-review: Check the fd number against FD_SETSIZE in PQsocketPoll

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7550](https://commitfest.postgresql.org/patch/7550/); [thread](https://www.postgresql.org/message-id/flat/985BA3B4-1281-451A-AE43-6ECCD87D75E4%40yandex-team.ru), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). One objection survived, maintenance-fit-1, and it is a fixable level 2 point about where the fix belongs. The guard itself replaces an out-of-bounds FD_SET() with an EINVAL failure and nobody objected to that. A committer who knows the libpq portability code has to decide between guarding the non-Windows select() fallback and removing it. Confidence is low because the thread has only the author's two messages and names no platform that compiles the branch.

Not checked: The code was not read. I could not check how the guard is placed relative to FD_SET(), how v2 excludes Windows, or whether other select() call sites have the same gap.

## Prior attempts

None found before 2026-10-29.

## Objections a reviewer is likely to raise

### 1. Guard goes into a branch no supported Unix may compile

v2 adds a new error path to the non-Windows select() fallback in PQsocketPoll(). If no buildfarm animal builds that branch, a later change to fe-misc.c can break it with no one noticing. The first report then comes from a user on a rare platform, and no committer can reproduce it.

Message #2 says the remaining case is "mostly theoretical", and the proposal names no supported platform that lacks poll(2). Code that is never compiled or run in testing costs review time now and can rot later. A committer would likely ask whether the right change is to require poll(2) on non-Windows platforms and drop that branch, which removes the risk instead of guarding it. The guard is small, so this is about where the fix goes.

**Question:** Which supported platform or buildfarm animal compiles the non-Windows select() fallback today, and if none does, why keep and extend that branch instead of removing it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/985BA3B4-1281-451A-AE43-6ECCD87D75E4%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/EA35FD13-9785-41D0-8D2F-FB3EE6A35BB0%40yandex-team.ru); `maintenance-fit-1`</sub>

## Questions for the author

1. Which supported platform or buildfarm animal compiles the non-Windows select() fallback in PQsocketPoll() today?
2. If no supported platform compiles that branch, why guard it instead of requiring poll(2) on non-Windows builds and removing it?
3. Do other select() call sites in libpq or the client tools call FD_SET() on an unchecked fd and need the same check?
4. Is the guard meant for backpatching, as the Bugfix tag suggests, or for master only?
5. How can the new EINVAL path be tested if no buildfarm animal builds the branch?

## Touch points

- **ABI / extension API:** PQsocketPoll() is a public libpq function and gains a new EINVAL failure case; its signature does not change.
- **security / privileges:** The patch replaces an out-of-bounds stack write in the client-side select() fallback with an error return.
- **backpatchability:** The CommitFest entry is tagged Bugfix, so a committer has to decide whether to backpatch the guard.
- **maintenance burden ("who maintains this in 10 years"):** The new error path sits in a select() branch that, per message #2, no known non-Windows platform compiles.

## Who would commit it, and when

- Owner: silent. Someone who knows libpq's socket code and the platform portability rules (which systems must have poll(2)) would have to own it. No reviewer has replied in the thread.
- Priority: fix design first; audience niche; review cost low. The patch is about 1.4 KB in one file, and message #2 calls the remaining case "mostly theoretical". The open point is whether to guard the non-Windows select() fallback or remove it, and that should be settled before code review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
