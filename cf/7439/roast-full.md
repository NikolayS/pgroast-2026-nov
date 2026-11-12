# Design pre-review: Use a hash table for the regexp cache

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7439](https://commitfest.postgresql.org/patch/7439/) (Needs review); authors: Tianyi Gao (gtydoit.gmail.com)
- Thread: [Use a hash table for the regexp cache](https://www.postgresql.org/message-id/flat/CANZS%2BG-i4TZ1CET0LXOSTQqWZQsLQbUY3BmV0XvWYz%3DPeO4ekg%40mail.gmail.com), started 2026-10-13
- Messages read: 3 of 3
- Patch prose read: v1-0001-Use-a-hash-table-for-the-regular-expression-cache.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

No surviving objection finds a flaw in the design of 0001. Both surviving objections are about demand and timing. With the limit kept at 32, message #1 shows gains only up to 32 patterns and no change at 33 or more (maintenance-fit-1). The thread cites no workload that uses more than 32 patterns per backend (demand-1). The list should agree on the sizing direction before 0001 is reviewed as groundwork.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the claim that an ERROR during compile or insert leaves the cache consistent could not be checked. The same holds for whether a caller's regex_t stays valid when its entry is evicted.

## The proposal, restated

**Problem.** Each backend caches at most 32 compiled regexps (MAX_CACHED_RES, fixed at compile time) in an array searched linearly. Workloads that cycle through more than 32 patterns recompile on every call, about 9x slower in the author's benchmark. The linear search also removes the gain from a larger limit, because a hit in a 1000-entry array costs about as much as compiling.

**Proposed design.** v1-0001 replaces the array with a simplehash table keyed by pattern, flags and collation, plus a dlist in LRU order, so lookup and move-to-front are O(1). The most recently used entry is compared before hashing, and each cached_re_str lives in its regexp's own memory context so the returned regex_t no longer moves. The limit stays at 32; a sizing GUC is left for a later patch.

**User-visible surface.**
- RE_compile_and_cache() in regex/regex.h: the returned regex_t now stays at a stable address while the cache is reorganized.

**What the author claims.**
- Behavior is unchanged: same limit of 32 and same move-to-front policy.
- No regression for a single repeated pattern (2.79 ms vs 2.84 ms) and about 11% faster at 32 patterns.
- With MAX_CACHED_RES=1024, a 1000-pattern workload is about 7.5 times faster than the array at the same limit.
- An ERROR during compile or insert leaves the cache consistent.
- A query-lifetime cache in fn_extra would be about 30% slower per short query and cannot cover SRFs, jsonpath, the planner or extensions.

**Thread state.** jian he proposed a query-level cache in fn_extra instead of the session cache; the author replied with measurements and caller cases against it. No one has reviewed the patch itself or answered the sizing question (memory GUC, count GUC, or a higher compiled-in limit).

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-13; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Groundwork commit has little value until the sizing question is settled

fixable · level 1b, rubric item 5 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A committer pushes 0001, which replaces a short array loop with a simplehash table plus a dlist. If the list never agrees on a sizing option, users with 33 or more patterns still recompile on every call. The project then maintains more complex cache code for about 11% at exactly 32 patterns.

With the default limit, message #1 shows gains only up to 32 patterns, and the 33, 64 and 1024 rows are unchanged. The large gains appear only with MAX_CACHED_RES=1024, which the patch does not ship. The commit message calls 0001 "groundwork", and message #1 leaves the next step open among three options with no reply yet. A committer would likely ask to see the sizing direction agreed, or posted as 0002, before taking on the extra structure.

Thread: [#1 (Tianyi Gao, 2026-10-13)](https://www.postgresql.org/message-id/CANZS%2BG-i4TZ1CET0LXOSTQqWZQsLQbUY3BmV0XvWYz%3DPeO4ekg%40mail.gmail.com), [#3 (Tianyi Gao, 2026-11-02)](https://www.postgresql.org/message-id/CANZS%2BG8cDsrf34ukTUrSzmMsTxkPkmWxBePxOw96_NHEfuOQAw%40mail.gmail.com)

**Question:** Which sizing option will follow, and if none is agreed, does 0001 alone help a real workload enough to justify the more complex cache?

Counter-check: held. Message #1's default-limit table shows no change at 33 or more patterns, and the commit message calls 0001 groundwork. simplehash and dlist are existing infrastructure, which lowers the cost, but no one has answered the sizing question.

### 2. No workload shown that cycles through more than 32 regexps

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer merges the new cache structure and then adds a sizing GUC. No user turns out to run more than 32 distinct patterns per backend, so every backend carries extra code and one more setting that nobody tunes.

The only evidence of demand is a wiki TODO entry and a 2008 thread (message #1). No field report, -general or -bugs thread, or production workload is cited for the 33+ case, which is the case where the 9x cost appears. The benchmark is synthetic: each query cycles through N patterns on purpose. Message #2 questions the session-level design itself, and nobody in the thread reports needing a larger cache.

Thread: [#1 (Tianyi Gao, 2026-10-13)](https://www.postgresql.org/message-id/CANZS%2BG-i4TZ1CET0LXOSTQqWZQsLQbUY3BmV0XvWYz%3DPeO4ekg%40mail.gmail.com), [#2 (jian he, 2026-11-01)](https://www.postgresql.org/message-id/CACJufxHL2KBstA%3D8D_KXRAXiJDYjNVqo_kkhE4kPHtaUSqQh_w%40mail.gmail.com), [#3 (Tianyi Gao, 2026-11-02)](https://www.postgresql.org/message-id/CANZS%2BG8cDsrf34ukTUrSzmMsTxkPkmWxBePxOw96_NHEfuOQAw%40mail.gmail.com)

**Question:** Which real applications or reports show backends regularly using more than 32 distinct regexps, and what recompile cost do they measure today?

Counter-check: held. The TODO entry and the 2008 thread show the interest has lasted, but the thread cites no workload that uses more than 32 patterns. The sizing GUC in the failure scenario is not part of 0001 and should not count against it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | RE_compile_and_cache() is exported in regex/regex.h, and the regex_t it returns now stays at a stable address. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | 0001 adds no GUC, but message #1 proposes a sizing GUC (by memory or by count) as the next step. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A simplehash table plus a dlist replaces a short array loop in regexp.c. |
| performance on the hot path | **yes** | Every regexp operator and function call goes through the cache lookup that the patch rewrites. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No workload shown that cycles through more than 32 regexps |

## 5. Questions the author must answer before anyone reviews code

1. Which sizing option (memory GUC, count GUC, or a higher compiled-in limit) will follow 0001, and can it be posted as 0002 in the same series?
2. Which real applications or reports show backends that regularly use more than 32 distinct regexps, and what recompile cost do they measure today?
3. If the limit stays at 32, does 0001 alone help a real workload enough to justify the more complex cache?
4. How much memory per backend does the hash table use compared with the array when only a few patterns are cached?
5. Do the benchmark results hold on a second platform and with more than one client?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the regex code in regexp.c and backend memory contexts would have to own it. No committer has spoken in the visible thread.
- **Liability:** The owner signs for cache consistency after an ERROR and for the lifetime of the regex_t returned to callers, including extensions, for as long as the cache exists.
- **Author follow-up:** The author answered message #2 within a day with measurements, offered to prototype the alternative, and said they will follow cfbot results.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A field report or production workload where backends cycle through more than 32 distinct regexps, with the recompile cost measured. Agreement on one sizing option, or a 0002 patch that implements it, would also change the answer.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** bundle
- **Audience:** unclear
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 16 KiB, local to regexp.c, and uses existing simplehash and dlist code. With the limit at 32 it gains about 11% at best, so it belongs in one series with the sizing patch that gives it its value.

## Appendix

### Refuted in the counter-check

- `demand-2` 0001 alone gives little benefit while the next step is undecided. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point with a more concrete request (agree the sizing direction or post it as 0002).

### Review angles

- prior-attempts: 0 objection(s). None of the listed precedents is this idea. [P8] reverted a subtrans XID cache for invalidation and wraparound reasons that do not apply to a regexp cache. [P7] and [P14] made a constant into a GUC, which 0001 does not do. The 2008 discussion cited in message #1 raised three concerns: measuring regexps in bytes, a dedicated count GUC, and O(N) search beyond about 1000 entries. Message #1 reports the first is gone and 0001 addresses the third. The GUC question is left for a later patch, so no earlier rejection reason applies to 0001.
- standards-compat: 0 objection(s). From the standards and compatibility angle, 0001 changes no SQL syntax, GUC, catalog, dump output or psql behaviour, and it keeps the same limit of 32 and the same move-to-front eviction. The only contract change is that RE_compile_and_cache() now returns a regex_t at a stable address, which weakens a constraint on callers and breaks none. The sizing GUC (memory or count) would need this review when it is proposed, but it is not part of this patch.
- replication-ondisk: 0 objection(s). The patch only changes a per-backend, in-memory cache in regexp.c, so it touches no persistent or cross-node state: no on-disk format, WAL, catalog, protocol or replication change. RE_compile_and_cache() keeps its signature, and its result now stays at one address, which only strengthens what extensions get; whether the internal struct is visible outside regexp.c could not be checked without the code.
- security: 0 objection(s). v1-0001 changes only the private lookup structure of a per-backend cache: same 32-entry limit, same key (pattern, flags, collation), no new privilege, view or shared state. The entry cap keeps hash collisions from user-chosen patterns small, and per-backend memory does not grow. A later sizing GUC would need its own review of the default and of who may raise it.
- maintenance-fit: 1 objection(s). Only one objection from this angle. The change is local to regexp.c, uses existing simplehash and dlist infrastructure, adds no GUC or catalog state, and keeps the single-pattern path to one comparison.
- already-exists: 0 objection(s). Postgres has no other way to get this result: MAX_CACHED_RES is a compile-time constant, and raising it makes lookups slow because the search is linear. No extension or hook can replace the backend regexp cache, and the patch reuses simplehash and dlist. The query-level fn_extra cache from message #2 was answered in message #3 with measurements and with callers that have no usable fn_extra.
- demand: 2 objection(s). The precedents listed concern walsender and subtransaction caches and do not apply to the demand question here.

### Precedents retrieved

- P1: [change regexp_substr first argument make tests more easier to understand](https://commitfest.postgresql.org/patch/4744/) (cf, Withdrawn, 2024-09-23) - judged not relevant; reason summarised from its thread
- P2: [Add regexp_positions(string text, pattern text, flags text) → setof record (starts integer\[\], lengths integer\[\])](https://commitfest.postgresql.org/patch/3027/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P3: [Allow regex operations in PL/Perl using UTF8 characters in non-UTF8 encoded databases](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Server-Side_Languages) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P4: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P5: [WIP: Track relation size in shared memory](https://commitfest.postgresql.org/patch/2933/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P7: [guc-ify the formerly hard-coded MAX_SEND_SIZE to max_wal_send](https://commitfest.postgresql.org/patch/958/) (cf, Returned with feedback, 2017-03-28) - used in the review; reason summarised from its thread
- P8: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - used in the review
- P9: [Revert "For all ppc compilers, implement pg_atomic_fetch_add_ with inline asm."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e9fae0696d9e3ff70a1438775ad9f786b854a5) (revert, Reverted, 2019-09-15) - judged not relevant
- P10: [Support regular expressions with nondeterministic collations](https://commitfest.postgresql.org/patch/5334/) (cf, Withdrawn, 2024-12-30) - judged not relevant; reason summarised from its thread
- P11: [replace magic num in struct cachedesc with CATCACHE_MAXKEYS](https://commitfest.postgresql.org/patch/5186/) (cf, Withdrawn, 2024-08-24) - judged not relevant; reason summarised from its thread
- P12: [JIT compilation per plan node](https://commitfest.postgresql.org/patch/4837/) (cf, Withdrawn, 2024-08-21) - judged not relevant; reason summarised from its thread
- P13: [Extracting only the columns needed for a query](https://commitfest.postgresql.org/patch/2331/) (cf, Withdrawn, 2020-07-01) - judged not relevant
- P14: [GUC-ify WALSender's MAX_SEND_SIZE constant](https://commitfest.postgresql.org/patch/4781/) (cf, Rejected, 2024-05-14) - used in the review
- P15: [Revert "Refactor compile-time assertion checks in c.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa650a99b4bfbe95e88f44070ae0df1818944a7) (revert, Reverted, 2020-03-23) - judged not relevant
- P16: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant
- P17: [Refactor compile-time assertion checks for C/C++](https://commitfest.postgresql.org/patch/2437/) (cf, Returned with feedback, 2020-03-23) - judged not relevant
- P18: [Restricting maximum keep segments by repslots](https://commitfest.postgresql.org/patch/1260/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P19: [Safer hash table initialization macro](https://commitfest.postgresql.org/patch/6276/) (cf, Withdrawn, 2026-03-09) - judged not relevant
- P20: [unnesting multirange data types](https://commitfest.postgresql.org/patch/3178/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P21: [Remove unnecessary 'always:' from CompilerWarnings task](https://commitfest.postgresql.org/patch/4550/) (cf, Withdrawn, 2023-11-13) - judged not relevant
- P22: [Subplan result cache](https://commitfest.postgresql.org/patch/1646/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P23: [Revert pg_bind_textdomain_codeset to a existant-but-empty function when ENABLE_NLS is not defined, for better compatibility of the backend with modules...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e43fd89762a5a894ea04f4fe74e05f95a1488664) (revert, Reverted, 2009-03-09) - judged not relevant
- P24: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 98 s; cost reported by the CLI: $1.06
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 59 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:56:17.954Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
