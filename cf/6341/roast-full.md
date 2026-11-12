# Design pre-review: Cleaning up PREPARE query strings?

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6341](https://commitfest.postgresql.org/patch/6341/) (Needs review); authors: Julien Rouhaud (rjuju)
- Thread: [Cleaning up PREPARE query strings?](https://www.postgresql.org/message-id/flat/aUwKEWGge5jWtaRX%40jrouhaud), started 2025-12-24
- Messages read: 19 of 19
- Patch prose read: v3-0001-Cleanup-explicit-PREPARE-query-strings.patch, v1-0001-Cleanup-explicit-PREPARE-query-strings.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Level 1b fails on demand-1 and already-exists-1. The thread shows one workload and no memory numbers, and it does not say why separate PREPAREs or pipelined Parse messages cannot serve it. Tom Lane raised this in message #2 and the demand side has no answer. The level 2 objections survive but are not evaluated here. Only 3 of 19 messages were read in full.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Level 2 was not evaluated. Without the code, I could not judge whether v4's split between stored text and parse locations can be made consistent on replan, or whether the query offset idea works.

## The proposal, restated

**Problem.** An explicit PREPARE inside a multi-statement query string stores the whole string, once per prepared statement. Applications that send all their PREPARE statements in one string at connection start waste memory this way. pg_prepared_statements also shows the whole string, so users must parse it again to find each query.

**Proposed design.** PREPARE uses CleanQuerytext(), as pg_stat_statements does, to cut the stored query string down to the statement's own text, based on the locations saved during parsing. Only explicit PREPARE changes, because doing it in CreateCachedPlan for all callers, including the extended protocol, caused too many issues. Sami Imseih's v4 keeps passing the original string to parse analysis so parse-time error cursors stay correct.

**User-visible surface.**
- pg_prepared_statements.statement shows only the PREPARE statement's own text, without the other statements, leading and trailing whitespace, or the final semicolon.
- A comment placed before the PREPARE keyword is dropped from the stored query text.
- Extensions that read the cached plan's query string, such as pg_stat_statements and auto_explain, get the shorter text.

**What the author claims.**
- Memory use drops for multi-statement strings that hold PREPARE, and the saving grows with the number of prepared statements.
- Overhead is minimal because the patch reuses parse locations and works only when space can be saved.
- An executor-level query offset could fix execution-time error positions without updating every location field in the parse tree.
- Comments for tools such as pg_hint_plan still work if placed after PREPARE ... AS.

**Thread state.** Tom Lane doubted it is worth it, since the change breaks error position fields, and he has not replied since; Sami Imseih reviewed and posted v4. Shinya Kato hit an assertion failure in queryjumblefuncs.c with v4 and pg_stat_statements after alter table and EXECUTE, which is still open, and CFbot reports a needed rebase.

## 2. Prior attempts

### Revert support for improved tracking of nested queries (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555)
- **Thread:** [https://www.postgresql.org/message-id/flat/18947-cdd2668beffe02bf@postgresql.org](https://www.postgresql.org/message-id/flat/18947-cdd2668beffe02bf@postgresql.org), [https://www.postgresql.org/message-id/flat/CAO6_XqqM6S9bQ2qd=75W+yKATwoazxSNhv5sjW06fjGAtHbTUA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAO6_XqqM6S9bQ2qd=75W+yKATwoazxSNhv5sjW06fjGAtHbTUA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/844a3b38-bbf1-4fb2-9fd6-f58c35c09917@pgbackrest.org](https://www.postgresql.org/message-id/flat/844a3b38-bbf1-4fb2-9fd6-f58c35c09917@pgbackrest.org)
- **Date:** decided 2025-06-12; started 2024-10-22
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This commit reverts the two following commits: - 499edb09741b, track more precisely query locations for nested statements. - 06450c7b8c70, a follow-up fix of 499edb09741b with query locations. The test introduced in this commit is not reverted. This is proving useful to track a problem that only pgaudit was able to detect. These prove to have issues with the tracking of SELECT statements, when these use multiple parenthesis which is something supported by the grammar. Incorrect location and lengths are causing pg_stat_statements to become confused, failing its job in query normalization with potential out-of-bound writes because the location and the length may not match with what can be handled. A lot of the query patterns discussed when this issue was reported have no test coverage in the main regression test suite, or the recovery test 027_stream_regress.pl would have caught the problems as pg_stat_statements is loaded by the node running the regression tests. A first step would be to improve the test coverage to stress more the query normalization logic. A different portion of this work was done in 45e0ba30fc40, with the addition of tests for nested queries. These can be left in the tree. They are useful to track the way inner queries are currently tracked by PGSS with non-top-level entries, and will be useful when reconsidering in the future the work reverted here.
- **Reason (quote):** "Incorrect location and lengths are causing pg_stat_statements to become confused, failing its job in query normalization with potential out-of-bound writes because the location and the length may not match with what can be handled."
- **What is different this time:** P13 changed how the parser records locations for nested statements. This proposal keeps parser locations and shortens the stored string in PREPARE, which can produce the same kind of mismatch between locations and text.


## 3. Objections a reviewer is likely to raise

### 1. No measured memory saving and no users beyond one workload

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer takes on a change that makes stored query text differ from the text that parse locations refer to. A few releases later an extension or a new executor error cursor reads the shorter string with offsets from the original one, and points at the wrong place or trips an assertion. The memory saving that justified the change was never measured.

The only use case is the author's own application, which sends all its PREPAREs in one string at connection start (message #1). The thread gives no memory numbers, and the cached plan and query trees are likely much larger than the duplicated text. Tom Lane's reply, "if that bothers you, don't do it that way" (message #2), was not answered on the demand side. Separate PREPAREs, or protocol-level Parse messages in a pipeline, avoid the duplication with no extra round trips.

Thread: [#1 (Julien Rouhaud, 2025-12-24)](https://www.postgresql.org/message-id/aUwKEWGge5jWtaRX%40jrouhaud), [#2 (Tom Lane, 2025-12-24)](https://www.postgresql.org/message-id/928501.1766593260%40sss.pgh.pa.us), [#3 (Sami Imseih, 2025-12-24)](https://www.postgresql.org/message-id/CAA5RZ0uco6Xxsgo6p8FakgRJjY3PhnjePWWmZWxJO2Sz6Xf%3DCA%40mail.gmail.com), [#19 (solai v, 2026-08-06)](https://www.postgresql.org/message-id/CAF0whucxZ_BzeHROewBa%3DfhFpx53_Eveb5Tnzk0k3ns5n95xXw%40mail.gmail.com)

**Question:** How much memory per session does this save for a realistic set of prepared statements, and which other users or tools report the problem that pipelined Parse messages do not already solve?

Counter-check: held. The thread has no memory numbers and one reported workload, and message #2 questions whether the gain is worth the cost. Sami Imseih's support and the readability gain in pg_prepared_statements help, but they do not show how common the need is.

### 2. Pipelined protocol-level Parse already avoids round trips with per-statement text

fixable · level 1b, rubric item 2 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An application prepares its fixed set of statements at connection start. It can send one Parse message per statement in a single pipeline, for example with libpq pipeline mode and PQsendPrepare. Each statement then stores only its own text, so the patch saves memory only for clients that pack SQL-level PREPARE into one simple-query string.

Message #1 gives one reason for the multi-statement string: avoiding extra round trips. Protocol pipelining already does that, and a Parse message accepts only one statement, so pg_prepared_statements shows clean text on that path. Message #2 gives the same advice in short form: "don't do it that way". The thread does not explain why pipelining or separate statements cannot serve this application, while the patch adds parse-location handling that has already crashed with pg_stat_statements in messages #5 and #18.

Thread: [#1 (Julien Rouhaud, 2025-12-24)](https://www.postgresql.org/message-id/aUwKEWGge5jWtaRX%40jrouhaud), [#2 (Tom Lane, 2025-12-24)](https://www.postgresql.org/message-id/928501.1766593260%40sss.pgh.pa.us), [#5 (Sami Imseih, 2026-01-13)](https://www.postgresql.org/message-id/CAA5RZ0sswD2h2w8FkVjToA%3D3oNYUb1mW-zuCD%3DJ6DDK2zuLuZg%40mail.gmail.com), [#18 (Shinya Kato, 2026-07-15)](https://www.postgresql.org/message-id/CAOzEurTYoinojY7a7GfJut3XsuR%3D26ZuF87SYzKV7RFcKMbvWA%40mail.gmail.com)

**Question:** Which applications or drivers cannot use pipelined Parse messages or separate PREPARE statements, and how much memory does the current behaviour cost them?

Counter-check: held. The thread gives round trips as the only reason for the multi-statement string, and it does not say why pipelined Parse messages or separate PREPAREs cannot serve. Tom Lane's remark in message #2 points the same way and has no answer.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Extensions that read the cached plan's query string, such as pg_stat_statements and auto_explain, get shorter text with changed locations. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The assertion failure in message #18 needs no special privilege when pg_stat_statements is loaded. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Stored text and parse locations must stay consistent on every re-analysis path and for future error-position work. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Separate PREPAREs or pipelined Parse messages on the client side already store per-statement text. |

## 5. Questions the author must answer before anyone reviews code

1. How much session memory does the patch save for a realistic set of prepared statements sent in one string?
2. Which applications or drivers cannot use pipelined Parse messages or separate PREPARE statements to avoid the extra round trips?
3. How do stmt_location and stmt_len stay consistent with the stored query string on replan after DDL, the case that fails in message #18?
4. How does an execution-time error cursor stay correct for a cleaned PREPARE string, and which test reaches executor_errposition()?
5. Can the stored text keep a comment placed before PREPARE, so that applications that tag statements need not change their SQL?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Tom Lane replied once in message #2 with doubt and has not returned. The owner would need to know the plan cache, parse locations and query jumbling.
- **Liability:** The owner signs for the match between stored query text and location fields on every replan path, for all extensions that read them, with no end date.
- **Author follow-up:** The author moved the entry across CommitFests up to 2026-10-01, but the assertion failure in message #18 has no reply and the patch needs a rebase.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: Memory numbers per session for a realistic batched-PREPARE workload, reports from other users or drivers that send PREPARE this way, and a reason why pipelined Parse messages or separate PREPAREs do not serve them.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The gain goes to clients that pack many SQL-level PREPAREs into one string, and the thread shows one such workload. The patch is small, but v4 still fails an assertion on replan and the error-position question from message #2 is open.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Leading comments on PREPARE disappear from stored text without opt-in. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Cached query string and saved statement locations can disagree on replan. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Planned execution-time error positions would need a second coordinate system. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Rewriting statement locations repeats the failure that caused a revert. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same replan mismatch with more detail on why v4 still fails.
- `standards-compat-2` Statement text also changes for single-statement PREPARE. Refuted (wrong facts): Message #19 used psql, which removes the final semicolon before sending, so it does not show the server stripping it. For a single statement CleanQuerytext() trims only leading and trailing whitespace, which leaves no concrete failure.
- `security-1` Stored query text no longer matches parse locations, crash reachable by any user. Refuted (not specific): Duplicate of replication-ondisk-1. The point that no special privilege is needed adds no separate defect beyond the same replan crash.
- `maintenance-fit-1` Cached query string and parse-tree locations stop matching after replan. Refuted (not specific): Duplicate of replication-ondisk-1. The claim that nothing fails loudly is also weak, because the assertion in queryjumblefuncs.c did fire.
- `maintenance-fit-3` Only explicit PREPARE changes, so query_string meaning depends on caller. Refuted (no failure scenario): Callers already differ today: SPI and the extended protocol store single-statement text, and consumers use stmt_location and stmt_len against whatever string they get. The only failure is a mismatch between those fields and the text, which replication-ondisk-1 covers.
- `maintenance-fit-4` Memory saving is shown for one pattern only, with no numbers. Refuted (not specific): Duplicate of demand-1, which makes the same point about missing memory numbers and a single reported workload.

### Review angles

- prior-attempts: 1 objection(s). [P12] and [P3] cover nearby areas: per-command copying of the whole query string into portals, and a memory limit GUC. Their recorded reasons do not apply here, and [P12] if anything supports the problem statement.
- standards-compat: 2 objection(s). Error-cursor correctness and the open assertion failure belong to other angles. From a compatibility view, only the two text-contract changes above apply.
- replication-ondisk: 1 objection(s). The proposal changes only backend-local, in-memory prepared statement text. It adds no on-disk format, WAL, catalog, replication or wire protocol change, so those parts of this angle have nothing to object to.
- security: 1 objection(s). The text shown in pg_prepared_statements gets shorter and stays visible only to the owning session, so it adds no new data exposure. It may even hide unrelated statements, such as ones with passwords, that the old full string exposed.
- maintenance-fit: 4 objection(s). Objections found. No visible committer supports the patch; Tom Lane doubted it in message #2 and has been silent since.
- already-exists: 1 objection(s). One objection from this angle. No extension or contrib module trims the stored PREPARE text, so the only existing alternative is a change on the client side.
- demand: 1 objection(s). The objection is "not now, show demand". It is not a claim that the problem does not exist.

### Precedents retrieved

- P1: [cursors with prepared statements](https://commitfest.postgresql.org/patch/1667/) (cf, Returned with feedback, 2018-12-01) - judged not relevant; reason summarised from its thread
- P2: [query logging of prepared statements](https://commitfest.postgresql.org/patch/2002/) (cf, Withdrawn, 2019-04-22) - judged not relevant; reason summarised from its thread
- P3: [New GUC prepared_statement_limit to limit memory used by prepared statements](https://commitfest.postgresql.org/patch/2241/) (cf, Returned with feedback, 2019-11-28) - used in the review; reason summarised from its thread
- P4: [Support prepared statement invalidation when result or argument types change](https://commitfest.postgresql.org/patch/4518/) (cf, Withdrawn, 2024-07-25) - judged not relevant; reason summarised from its thread
- P5: [Implicit prepare of statements: replace literals with parameters and store cached plans](https://commitfest.postgresql.org/patch/1150/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P6: [Adding column "mem_usage" to view pg_prepared_statements](https://commitfest.postgresql.org/patch/2223/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P7: [Report the query string that caused a memory error under Valgrind](https://commitfest.postgresql.org/patch/4156/) (cf, Withdrawn, 2023-01-31) - judged not relevant; reason summarised from its thread
- P8: [The Prepare statement with the skewed parameter uses the custom plan](https://commitfest.postgresql.org/patch/4839/) (cf, Withdrawn, 2024-09-22) - judged not relevant; reason summarised from its thread
- P9: [Enforce typmod for function inputs, function results and parameters for spi_prepare'd statements called from PLs](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Functions) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P10: [Revert "Cope with NULL query string in ExecInitParallelPlan()."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=83efce7a1ebc5bae79617ddba12a64790141725c) (revert, Reverted, 2021-04-15) - judged not relevant
- P11: [Revert current_query() change to use debug_query_string again; add comment.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1c175cd76a9db7d6114139a28f838697e88306ba) (revert, Reverted, 2009-01-08) - judged not relevant
- P12: [Revert my bad decision of about a year ago to make PortalDefineQuery responsible for copying the query string into the new Portal.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1591fcbec77f6544a0f665758bae912c68e6fa42) (revert, Reverted, 2008-04-02) - used in the review
- P13: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - used in the review
- P14: [\[PATCH\] Add UPDATE WHERE OFFSET IN clause](https://commitfest.postgresql.org/patch/3531/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P15: [OFFSET optimisation for IndexScan using visibility map](https://commitfest.postgresql.org/patch/1513/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P16: [Optionally record Plan IDs to track plan changes for a query](https://commitfest.postgresql.org/patch/5486/) (cf, Returned with feedback, 2025-03-24) - judged not relevant
- P17: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P18: [Optimize planner memory consumption for huge arrays](https://commitfest.postgresql.org/patch/4825/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P19: [Allow queries in WHEN expression of FOR EACH STATEMENT triggers](https://commitfest.postgresql.org/patch/2648/) (cf, Withdrawn, 2021-01-13) - judged not relevant
- P20: [loop over unbound cursor](https://commitfest.postgresql.org/patch/2558/) (cf, Withdrawn, 2020-08-10) - judged not relevant
- P21: [Allow queries in WHEN expression of FOR EACH STATEMENT triggers](https://commitfest.postgresql.org/patch/3148/) (cf, Withdrawn, 2021-10-03) - judged not relevant
- P22: [Allow TIMESTAMP WITH TIME ZONE to store the original timezone information, either zone name or offset from UTC](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Dates_and_Times) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P23: [PARALLEL flag for cursors](https://commitfest.postgresql.org/patch/1362/) (cf, Returned with feedback, 2018-02-07) - judged not relevant
- P24: [Modify the planner to better estimate caching effects](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 20; wall time: 192 s; cost reported by the CLI: $2.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 104 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:49:11.086Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
