# Design pre-review: Logging plan of the currently running query

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5330](https://commitfest.postgresql.org/patch/5330/); [thread](https://www.postgresql.org/message-id/flat/d68c3ae31672664876b22d2dcbb526d2%40postgrespro.ru), 197 of 197 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the node-wrapping design, which Robert Haas accepted in the thread. The remaining points are fixable. maintenance-fit-2 asks for overhead numbers that reviewers requested and the thread never shows. standards-compat-1, maintenance-fit-1 and replication-ondisk-1 concern the 0002 request-ID patch, which no reviewer has answered yet.

Not checked: Without the code, this review could not check that re-arming ExecProcNodeFirst is safe for every node type and for cursors. It also could not check how the 0002 request slot behaves under concurrent callers.

## Prior attempts

- [interrogate other backends for plans of the queries they are running](https://commitfest.postgresql.org/patch/352/) (same idea; Returned with feedback, 2015-12-24). reason not recorded Different now: P1 also asked another backend for the plan of its running query, but the corpus records no reason for its return. This proposal logs from the next node entry after CHECK_FOR_INTERRUPTS, a design the reviewers in this thread accepted.
- [Allow auto_explain to log plans before queries are executed](https://www.postgresql.org/message-id/flat/20200227023518.7331aa1470a57b450dc11c14@sraoss.co.jp) (similar idea; Returned with feedback, 2020-04-08). "There was some discussion but no clear consensus on what should really be done." ([message](https://www.postgresql.org/message-id/CAOBaU_bfnj9QU1eqWrJCAfDo%2BPcpZg0cNfwVfvt6wfpD5Ngdpw%40mail.gmail.com)) Different now: P2 added an auto_explain option that logged every plan before execution and stalled on its interaction with ANALYZE-style options. This proposal works on demand for one backend, uses default EXPLAIN options and adds no GUC.

## Objections a reviewer is likely to raise

### 1. No overhead numbers for the per-query bookkeeping every backend pays

An OLTP site running short pgbench-style queries upgrades. Every ExecutorRun now saves and restores a QueryDesc pointer in TransactionStateData and clears LogQueryPlanPending, and every node's first call passes an extra test. Any regression shows up only after release.

The claim of near-zero cost covers only the interrupt-armed path (message #118). The queryDesc save and restore in standard_ExecutorRun and the abort-time resets run for all queries (messages #152, #159). Reviewers asked for time and memory benchmarks in messages #1 and #129, and the author agreed in message #139. The thread shows no numbers.

**Question:** What do pgbench -S and a nested PL/pgSQL workload measure with and without 0001, with the feature never invoked?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/d68c3ae31672664876b22d2dcbb526d2%40postgrespro.ru), [#118](https://www.postgresql.org/message-id/CA%2BTgmoZVYG%2BOt%3DyPg3HAr2H%3D2fPvQOyahrx5qwaRf5cqkM7PAQ%40mail.gmail.com), [#129](https://www.postgresql.org/message-id/CADE6Lvi1yJYF8KgK9ofin50bRKxByqSHufxshj0pLC7dfoZD9g%40mail.gmail.com), [#139](https://www.postgresql.org/message-id/e04b16ddfad83bab3632617e8cda9b42%40oss.nttdata.com), [#152](https://www.postgresql.org/message-id/CA%2BTgmoYBLNSNGSumcDZB4ymVfy-aHyFa_i4n9MHRpPOLp5m%2BtQ%40mail.gmail.com), [#159](https://www.postgresql.org/message-id/7c7ca3bc345b28fa4b903bd49fdce519%40oss.nttdata.com); `maintenance-fit-2`</sub>

### 2. Return type differs between 0001 and 0002, and from sibling pg_log_* functions

0001 is committed for one release with a boolean result, and 0002 lands in a later release with an integer result. A monitoring script that runs `select ... where pg_log_query_plan(pid)` then fails after upgrade with a type error. A tool that treats pg_log_* functions alike also has to special-case this one.

The brief says 0001 returns boolean and 0002 returns a request ID. Message #197 posts them as separately reviewable steps. pg_log_backend_memory_contexts() and the pg_cancel_backend() family return boolean and report failure as false plus a WARNING, as message #138 shows. The proposal does not say what 0002 returns when the target is not a client backend, and no reviewer has accepted the new return type.

**Question:** Will 0001 and 0002 be committed together so the return type is set once, and what does 0002 return when no request is queued?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#138](https://www.postgresql.org/message-id/8de51267059a43f53848c69a2bd9bc3b%40oss.nttdata.com), [#196](https://www.postgresql.org/message-id/86e6450e0ef731242f5b526bf61b34bc%40oss.nttdata.com), [#197](https://www.postgresql.org/message-id/6e213901e12bbd2c8e0856250369f728%40oss.nttdata.com); `standards-compat-1`</sub>

### 3. Request-ID shared memory in 0002 sets a protocol no reviewer has accepted

Two monitoring tools call pg_log_query_plan() for the same PID within a few milliseconds, and the target has only one request slot. One request ID is overwritten or rejected, so that tool waits for a log line that never appears. Users find out when the automation built on the ID stalls.

Message #197 adds one shared-memory slot per backend slot and a cluster-wide counter. It also changes the return type from boolean to a request ID. The author presents the slot as the future carrier for EXPLAIN options (messages #196, #197), so the layout and return type become a long-term commitment. No reviewer has replied to v61, and pg_log_backend_memory_contexts() still returns boolean, so the two sibling functions would differ.

**Question:** What happens when two callers target the same backend before it serves the first request, and why is a per-backend slot the right shape for later options?

<sub>fixable, level 2, design; messages [#194](https://www.postgresql.org/message-id/d9b5a15b2c996673da908704853005e6%40oss.nttdata.com), [#195](https://www.postgresql.org/message-id/9a72f9d4-4afe-40d1-8208-7262d437e73d%40gmail.com), [#196](https://www.postgresql.org/message-id/86e6450e0ef731242f5b526bf61b34bc%40oss.nttdata.com), [#197](https://www.postgresql.org/message-id/6e213901e12bbd2c8e0856250369f728%40oss.nttdata.com); `maintenance-fit-1`</sub>

### 4. Request IDs restart after a server restart and repeat in the log

A monitoring tool calls pg_log_query_plan() and gets ID 3, then scans the server log for "request ID 3". The server restarted earlier that day, and the same log file holds an older entry with ID 3. The tool picks the wrong plan and nothing tells it so.

Message #197 says 0002 creates request IDs from a cluster-wide counter in shared memory. Shared memory is reset at startup and after crash recovery, so the counter starts again from the beginning while the log files are kept. Each hot standby also has its own counter, so a central log collector sees the same ID from several nodes. Matching each call to its log record was the stated purpose of 0002 (messages #193 to #195), and that matching is unreliable unless the ID is unique.

**Question:** How does an automation tool match a returned request ID to exactly one log record across restarts, crash recovery and standbys?

<sub>fixable, level 2, design; messages [#193](https://www.postgresql.org/message-id/4b22a159-a624-4d3b-b304-1dbd974caf79%40gmail.com), [#194](https://www.postgresql.org/message-id/d9b5a15b2c996673da908704853005e6%40oss.nttdata.com), [#195](https://www.postgresql.org/message-id/9a72f9d4-4afe-40d1-8208-7262d437e73d%40gmail.com), [#197](https://www.postgresql.org/message-id/6e213901e12bbd2c8e0856250369f728%40oss.nttdata.com); `replication-ondisk-1`</sub>

## Questions for the author

1. What do pgbench -S and a nested PL/pgSQL workload measure with and without 0001 when pg_log_query_plan() is never called?
2. Will 0001 and 0002 be committed together so the return type is set once, and what does 0002 return when the target is not a client backend?
3. What does a second caller get when it targets a backend whose request slot still holds an unserved request?
4. Within what scope is a request ID unique, and how does a tool match one log record after a server restart resets the counter?
5. Does clearing LogQueryPlanPending at the end of ExecutorRun work for cursors and the extended query protocol, where one query spans several ExecutorRun calls?

## Touch points

- **catalog changes:** The patch adds a pg_proc entry for pg_log_query_plan(), and 0002 changes its return type.
- **ABI / extension API:** The patch adds the ExplainState field es->running and exports ExplainStringAssemble(), which auto_explain also uses.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Return type differs between 0001 and 0002, and from sibling pg_log_* functions
- **security / privileges:** The function is superuser-only by default and grantable, and it writes query text and plans of other sessions to the server log.
- **backpatchability:** see objection standards-compat-1: Return type differs between 0001 and 0002, and from sibling pg_log_* functions
- **maintenance burden ("who maintains this in 10 years"):** The patch adds explain_running.c, re-arming of ExecProcNodeFirst on every plan node, and in 0002 shared-memory request slots.
- **performance on the hot path:** Every ExecutorRun saves and restores a QueryDesc pointer and clears LogQueryPlanPending, even when nobody calls the function.

## Who would commit it, and when

- Owner: qualified committer engaged. The owner must know executor node dispatch and interrupt handling. Robert Haas reviews in the thread and accepted the node-wrapping design.
- Priority: proceed; audience a sizeable minority; review cost high. DBAs who investigate slow queries in production have asked for this since 2021, and a committer accepted the core design. The patch touches the executor, procsignal and EXPLAIN code, so review needs care, and 0002 still needs a first answer.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
