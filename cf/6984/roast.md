# Design pre-review: [PATCH v1 0/7] Wait event timing and tracing instrumentation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6984](https://commitfest.postgresql.org/patch/6984/); [thread](https://www.postgresql.org/message-id/flat/CAPHG-0mAOn05ae6Kqx1wHXxzOk4E5W7ajjd%3DQBhgkR7a0uyQmw%40mail.gmail.com), 19 of 19 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The series fails at level 1b, so the design objections are not ranked. Messages #13 and #18 ask to settle first whether core observes waits by sampling, at selected sites, or at every transition, and the thread shows no agreement (maintenance-fit-4). Message #19 reproduces the main result with an external tracer on stock PG18, so the shown gain is lower cost and no host access (already-exists-2). The proposal does not say why the collector must be in contrib (already-exists-1).

Not checked: No code was read. The cost with no consumer loaded, the code-size growth at each site, and the safety of the lock-free slot claiming and the seqlock ring could not be judged; the level 2 objections were not evaluated.

## Prior attempts

- [Add accumulated statistics for wait event](https://www.postgresql.org/message-id/flat/2022793411.102298.1532329482584.JavaMail.jboss@ep1ml402) (similar idea; Returned with feedback, 2020-08-02). "Since the thread has stalled and there is no updated patch I've marked this entry Returned with Feedback." ([message](https://www.postgresql.org/message-id/25597647-59D2-413F-B7E9-920AF57DD51B%40yesql.se)) Different now: P3 kept per-backend wait counts and accumulated times in core with a hash table and a track_wait_timing setting. This series moves collection to a contrib module behind hooks, adds a trace ring with statement attribution, and reports about 2 ns of cost with no consumer loaded.

## Objections a reviewer is likely to raise

### 1. Every-transition observation chosen before the observation contract is agreed

Core commits hooks on every wait transition. Later the project adds the configurable sampler Andres describes, and then has two overlapping wait observation mechanisms. The hooks cannot be removed without breaking extensions that came to depend on them.

Message #13 asks to settle first whether the contract is sampling, selected extended sites, or every transition. Message #18 rejects the premise that wait timing alone is useful and prefers sampling with state context. The overhead figures in message #19 come from v11, and v12 to v14 were not measured again. A hook API offered to extensions is hard to withdraw once it ships.

**Question:** Why should core commit permanent every-transition hooks before the project agrees whether a sampler would answer the same incident questions?

<sub>fixable, level 1b, desirability; messages [#13](https://www.postgresql.org/message-id/9F597BE1-53A7-4615-A2D4-60AF0CAE48E6%40yandex-team.ru), [#18](https://www.postgresql.org/message-id/2s45loetq2c3y3vyecm5oos36vdr6dv7wlupddjx63owhcgwfi%40ia6jtqqs26kb), [#19](https://www.postgresql.org/message-id/CAPHG-0%3Dp4cX9gy5KKX_SicPdE8R_MoeiVAUGU%2B_hu3fXOi_K4A%40mail.gmail.com); `maintenance-fit-4`</sub>

### 2. Exact per-wait timing is already possible on stock builds

A DBA who can run eBPF on the host already gets exact per-wait durations from my_wait_event_info on an unpatched server, as the author did on stock PG18. The proposal adds a permanent API and 94 converted sites, and its added value is lower cost per transition and use without host access.

Message #19 reproduces the main result, Lock:transactionid dominating slow requests, with an external eBPF tracer on stock PG18. Message #1 gives that tracer's cost as about 200-300 ns per transition. So the gap is overhead and deployment, and is not capability. Users on managed services still depend on the provider loading the module, so the hook does not by itself close the deployment gap.

**Question:** Which incident could not be diagnosed with the existing external tracer plus pg_wait_sampling, track_io_timing and log_lock_waits, and why is the cost difference decisive?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAPHG-0mAOn05ae6Kqx1wHXxzOk4E5W7ajjd%3DQBhgkR7a0uyQmw%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/9F597BE1-53A7-4615-A2D4-60AF0CAE48E6%40yandex-team.ru), [#19](https://www.postgresql.org/message-id/CAPHG-0%3Dp4cX9gy5KKX_SicPdE8R_MoeiVAUGU%2B_hu3fXOi_K4A%40mail.gmail.com); `already-exists-2`</sub>

### 3. Collector module belongs out of tree once the hook exists

A future patch changes protocol handling, the executor hooks or transaction callbacks. The contrib module's statement-attribution state machine silently misplaces waits, and the committer who touched postgres.c now owns fixing a module they never reviewed. Users find out when pg_wait_event_trace_by_statement() puts lock waits in the wrong bucket.

Patches 0004 and 0005 are about 480 KiB and use only the hook from 0001 plus existing extension hooks. pg_wait_sampling already shows that a wait collector can be maintained out of tree. In message #13, Andrey suggested a small core hook with a separately maintained collector, and the proposal does not say why contrib placement is needed. Message #16 already lists four ambiguous attribution cases.

**Question:** What does contrib placement give pg_wait_event_tracing that an out-of-tree extension using the same hook cannot have?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#13](https://www.postgresql.org/message-id/9F597BE1-53A7-4615-A2D4-60AF0CAE48E6%40yandex-team.ru), [#16](https://www.postgresql.org/message-id/CAPHG-0moXufhp9ssiVHE%2B83A7SyTApTxf4zMr4h7X6%2BtEDTrEQ%40mail.gmail.com), [#19](https://www.postgresql.org/message-id/CAPHG-0%3Dp4cX9gy5KKX_SicPdE8R_MoeiVAUGU%2B_hu3fXOi_K4A%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Why should core add hooks on every wait transition before the list agrees whether a configurable sampler answers the same incident questions?
2. Which incident could not be diagnosed with the external eBPF tracer plus pg_wait_sampling, track_io_timing and log_lock_waits, and why is the cost difference decisive?
3. What does contrib placement give pg_wait_event_tracing that an out-of-tree extension using the same hook cannot have?
4. Which individual wait sites justify the timed pair on their own analysis, and does the use case still hold if only those are converted?
5. What are the no-consumer and capture-on overhead numbers when measured on v14 instead of v11?

## Touch points

- **WAL / replication / standby:** v14-0002 converts wait sites in xlog.c, walwriter, walsender, walreceiver and standby.c to the hooked pair; no WAL format change.
- **ABI / extension API:** v14-0001 adds wait_event_begin_hook, wait_event_end_hook and a second public reporting pair with a documented hook contract.
- **new GUC:** The module adds pg_wait_event_tracing.capture and two other GUCs.
- **security / privileges:** SUSET capture GUC, cross-backend trace reads by pg_read_all_stats, and reset functions with pg_signal_backend-style checks.
- **maintenance burden ("who maintains this in 10 years"):** Every future wait site must choose between two pairs, and contrib gains a large collector tied to executor and protocol hooks.
- **performance on the hot path:** All 94 wait sites, including LWLock, spinlock and I/O waits, gain a hook pointer test.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #13 suggests a small core hook with the collector maintained outside the tree; an external eBPF tracer gives exact timing today.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund and Michael Paquier take part in the thread, but Andres Freund opposes the hooks and no committer supports the current shape. The owner would need to know the wait event, LWLock and I/O paths.
- Priority: needs design rethink; audience a sizeable minority; review cost high. The goal has support from DBAs in the thread, but the series converts 94 sites and adds about 480 KiB of contrib patches while a committer who knows the area opposes hooks on all sites. Agreement on sampling, selected sites or every transition must come before code review.

---

[roast-full.md](roast-full.md) has 5 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 4.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
