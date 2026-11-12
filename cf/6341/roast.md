# Design pre-review: Cleaning up PREPARE query strings?

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6341](https://commitfest.postgresql.org/patch/6341/); [thread](https://www.postgresql.org/message-id/flat/aUwKEWGge5jWtaRX%40jrouhaud), 19 of 19 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Level 1b fails on demand-1 and already-exists-1. The thread shows one workload and no memory numbers, and it does not say why separate PREPAREs or pipelined Parse messages cannot serve it. Tom Lane raised this in message #2 and the demand side has no answer. The level 2 objections survive but are not evaluated here. Only 3 of 19 messages were read in full.

Not checked: Level 2 was not evaluated. Without the code, I could not judge whether v4's split between stored text and parse locations can be made consistent on replan, or whether the query offset idea works.

## Prior attempts

- [Revert support for improved tracking of nested queries](https://www.postgresql.org/message-id/flat/18947-cdd2668beffe02bf@postgresql.org) (related; Reverted, 2025-06-12). "Incorrect location and lengths are causing pg_stat_statements to become confused, failing its job in query normalization with potential out-of-bound writes because the location and the length may not match with what can be handled." Different now: P13 changed how the parser records locations for nested statements. This proposal keeps parser locations and shortens the stored string in PREPARE, which can produce the same kind of mismatch between locations and text.

## Objections a reviewer is likely to raise

### 1. No measured memory saving and no users beyond one workload

A committer takes on a change that makes stored query text differ from the text that parse locations refer to. A few releases later an extension or a new executor error cursor reads the shorter string with offsets from the original one, and points at the wrong place or trips an assertion. The memory saving that justified the change was never measured.

The only use case is the author's own application, which sends all its PREPAREs in one string at connection start (message #1). The thread gives no memory numbers, and the cached plan and query trees are likely much larger than the duplicated text. Tom Lane's reply, "if that bothers you, don't do it that way" (message #2), was not answered on the demand side. Separate PREPAREs, or protocol-level Parse messages in a pipeline, avoid the duplication with no extra round trips.

**Question:** How much memory per session does this save for a realistic set of prepared statements, and which other users or tools report the problem that pipelined Parse messages do not already solve?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/aUwKEWGge5jWtaRX%40jrouhaud), [#2](https://www.postgresql.org/message-id/928501.1766593260%40sss.pgh.pa.us), [#3](https://www.postgresql.org/message-id/CAA5RZ0uco6Xxsgo6p8FakgRJjY3PhnjePWWmZWxJO2Sz6Xf%3DCA%40mail.gmail.com), [#19](https://www.postgresql.org/message-id/CAF0whucxZ_BzeHROewBa%3DfhFpx53_Eveb5Tnzk0k3ns5n95xXw%40mail.gmail.com); `demand-1`</sub>

### 2. Pipelined protocol-level Parse already avoids round trips with per-statement text

An application prepares its fixed set of statements at connection start. It can send one Parse message per statement in a single pipeline, for example with libpq pipeline mode and PQsendPrepare. Each statement then stores only its own text, so the patch saves memory only for clients that pack SQL-level PREPARE into one simple-query string.

Message #1 gives one reason for the multi-statement string: avoiding extra round trips. Protocol pipelining already does that, and a Parse message accepts only one statement, so pg_prepared_statements shows clean text on that path. Message #2 gives the same advice in short form: "don't do it that way". The thread does not explain why pipelining or separate statements cannot serve this application, while the patch adds parse-location handling that has already crashed with pg_stat_statements in messages #5 and #18.

**Question:** Which applications or drivers cannot use pipelined Parse messages or separate PREPARE statements, and how much memory does the current behaviour cost them?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/aUwKEWGge5jWtaRX%40jrouhaud), [#2](https://www.postgresql.org/message-id/928501.1766593260%40sss.pgh.pa.us), [#5](https://www.postgresql.org/message-id/CAA5RZ0sswD2h2w8FkVjToA%3D3oNYUb1mW-zuCD%3DJ6DDK2zuLuZg%40mail.gmail.com), [#18](https://www.postgresql.org/message-id/CAOzEurTYoinojY7a7GfJut3XsuR%3D26ZuF87SYzKV7RFcKMbvWA%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. How much session memory does the patch save for a realistic set of prepared statements sent in one string?
2. Which applications or drivers cannot use pipelined Parse messages or separate PREPARE statements to avoid the extra round trips?
3. How do stmt_location and stmt_len stay consistent with the stored query string on replan after DDL, the case that fails in message #18?
4. How does an execution-time error cursor stay correct for a cleaned PREPARE string, and which test reaches executor_errposition()?
5. Can the stored text keep a comment placed before PREPARE, so that applications that tag statements need not change their SQL?

## Touch points

- **ABI / extension API:** Extensions that read the cached plan's query string, such as pg_stat_statements and auto_explain, get shorter text with changed locations.
- **security / privileges:** The assertion failure in message #18 needs no special privilege when pg_stat_statements is loaded.
- **maintenance burden ("who maintains this in 10 years"):** Stored text and parse locations must stay consistent on every re-analysis path and for future error-position work.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Separate PREPAREs or pipelined Parse messages on the client side already store per-statement text.

## Who would commit it, and when

- Owner: silent. Tom Lane replied once in message #2 with doubt and has not returned. The owner would need to know the plan cache, parse locations and query jumbling.
- Not now, show demand first: Memory numbers per session for a realistic batched-PREPARE workload, reports from other users or drivers that send PREPARE this way, and a reason why pipelined Parse messages or separate PREPAREs do not serve them.
- Priority: needs design rethink; audience niche; review cost medium. The gain goes to clients that pack many SQL-level PREPAREs into one string, and the thread shows one such workload. The patch is small, but v4 still fails an assertion on replan and the error-position question from message #2 is open.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
