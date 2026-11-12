# Design pre-review: Fall back to serial vacuum when out of DSM segments

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7380](https://commitfest.postgresql.org/patch/7380/); [thread](https://www.postgresql.org/message-id/flat/CALj2ACVjBN03Fq4XWgKH1zOfkapK6-P3ufNLHUHnZcW1zPm%2BMg%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). No fundamental objection survived for HEAD. The fallback copies what parallel index builds do, adds no user-visible surface, and costs nothing for users who do not run parallel vacuum. The one surviving objection, demand-1, is fixable and bears mostly on the "stable" target, which the thread does not argue. A committer who knows parallel vacuum must decide on the backpatch and on removing the nworkers check from cc053b6e127.

Not checked: The follow-up patch and messages #4 to #7 were not shown. Without the code, it is not possible to confirm that removing the nworkers check is safe or that every caller handles a NULL return from parallel_vacuum_init().

## Prior attempts

None found before 2026-09-28.

## Objections a reviewer is likely to raise

### 1. No field report of DSM exhaustion breaking parallel vacuum

A committer pushes the fallback, possibly to stable branches as the CommitFest target suggests. No user who reported the error exists to confirm that it helps. Users who do run out of DSM slots, and still have one slot left, still see "too many dynamic shared memory segments" from the dead-TID allocation.

The only evidence is a synthetic test in message #4, where another session holds every DSM slot. Message #7 calls DSM exhaustion very rare and says the reviewer had never seen it. Message #6 says the error is acceptable because it can happen anyway, and message #7 agrees that the one-slot-left case still fails. So the fix covers a narrow case with no reported users. The change is small, but a Bugfix tag with target "stable" needs more than a synthetic case.

**Question:** Which production systems have seen parallel vacuum or parallel autovacuum fail with "too many dynamic shared memory segments", and would the all-slots-taken fallback have covered them?

<sub>fixable, level 1b, demand; messages [#4](https://www.postgresql.org/message-id/CAJTYsWVvGOGmyXVjjOwC4-R31Lp1iay27jR5tukr%2BCTh2JefMA%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CALj2ACUO8CFQoN9PRenxiuYHXE1hVBHd2_OqHvEUagXO-C-VdQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAGRkXqQHk9VjvXyRwTvOnJBv1K276uXCyDLb0kDvNjSHFFMeAQ%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Is this change meant for HEAD only, or for backpatch to 19 as the "stable" target on the CommitFest entry suggests?
2. Which production systems have seen parallel vacuum or parallel autovacuum fail with "too many dynamic shared memory segments", and would the all-slots-taken fallback have covered them?
3. What happens when the parallel context gets a segment but the later dead-TID DSM allocation fails: does vacuum still raise the error, and is that accepted?
4. Which case did the nworkers check from cc053b6e127 cover, and why can it not occur once parallel_vacuum_init() returns NULL for a NULL pcxt->seg?
5. Does vacuum log or report that it fell back to serial because no DSM segment was free?

## Touch points

- **backpatchability:** The CommitFest entry is tagged Bugfix with target "stable", but the thread discusses only HEAD.
- **maintenance burden ("who maintains this in 10 years"):** The patch removes the nworkers check that crash fix cc053b6e127 added in vacuumparallel.c.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No field report of DSM exhaustion breaking parallel vacuum

## Who would commit it, and when

- Owner: qualified committer engaged. Masahiko Sawada pushed the related crash fix in message #3. The follow-up needs a committer who knows parallel vacuum and the DSM fallback in InitializeParallelDSM().
- Priority: proceed; audience niche; review cost low. The thread shows DSM exhaustion only in a synthetic test, so few users benefit. The change is local to parallel_vacuum_init() and follows the existing parallel index build fallback; the follow-up patch itself was not read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
