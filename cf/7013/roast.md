# Design pre-review: Use Boyer-Moore-Horspool for simple LIKE contains patterns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7013](https://commitfest.postgresql.org/patch/7013/); [thread](https://www.postgresql.org/message-id/flat/CAEah3%3DOZ2NQ5cZZ_DHKhDHVYQb4jxRXZhBmhaayhVVknJUEwrA%40mail.gmail.com), 14 of 14 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The decisive objection is maintenance-fit-1. Message #13 measures a steady 2.7x slowdown on 4-symbol data, message #12 claims no bound, and affected users have no way to opt out. The list has to agree whether that trade is acceptable before code review makes sense. demand-1 adds that all evidence is synthetic and that v5 limits the gain to Const patterns. The goal itself drew no objection.

Not checked: The code was not read, so the escape handling, UTF-8 byte-boundary safety and the fn_extra caching rules were not checked. Only 3 of 14 messages were shown in full; the later numbers rest on the objections' citations.

## Prior attempts

None found before 2026-07-14.

## Objections a reviewer is likely to raise

### 1. Low-alphabet data gets slower after upgrade, with no way to opt out

A site stores DNA sequences or other 4-symbol strings in a text column and runs col LIKE '%<64-byte literal>%'. After a major-version upgrade the same scans run about 2.7 times slower. Nothing in EXPLAIN shows the change, and no setting turns it off.

In message #13, v5 still reaches a steady 2.7x regression on a 4-symbol alphabet at large input sizes. Message #12 reports up to 1.8x on repetitive LATIN1 data, and the author claims no bound. Message #11 shows that each fallback trades one regression class for another. The thresholds behind these numbers (6 bytes for single-byte, 4 for UTF-8) come from AArch64 SQL runs only. A committer would likely ask who decided this trade is acceptable for users who never asked for the change.

**Question:** What do x86-64 SQL-level numbers show for v6 on low-alphabet data, and how would an affected user get the old behaviour back?

<sub>fixable, level 2, performance on the hot path; messages [#5](https://www.postgresql.org/message-id/CABXr29E0Pr2T27Wv%2BwS_jpiRtXy8KyadaJV7Udz4iP1m1-J%3DBw%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CAEah3%3DNW-_M7D_LqOUi4vi-S7LRFiv7c3p8WwuyyiLVOAy77cQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAEah3%3DPrc-BGCLYf9uhJi40ddhC-uU3zBAp35D6z3D-S_zdjbg%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CABXr29Hhwg3UkMa_yyZ029qLF2RDsjW3ud141tT4fi_SToNQNw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Const-only restriction may leave out most real contains-search queries

An application sends `where name like $1` through JDBC, an ORM or PL/pgSQL with the search term bound as a parameter. Since v5, that pattern takes the existing matcher on the first call and never uses BMH. The user upgrades, sees no speedup, and still carries the low-alphabet regression risk wherever literals are inlined.

Message #12 limits BMH to Const patterns, after message #4 found stale-pattern bugs with Params, so the parameterized case no longer benefits. All demand evidence is synthetic: generated 1M-row tables and pg_attribute loops (message #1), plus matcher matrices (messages #5, #9, #13), with no field report or user request. Users with slow contains-searches on large tables can already use a pg_trgm index, which this patch does not change. The thread gives no numbers on how much of the stated benefit survives the Const-only rule.

**Question:** Which real workloads send `LIKE '%literal%'` with a literal Const of at least 4 or 6 bytes over sequential scans, and how much time do they spend in the matcher?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAEah3%3DOZ2NQ5cZZ_DHKhDHVYQb4jxRXZhBmhaayhVVknJUEwrA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/024125bd-de8a-49d7-9523-3a22131aa3bd%40gmail.com), [#12](https://www.postgresql.org/message-id/CAEah3%3DPrc-BGCLYf9uhJi40ddhC-uU3zBAp35D6z3D-S_zdjbg%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CABXr29Hhwg3UkMa_yyZ029qLF2RDsjW3ud141tT4fi_SToNQNw%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. What do SQL-level x86-64 benchmarks of v6 show for low-alphabet and repetitive input, compared with the existing matcher?
2. What is the worst-case slowdown of v6 against the existing matcher, and which inputs produce it?
3. How would a user whose data hits the 2.7x regression get the old matcher back?
4. Which real workloads run LIKE '%literal%' with a Const literal of at least 4 or 6 bytes over sequential scans, and how much time do they spend in the matcher?
5. What did the earlier BMH/LIKE threads linked in message #5 conclude, and how does v6 answer them?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch adds like_bmh.c with measured length thresholds and a second BMH search loop beside the one in varlena.c.
- **performance on the hot path:** textlike and namelike gain a dispatch step on every call, and BMH is slower on low-alphabet and repetitive input.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Const-only restriction may leave out most real contains-search queries

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the LIKE matcher in utils/adt and its encoding and collation rules. No committer has replied in the visible thread.
- Priority: fix design first; audience a sizeable minority; review cost medium. The query shape is common and reviewers engaged, but v5 still shows a 2.7x regression with no stated bound and no opt-out. The patch is about 23 KiB within utils/adt, but judging it needs benchmarks across encodings and architectures.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
