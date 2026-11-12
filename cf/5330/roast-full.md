# Design pre-review: Logging plan of the currently running query

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5330](https://commitfest.postgresql.org/patch/5330/) (Needs review); authors: Atsushi Torikoshi (atsushitorikoshi)
- Thread: [Re: RFC: Logging plan of the running query](https://www.postgresql.org/message-id/flat/d68c3ae31672664876b22d2dcbb526d2%40postgrespro.ru), started 2021-11-02
- Messages read: 197 of 197
- Patch prose read: v13-0002-log-running-query-plan.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the node-wrapping design, which Robert Haas accepted in the thread. The remaining points are fixable. maintenance-fit-2 asks for overhead numbers that reviewers requested and the thread never shows. standards-compat-1, maintenance-fit-1 and replication-ondisk-1 concern the 0002 request-ID patch, which no reviewer has answered yet.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check that re-arming ExecProcNodeFirst is safe for every node type and for cursors. It also could not check how the 0002 request slot behaves under concurrent callers.

## The proposal, restated

**Problem.** A DBA cannot see the plan of a long-running query until it finishes, and production systems usually do not allow a debugger. The patch targets people investigating slow queries in production.

**Proposed design.** pg_log_query_plan(pid) sends a signal to a client backend. At the next CHECK_FOR_INTERRUPTS the target re-arms ExecProcNodeFirst on every node of its current plan tree, and the first node entered then writes a plain EXPLAIN (no ANALYZE) of the innermost running query to the server log. A v61-0002 patch adds request IDs kept in shared memory per backend slot, returned to the caller and printed in the log line.

**User-visible surface.**
- SQL function pg_log_query_plan(pid integer), superuser-only by default and grantable; returns boolean in 0001 and a request ID in 0002
- New LOG_SERVER_ONLY messages that carry the query text, queryid, plan, and in 0002 the request ID
- New ExplainState field es->running, new explain_running.c, and an exported ExplainStringAssemble() that auto_explain also uses
- New injection point log-query-interrupt and a TAP test, 014_pg_log_query_plan.pl

**What the author claims.**
- Users need to see plans of long-running production queries without waiting for them to finish or using a debugger.
- Logging from a node entry point is safe, while running EXPLAIN directly inside the interrupt handler is not.
- Backends that never receive the request pay almost no cost, because the extra check only runs after the interrupt arrives.
- Repeated calls every 0.1s during make installcheck caused no crashes, and the TAP test passed 300 runs.
- The scope is kept narrow on purpose, and progress tracking, EXPLAIN options and ANALYZE are left to later patches.

**Thread state.** Robert Haas and Andrei Lepikhov are the active reviewers, and Haas has accepted the node-wrapping design. The last message posts v61 with the request-ID 0002 patch and asks for opinions, and no reply is visible.

## 2. Prior attempts

### interrogate other backends for plans of the queries they are running (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/352/](https://commitfest.postgresql.org/patch/352/)
- **Date:** decided 2015-12-24; started 2015-08-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P1 also asked another backend for the plan of its running query, but the corpus records no reason for its return. This proposal logs from the next node entry after CHECK_FOR_INTERRUPTS, a design the reviewers in this thread accepted.

### Allow auto_explain to log plans before queries are executed (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2480/](https://commitfest.postgresql.org/patch/2480/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20200227023518.7331aa1470a57b450dc11c14@sraoss.co.jp](https://www.postgresql.org/message-id/flat/20200227023518.7331aa1470a57b450dc11c14@sraoss.co.jp)
- **Date:** decided 2020-04-08; started 2020-02-26
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Julien Rouhaud (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers discussed the behavior but did not agree on a design. Julien Rouhaud said the final plan is still needed when log_analyze, log_buffers or log_timing are on, and suggested a cost threshold for logging before execution. The thread moved to Waiting on Author, made no further progress, and was returned with feedback.
- **Reason (quote):** "There was some discussion but no clear consensus on what should really be done." ([message](https://www.postgresql.org/message-id/CAOBaU_bfnj9QU1eqWrJCAfDo%2BPcpZg0cNfwVfvt6wfpD5Ngdpw%40mail.gmail.com))
- **What is different this time:** P2 added an auto_explain option that logged every plan before execution and stalled on its interaction with ANALYZE-style options. This proposal works on demand for one backend, uses default EXPLAIN options and adds no GUC.


## 3. Objections a reviewer is likely to raise

### 1. No overhead numbers for the per-query bookkeeping every backend pays

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-2`

An OLTP site running short pgbench-style queries upgrades. Every ExecutorRun now saves and restores a QueryDesc pointer in TransactionStateData and clears LogQueryPlanPending, and every node's first call passes an extra test. Any regression shows up only after release.

The claim of near-zero cost covers only the interrupt-armed path (message #118). The queryDesc save and restore in standard_ExecutorRun and the abort-time resets run for all queries (messages #152, #159). Reviewers asked for time and memory benchmarks in messages #1 and #129, and the author agreed in message #139. The thread shows no numbers.

Thread: [#1 (Ekaterina Sokolova, 2021-11-02)](https://www.postgresql.org/message-id/d68c3ae31672664876b22d2dcbb526d2%40postgrespro.ru), [#118 (Robert Haas, 2024-03-04)](https://www.postgresql.org/message-id/CA%2BTgmoZVYG%2BOt%3DyPg3HAr2H%3D2fPvQOyahrx5qwaRf5cqkM7PAQ%40mail.gmail.com), [#129 (Sadeq Dousti, 2025-03-31)](https://www.postgresql.org/message-id/CADE6Lvi1yJYF8KgK9ofin50bRKxByqSHufxshj0pLC7dfoZD9g%40mail.gmail.com), [#139 (torikoshia, 2025-04-03)](https://www.postgresql.org/message-id/e04b16ddfad83bab3632617e8cda9b42%40oss.nttdata.com), [#152 (Robert Haas, 2025-09-09)](https://www.postgresql.org/message-id/CA%2BTgmoYBLNSNGSumcDZB4ymVfy-aHyFa_i4n9MHRpPOLp5m%2BtQ%40mail.gmail.com), [#159 (torikoshia, 2025-10-01)](https://www.postgresql.org/message-id/7c7ca3bc345b28fa4b903bd49fdce519%40oss.nttdata.com)

**Question:** What do pgbench -S and a nested PL/pgSQL workload measure with and without 0001, with the feature never invoked?

Counter-check: held. The added work per ExecutorRun is small, but it runs for every query, and reviewers asked for numbers in messages #1 and #129. The author agreed in message #139 and the thread shows no measurement.

### 2. Return type differs between 0001 and 0002, and from sibling pg_log_* functions

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, backpatchability, catalog changes · `standards-compat-1`

0001 is committed for one release with a boolean result, and 0002 lands in a later release with an integer result. A monitoring script that runs `select ... where pg_log_query_plan(pid)` then fails after upgrade with a type error. A tool that treats pg_log_* functions alike also has to special-case this one.

The brief says 0001 returns boolean and 0002 returns a request ID. Message #197 posts them as separately reviewable steps. pg_log_backend_memory_contexts() and the pg_cancel_backend() family return boolean and report failure as false plus a WARNING, as message #138 shows. The proposal does not say what 0002 returns when the target is not a client backend, and no reviewer has accepted the new return type.

Thread: [#138 (torikoshia, 2025-04-03)](https://www.postgresql.org/message-id/8de51267059a43f53848c69a2bd9bc3b%40oss.nttdata.com), [#196 (torikoshia, 2026-09-25)](https://www.postgresql.org/message-id/86e6450e0ef731242f5b526bf61b34bc%40oss.nttdata.com), [#197 (torikoshia, 2026-10-19)](https://www.postgresql.org/message-id/6e213901e12bbd2c8e0856250369f728%40oss.nttdata.com)

**Question:** Will 0001 and 0002 be committed together so the return type is set once, and what does 0002 return when no request is queued?

Counter-check: held. v61 posts 0002 as a separately reviewable step that changes the result from boolean to a request ID, and no reviewer has accepted that. The thread does not say what 0002 returns when the target is not a client backend.

### 3. Request-ID shared memory in 0002 sets a protocol no reviewer has accepted

fixable · level 2, rubric item 6 · design, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

Two monitoring tools call pg_log_query_plan() for the same PID within a few milliseconds, and the target has only one request slot. One request ID is overwritten or rejected, so that tool waits for a log line that never appears. Users find out when the automation built on the ID stalls.

Message #197 adds one shared-memory slot per backend slot and a cluster-wide counter. It also changes the return type from boolean to a request ID. The author presents the slot as the future carrier for EXPLAIN options (messages #196, #197), so the layout and return type become a long-term commitment. No reviewer has replied to v61, and pg_log_backend_memory_contexts() still returns boolean, so the two sibling functions would differ.

Thread: [#194 (torikoshia, 2026-09-08)](https://www.postgresql.org/message-id/d9b5a15b2c996673da908704853005e6%40oss.nttdata.com), [#195 (Andrei Lepikhov, 2026-09-18)](https://www.postgresql.org/message-id/9a72f9d4-4afe-40d1-8208-7262d437e73d%40gmail.com), [#196 (torikoshia, 2026-09-25)](https://www.postgresql.org/message-id/86e6450e0ef731242f5b526bf61b34bc%40oss.nttdata.com), [#197 (torikoshia, 2026-10-19)](https://www.postgresql.org/message-id/6e213901e12bbd2c8e0856250369f728%40oss.nttdata.com)

**Question:** What happens when two callers target the same backend before it serves the first request, and why is a per-backend slot the right shape for later options?

Counter-check: held. Message #197 describes one request slot per backend slot, with a state and a spinlock, but does not say what a second caller gets while a request is pending. No reviewer has replied to v61, so the question is open.

### 4. Request IDs restart after a server restart and repeat in the log

fixable · level 2, rubric item 14 · design, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A monitoring tool calls pg_log_query_plan() and gets ID 3, then scans the server log for "request ID 3". The server restarted earlier that day, and the same log file holds an older entry with ID 3. The tool picks the wrong plan and nothing tells it so.

Message #197 says 0002 creates request IDs from a cluster-wide counter in shared memory. Shared memory is reset at startup and after crash recovery, so the counter starts again from the beginning while the log files are kept. Each hot standby also has its own counter, so a central log collector sees the same ID from several nodes. Matching each call to its log record was the stated purpose of 0002 (messages #193 to #195), and that matching is unreliable unless the ID is unique.

Thread: [#193 (Andrei Lepikhov, 2026-08-31)](https://www.postgresql.org/message-id/4b22a159-a624-4d3b-b304-1dbd974caf79%40gmail.com), [#194 (torikoshia, 2026-09-08)](https://www.postgresql.org/message-id/d9b5a15b2c996673da908704853005e6%40oss.nttdata.com), [#195 (Andrei Lepikhov, 2026-09-18)](https://www.postgresql.org/message-id/9a72f9d4-4afe-40d1-8208-7262d437e73d%40gmail.com), [#197 (torikoshia, 2026-10-19)](https://www.postgresql.org/message-id/6e213901e12bbd2c8e0856250369f728%40oss.nttdata.com)

**Question:** How does an automation tool match a returned request ID to exactly one log record across restarts, crash recovery and standbys?

Counter-check: held. Message #197 keeps the counter in shared memory allocated at server startup, so IDs repeat after a restart, and the proposal does not state their scope. The log line also carries the PID, so a documented scope may be enough; the standby point adds little.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds a pg_proc entry for pg_log_query_plan(), and 0002 changes its return type. |
| ABI / extension API | **yes** | The patch adds the ExplainState field es->running and exports ExplainStringAssemble(), which auto_explain also uses. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Return type differs between 0001 and 0002, and from sibling pg_log_* functions |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The function is superuser-only by default and grantable, and it writes query text and plans of other sessions to the server log. |
| backpatchability | **yes** | see objection standards-compat-1: Return type differs between 0001 and 0002, and from sibling pg_log_* functions |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds explain_running.c, re-arming of ExecProcNodeFirst on every plan node, and in 0002 shared-memory request slots. |
| performance on the hot path | **yes** | Every ExecutorRun saves and restores a QueryDesc pointer and clears LogQueryPlanPending, even when nobody calls the function. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What do pgbench -S and a nested PL/pgSQL workload measure with and without 0001 when pg_log_query_plan() is never called?
2. Will 0001 and 0002 be committed together so the return type is set once, and what does 0002 return when the target is not a client backend?
3. What does a second caller get when it targets a backend whose request slot still holds an unserved request?
4. Within what scope is a request ID unique, and how does a tool match one log record after a server restart resets the counter?
5. Does clearing LogQueryPlanPending at the end of ExecutorRun work for cursors and the extended query protocol, where one query spans several ExecutorRun calls?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The owner must know executor node dispatch and interrupt handling. Robert Haas reviews in the thread and accepted the node-wrapping design.
- **Liability:** The owner signs for the re-arming of ExecProcNodeFirst across all plan node types and for the per-query QueryDesc tracking, for as long as the executor keeps this shape. With 0002, the owner also signs for the shared-memory request slots and the return type.
- **Author follow-up:** The author has kept the entry alive through 11 CommitFests, rebased it repeatedly and posted v61, which suggests the author stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** DBAs who investigate slow queries in production have asked for this since 2021, and a committer accepted the core design. The patch touches the executor, procsignal and EXPLAIN code, so review needs care, and 0002 still needs a first answer.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Request ID and queryid in the log do not uniquely identify a request. Refuted (not specific): Duplicate of replication-ondisk-1 for the repeated request IDs. The queryid 0 part matches existing practice: auto_explain and log_line_prefix also show no query ID when compute_query_id is off.
- `replication-ondisk-2` Return type of pg_log_query_plan() changes between 0001 and 0002. Refuted (not specific): Duplicate of standards-compat-1, which covers the same return type change and also asks what 0002 returns on failure.
- `maintenance-fit-3` A third copy of EXPLAIN option handling to keep in sync. Refuted (no failure scenario): Since v56 the patch uses the NewExplainState() defaults and shares ExplainStringAssemble() with auto_explain, so it adds no new option list. The drift is in existing auto_explain code, which message #134 says is not this patch's fault, and 0003 is not yet proposed.

### Review angles

- prior-attempts: 0 objection(s). No precedent reason applies. P1 has no recorded reason. P2 stalled on design choices for an auto_explain option, which this proposal does not use. The earlier designs abandoned inside this thread are settled: shm_mq messaging (messages #1, #3), EXPLAIN inside the interrupt handler (message #146), and the need for prior configuration (messages #131, #142).
- standards-compat: 2 objection(s). Other contract points were settled in the thread: the PID in the message stays for consistency (#158), the function name was kept (#176), and EXPLAIN options use the defaults (#174).
- replication-ondisk: 2 objection(s). The proposal adds no on-disk format, WAL record or wire protocol change. Its shared state is limited to the 0002 shared-memory request slots and the new pg_proc entry.
- security: 0 objection(s). No security objection that would survive review. pg_log_query_plan() is superuser-only by default and needs an explicit GRANT, and output goes only to the server log at LOG_SERVER_ONLY. Message #173 and message #174 tied the default to who can read the log. That keeps the plan, query text and queryid visible only to roles that can already read the log. A grantee cannot read other users' plans through the function itself. Repeated calls only fill the log and burn CPU in the target backend, and the superuser accepts that risk when granting. The 0002 request-ID slots use a fixed amount of shared memory per backend slot, so an unprivileged user cannot make them grow. Not checked without the code: whether a grantee can overwrite or reuse another caller's request slot in 0002, and whether custom-plan constants or RLS quals in the logged plan go beyond what log_statement settings allow.
- maintenance-fit: 3 objection(s). The core node-wrapping design has a qualified owner. Robert Haas reviewed it repeatedly and accepted it, Andres Freund called it acceptable (message #123), and the tree walk now uses planstate_tree_walker (message #190).
- already-exists: 0 objection(s). No existing way does this job. auto_explain logs a plan only after the query finishes, and the log_before_query idea [P2] was returned without a design. pg_query_state needs a patched core for custom signals (message #146). A debugger is often not allowed in production. A smaller hook-only design would still need the same core work to re-arm ExecProcNodeFirst and track the current QueryDesc (messages #181, #185).
- demand: 0 objection(s). The thread shows demand that has lasted since 2021: the pg_query_state extension, a parallel progressive EXPLAIN effort, a PGConf.EU talk, and several independent reviewers (messages #129, #137, #145, #170). The demand for the 0002 request IDs rests on one reviewer's automation case (message #193), but its cost is small and falls on the feature.

### Precedents retrieved

- P1: [interrogate other backends for plans of the queries they are running](https://commitfest.postgresql.org/patch/352/) (cf, Returned with feedback, 2015-12-24) - used in the review
- P2: [Allow auto_explain to log plans before queries are executed](https://commitfest.postgresql.org/patch/2480/) (cf, Returned with feedback, 2020-04-08) - used in the review; reason summarised from its thread
- P3: [Exclude partition constaint checking in query execution plan for partitioned table queries](https://commitfest.postgresql.org/patch/1264/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P4: [query logging of prepared statements](https://commitfest.postgresql.org/patch/2002/) (cf, Withdrawn, 2019-04-22) - judged not relevant; reason summarised from its thread
- P5: [More stable query plans via more predictable column statistics](https://commitfest.postgresql.org/patch/434/) (cf, Returned with feedback, 2016-02-03) - judged not relevant; reason summarised from its thread
- P6: [File content logging during execution of COPY queries](https://commitfest.postgresql.org/patch/828/) (cf, Rejected, 2016-12-02) - judged not relevant; reason summarised from its thread
- P7: [CLUSTER command progress monitor](https://commitfest.postgresql.org/patch/1268/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P8: [VACUUM Progress Checker](https://commitfest.postgresql.org/patch/349/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P9: [ANALYZE command progress checker](https://commitfest.postgresql.org/patch/1053/) (cf, Returned with feedback, 2017-04-05) - judged not relevant; reason summarised from its thread
- P10: [ANALYZE command progress checker](https://commitfest.postgresql.org/patch/1104/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P11: [Progress reporting for COPY command](https://commitfest.postgresql.org/patch/2607/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P12: [Log query parameters for terminated execute](https://commitfest.postgresql.org/patch/1693/) (cf, Rejected, 2018-07-30) - judged not relevant
- P13: [logical streaming for large in-progress transactions](https://commitfest.postgresql.org/patch/1429/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P14: [Display individual query in pg_stat_activity](https://commitfest.postgresql.org/patch/2656/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P15: [Optional message to signalled process in pg_terminate_backend()/pg_cancel_backend()](https://commitfest.postgresql.org/patch/1174/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P16: [Addition of authenticated ID to pg_stat_activity](https://commitfest.postgresql.org/patch/3095/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P17: [Revert "Distinguish printf-like functions that support %m from those that don't."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=46b5e7c4b5befbf6ac86d827a3a58f1f02c7338e) (revert, Reverted, 2018-08-12) - judged not relevant
- P18: [Revert my bad decision of about a year ago to make PortalDefineQuery responsible for copying the query string into the new Portal.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1591fcbec77f6544a0f665758bae912c68e6fa42) (revert, Reverted, 2008-04-02) - judged not relevant
- P19: [Allow queries in WHEN expression of FOR EACH STATEMENT triggers](https://commitfest.postgresql.org/patch/2648/) (cf, Withdrawn, 2021-01-13) - judged not relevant
- P20: [Waits monitoring](https://commitfest.postgresql.org/patch/297/) (cf, Returned with feedback, 2015-12-24) - judged not relevant
- P21: [Show temp table namespace xid in pg_stat_activity](https://commitfest.postgresql.org/patch/2016/) (cf, Returned with feedback, 2019-04-09) - judged not relevant
- P22: [Make executor's Range Table an array instead of a List](https://commitfest.postgresql.org/patch/1771/) (cf, Returned with feedback, 2018-08-23) - judged not relevant
- P23: [Allow queries in WHEN expression of FOR EACH STATEMENT triggers](https://commitfest.postgresql.org/patch/3148/) (cf, Withdrawn, 2021-10-03) - judged not relevant
- P24: [Extend comments on signal handling and cross-reference comments on signal handling functions](https://commitfest.postgresql.org/patch/2951/) (cf, Returned with feedback, 2021-09-27) - judged not relevant

### Warnings

- 106 long messages in the middle of the thread were left out of the prompts

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 149 s; cost reported by the CLI: $7.30
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 85 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:31:48.770Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
