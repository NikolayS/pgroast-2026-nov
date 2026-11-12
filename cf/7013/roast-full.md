# Design pre-review: Use Boyer-Moore-Horspool for simple LIKE contains patterns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7013](https://commitfest.postgresql.org/patch/7013/) (Needs review); authors: Atsushi Ogawa (atsushio)
- Thread: [\[PATCH\] Use Boyer-Moore-Horspool for simple LIKE contains patterns](https://www.postgresql.org/message-id/flat/CAEah3%3DOZ2NQ5cZZ_DHKhDHVYQb4jxRXZhBmhaayhVVknJUEwrA%40mail.gmail.com), started 2026-07-14
- Messages read: 14 of 14
- Patch prose read: Use-Boyer-Moore-Horspool-for-simple-LIKE-patterns.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The decisive objection is maintenance-fit-1. Message #13 measures a steady 2.7x slowdown on 4-symbol data, message #12 claims no bound, and affected users have no way to opt out. The list has to agree whether that trade is acceptable before code review makes sense. demand-1 adds that all evidence is synthetic and that v5 limits the gain to Const patterns. The goal itself drew no objection.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the escape handling, UTF-8 byte-boundary safety and the fn_extra caching rules were not checked. Only 3 of 14 messages were shown in full; the later numbers rest on the objections' citations.

## The proposal, restated

**Problem.** col LIKE '%literal%' is a common query shape, and the existing matcher tries each input position in turn, which is O(text * pattern) in the worst case. The author wants faster contains-searches for users who run such queries with medium or long literals.

**Proposed design.** textlike and namelike check the pattern on the first call and cache either a BMH search state or a generic marker in FmgrInfo.fn_extra. As of v5/v6, only Const patterns use BMH, with a single literal between leading and trailing %, deterministic collation, and single-byte or UTF-8 encoding. The search checks the candidate's last byte, then compares the rest left to right.

**What the author claims.**
- Constant-pattern contains-searches are about 19% to 57% faster in the v1 benchmark, with the gain growing with literal length.
- Patterns that do not qualify cost at most about 1% more in wall-clock time.
- Planner behavior and index-path choice, including pg_trgm, do not change.
- v5 still has narrow regressions, up to about 1.8x on repetitive LATIN1 input; the author does not claim a no-regression bound.
- The existing SB_MatchText, UTF8_MatchText and MB_MatchText code stays unchanged.

**Thread state.** Greg Sabino Mullane and Haibo Yan reviewed several versions, and Bryan Green found a stale-pattern bug with PL/pgSQL variables, fixed in v3. Haibo Yan called v5 ready for community review, and v6 addresses his minor points, but no committer has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-14; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Low-alphabet data gets slower after upgrade, with no way to opt out

fixable · level 2, rubric item 8 · performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A site stores DNA sequences or other 4-symbol strings in a text column and runs col LIKE '%<64-byte literal>%'. After a major-version upgrade the same scans run about 2.7 times slower. Nothing in EXPLAIN shows the change, and no setting turns it off.

In message #13, v5 still reaches a steady 2.7x regression on a 4-symbol alphabet at large input sizes. Message #12 reports up to 1.8x on repetitive LATIN1 data, and the author claims no bound. Message #11 shows that each fallback trades one regression class for another. The thresholds behind these numbers (6 bytes for single-byte, 4 for UTF-8) come from AArch64 SQL runs only. A committer would likely ask who decided this trade is acceptable for users who never asked for the change.

Thread: [#5 (Haibo Yan, 2026-09-15)](https://www.postgresql.org/message-id/CABXr29E0Pr2T27Wv%2BwS_jpiRtXy8KyadaJV7Udz4iP1m1-J%3DBw%40mail.gmail.com), [#11 (Atsushi Ogawa, 2026-09-19)](https://www.postgresql.org/message-id/CAEah3%3DNW-_M7D_LqOUi4vi-S7LRFiv7c3p8WwuyyiLVOAy77cQ%40mail.gmail.com), [#12 (Atsushi Ogawa, 2026-09-28)](https://www.postgresql.org/message-id/CAEah3%3DPrc-BGCLYf9uhJi40ddhC-uU3zBAp35D6z3D-S_zdjbg%40mail.gmail.com), [#13 (Haibo Yan, 2026-09-29)](https://www.postgresql.org/message-id/CABXr29Hhwg3UkMa_yyZ029qLF2RDsjW3ud141tT4fi_SToNQNw%40mail.gmail.com)

**Question:** What do x86-64 SQL-level numbers show for v6 on low-alphabet data, and how would an affected user get the old behaviour back?

Counter-check: held. Message #13 measures a steady 2.7x regression on a 4-symbol alphabet, and message #12 claims no bound. The thread shows no SQL-level x86-64 numbers for v5/v6, and nobody has decided whether the regression is acceptable.

### 2. Const-only restriction may leave out most real contains-search queries

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

An application sends `where name like $1` through JDBC, an ORM or PL/pgSQL with the search term bound as a parameter. Since v5, that pattern takes the existing matcher on the first call and never uses BMH. The user upgrades, sees no speedup, and still carries the low-alphabet regression risk wherever literals are inlined.

Message #12 limits BMH to Const patterns, after message #4 found stale-pattern bugs with Params, so the parameterized case no longer benefits. All demand evidence is synthetic: generated 1M-row tables and pg_attribute loops (message #1), plus matcher matrices (messages #5, #9, #13), with no field report or user request. Users with slow contains-searches on large tables can already use a pg_trgm index, which this patch does not change. The thread gives no numbers on how much of the stated benefit survives the Const-only rule.

Thread: [#1 (小川淳, 2026-07-14)](https://www.postgresql.org/message-id/CAEah3%3DOZ2NQ5cZZ_DHKhDHVYQb4jxRXZhBmhaayhVVknJUEwrA%40mail.gmail.com), [#4 (Bryan Green, 2026-09-14)](https://www.postgresql.org/message-id/024125bd-de8a-49d7-9523-3a22131aa3bd%40gmail.com), [#12 (Atsushi Ogawa, 2026-09-28)](https://www.postgresql.org/message-id/CAEah3%3DPrc-BGCLYf9uhJi40ddhC-uU3zBAp35D6z3D-S_zdjbg%40mail.gmail.com), [#13 (Haibo Yan, 2026-09-29)](https://www.postgresql.org/message-id/CABXr29Hhwg3UkMa_yyZ029qLF2RDsjW3ud141tT4fi_SToNQNw%40mail.gmail.com)

**Question:** Which real workloads send `LIKE '%literal%'` with a literal Const of at least 4 or 6 bytes over sequential scans, and how much time do they spend in the matcher?

Counter-check: held. All evidence in the thread is synthetic, and no message reports a real workload. v5 removed BMH for Params, and the thread gives no numbers on how much of the benefit that loses.

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
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds like_bmh.c with measured length thresholds and a second BMH search loop beside the one in varlena.c. |
| performance on the hot path | **yes** | textlike and namelike gain a dispatch step on every call, and BMH is slower on low-alphabet and repetitive input. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Const-only restriction may leave out most real contains-search queries |

## 5. Questions the author must answer before anyone reviews code

1. What do SQL-level x86-64 benchmarks of v6 show for low-alphabet and repetitive input, compared with the existing matcher?
2. What is the worst-case slowdown of v6 against the existing matcher, and which inputs produce it?
3. How would a user whose data hits the 2.7x regression get the old matcher back?
4. Which real workloads run LIKE '%literal%' with a Const literal of at least 4 or 6 bytes over sequential scans, and how much time do they spend in the matcher?
5. What did the earlier BMH/LIKE threads linked in message #5 conclude, and how does v6 answer them?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the LIKE matcher in utils/adt and its encoding and collation rules. No committer has replied in the visible thread.
- **Liability:** The owner signs for byte-level match correctness across encodings and collations, and for every later report of a slower contains-search, with thresholds tuned on AArch64 only.
- **Author follow-up:** The author answered the first review within three days and posted six versions, each addressing reviewer findings.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The query shape is common and reviewers engaged, but v5 still shows a 2.7x regression with no stated bound and no opt-out. The patch is about 23 KiB within utils/adt, but judging it needs benchmarks across encodings and architectures.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Earlier BMH/LIKE concerns linked in message #5 are never answered. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same 2.7x low-alphabet regression with the same failure scenario. The material does not show what the linked earlier discussions say, so they cannot support a separate objection.
- `maintenance-fit-2` Second Boyer-Moore-Horspool implementation in utils/adt. Refuted (already handled): Message #2 raised this and saw no advantage in refactoring; message #3 gave the reasons and added a cross-reference comment. Since v5 the LIKE loop differs by design (guard byte first, left-to-right compare, preloaded shift), so a varlena.c fix would not carry over unchanged.
- `maintenance-fit-3` Every textlike call pays for dispatch; only literal Const patterns gain. Refuted (already handled): Message #1 measures the fallback cost at about 1% or less, and message #3 shows +0.3% for v2. v5 caches the generic marker on the first call, as v1 did, with no per-row revalidation. The share-of-constants question is a duplicate of demand-1.

### Review angles

- prior-attempts: 1 objection(s). The precedent index returned no matching attempt. Message #5 links earlier BMH/LIKE discussions, but the material does not report their content, so no prior attempt entry is listed.
- standards-compat: 0 objection(s). The proposal adds no syntax, keyword, GUC, catalog object, psql feature or dump state, so nothing needs a pg_dump round-trip. Results stay the same as the existing matcher: nondeterministic collations and non-UTF-8 multibyte encodings keep the old path, and differential tests matched before and after the patch. The remaining issues are performance and design, which other angles cover.
- replication-ondisk: 0 objection(s). The proposal changes no persistent or cross-node state: no on-disk format, WAL record, catalog, wire message or existing struct changes. The BMH state lives only in per-backend FmgrInfo.fn_extra memory and LIKE results do not change, so standbys, subscribers and pg_upgrade are unaffected.
- security: 0 objection(s). The patch changes only how textlike and namelike match a constant pattern, so it adds no privilege, role, GUC, catalog entry or new exposure. Bryan Green audited it for security (message #4), and the stale-pattern bug he found was a correctness issue, fixed in v3. The remaining slowdown is a constant factor of about 2.7x, linear in input size (message #13), so it does not give an unprivileged user a new denial-of-service path. Leakproofness and RLS behaviour are unchanged because the functions and the planner are the same.
- maintenance-fit: 3 objection(s). Not applicable. Message #5 links earlier BMH/LIKE threads that the material does not include, so they could not be weighed. No committer has engaged in the visible thread, which bears on committability only.
- already-exists: 0 objection(s). Postgres already has two ways around slow '%literal%' searches. strpos() uses the BMH search in varlena.c, but every LIKE query, including generated ones, would have to be rewritten. A pg_trgm index avoids most scans but still rechecks rows with the LIKE matcher. Neither covers what the patch speeds up, and the thread settled the duplicate BMH code in messages #2 and #3.
- demand: 1 objection(s). The query shape is common and three reviewers engaged, but the material shows no user asking for this. The Const-only rule in v5 may shrink the group that benefits.

### Precedents retrieved

- P1: [pgbench: faster version of tpcb-like transaction](https://commitfest.postgresql.org/patch/1257/) (cf, Returned with feedback, 2017-10-01) - judged not relevant; reason summarised from its thread
- P2: [Allow LISTEN on patterns](https://commitfest.postgresql.org/patch/5630/) (cf, Withdrawn, 2025-03-07) - judged not relevant; reason summarised from its thread
- P3: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P4: [Make deparsing of column defaults faster](https://commitfest.postgresql.org/patch/1657/) (cf, Returned with feedback, 2018-09-24) - judged not relevant; reason summarised from its thread
- P5: [Faster pglz compression](https://commitfest.postgresql.org/patch/2897/) (cf, Withdrawn, 2023-02-28) - judged not relevant; reason summarised from its thread
- P6: [Pattern based listeners for asynchronous messaging (LISTEN/NOTIFY)](https://commitfest.postgresql.org/patch/1218/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P7: [Fix to_char() behaviour with 'WW-D' pattern](https://commitfest.postgresql.org/patch/2388/) (cf, Returned with feedback, 2020-07-10) - judged not relevant
- P8: [Speed up transaction completion faster after many relations are accessed in a transaction](https://commitfest.postgresql.org/patch/1993/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P9: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P10: [Add a greedy join search algorithm to handle large join problems](https://commitfest.postgresql.org/patch/6486/) (cf, Withdrawn, 2026-02-14) - judged not relevant; reason summarised from its thread
- P11: [change regexp_substr first argument make tests more easier to understand](https://commitfest.postgresql.org/patch/4744/) (cf, Withdrawn, 2024-09-23) - judged not relevant; reason summarised from its thread
- P12: [Revert some careless search-and-replace: "ADD" in comment text should not be replaced with "ADD_P".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=12119188fe04cd20a6f1317f62886b685ae71c59) (revert, Reverted, 2005-12-29) - judged not relevant
- P13: [Skip replicating the tables specified in except table option](https://commitfest.postgresql.org/patch/3646/) (cf, Withdrawn, 2024-01-09) - judged not relevant; reason summarised from its thread
- P14: [Add CREATE SCHEMA ... LIKE support](https://commitfest.postgresql.org/patch/6481/) (cf, Returned with feedback, 2026-04-06) - judged not relevant
- P15: [create table like: ACCESS METHOD](https://commitfest.postgresql.org/patch/2865/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P16: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant
- P17: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P18: [Track skipped vacuum and analyze in pg_stat_all_tables](https://commitfest.postgresql.org/patch/6640/) (cf, Withdrawn, 2026-04-03) - judged not relevant
- P19: [Improved crc32c performance with AVX-512 algorithm.](https://commitfest.postgresql.org/patch/5100/) (cf, Returned with feedback, 2025-04-02) - judged not relevant
- P20: [Add uri percent-encoding for binary data](https://commitfest.postgresql.org/patch/2299/) (cf, Returned with feedback, 2020-07-29) - judged not relevant
- P21: [Optimize numeric.c mul_var() using the Karatsuba algorithm](https://commitfest.postgresql.org/patch/4930/) (cf, Withdrawn, 2024-09-04) - judged not relevant
- P22: [autovacuum may skip tables when session_authorization/role is set on database](https://commitfest.postgresql.org/patch/4702/) (cf, Returned with feedback, 2024-01-11) - judged not relevant
- P23: [vector search support](https://commitfest.postgresql.org/patch/4292/) (cf, Returned with feedback, 2023-11-07) - judged not relevant
- P24: [Contains and is contained by operators of inet datatypes](https://commitfest.postgresql.org/patch/805/) (cf, Rejected, 2016-11-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 111 s; cost reported by the CLI: $2.55
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 68 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:15:22.406Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
