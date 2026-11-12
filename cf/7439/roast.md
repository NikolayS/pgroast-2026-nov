# Design pre-review: Use a hash table for the regexp cache

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7439](https://commitfest.postgresql.org/patch/7439/); [thread](https://www.postgresql.org/message-id/flat/CANZS%2BG-i4TZ1CET0LXOSTQqWZQsLQbUY3BmV0XvWYz%3DPeO4ekg%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). No surviving objection finds a flaw in the design of 0001. Both surviving objections are about demand and timing. With the limit kept at 32, message #1 shows gains only up to 32 patterns and no change at 33 or more (maintenance-fit-1). The thread cites no workload that uses more than 32 patterns per backend (demand-1). The list should agree on the sizing direction before 0001 is reviewed as groundwork.

Not checked: Without the code, the claim that an ERROR during compile or insert leaves the cache consistent could not be checked. The same holds for whether a caller's regex_t stays valid when its entry is evicted.

## Prior attempts

None found before 2026-10-13.

## Objections a reviewer is likely to raise

### 1. Groundwork commit has little value until the sizing question is settled

A committer pushes 0001, which replaces a short array loop with a simplehash table plus a dlist. If the list never agrees on a sizing option, users with 33 or more patterns still recompile on every call. The project then maintains more complex cache code for about 11% at exactly 32 patterns.

With the default limit, message #1 shows gains only up to 32 patterns, and the 33, 64 and 1024 rows are unchanged. The large gains appear only with MAX_CACHED_RES=1024, which the patch does not ship. The commit message calls 0001 "groundwork", and message #1 leaves the next step open among three options with no reply yet. A committer would likely ask to see the sizing direction agreed, or posted as 0002, before taking on the extra structure.

**Question:** Which sizing option will follow, and if none is agreed, does 0001 alone help a real workload enough to justify the more complex cache?

<sub>fixable, level 1b, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CANZS%2BG-i4TZ1CET0LXOSTQqWZQsLQbUY3BmV0XvWYz%3DPeO4ekg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CANZS%2BG8cDsrf34ukTUrSzmMsTxkPkmWxBePxOw96_NHEfuOQAw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. No workload shown that cycles through more than 32 regexps

A committer merges the new cache structure and then adds a sizing GUC. No user turns out to run more than 32 distinct patterns per backend, so every backend carries extra code and one more setting that nobody tunes.

The only evidence of demand is a wiki TODO entry and a 2008 thread (message #1). No field report, -general or -bugs thread, or production workload is cited for the 33+ case, which is the case where the 9x cost appears. The benchmark is synthetic: each query cycles through N patterns on purpose. Message #2 questions the session-level design itself, and nobody in the thread reports needing a larger cache.

**Question:** Which real applications or reports show backends regularly using more than 32 distinct regexps, and what recompile cost do they measure today?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CANZS%2BG-i4TZ1CET0LXOSTQqWZQsLQbUY3BmV0XvWYz%3DPeO4ekg%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CACJufxHL2KBstA%3D8D_KXRAXiJDYjNVqo_kkhE4kPHtaUSqQh_w%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CANZS%2BG8cDsrf34ukTUrSzmMsTxkPkmWxBePxOw96_NHEfuOQAw%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which sizing option (memory GUC, count GUC, or a higher compiled-in limit) will follow 0001, and can it be posted as 0002 in the same series?
2. Which real applications or reports show backends that regularly use more than 32 distinct regexps, and what recompile cost do they measure today?
3. If the limit stays at 32, does 0001 alone help a real workload enough to justify the more complex cache?
4. How much memory per backend does the hash table use compared with the array when only a few patterns are cached?
5. Do the benchmark results hold on a second platform and with more than one client?

## Touch points

- **ABI / extension API:** RE_compile_and_cache() is exported in regex/regex.h, and the regex_t it returns now stays at a stable address.
- **new GUC:** 0001 adds no GUC, but message #1 proposes a sizing GUC (by memory or by count) as the next step.
- **maintenance burden ("who maintains this in 10 years"):** A simplehash table plus a dlist replaces a short array loop in regexp.c.
- **performance on the hot path:** Every regexp operator and function call goes through the cache lookup that the patch rewrites.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No workload shown that cycles through more than 32 regexps

## Who would commit it, and when

- Owner: silent. Someone who knows the regex code in regexp.c and backend memory contexts would have to own it. No committer has spoken in the visible thread.
- Not now, show demand first: A field report or production workload where backends cycle through more than 32 distinct regexps, with the recompile cost measured. Agreement on one sizing option, or a 0002 patch that implements it, would also change the answer.
- Priority: bundle; audience unclear; review cost low. The patch is about 16 KiB, local to regexp.c, and uses existing simplehash and dlist code. With the limit at 32 it gains about 11% at best, so it belongs in one series with the sizing patch that gives it its value.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
