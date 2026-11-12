# Design pre-review: Bypassing cursors in postgres_fdw to enable parallel plans

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6233](https://commitfest.postgresql.org/patch/6233/) (Needs review); authors: Rafia Sabih (rafia.sabih)
- Thread: [Bypassing cursors in postgres_fdw to enable parallel plans](https://www.postgresql.org/message-id/flat/CA%2BFpmFcmO5ctjYgQxSomJC%3DmCugqPo%2B51Le2wdxX0kWxjvBBig%40mail.gmail.com), started 2025-01-06
- Messages read: 58 of 58
- Patch prose read: v19-0001-postgres_fdw-Rename-cursor_exists-flag-to-scan_i.patch, v19-0002-postgres_fdw-Add-streaming_fetch-option-for-curs.patch, 0001-Add-a-fetch-mechanism-without-cursors.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has no surviving fundamental objection, but the buffering design in v19 fails at level 2. Message #58 shows that a drain runs type input functions under another statement's user and search_path (security-1), and that a failed outer portal leaves the drain state pointing at freed memory (replication-ondisk-1). The reviewing committer asked for a redesign, and no patch or reply for it exists yet. Both points look solvable, so this is a design question and the idea stands.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The redesign with raw PGresult spools exists only as a sketch in message #58, so its cleanup rules and memory or disk use could not be judged. Only the first 3 of 58 messages and the patch prose were read; no code was read.

## The proposal, restated

**Problem.** postgres_fdw reads remote rows through a cursor, so the remote server never uses a parallel plan for the scan. Users with large foreign scans lose remote parallelism. A later reply adds that remote endpoints without cursor support, such as a DuckDB-based one, cannot serve postgres_fdw today.

**Proposed design.** A new boolean option streaming_fetch, set per server or per table, runs the remote query with libpq chunked-rows mode and uses fetch_size as the chunk size. A connection carries one stream at a time, so when another scan or statement needs the connection, postgres_fdw drains the active scan into a work_mem-bounded tuplestore and replays it later. Scans that fetch ctid or lock rows still use a cursor, and async execution is disabled in this mode.

**User-visible surface.**
- New server and foreign table option streaming_fetch (boolean, default false); the table setting overrides the server setting.
- EXPLAIN VERBOSE shows "Streaming Fetch: true" for scans that use the mode.
- async_capable is disabled automatically when streaming_fetch is enabled.
- New error when a scan reads from a tuplestore that lost rows during an interrupted drain.

**What the author claims.**
- Remote parallel plans become possible and can speed up large scans a lot.
- A count(*) over about 1 million rows took about 25 ms with the mode versus 63 ms with a cursor.
- Nested or interleaved scans on one connection stay correct without opening a second connection.
- For queries that read few rows, early stops and rescans can make the mode slower than cursors.

**Thread state.** Robert Haas has reviewed every version through v19. On 2026-11-09 he asked for a redesign, because type input functions run during a drain under another statement's security context, and a failed outer portal can leave dangling resource owner pointers. No reply to that review is visible.

## 2. Prior attempts

### PARALLEL flag for cursors (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1362/](https://commitfest.postgresql.org/patch/1362/)
- **Thread:** [https://www.postgresql.org/message-id/flat/98760523-a0d2-8705-38e3-c4602ecf2448@2ndquadrant.com](https://www.postgresql.org/message-id/flat/98760523-a0d2-8705-38e3-c4602ecf2448@2ndquadrant.com)
- **Date:** decided 2018-02-07; started 2017-10-14
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P11 tried to let cursors themselves run parallel plans in core. This proposal leaves cursors alone and avoids them in postgres_fdw through libpq chunked-rows mode.


## 3. Objections a reviewer is likely to raise

### 1. Drain runs type input functions under another statement's security context

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 7 · security / privileges · `security-1`

Alice scans a foreign table whose column is a domain with a SQL-language check. Mid-scan she calls Bob's SECURITY DEFINER function, which uses the same foreign server and drains Alice's scan. Alice's domain checks then run as Bob with Bob's search_path, and an error is attributed to Bob's statement.

The drain converts remote rows to tuples, so domain input can run arbitrary SQL at a point chosen by whichever statement next needs the connection. Message #58 shows this with a reentrancy test and calls it not OK from a security perspective. The v19 design keeps this behaviour, and no reply or fix is visible. Message #58 says the remedy replaces the tuplestore with raw PGresult spooling, so the change is to the buffering design itself.

Thread: [#58 (Robert Haas, 2026-11-09)](https://www.postgresql.org/message-id/CA%2BTgmobeqkqL0KdgDM57q41ZeuKB%3DCZt%3DNo3yfDNRTt9V-9HAw%40mail.gmail.com)

**Question:** How does the redesign make sure that every type input and domain check runs only when its own scan returns the row, under that scan's user and search_path?

Counter-check: held. Message #58 confirms that v19 runs type input functions during a drain under another statement's user and search_path, and no reply is visible. The same message outlines a redesign that delays type conversion, so a new patch version could solve it.

### 2. Drained rows spilled to disk lack a defined owner across subtransaction abort

fixable · level 2, rubric item 14 · design, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A user fetches from a cursor opened at an outer level, a subtransaction fails, and AtSubAbort_Portals frees the resource owner that owns the drain's temp file. A later scan on the same connection drains into that freed resource owner, or reads a freed active_scan pointer. The backend errors out or crashes.

In message #58, Robert Haas found that the scan's resource owner and the connection's active_scan pointer can outlive the memory they point to after an outer portal fails. His proposal of spool IDs, a transaction-lifespan hash table and per-spool ActivePortal tracking is only a sketch. No reply or patch shows when each spool and its temp files are created and removed on subtransaction abort, portal failure, rescan or connection close. As of v19, the drain state therefore has no proven cleanup point.

Thread: [#56 (Robert Haas, 2026-10-09)](https://www.postgresql.org/message-id/CA%2BTgmobcAQ6XZA3yjipTFxepVi-xfMLFz6c8BjqYO6kMN6Byhw%40mail.gmail.com), [#58 (Robert Haas, 2026-11-09)](https://www.postgresql.org/message-id/CA%2BTgmobeqkqL0KdgDM57q41ZeuKB%3DCZt%3DNo3yfDNRTt9V-9HAw%40mail.gmail.com)

**Question:** For each drain spool and its temp files, which resource owner or callback removes it on subtransaction abort, outer-portal failure, rescan and connection loss, and what does a later drain do when the spool is gone?

Counter-check: held. Message #58 shows that a failed outer portal leaves active_scan and its resource owner pointing at freed memory in v19. The spool-ID remedy is a sketch with no patch or reply, so the cleanup points are not yet defined.

### 3. Every query-sending path must remember to drain the active scan

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a new place in postgres_fdw that sends SQL on a cached connection, such as a new callback, a new transaction-control command or an analyze helper, without calling the drain function. A user with streaming_fetch on runs a nested scan through that path. The query fails or returns wrong rows, and nothing catches it unless a test happens to cover that path with an overlapping scan.

The patch adds a second "connection is busy" mechanism next to pendingAreq, and each one needs its own handling at every site that sends a query. Review found missed sites several times: execute_foreign_modify, fetch_more_data_begin, GetConnection and the ANALYZE helpers in message #47, then RELEASE SAVEPOINT in pgfdw_subxact_callback in messages #50 and #56. Making pgfdw_exec_query drain by itself (message #48) covers only the callers that go through it. Message #47 suggested unifying with process_pending_request, and the material does not show that this was done.

Thread: [#47 (Robert Haas, 2026-08-11)](https://www.postgresql.org/message-id/CA%2BTgmoYh8Mm0WkQq4b26Mt2bFNOGFiG3DR6dC9puhGcM7DQVYw%40mail.gmail.com), [#48 (Rafia Sabih, 2026-08-21)](https://www.postgresql.org/message-id/CA%2BFpmFe-suHRoOZPcPmyDO4jSVTNbFPGRAbwEe7KmpG3ofj4kQ%40mail.gmail.com), [#50 (Robert Haas, 2026-08-27)](https://www.postgresql.org/message-id/CA%2BTgmoa-FXH9jZaxm039eWf8%3DnhnoBfDS%3DoeKfm50sbGEAZ58g%40mail.gmail.com), [#56 (Robert Haas, 2026-10-09)](https://www.postgresql.org/message-id/CA%2BTgmobcAQ6XZA3yjipTFxepVi-xfMLFz6c8BjqYO6kMN6Byhw%40mail.gmail.com)

**Question:** Can the async pendingAreq handling and the streaming drain share one path, so that a missed call fails loudly in every build?

Counter-check: held. Review found missed drain sites in messages #47, #50 and #56. Making pgfdw_exec_query drain by itself covers only its callers, and the thread does not show the suggested unification with process_pending_request.

### 4. streaming_fetch silently overrides an explicit async_capable setting

fixable · level 2, rubric item 12 · design, psql UX conventions, planner behaviour change · `standards-compat-1`

A DBA sets async_capable 'true' on a server for Append over partitioned foreign tables, then sets streaming_fetch 'true' on one table. Async Append stops for that table with no notice or error. The DBA finds out only when queries slow down and EXPLAIN shows no Async Foreign Scan.

The commit message and doc hunk say async execution "is disabled automatically" when streaming_fetch is on. Postgres options usually either take effect or are rejected at validation, not quietly overridden by a sibling option. Message #20 made this choice to avoid touching a GUC, but no message weighs a reject-at-validation or warn-on-conflict rule. Table and server levels make it worse, because the two options can be set on different objects by different people.

Thread: [#20 (Rafia Sabih, 2026-04-02)](https://www.postgresql.org/message-id/CA%2BFpmFeowaWiXUYAGxDsoXcDHQWL9me%2BRxPtc%2Bhz92NU%2BfCYpg%40mail.gmail.com)

**Question:** Why should a conflicting async_capable setting be ignored silently, instead of rejected or reported when the options are set or the plan is built?

Counter-check: held. The doc hunk states the override, but the thread never weighs rejecting or reporting the conflict. Because the two options can sit on different objects, a user can lose async Append without any message.

### 5. Benefit shown only on synthetic tests, with mixed results

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A user turns on streaming_fetch for a large foreign scan, expecting remote parallelism to help. The scan runs slower than the cursor path, as in message #12 and message #14. Meanwhile postgres_fdw maintainers must keep the drain logic correct in every place that sends a query on a connection.

The gain rests on one count(*) test of about 1 million rows (message #11), and the only independent measurement found the mode 20-30% slower on 20 million rows (messages #12 and #14). No field report or production workload is cited. The only outside user is one endpoint without cursor support (message #52), which is a different need from remote parallelism. The cost is a 95 KiB patch whose drain logic touches transaction callbacks, rescans and every query-sending path (messages #47, #50 and #58), and that cost falls on everyone who maintains postgres_fdw.

Thread: [#11 (Rafia Sabih, 2025-11-14)](https://www.postgresql.org/message-id/CA%2BFpmFfKtaRHcZ%3DtVQPRt0qt7xMGCotgbWZ9H0fpLoKA8%3DZJjA%40mail.gmail.com), [#12 (KENAN YILMAZ, 2025-11-25)](https://www.postgresql.org/message-id/CAEV8nunoatq-bGLo%3DmD%3DWNAFers%2Bf4hqnxN-7gw-M%3DCg9qOqqA%40mail.gmail.com), [#14 (KENAN YILMAZ, 2025-11-28)](https://www.postgresql.org/message-id/CAEV8nunSKsoWm-mpAvDMdwzy1V3_BrMSBJxprPjZ%2B%2B76qV8cRw%40mail.gmail.com), [#47 (Robert Haas, 2026-08-11)](https://www.postgresql.org/message-id/CA%2BTgmoYh8Mm0WkQq4b26Mt2bFNOGFiG3DR6dC9puhGcM7DQVYw%40mail.gmail.com), [#50 (Robert Haas, 2026-08-27)](https://www.postgresql.org/message-id/CA%2BTgmoa-FXH9jZaxm039eWf8%3DnhnoBfDS%3DoeKfm50sbGEAZ58g%40mail.gmail.com), [#52 (Jelte Fennema-Nio, 2026-09-02)](https://www.postgresql.org/message-id/CAGECzQSTwxPsLrT9OUO_i8BUv958UeaTL9yHodb%3DDthbzZgQHA%40mail.gmail.com), [#58 (Robert Haas, 2026-11-09)](https://www.postgresql.org/message-id/CA%2BTgmobeqkqL0KdgDM57q41ZeuKB%3DCZt%3DNo3yfDNRTt9V-9HAw%40mail.gmail.com)

**Question:** Which real workloads, with measured numbers, run faster with streaming_fetch, and why was the 20 million row scan slower?

Counter-check: held. Message #10 asked for best-case and worst-case measurements. The thread has one count(*) gain and an unexplained slower result on 20 million rows, though that test returned zero rows on a cold cache and so says little about the mode in general.

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
| planner behaviour change | **yes** | streaming_fetch disables async execution for the scan and lets the remote server choose a parallel plan. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | see objection standards-compat-1: streaming_fetch silently overrides an explicit async_capable setting |
| security / privileges | **yes** | A drain runs type input and domain checks under the user and search_path of another statement (message #58). |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second fetch path in postgres_fdw needs a drain at every place that sends a query on a cached connection. |
| performance on the hot path | **yes** | With the option on, early stops and rescans must read and discard the rest of the remote result; the default path is unchanged. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Benefit shown only on synthetic tests, with mixed results |

## 5. Questions the author must answer before anyone reviews code

1. How does the redesign make every type input function and domain check run only when its own scan returns the row, under that scan's user and search_path?
2. Which resource owner or callback removes each drain spool and its temp files on subtransaction abort, outer-portal failure, rescan and connection loss?
3. Can the streaming drain and the async pendingAreq handling share one path, so that a query-sending site that skips the drain fails in every build?
4. Why does streaming_fetch silently disable a conflicting async_capable setting instead of rejecting or reporting the conflict?
5. Which workloads run faster with streaming_fetch, with measured numbers, and why was the 20 million row scan in messages #12 and #14 slower?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Robert Haas has reviewed every version through v19 and outlined the redesign in message #58. The owner must know postgres_fdw connection handling, resource owners and portal cleanup in subtransactions.
- **Liability:** A second fetch path in postgres_fdw, kept for as long as the option exists. The owner signs for drain correctness at every query-sending site, spill file cleanup, and the security context of delayed type conversion.
- **Author follow-up:** The author posted new versions through v19 over almost two years. No reply to the redesign request in message #58 is visible, but that message is three days old.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Users with large foreign scans gain remote parallelism, and a committer is engaged. The 95 KiB patch needs a new buffering design before more review makes sense, because v19 has an open security problem and an open resource owner problem.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` The option does not guarantee cursor-free access for non-Postgres endpoints. Refuted (no failure scenario): The proposal states remote parallelism as its goal and documents the cursor fallback for ctid and row-locking scans. Cursor-free access for non-Postgres endpoints is a third-party remark in message #52, and postgres_fdw makes no promise about such endpoints.
- `standards-compat-3` A per-table switch for a choice that depends on query shape. Refuted (already handled): Server and table options are how postgres_fdw already exposes fetch_size and async_capable, and the table setting overrides the server setting. The default is false, the docs state the few-rows cost, and EXPLAIN VERBOSE shows which scans use the mode.
- `security-2` Failed outer portal leaves active scan with freed resource owner. Refuted (not specific): Duplicate of replication-ondisk-1, which covers the same freed resource owner and dangling active_scan pointer and also asks for the cleanup points.
- `maintenance-fit-2` Pending redesign adds feature-specific spool machinery with one engaged reviewer. Refuted (not specific): A committer who knows the area has reviewed every version and proposed the redesign, which the rubric counts as lowering rejection confidence. The remaining points duplicate replication-ondisk-1, security-1 and demand-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent's recorded reason applies. P3 died for starting remote execution in ExecInitNode, and nothing shows this proposal does that. P1 was set aside for a parallel-worker solution that this proposal builds on. P11 records no reason, and the thread cites no earlier discussion. The open problems are the type-input security context and the failed-portal resource owners from message #58, which belong to other angles.
- standards-compat: 3 objection(s). pg_dump round-trip and EXPLAIN property format were checked: generic FDW options dump like existing ones, and the EXPLAIN format was raised and changed in message #39.
- replication-ondisk: 1 objection(s). The option is a generic FDW option, so it needs no catalog change, on-disk format change, WAL change or protocol change, and pg_dump already handles it. The only persistent state is the temp files written when a drain spills to disk, and the one objection covers their cleanup.
- security: 2 objection(s). Other aspects need no new privilege: the option is set by the owner of the server or table, and the remote query runs under the existing user mapping.
- maintenance-fit: 2 objection(s). Non-users pay no clear hot-path cost, since the option defaults to false and Jelte's DuckDB report (message #52) shows real demand. So these objections concern long-run maintenance and are not a reason to reject the goal.
- already-exists: 0 objection(s). No existing feature does this job. A remote cursor rules out a parallel plan on the remote side, a larger fetch_size does not help, and dblink() gives up pushdown, joins and planner integration. Cursor-less endpoints such as the DuckDB-based one in message #52 have no other route. The draining overlap with pendingAreq was raised and handled in message #47.
- demand: 1 objection(s). Remote parallelism through postgres_fdw is a plausible need, and a committer has reviewed every version. The objection asks for evidence of demand, not for rejection.

### Precedents retrieved

- P1: [Asynchronous execution on postgres-fdw](https://commitfest.postgresql.org/patch/291/) (cf, Rejected, 2015-08-10) - used in the review; reason summarised from its thread
- P2: [Push down more UPDATEs/DELETEs in postgres_fdw](https://commitfest.postgresql.org/patch/791/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P3: [Async execution of postgres_fdw.](https://commitfest.postgresql.org/patch/32/) (cf, Rejected, 2015-05-09) - used in the review; reason summarised from its thread
- P4: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P5: [postgres_fdw: evaluate placeholdervars on remote server](https://commitfest.postgresql.org/patch/1055/) (cf, Returned with feedback, 2017-09-27) - judged not relevant; reason summarised from its thread
- P6: [postgres_fdw super user checks](https://commitfest.postgresql.org/patch/824/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P7: [postgres_fdw: support parameterized foreign joins](https://commitfest.postgresql.org/patch/1042/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P8: [Kerberos delegation support in libpq and postgres_fdw](https://commitfest.postgresql.org/patch/3265/) (cf, Returned with feedback, 2021-11-26) - judged not relevant; reason summarised from its thread
- P9: [Trigger failsafe when VACUUM is run in single user mode](https://commitfest.postgresql.org/patch/3467/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P10: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P11: [PARALLEL flag for cursors](https://commitfest.postgresql.org/patch/1362/) (cf, Returned with feedback, 2018-02-07) - used in the review
- P12: [support tid range scan in parallel](https://commitfest.postgresql.org/patch/4982/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P13: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P14: [Allow pageinspect's bt_page_stats function to return a set of rows instead of a single row](https://commitfest.postgresql.org/patch/3713/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P15: [Revert misguided change to postgres_fdw FOR UPDATE/SHARE code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8ec8760fc87ecde0516e511f1c55aec627b01ea7) (revert, Reverted, 2014-12-12) - judged not relevant
- P16: [Revert "Temporarily instrument postgres_fdw test to look for statistics changes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=04e7ecadf64d18f67ca1d0632d8ab71f120ca5e3) (revert, Reverted, 2018-03-08) - judged not relevant
- P17: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P18: [Allow parallel plan for referential integrity checks](https://commitfest.postgresql.org/patch/3539/) (cf, Returned with feedback, 2023-07-04) - judged not relevant
- P19: [Allow parallel plan for referential integrity checks](https://commitfest.postgresql.org/patch/4507/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P20: [Batch (pipelined) mode for libpq](https://commitfest.postgresql.org/patch/634/) (cf, Returned with feedback, 2016-11-22) - judged not relevant
- P21: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1024/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P22: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1317/) (cf, Returned with feedback, 2018-11-29) - judged not relevant
- P23: [Revert changes for SSL compression in libpq](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0ba71107efeeccde9158f47118f95043afdca0bb) (revert, Reverted, 2021-03-10) - judged not relevant
- P24: [Parallel-aware file_fdw](https://commitfest.postgresql.org/patch/2108/) (cf, Returned with feedback, 2019-09-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 163 s; cost reported by the CLI: $5.61
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 92 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:46:36.488Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
