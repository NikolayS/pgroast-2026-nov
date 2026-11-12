# Design pre-review: Enlarge the syslogger pipe on Linux (F_SETPIPE_SZ)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7390](https://commitfest.postgresql.org/patch/7390/) (Needs review); authors: Sehrope Sarkuni (sehrope)
- Thread: [Enlarge the syslogger pipe on Linux (F_SETPIPE_SZ)](https://www.postgresql.org/message-id/flat/CAH7T-arLtnrbM5ynVAeXYxtK%3DP012xH4%2BcrCm7w0efZq7%2BRztQ%40mail.gmail.com), started 2026-09-16
- Messages read: 1 of 1
- Patch prose read: 0001-Enlarge-the-syslogger-pipe-with-F_SETPIPE_SZ.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived the panel, so nothing stops the proposal at levels 1a, 1b or 2. The problem is stated, message #1 gives measurements for statement logging and bursts, and the change adds no user-visible surface and falls back to the kernel default. Confidence is low because only one message exists, nobody has replied, and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the retry sizes, the handling of a failed or partly granted request, and any reporting of the final pipe size could not be checked. The effect of kernel per-user pipe limits also rests on the code and on testing.

## The proposal, restated

**Problem.** Backends block in write() when the 64 KiB syslogger pipe fills, and it fills fast when several backends log multi-chunk messages at once. The author targets chatty logging: statement logging, pgaudit, auto_explain and similar.

**Proposed design.** At pipe creation the syslogger asks the kernel for 1 MiB with F_SETPIPE_SZ, then retries smaller sizes if the kernel refuses. It never shrinks the pipe below what the kernel already gave it. If the fcntl is missing or every request fails, the default stays.

**What the author claims.**
- At 96 clients with log_min_duration_statement = 0, p99 latency drops from about 10.7 ms to 2.0 ms, but p50 rises from 387 to 509 us.
- Collector CPU drops by about a third because it wakes less often and reads more per wakeup.
- In RAISE LOG burst tests the patch won every pair on average latency, up to about 8 times lower.
- Throughput stays within 1% when the collector is saturated, and 256 KiB gave only a third of the burst gain.
- No GUC is needed because 1 MiB is the default unprivileged maximum and the buffer exists once per cluster.

**Thread state.** Only the first message is visible. Nobody has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-16; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

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
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a Linux-only F_SETPIPE_SZ path with a retry loop in syslogger.c, and leaves Windows unchanged. |
| performance on the hot path | **yes** | The pipe size changes how long logging backends block in write(); p50 rises from 387 to 509 us at 96 clients. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the higher p50 at 64 and 96 clients (387 to 509 us at 96) an accepted trade-off, and does it appear when the box is not oversubscribed?
2. Which smaller sizes does the retry try, and does the syslogger report the size it got or a failed request?
3. Can pipe-max-size or the per-user pipe-user-pages-soft limit make the 1 MiB request fail, or shrink other pipes of the postgres user, on common distributions?
4. Do the results hold with logs on a real disk instead of tmpfs, where the collector's write speed differs?
5. Does a 1 MiB buffer change how much log data is lost or reordered when the syslogger dies or restarts?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the syslogger and postmaster startup code, and Linux pipe semantics, would have to own it. Nobody has replied in the visible thread.
- **Liability:** The owner signs for a Linux-only fcntl path in syslogger.c and for a latency shift under heavy logging: lower p99, higher p50 when oversubscribed. The path is small and falls back to the kernel default, so the long-term burden is low.
- **Author follow-up:** No sign either way. The author says he cannot test the Windows CreatePipe variant.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 2.5 KiB in one file with no GUC or other user-visible surface, and message #1 shows gains for statement logging, pgaudit and auto_explain workloads. The open points are the p50 trade-off and kernel limit behaviour, which a reviewer can settle quickly.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The precedent list has no earlier proposal to enlarge the syslogger pipe or use F_SETPIPE_SZ, and the thread does not point to one. From the prior-attempts angle there is no recorded rejection reason to apply, so no precedent found.
- standards-compat: 0 objection(s). From the standards and compatibility angle the patch has no user-facing contract to judge: no syntax, no GUC, no catalog entry, no pg_dump state and no psql change, and it falls back to the kernel default when the fcntl fails. The higher p50 latency at 64 and 96 clients and the unstated retry sizes belong to the performance and design angles.
- replication-ondisk: 0 objection(s). The patch changes only the kernel buffer size of the syslogger pipe in syslogger.c, with no on-disk format, WAL, catalog, wire protocol or ABI change. The postmaster holds both pipe ends across syslogger restarts, so buffered log data survives a collector restart as before, and nothing persistent needs pg_upgrade or replication handling.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. The pipe stays internal to the cluster, owned by the postgres OS user, and no new SQL surface, setting or role is added. A database user who floods the log with RAISE LOG could already block backends, and a larger buffer reduces that blocking instead of adding a new denial-of-service path. The extra kernel memory is one fixed 1 MiB buffer. Without the code I could not check one thing: whether the request counts against the postgres user's pipe-user-pages-soft limit in a way that shrinks other pipes, such as archive_command. That is an operations question, not a security one.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is no objection worth sending. The patch is a best-effort fcntl in one file with no GUC, catalog, protocol or API surface, and it falls back to current behaviour. Its costs (one kernel buffer per cluster, a higher p50 under oversubscription) fall on clusters that log heavily, which are the clusters that gain. Two points cannot be checked without the code: which retry sizes the patch tries, and whether it reports the size it got.
- already-exists: 0 objection(s). Postgres has no existing setting, view or idiom that sets the syslogger pipe size. On Linux the size can only be changed with fcntl on the open pipe, and pipe-max-size caps requests without changing the default, so an administrator cannot get this from outside. The pipe is created in the postmaster before extensions could act on it. Switching log_destination to syslog avoids the pipe but changes delivery and loss behaviour, so it does not do the same job.
- demand: 0 objection(s). No demand objection. Statement logging, pgaudit and auto_explain are common workloads, message #1 shows measured gains for them, and the cost (one 1 MiB kernel buffer, no GUC, no UI) falls only on clusters that log heavily. The p50 trade-off at 64 and 96 clients and the missing retry details belong to the design and performance angles.

### Precedents retrieved

- P1: [Revert "postmaster: Start syslogger earlier".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=833451552925d0175e1e15128e411ddef9a36996) (revert, Reverted, 2019-05-19) - judged not relevant
- P2: [Improve logrotate test so that it meaningfully exercises syslogger.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9d5c22d28f8c161aa729d841aeec936c646044d4) (revert, Partially reverted, 2019-05-19) - judged not relevant
- P3: [Allow FunctionScans to pipeline results](https://commitfest.postgresql.org/patch/2372/) (cf, Withdrawn, 2020-03-23) - judged not relevant
- P4: [Batch (pipelined) mode for libpq](https://commitfest.postgresql.org/patch/634/) (cf, Returned with feedback, 2016-11-22) - judged not relevant
- P5: [Revert dup2() checking in syslogger.c](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=aef61bf433a9e9b6e2d98b0fdcce8562c3ad526f) (revert, Reverted, 2014-01-28) - judged not relevant
- P6: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1024/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P7: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1317/) (cf, Returned with feedback, 2018-11-29) - judged not relevant; reason summarised from its thread
- P8: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [logicalctl: release LogicalDecodingControlLock before logging](https://commitfest.postgresql.org/patch/6477/) (cf, Withdrawn, 2026-03-04) - judged not relevant; reason summarised from its thread
- P10: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - judged not relevant; reason summarised from its thread
- P11: [Adding linux cachestat in file Readv and Prefetch](https://commitfest.postgresql.org/patch/4779/) (cf, Rejected, 2024-03-14) - judged not relevant; reason summarised from its thread
- P12: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P13: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P14: [Fix log_line_prefix to display the transaction id (%x) for statements not in a transaction block](https://commitfest.postgresql.org/patch/4606/) (cf, Withdrawn, 2024-05-18) - judged not relevant
- P15: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P16: [Improve logicalrep_worker_launch() logic](https://commitfest.postgresql.org/patch/5734/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P17: [Flexible configuration for full-text search](https://commitfest.postgresql.org/patch/1366/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P18: [Windows meson build](https://commitfest.postgresql.org/patch/5522/) (cf, Returned with feedback, 2026-03-27) - judged not relevant
- P19: [Fix creation of empty .log files during log rotation](https://commitfest.postgresql.org/patch/5365/) (cf, Rejected, 2026-01-08) - judged not relevant
- P20: [Flexible configuration for full-text search](https://commitfest.postgresql.org/patch/1421/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P21: [Implement <null treatment> for window functions](https://commitfest.postgresql.org/patch/2628/) (cf, Returned with feedback, 2021-03-25) - judged not relevant
- P22: [Escaping from blocked send() by pg_terminate_backend().](https://commitfest.postgresql.org/patch/84/) (cf, Rejected, 2015-01-12) - judged not relevant
- P23: [Switch logtape and tuplesort to use int64 instead of long](https://commitfest.postgresql.org/patch/4579/) (cf, Withdrawn, 2023-09-26) - judged not relevant
- P24: [Fix autovacuum freeze bug where weight can lower a table's freeze score](https://commitfest.postgresql.org/patch/7242/) (cf, Withdrawn, 2026-09-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 92 s; cost reported by the CLI: $0.94
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:49:32.251Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
