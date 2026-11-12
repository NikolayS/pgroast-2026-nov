# Design pre-review: [PATCH v1 0/7] Wait event timing and tracing instrumentation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6984](https://commitfest.postgresql.org/patch/6984/) (Needs review); authors: Dmitry Fomin (dmitryf)
- Thread: [\[PATCH v1 0/7\] Wait event timing and tracing instrumentation](https://www.postgresql.org/message-id/flat/CAPHG-0mAOn05ae6Kqx1wHXxzOk4E5W7ajjd%3DQBhgkR7a0uyQmw%40mail.gmail.com), started 2026-07-03
- Messages read: 19 of 19
- Patch prose read: v14-0002-Convert-wait_start-end-call-sites-to-the-timed-p.patch, v14-0003-test_wait_hook-test-module-for-the-wait-event-ho.patch, v14-0001-Add-begin-end-hooks-for-timed-wait-events.patch, v14-0004-pg_wait_event_tracing-statistics-level.patch, v14-0005-pg_wait_event_tracing-trace-level.patch, v1-0003-wait_event_timing-expose-overflow-counters-and-ad.patch, v1-0001-wait_event_timing-add-enable-wait-event-timing-fl.patch, v1-0002-wait_event_timing-record-per-backend-wait-event-s.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The series fails at level 1b, so the design objections are not ranked. Messages #13 and #18 ask to settle first whether core observes waits by sampling, at selected sites, or at every transition, and the thread shows no agreement (maintenance-fit-4). Message #19 reproduces the main result with an external tracer on stock PG18, so the shown gain is lower cost and no host access (already-exists-2). The proposal does not say why the collector must be in contrib (already-exists-1).

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: No code was read. The cost with no consumer loaded, the code-size growth at each site, and the safety of the lock-free slot claiming and the seqlock ring could not be judged; the level 2 objections were not evaluated.

## The proposal, restated

**Problem.** Postgres shows wait events in pg_stat_activity only as snapshots, so users cannot see how long backends spend in each wait or the order of a session's waits. Sampling tools such as pg_wait_sampling miss short waits and cannot rebuild one slow request. The target users are DBAs who are diagnosing incidents.

**Proposed design.** The current series (v14) adds wait_event_begin_hook and wait_event_end_hook, called from a new pair pgstat_report_wait_start_timed()/pgstat_report_wait_end_timed(), and converts all 94 core wait sites to the timed pair. A new module, contrib/pg_wait_event_tracing, uses the hooks to keep per-backend statistics (count, total, max, histogram) and a per-session trace ring with query-attribution markers. The first version (v1 to v6) did this in core behind a configure flag.

**User-visible surface.**
- C API: wait_event_begin_hook, wait_event_end_hook, pgstat_report_wait_start_timed(), pgstat_report_wait_end_timed(), and a documented hook contract
- New contrib extension pg_wait_event_tracing, which must be in shared_preload_libraries
- GUC pg_wait_event_tracing.capture (off | stats | trace, SUSET), plus two other module GUCs
- SQL views and functions for statistics, resets, capacity, orphaned-ring cleanup and the trace ring
- pg_wait_event_trace_by_statement(), which groups waits by statement and returns a phase column and <idle>/<no-statement> buckets

**What the author claims.**
- With no consumer loaded, the timed pair adds about 2 ns per wait pair on an already-set latch and no resolved end-to-end TPS difference.
- With capture on, a lock-heavy workload loses about 3.3% TPS and other pgbench workloads lose up to about 1%.
- Exact per-wait timing explained a slow-request tail in a test where pg_wait_sampling pointed to the wrong waits.
- Timing only some wait sites would leave unexplained gaps, so the author converts all sites.
- Guillaume Lelarge and Kirk Wolak say the feature is useful.

**Thread state.** Andres Freund is "dead set against" the hooks, objects to converting every wait site, and prefers a configurable wait event sampler; the author answered but kept all sites converted. Michael Paquier and Andrey Borodin raised cost and observation-contract concerns, and no committer supports the current shape.

## 2. Prior attempts

### Add accumulated statistics for wait event (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2332/](https://commitfest.postgresql.org/patch/2332/)
- **Thread:** [https://www.postgresql.org/message-id/flat/2022793411.102298.1532329482584.JavaMail.jboss@ep1ml402](https://www.postgresql.org/message-id/flat/2022793411.102298.1532329482584.JavaMail.jboss@ep1ml402)
- **Date:** decided 2020-08-02; started 2018-07-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier, Kyotaro Horiguchi, Daniel Gustafsson (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The patch stopped applying to HEAD. The author said there was no time to rebase it in that commitfest, so the entry was returned with feedback. Earlier, Michael Paquier argued that client-side sampling of pg_stat_activity finds bottlenecks at lower cost than server-side counters for all backends. Benchmarks showed up to about 10% overhead in read-only tests, and Kyotaro Horiguchi found frequent hash lookups even with timing off.
- **Reason (quote):** "Since the thread has stalled and there is no updated patch I've marked this entry Returned with Feedback." ([message](https://www.postgresql.org/message-id/25597647-59D2-413F-B7E9-920AF57DD51B%40yesql.se))
- **What is different this time:** P3 kept per-backend wait counts and accumulated times in core with a hash table and a track_wait_timing setting. This series moves collection to a contrib module behind hooks, adds a trace ring with statement attribution, and reports about 2 ns of cost with no consumer loaded.


## 3. Objections a reviewer is likely to raise

### 1. Every-transition observation chosen before the observation contract is agreed

fixable · level 1b, rubric item 6 · desirability, performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-4`

Core commits hooks on every wait transition. Later the project adds the configurable sampler Andres describes, and then has two overlapping wait observation mechanisms. The hooks cannot be removed without breaking extensions that came to depend on them.

Message #13 asks to settle first whether the contract is sampling, selected extended sites, or every transition. Message #18 rejects the premise that wait timing alone is useful and prefers sampling with state context. The overhead figures in message #19 come from v11, and v12 to v14 were not measured again. A hook API offered to extensions is hard to withdraw once it ships.

Thread: [#13 (Andrey Borodin, 2026-09-05)](https://www.postgresql.org/message-id/9F597BE1-53A7-4615-A2D4-60AF0CAE48E6%40yandex-team.ru), [#18 (Andres Freund, 2026-10-28)](https://www.postgresql.org/message-id/2s45loetq2c3y3vyecm5oos36vdr6dv7wlupddjx63owhcgwfi%40ia6jtqqs26kb), [#19 (Dmitry Fomin, 2026-11-02)](https://www.postgresql.org/message-id/CAPHG-0%3Dp4cX9gy5KKX_SicPdE8R_MoeiVAUGU%2B_hu3fXOi_K4A%40mail.gmail.com)

**Question:** Why should core commit permanent every-transition hooks before the project agrees whether a sampler would answer the same incident questions?

Counter-check: held. Message #13 asks to settle the observation contract first, and message #18 prefers a sampler; the thread shows no agreement. Hooks can change between major releases, so the regret is smaller than stated, but the open question stands.

### 2. Exact per-wait timing is already possible on stock builds

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved) · `already-exists-2`

A DBA who can run eBPF on the host already gets exact per-wait durations from my_wait_event_info on an unpatched server, as the author did on stock PG18. The proposal adds a permanent API and 94 converted sites, and its added value is lower cost per transition and use without host access.

Message #19 reproduces the main result, Lock:transactionid dominating slow requests, with an external eBPF tracer on stock PG18. Message #1 gives that tracer's cost as about 200-300 ns per transition. So the gap is overhead and deployment, and is not capability. Users on managed services still depend on the provider loading the module, so the hook does not by itself close the deployment gap.

Thread: [#1 (Dmitry Fomin, 2026-07-03)](https://www.postgresql.org/message-id/CAPHG-0mAOn05ae6Kqx1wHXxzOk4E5W7ajjd%3DQBhgkR7a0uyQmw%40mail.gmail.com), [#13 (Andrey Borodin, 2026-09-05)](https://www.postgresql.org/message-id/9F597BE1-53A7-4615-A2D4-60AF0CAE48E6%40yandex-team.ru), [#19 (Dmitry Fomin, 2026-11-02)](https://www.postgresql.org/message-id/CAPHG-0%3Dp4cX9gy5KKX_SicPdE8R_MoeiVAUGU%2B_hu3fXOi_K4A%40mail.gmail.com)

**Question:** Which incident could not be diagnosed with the existing external tracer plus pg_wait_sampling, track_io_timing and log_lock_waits, and why is the cost difference decisive?

Counter-check: held. Message #19 reproduces the main result with an external eBPF tracer on stock PG18, so the gap is cost and host access. The thread shows two short supporting replies and no field reports from users who cannot use that tracer.

### 3. Collector module belongs out of tree once the hook exists

fixable · level 1b, rubric item 4 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A future patch changes protocol handling, the executor hooks or transaction callbacks. The contrib module's statement-attribution state machine silently misplaces waits, and the committer who touched postgres.c now owns fixing a module they never reviewed. Users find out when pg_wait_event_trace_by_statement() puts lock waits in the wrong bucket.

Patches 0004 and 0005 are about 480 KiB and use only the hook from 0001 plus existing extension hooks. pg_wait_sampling already shows that a wait collector can be maintained out of tree. In message #13, Andrey suggested a small core hook with a separately maintained collector, and the proposal does not say why contrib placement is needed. Message #16 already lists four ambiguous attribution cases.

Thread: [#13 (Andrey Borodin, 2026-09-05)](https://www.postgresql.org/message-id/9F597BE1-53A7-4615-A2D4-60AF0CAE48E6%40yandex-team.ru), [#16 (Dmitry Fomin, 2026-10-23)](https://www.postgresql.org/message-id/CAPHG-0moXufhp9ssiVHE%2B83A7SyTApTxf4zMr4h7X6%2BtEDTrEQ%40mail.gmail.com), [#19 (Dmitry Fomin, 2026-11-02)](https://www.postgresql.org/message-id/CAPHG-0%3Dp4cX9gy5KKX_SicPdE8R_MoeiVAUGU%2B_hu3fXOi_K4A%40mail.gmail.com)

**Question:** What does contrib placement give pg_wait_event_tracing that an out-of-tree extension using the same hook cannot have?

Counter-check: held. Message #13 notes the deployment value of in-tree diagnostics, but the same message proposes a small core hook with a separately maintained collector. The proposal does not say what contrib placement adds over that.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | v14-0002 converts wait sites in xlog.c, walwriter, walsender, walreceiver and standby.c to the hooked pair; no WAL format change. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v14-0001 adds wait_event_begin_hook, wait_event_end_hook and a second public reporting pair with a documented hook contract. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The module adds pg_wait_event_tracing.capture and two other GUCs. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | SUSET capture GUC, cross-backend trace reads by pg_read_all_stats, and reset functions with pg_signal_backend-style checks. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every future wait site must choose between two pairs, and contrib gains a large collector tied to executor and protocol hooks. |
| performance on the hot path | **yes** | All 94 wait sites, including LWLock, spinlock and I/O waits, gain a hook pointer test. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #13 suggests a small core hook with the collector maintained outside the tree; an external eBPF tracer gives exact timing today. |

## 5. Questions the author must answer before anyone reviews code

1. Why should core add hooks on every wait transition before the list agrees whether a configurable sampler answers the same incident questions?
2. Which incident could not be diagnosed with the external eBPF tracer plus pg_wait_sampling, track_io_timing and log_lock_waits, and why is the cost difference decisive?
3. What does contrib placement give pg_wait_event_tracing that an out-of-tree extension using the same hook cannot have?
4. Which individual wait sites justify the timed pair on their own analysis, and does the use case still hold if only those are converted?
5. What are the no-consumer and capture-on overhead numbers when measured on v14 instead of v11?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund and Michael Paquier take part in the thread, but Andres Freund opposes the hooks and no committer supports the current shape. The owner would need to know the wait event, LWLock and I/O paths.
- **Liability:** The owner signs for a hook at every wait site, including sites inside critical sections, with a contract that only documentation enforces. The owner also signs for a large contrib module whose statement attribution depends on executor, protocol and transaction hooks, for as long as it ships.
- **Author follow-up:** The author posted 14 versions in about four months and answered each review, which suggests the author stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The goal has support from DBAs in the thread, but the series converts 94 sites and adds about 480 KiB of contrib patches while a committer who knows the area opposes hooks on all sites. Agreement on sampling, selected sites or every transition must come before code review.

## Appendix

### Held up in the counter-check, not listed above

- `prior-attempts-1` (fixable, level 2) Converts all wait sites, where earlier discussion allowed only selected ones. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-1` (fixable, level 2) A second wait-reporting pair splits one API into two. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) Contrib functions take names from the core pg_stat_ namespace. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-2` (fixable, level 2) Orphaned trace rings stay in DSA until a superuser sweeps them. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Hook contract is unenforced in critical sections and during release. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-3` By-statement attribution is a heuristic exposed as a stable SQL result. Refuted (no failure scenario): The function reads a fixed-size ring that is overwritten, and the documentation lists the four ambiguous cases. Contrib functions change results between extension versions today, and Postgres minor releases do not usually change such rules.
- `replication-ondisk-1` Timed hooks run inside WAL and control file critical sections. Refuted (not specific): Duplicate of maintenance-fit-2. Both describe a third-party hook that breaks the documented contract in a critical section; the per-site question is covered by prior-attempts-1.
- `maintenance-fit-1` Converting all 94 wait sites makes every wait site a hook site. Refuted (not specific): Duplicate of prior-attempts-1 for the conversion of all sites, and of standards-compat-1 for the choice between two pairs. It adds no separate point, and its fatal severity does not hold because converting fewer sites is possible.
- `maintenance-fit-3` Large contrib collector with no committer taking ownership. Refuted (not specific): Duplicate of already-exists-1 for the contrib placement. The missing committer owner belongs in the committability field, and the rubric says it must not raise the verdict.
- `demand-1` Exact per-wait timing already works on stock Postgres via external tracing. Refuted (not specific): Duplicate of already-exists-2. P3 must not be used as support: it was returned because the patch stalled without a rebase and showed about 10% overhead, which says nothing about demand here.
- `demand-2` Cost of converting every wait site falls on all users. Refuted (not specific): Duplicate of prior-attempts-1 for the choice at each new wait site and the code size. For the run-time part, messages #14 and #15 measure about 2 ns per pair with no consumer and no resolved TPS difference.

### Review angles

- prior-attempts: 1 objection(s). P3 was recorded as returned because it stalled, so its reason does not carry over. P20 and P7 share only keywords or the hook mechanism and are not listed.
- standards-compat: 3 objection(s). Three objections from the standards and compatibility angle.
- replication-ondisk: 2 objection(s). The series adds no WAL records, no on-disk format change, no core catalog change and no protocol change. Only the hook placement on durable-state paths and the orphaned-ring lifecycle touch this angle.
- security: 0 objection(s). From the security angle, the proposal has no gap that could support an objection. Capture defaults to off and is SUSET, with delegation through GRANT SET. Cross-backend reads of statistics and trace rings require pg_read_all_stats, which already sees more through pg_stat_activity and pg_stat_statements. The reset functions follow pg_signal_backend rules, and reset-all and the orphaned-ring sweep check superuser() in C. The orphaned-ring memory is bounded by ProcNumbers times ring size, and only a privileged setting can enable it. The hook contract is a safety concern for another angle, and these points need the code to confirm.
- maintenance-fit: 4 objection(s)
- already-exists: 2 objection(s). Two objections from this angle. Both concern where the collector lives and what it adds over existing tools; the hook design and its overhead are left to other angles.
- demand: 2 objection(s). Some demand exists: one consultant tested the patch and wants it, and the author describes real incidents. The material does not show demand beyond a few people or a lack of a workable external tool.

### Precedents retrieved

- P1: [Add new wait event to XactLockTableWait](https://commitfest.postgresql.org/patch/5804/) (cf, Withdrawn, 2026-03-31) - judged not relevant; reason summarised from its thread
- P2: [Fix cleanup of WSA events in WIN32 wait event code](https://commitfest.postgresql.org/patch/904/) (cf, Rejected, 2016-12-27) - judged not relevant
- P3: [Add accumulated statistics for wait event](https://commitfest.postgresql.org/patch/2332/) (cf, Returned with feedback, 2020-08-02) - used in the review; reason summarised from its thread
- P4: [pg_stat_activity: avoid showing state=active with wait_event=ClientRead](https://commitfest.postgresql.org/patch/3760/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P5: [Fix Windows network event handling](https://commitfest.postgresql.org/patch/3523/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P6: [Revert "Display the time when the process started waiting for the lock, in pg_locks."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=890d2182a2c425aaa80f9bf9f7116d31e0c6538e) (revert, Reverted, 2021-02-09) - judged not relevant
- P7: [Xact event callback at command start](https://commitfest.postgresql.org/patch/2860/) (cf, Rejected, 2021-09-01) - used in the review; reason summarised from its thread
- P8: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P9: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P10: [Eagerly evict bulkwrite strategy ring](https://commitfest.postgresql.org/patch/5919/) (cf, Withdrawn, 2026-02-19) - judged not relevant; reason summarised from its thread
- P11: [Backpatching injection point core facilities to REL_17_STABLE](https://commitfest.postgresql.org/patch/5865/) (cf, Withdrawn, 2025-08-17) - judged not relevant; reason summarised from its thread
- P12: [Revert "Don't truncate database and user names in startup packets."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0ff56e2d3ff1db3de727b33b2dac985ccc43ef8) (revert, Reverted, 2024-12-12) - judged not relevant
- P13: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P14: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - judged not relevant
- P15: [Send duration output to separate files](https://commitfest.postgresql.org/patch/5125/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P16: [pg_tracing](https://commitfest.postgresql.org/patch/4456/) (cf, Withdrawn, 2024-07-10) - judged not relevant
- P17: [Fix around conn_duration in pgbench](https://commitfest.postgresql.org/patch/3167/) (cf, Withdrawn, 2021-06-15) - judged not relevant
- P18: [Revert: Implement pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=772faafca1b288c4dd66b7150a7831c27b768003) (revert, Reverted, 2024-04-11) - judged not relevant
- P19: [Do not exit on postmaster death event inside CRIT sections.](https://commitfest.postgresql.org/patch/5964/) (cf, Returned with feedback, 2025-08-28) - judged not relevant
- P20: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - used in the review
- P21: [Connection string usage for Core Postgresql client applications](https://commitfest.postgresql.org/patch/2354/) (cf, Withdrawn, 2020-01-21) - judged not relevant
- P22: [minimizing pg_stat_statements.track='none' performance overhead](https://commitfest.postgresql.org/patch/2092/) (cf, Withdrawn, 2019-04-15) - judged not relevant
- P23: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P24: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0004-ci-build-one-task-with-enable-wait-event-timing.patch not read
- attachment limit (8) reached; v1-0005-wait_event_timing-allocate-the-per-backend-array-.patch not read
- attachment limit (8) reached; v1-0007-wait_event_timing-add-query-attribution-markers-t.patch not read
- attachment limit (8) reached; v1-0006-wait_event_timing-add-trace-level-with-a-per-sess.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 210 s; cost reported by the CLI: $7.23
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 125 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:12:39.849Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
