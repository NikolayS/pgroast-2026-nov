# Design pre-review: Speed up lpad() and rpad()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7340](https://commitfest.postgresql.org/patch/7340/) (Needs review); authors: Sehrope Sarkuni (sehrope)
- Thread: [Speed up lpad() and rpad() for one-byte padding strings](https://www.postgresql.org/message-id/flat/CAH7T-apj%2BpFg9bRkXVGfGejS2Uu4WzdMd7KoLKTW11mdsrt1Ew%40mail.gmail.com), started 2026-09-23
- Messages read: 12 of 12
- Patch prose read: v5-0001-Factor-the-padding-loop-out-of-lpad-and-rpad.patch, v5-0003-Use-repeat_bytes-in-repeat.patch, v5-0002-Copy-whole-repetitions-of-the-pad-string-at-once-.patch, 0001-Use-memset-for-one-byte-padding-in-lpad-and-rpad.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and no level fails. A committer co-wrote the series and Heikki asked for the shared design. Three fixable points are open: demand-2 and maintenance-fit-2 rest on the regressions measured in message #12, which have no reply, and demand-1 on the lack of a production workload with large N. The committer who signs must weigh these.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check how cheaply append_padding() and repeat_bytes() could add a small-N or large-source path. It also could not check pad validation for the partial tail.

## The proposal, restated

**Problem.** lpad() and rpad() call pg_mblen_range() and memcpy() once per padding character, so large pads are slow; repeat() copies once per repetition. The author points to large N in test data, benchmarks and reproducers, such as join_hash.sql using rpad('', 320000, 'x'). John Naylor says real uses rarely exceed N of 100, where no difference is measured.

**Proposed design.** v5 moves the padding loop of lpad() and rpad() into a shared helper, append_padding(), which counts the pad characters once, writes whole repetitions with a new repeat_bytes() and copies the partial tail as bytes. repeat_bytes() copies the source once, doubles the copied region to at least 16 kB, then copies that block repeatedly with CHECK_FOR_INTERRUPTS() between copies. repeat() also uses repeat_bytes(). The first one-byte memset() fast path was dropped.

**User-visible surface.**
- Long lpad() and rpad() calls become cancellable; on master they ignore statement_timeout until the call ends.

**What the author claims.**
- rpad and lpad with a 1M result drop from about 4.8 ms to about 0.11 ms, for any pad length.
- repeat() is about 4 times faster for 1-byte sources at large N and at parity for sources of 100 bytes or more.
- Output and pad validation are unchanged from master, including errors for invalid pad strings.
- A 16 kB block performs the same as blocks up to 4 MB on the author's machine; 16 MB is slower.
- The doubling idea for repeat() comes from a separate thread whose memset()-only v2 is Ready for Committer.

**Thread state.** Nathan Bossart co-wrote the series and is committer; Heikki asked that repeat(), lpad() and rpad() share one implementation, which v5 does. Manu (Zen 5) and shihao zhong (M2 Max) report gains but also regressions: short lpad up to three pad characters, and repeat() with 1000-byte source at 10M is 24% slower on M2 Max.

## 2. Prior attempts

### optimize several list functions with SIMD intrinsics (related)

- **Link:** [https://commitfest.postgresql.org/patch/4238/](https://commitfest.postgresql.org/patch/4238/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20230308002502.GA3378449@nathanxps13](https://www.postgresql.org/message-id/flat/20230308002502.GA3378449@nathanxps13)
- **Date:** decided 2023-07-10; started 2023-03-08
- **Outcome:** Returned with feedback (cf)
- **Who objected:** John Naylor, Heikki Linnakangas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** No workload was shown where these List functions matter, and the author agreed that a profile showing them should come first. Heikki Linnakangas checked callers such as tlist_member and list_delete_ptr, found none in hot code paths where the change would make a noticeable difference, and returned the patch with feedback.
- **Reason (quote):** "Grepping for "tlist_member" and "list_delete_ptr", I don't see any callers in hot codepaths where this could make a noticeable difference." ([message](https://www.postgresql.org/message-id/1184a66b-efa4-6fc8-ba8c-72db6f3b1352%40iki.fi))
- **What is different this time:** P4 changed List searches and showed no hot callers. This proposal shows large measured gains, but only at N far above what message #7 calls typical, so the question of a real workload is shared.


## 3. Objections a reviewer is likely to raise

### 1. Short-pad callers pay so that large-pad callers gain

fixable · level 1b, rubric item 4 · desirability, performance on the hot path · `demand-2`

A reporting query formats millions of rows with lpad(code, 7, '0') on ARM. Each call now runs more instructions than on master, so the common formatting case gets slower, and nobody notices until a release-to-release comparison.

Message #12 measures lpad(s, 7, '0') at 772 instructions per call against 694 on master, and reports that padding of up to three characters costs more. That size is the usual case John Naylor describes in message #7. Message #12 also finds repeat() with a 1000-byte source 24% slower at 10M. The thread has no reply to these numbers, so the trade between common small calls and rare large calls has not been weighed.

Thread: [#7 (John Naylor, 2026-09-24)](https://www.postgresql.org/message-id/CANWCAZaB8NsQJrKc-pxRTj_%3DC8CBOTDEWkPW0Eur%3DymUSbUwZQ%40mail.gmail.com), [#12 (shihao zhong, 2026-10-03)](https://www.postgresql.org/message-id/CAGRkXqQ%2BFM6OMnZo%3Da-p2xqXSGGwC6VD4C7hmsJEASiyFBecrQ%40mail.gmail.com)

**Question:** Can the patch keep short pads, up to about 16 characters, at master's cost on both x86 and ARM, with timings for a typical row-formatting workload?

Counter-check: held. Message #12 reports more instructions for pads of up to three characters, and the thread has no reply. The x86 timings in message #4 show no difference at N = 10, so the size of the cost is open, but the trade has not been weighed.

### 2. Speedup targets large N, which the thread shows mainly in test data

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer pushes the shared doubling helper for lpad(), rpad() and repeat(). Typical callers, such as lpad(id, 7, '0'), see no change, so Postgres takes on a new copy strategy and a tuned 16 kB block for a gain few production queries see.

In message #7, John Naylor says real N rarely exceeds 100, and the tables in message #1 show no difference up to N = 100. Message #8 answers with join_hash.sql, benchmarks and reproducers, all of which build test data once and do not run in production loops. No field report, -general thread or user workload is cited, and message #8 got no reply. The cancellation gain from message #11 is real, but adding CHECK_FOR_INTERRUPTS() to the existing loop would also give it.

**Precedents.**
- [optimize several list functions with SIMD intrinsics](https://commitfest.postgresql.org/patch/4238/) (Returned with feedback, 2023-07-10); thread: [link](https://www.postgresql.org/message-id/flat/20230308002502.GA3378449@nathanxps13)

Thread: [#1 (Sehrope Sarkuni, 2026-09-23)](https://www.postgresql.org/message-id/CAH7T-apj%2BpFg9bRkXVGfGejS2Uu4WzdMd7KoLKTW11mdsrt1Ew%40mail.gmail.com), [#7 (John Naylor, 2026-09-24)](https://www.postgresql.org/message-id/CANWCAZaB8NsQJrKc-pxRTj_%3DC8CBOTDEWkPW0Eur%3DymUSbUwZQ%40mail.gmail.com), [#8 (Sehrope Sarkuni, 2026-09-24)](https://www.postgresql.org/message-id/CAH7T-aoGV1tTdYy0OrDxkgkue%3Du7wTRxztTLPxUs0OZ%3DL2hF%3Dg%40mail.gmail.com), [#11 (Manu, 2026-09-25)](https://www.postgresql.org/message-id/179029949414.497877.6537870082909473028%40gmail.com)

**Question:** Which production workloads call lpad(), rpad() or repeat() with results large enough to matter, and how much time do they spend there today?

Counter-check: held. Message #8 grants that formatting uses have small N and names only test data, benchmarks and reproducers. An engaged committer co-wrote the series, which lowers confidence, but the thread shows no production workload.

### 3. Fixed 16 kB block makes some repeat() calls slower

fixable · level 2, rubric item 15 · maintenance burden ("who maintains this in 10 years"), performance on the hot path · `maintenance-fit-2`

A user on Apple Silicon builds 10M test values with repeat() from a 1000-byte source and gets results 24% slower after upgrading. Later reports from other CPUs or glibc versions ask for a different block size. The project then has to tune a constant that depends on memcpy() behaviour it does not control.

On master, repeat() with a large source was already at memcpy() speed, and message #12 shows v5 is 24% slower there, which message #12 attributes to memory cost and says block size does not fix. The 16 kB value rests on one machine (message #10) plus Zen 5 (message #11). The repeat() thread found a different best size on Intel, tied to glibc's non-temporal threshold. Nothing in the tree would detect a later regression on another platform.

Thread: [#10 (Sehrope Sarkuni, 2026-09-24)](https://www.postgresql.org/message-id/CAH7T-aq%2BahEAKamY1JU2jN6X_N3OLD63kJ7jfn9YyRVNd1GkBA%40mail.gmail.com), [#11 (Manu, 2026-09-25)](https://www.postgresql.org/message-id/179029949414.497877.6537870082909473028%40gmail.com), [#12 (shihao zhong, 2026-10-03)](https://www.postgresql.org/message-id/CAGRkXqQ%2BFM6OMnZo%3Da-p2xqXSGGwC6VD4C7hmsJEASiyFBecrQ%40mail.gmail.com)

**Question:** Can repeat_bytes() keep master's per-copy loop when the source is already large, so that no measured platform regresses, and what policy decides the block size on future hardware?

Counter-check: held. Message #12 measures repeat() with a 1000-byte source at 10M as 24% slower on M2 Max, and the thread has no reply. Message #11 shows 16 kB costs nothing on Zen 5, but no Intel machine ran v5.

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
| maintenance burden ("who maintains this in 10 years") | **yes** | repeat_bytes() carries a fixed 16 kB block size tuned on three machines. |
| performance on the hot path | **yes** | The series rewrites the copy loops of lpad(), rpad() and repeat(), which run once per row in formatting queries. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Speedup targets large N, which the thread shows mainly in test data |

## 5. Questions the author must answer before anyone reviews code

1. Can v5 keep pads of a few characters at master's cost on both x86 and ARM, shown with timings for a row-formatting query such as lpad(s, 7, '0')?
2. Which production workloads call lpad(), rpad() or repeat() with results large enough to matter, and how much time do they spend there today?
3. Can repeat_bytes() keep master's per-copy loop for large sources, so that repeat() with a 1000-byte source at 10M is not slower on M2 Max?
4. How does v5 with the 16 kB block perform on an Intel machine, where the repeat() thread found a different best size?
5. How will this series be ordered or merged with the Ready for Committer repeat() patch from the other thread?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Nathan Bossart co-wrote the series and is listed as committer on the CommitFest entry.
- **Liability:** The owner signs for the 16 kB block size and for any later platform report of slower short pads or large-source repeat() calls. Nothing in the tree detects such a regression.
- **Author follow-up:** The author revised the series up to v5 in message #10 after review. Message #12, the last message, has no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The series is three small patches in oracle_compat.c with a committer engaged, and it makes long lpad() and rpad() calls cancellable. The speed gain reaches only large N, and the regressions from message #12 need an answer before commit.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Common short-pad calls pay so that rare large pads gain. Refuted (not specific): Duplicate of demand-2, which states the same short-pad regression from message #12 without relying on [P4]. [P4] concerned List searches with no hot callers, so it does not support a claim of a measured regression.
- `maintenance-fit-1` Short lpad() and rpad() calls pay for a large-N speedup. Refuted (not specific): Duplicate of demand-2. Both rest on the 772 against 694 instruction count in message #12 and ask for the same small-N path.
- `already-exists-1` For bulk test data, repeat() already covers the large-N rpad() case. Refuted (not specific): Duplicate of demand-1, which asks the same question about who needs large N. In message #9 a committer asked that repeat(), lpad() and rpad() share one implementation, so the proposal does not add a separate second fast path.

### Review angles

- prior-attempts: 1 objection(s). The only earlier discussion named in the thread is the repeat() thread (message #9), which is still open and not a rejection. P12 and P8 died on unmeasurable gains, which does not apply to this proposal.
- standards-compat: 0 objection(s). The series adds no syntax, GUC, catalog object or dump state, and Manu's comparison across three encodings in message #11 shows identical output and errors. The only visible change is that long lpad() and rpad() calls become cancellable, which matches what repeat() already does on master. Invalid pad strings were not in that comparison; the encoding.sql tests cover them.
- replication-ondisk: 0 objection(s). The patch has no persistent or cross-node state. It changes in-memory copy loops in lpad(), rpad() and repeat() in oracle_compat.c, with no WAL, catalog, on-disk, protocol or exported-ABI change. Manu checked output, md5 of every result and error messages against master in UTF8, EUC_JP and SQL_ASCII databases (message #11), so stored results, indexes and replicas see identical values.
- security: 0 objection(s). The proposal changes no privilege, catalog or exposure. The commit message for 0002 says pad validation is unchanged, and message #11 reports identical output and errors across UTF8, EUC_JP and SQL_ASCII. The new cancellation check reduces the existing denial-of-service exposure from long lpad() and rpad() calls.
- maintenance-fit: 2 objection(s). The change is contained in oracle_compat.c, a committer co-wrote it, and it adds cancellation to lpad() and rpad(). The objections are about regressions in cases that are not the target, not about fit in core.
- already-exists: 1 objection(s). One objection only. The cancellation gain for long lpad()/rpad() calls in message #11 has no existing alternative, so it is not raised here.
- demand: 2 objection(s). Demand is the weak point, but not fatal. A committer co-wrote the series, Heikki asked for the shared design, and the cancellation fix has its own value, so this reads as 'show demand and fix the short-pad regression', not a rejection.

### Precedents retrieved

- P1: [Fix slotsync worker busy loop causing repeated log messages](https://commitfest.postgresql.org/patch/6546/) (cf, Withdrawn, 2026-05-01) - judged not relevant; reason summarised from its thread
- P2: [Precompute string lengths in PerformRadiusTransaction](https://commitfest.postgresql.org/patch/6377/) (cf, Withdrawn, 2026-03-25) - judged not relevant; reason summarised from its thread
- P3: [Revert "Add missing padding from MemoryChunk struct"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5495796ad12aae971137d184ff25add8f6b03968) (revert, Reverted, 2022-08-30) - judged not relevant
- P4: [optimize several list functions with SIMD intrinsics](https://commitfest.postgresql.org/patch/4238/) (cf, Returned with feedback, 2023-07-10) - used in the review; reason summarised from its thread
- P5: [Revert "isolationtester: don't repeat the is-it-waiting query when retrying a step."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dca369320f6023b55feb49f281d394181fc57903) (revert, Reverted, 2016-02-12) - judged not relevant
- P6: [Revert idea of zer-padding padding session id in log_line_prefix](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b06e6aba8773df8040b357a1cb058879ed2d94f) (revert, Reverted, 2013-05-06) - judged not relevant
- P7: [Decoupling our alignment assumptions about int64 and double](https://commitfest.postgresql.org/patch/6441/) (cf, Withdrawn, 2026-02-03) - judged not relevant; reason summarised from its thread
- P8: [micro-optimize nbtcompare.c routines](https://commitfest.postgresql.org/patch/5273/) (cf, Withdrawn, 2024-09-27) - used in the review; reason summarised from its thread
- P9: [Revert "Refactor CopyAttributeOut{CSV,Text}() to use a callback in COPY TO"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aa8324b81fa4979ffdc6ccf81d560eac9446948) (revert, Reverted, 2024-02-06) - judged not relevant
- P10: [Refactoring MemSet/memset api usage](https://commitfest.postgresql.org/patch/3645/) (cf, Returned with feedback, 2022-09-30) - judged not relevant; reason summarised from its thread
- P11: [Subtrans performance smoothing](https://commitfest.postgresql.org/patch/3792/) (cf, Withdrawn, 2022-09-06) - judged not relevant; reason summarised from its thread
- P12: [Avoid multiple calls to memcpy (src/backend/access/index/genam.c)](https://commitfest.postgresql.org/patch/6586/) (cf, Withdrawn, 2026-03-25) - used in the review; reason summarised from its thread
- P13: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant
- P14: [Resume vacuum and autovacuum from interruption and cancellation](https://commitfest.postgresql.org/patch/2211/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P15: [KnownAssignedXidsGetAndSetXmin performance](https://commitfest.postgresql.org/patch/3271/) (cf, Returned with feedback, 2022-11-15) - judged not relevant
- P16: [Fixes wrong memset api usage](https://commitfest.postgresql.org/patch/3640/) (cf, Rejected, 2022-05-18) - judged not relevant
- P17: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - judged not relevant
- P18: [Revert "Optimize various aggregate deserialization functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f3b56eea23554e1756a26080db273156f23f4f2) (revert, Reverted, 2023-10-10) - judged not relevant
- P19: [Speed up LockReleaseAll](https://commitfest.postgresql.org/patch/3220/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P20: [Speed up releasing of locks](https://commitfest.postgresql.org/patch/3501/) (cf, Withdrawn, 2024-05-08) - judged not relevant
- P21: [Allow COPY "text" to output a header and add header matching mode to COPY FROM](https://commitfest.postgresql.org/patch/2504/) (cf, Returned with feedback, 2020-09-29) - judged not relevant
- P22: [SLOPE - Planner optimizations on monotonic expressions.](https://commitfest.postgresql.org/patch/6653/) (cf, Withdrawn, 2026-07-20) - judged not relevant
- P23: [Remove extra Sort node above a btree-compatible Index Scan](https://commitfest.postgresql.org/patch/5606/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P24: [pgbench: allow to cancel queries during benchmark](https://commitfest.postgresql.org/patch/4399/) (cf, Returned with feedback, 2024-10-07) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 148 s; cost reported by the CLI: $2.23
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 76 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:42:09.522Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
