# Design pre-review: Bypassing cursors in postgres_fdw to enable parallel plans

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6233](https://commitfest.postgresql.org/patch/6233/); [thread](https://www.postgresql.org/message-id/flat/CA%2BFpmFcmO5ctjYgQxSomJC%3DmCugqPo%2B51Le2wdxX0kWxjvBBig%40mail.gmail.com), 58 of 58 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has no surviving fundamental objection, but the buffering design in v19 fails at level 2. Message #58 shows that a drain runs type input functions under another statement's user and search_path (security-1), and that a failed outer portal leaves the drain state pointing at freed memory (replication-ondisk-1). The reviewing committer asked for a redesign, and no patch or reply for it exists yet. Both points look solvable, so this is a design question and the idea stands.

Not checked: The redesign with raw PGresult spools exists only as a sketch in message #58, so its cleanup rules and memory or disk use could not be judged. Only the first 3 of 58 messages and the patch prose were read; no code was read.

## Prior attempts

- [PARALLEL flag for cursors](https://www.postgresql.org/message-id/flat/98760523-a0d2-8705-38e3-c4602ecf2448@2ndquadrant.com) (similar idea; Returned with feedback, 2018-02-07). reason not recorded Different now: P11 tried to let cursors themselves run parallel plans in core. This proposal leaves cursors alone and avoids them in postgres_fdw through libpq chunked-rows mode.

## Objections a reviewer is likely to raise

### 1. Drain runs type input functions under another statement's security context

Alice scans a foreign table whose column is a domain with a SQL-language check. Mid-scan she calls Bob's SECURITY DEFINER function, which uses the same foreign server and drains Alice's scan. Alice's domain checks then run as Bob with Bob's search_path, and an error is attributed to Bob's statement.

The drain converts remote rows to tuples, so domain input can run arbitrary SQL at a point chosen by whichever statement next needs the connection. Message #58 shows this with a reentrancy test and calls it not OK from a security perspective. The v19 design keeps this behaviour, and no reply or fix is visible. Message #58 says the remedy replaces the tuplestore with raw PGresult spooling, so the change is to the buffering design itself.

**Question:** How does the redesign make sure that every type input and domain check runs only when its own scan returns the row, under that scan's user and search_path?

<sub>fixable, level 2, security / privileges; messages [#58](https://www.postgresql.org/message-id/CA%2BTgmobeqkqL0KdgDM57q41ZeuKB%3DCZt%3DNo3yfDNRTt9V-9HAw%40mail.gmail.com); `security-1`</sub>

### 2. Drained rows spilled to disk lack a defined owner across subtransaction abort

A user fetches from a cursor opened at an outer level, a subtransaction fails, and AtSubAbort_Portals frees the resource owner that owns the drain's temp file. A later scan on the same connection drains into that freed resource owner, or reads a freed active_scan pointer. The backend errors out or crashes.

In message #58, Robert Haas found that the scan's resource owner and the connection's active_scan pointer can outlive the memory they point to after an outer portal fails. His proposal of spool IDs, a transaction-lifespan hash table and per-spool ActivePortal tracking is only a sketch. No reply or patch shows when each spool and its temp files are created and removed on subtransaction abort, portal failure, rescan or connection close. As of v19, the drain state therefore has no proven cleanup point.

**Question:** For each drain spool and its temp files, which resource owner or callback removes it on subtransaction abort, outer-portal failure, rescan and connection loss, and what does a later drain do when the spool is gone?

<sub>fixable, level 2, design; messages [#56](https://www.postgresql.org/message-id/CA%2BTgmobcAQ6XZA3yjipTFxepVi-xfMLFz6c8BjqYO6kMN6Byhw%40mail.gmail.com), [#58](https://www.postgresql.org/message-id/CA%2BTgmobeqkqL0KdgDM57q41ZeuKB%3DCZt%3DNo3yfDNRTt9V-9HAw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 3. Every query-sending path must remember to drain the active scan

A later patch adds a new place in postgres_fdw that sends SQL on a cached connection, such as a new callback, a new transaction-control command or an analyze helper, without calling the drain function. A user with streaming_fetch on runs a nested scan through that path. The query fails or returns wrong rows, and nothing catches it unless a test happens to cover that path with an overlapping scan.

The patch adds a second "connection is busy" mechanism next to pendingAreq, and each one needs its own handling at every site that sends a query. Review found missed sites several times: execute_foreign_modify, fetch_more_data_begin, GetConnection and the ANALYZE helpers in message #47, then RELEASE SAVEPOINT in pgfdw_subxact_callback in messages #50 and #56. Making pgfdw_exec_query drain by itself (message #48) covers only the callers that go through it. Message #47 suggested unifying with process_pending_request, and the material does not show that this was done.

**Question:** Can the async pendingAreq handling and the streaming drain share one path, so that a missed call fails loudly in every build?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#47](https://www.postgresql.org/message-id/CA%2BTgmoYh8Mm0WkQq4b26Mt2bFNOGFiG3DR6dC9puhGcM7DQVYw%40mail.gmail.com), [#48](https://www.postgresql.org/message-id/CA%2BFpmFe-suHRoOZPcPmyDO4jSVTNbFPGRAbwEe7KmpG3ofj4kQ%40mail.gmail.com), [#50](https://www.postgresql.org/message-id/CA%2BTgmoa-FXH9jZaxm039eWf8%3DnhnoBfDS%3DoeKfm50sbGEAZ58g%40mail.gmail.com), [#56](https://www.postgresql.org/message-id/CA%2BTgmobcAQ6XZA3yjipTFxepVi-xfMLFz6c8BjqYO6kMN6Byhw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. streaming_fetch silently overrides an explicit async_capable setting

A DBA sets async_capable 'true' on a server for Append over partitioned foreign tables, then sets streaming_fetch 'true' on one table. Async Append stops for that table with no notice or error. The DBA finds out only when queries slow down and EXPLAIN shows no Async Foreign Scan.

The commit message and doc hunk say async execution "is disabled automatically" when streaming_fetch is on. Postgres options usually either take effect or are rejected at validation, not quietly overridden by a sibling option. Message #20 made this choice to avoid touching a GUC, but no message weighs a reject-at-validation or warn-on-conflict rule. Table and server levels make it worse, because the two options can be set on different objects by different people.

**Question:** Why should a conflicting async_capable setting be ignored silently, instead of rejected or reported when the options are set or the plan is built?

<sub>fixable, level 2, design; messages [#20](https://www.postgresql.org/message-id/CA%2BFpmFeowaWiXUYAGxDsoXcDHQWL9me%2BRxPtc%2Bhz92NU%2BfCYpg%40mail.gmail.com); `standards-compat-1`</sub>

### 5. Benefit shown only on synthetic tests, with mixed results

A user turns on streaming_fetch for a large foreign scan, expecting remote parallelism to help. The scan runs slower than the cursor path, as in message #12 and message #14. Meanwhile postgres_fdw maintainers must keep the drain logic correct in every place that sends a query on a connection.

The gain rests on one count(*) test of about 1 million rows (message #11), and the only independent measurement found the mode 20-30% slower on 20 million rows (messages #12 and #14). No field report or production workload is cited. The only outside user is one endpoint without cursor support (message #52), which is a different need from remote parallelism. The cost is a 95 KiB patch whose drain logic touches transaction callbacks, rescans and every query-sending path (messages #47, #50 and #58), and that cost falls on everyone who maintains postgres_fdw.

**Question:** Which real workloads, with measured numbers, run faster with streaming_fetch, and why was the 20 million row scan slower?

<sub>fixable, level 1b, demand; messages [#11](https://www.postgresql.org/message-id/CA%2BFpmFfKtaRHcZ%3DtVQPRt0qt7xMGCotgbWZ9H0fpLoKA8%3DZJjA%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAEV8nunoatq-bGLo%3DmD%3DWNAFers%2Bf4hqnxN-7gw-M%3DCg9qOqqA%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAEV8nunSKsoWm-mpAvDMdwzy1V3_BrMSBJxprPjZ%2B%2B76qV8cRw%40mail.gmail.com), [#47](https://www.postgresql.org/message-id/CA%2BTgmoYh8Mm0WkQq4b26Mt2bFNOGFiG3DR6dC9puhGcM7DQVYw%40mail.gmail.com), [#50](https://www.postgresql.org/message-id/CA%2BTgmoa-FXH9jZaxm039eWf8%3DnhnoBfDS%3DoeKfm50sbGEAZ58g%40mail.gmail.com), [#52](https://www.postgresql.org/message-id/CAGECzQSTwxPsLrT9OUO_i8BUv958UeaTL9yHodb%3DDthbzZgQHA%40mail.gmail.com), [#58](https://www.postgresql.org/message-id/CA%2BTgmobeqkqL0KdgDM57q41ZeuKB%3DCZt%3DNo3yfDNRTt9V-9HAw%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. How does the redesign make every type input function and domain check run only when its own scan returns the row, under that scan's user and search_path?
2. Which resource owner or callback removes each drain spool and its temp files on subtransaction abort, outer-portal failure, rescan and connection loss?
3. Can the streaming drain and the async pendingAreq handling share one path, so that a query-sending site that skips the drain fails in every build?
4. Why does streaming_fetch silently disable a conflicting async_capable setting instead of rejecting or reporting the conflict?
5. Which workloads run faster with streaming_fetch, with measured numbers, and why was the 20 million row scan in messages #12 and #14 slower?

## Touch points

- **planner behaviour change:** streaming_fetch disables async execution for the scan and lets the remote server choose a parallel plan.
- **psql UX conventions:** see objection standards-compat-1: streaming_fetch silently overrides an explicit async_capable setting
- **security / privileges:** A drain runs type input and domain checks under the user and search_path of another statement (message #58).
- **maintenance burden ("who maintains this in 10 years"):** A second fetch path in postgres_fdw needs a drain at every place that sends a query on a cached connection.
- **performance on the hot path:** With the option on, early stops and rescans must read and discard the rest of the remote result; the default path is unchanged.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Benefit shown only on synthetic tests, with mixed results

## Who would commit it, and when

- Owner: qualified committer engaged. Robert Haas has reviewed every version through v19 and outlined the redesign in message #58. The owner must know postgres_fdw connection handling, resource owners and portal cleanup in subtransactions.
- Priority: fix design first; audience a sizeable minority; review cost high. Users with large foreign scans gain remote parallelism, and a committer is engaged. The 95 KiB patch needs a new buffering design before more review makes sense, because v19 has an open security problem and an open resource owner problem.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
