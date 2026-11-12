# Design pre-review: log_postmaster_stats

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6804](https://commitfest.postgresql.org/patch/6804/) (Needs review); authors: Jakub Wartak (jakub.wartak.tomtom)
- Thread: [log_postmaster_stats](https://www.postgresql.org/message-id/flat/CAKZiRmxeTNAe_VZ9m%3Drt%3D5155mvL%3DCFsQ4ENJWhuUdgMe%3D_Udg%40mail.gmail.com), started 2026-05-27
- Messages read: 9 of 9
- Patch prose read: v4-0001-Add-log_postmaster_overloads-and-log_postmaster_e.patch, v1-0001-Add-log_postmaster_stats.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The connection-rate threshold has support in the thread, so the goal is not in question. The CPU-based log_postmaster_overloads is: message #7 asked to drop it, and v3 and v4 keep it without answering the misfire concern (standards-compat-1), the missing test (maintenance-fit-1) or the overlap with per-process OS monitoring (already-exists-1). The list needs to settle whether that GUC stays before code review makes sense. The duplicate lines and the GUC name are smaller, fixable points.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check how the lateness rule is computed, where the counters sit in the connection accept path, or what the TAP test covers. Only 3 of 9 messages were shown in full.

## The proposal, restated

**Problem.** DBAs cannot tell afterwards how busy the postmaster was, because log_connections is too costly to keep on and pg_stat_database counters hold no history or CPU usage. A single postmaster at 100% CPU is easy to miss on large machines, so connection storms, slow fork() with very large shared_buffers, and parallel worker launch floods go undetected.

**Proposed design.** The postmaster checks its counters and getrusage(RUSAGE_SELF) once per second. It logs a stats line when it used more than 75% of one CPU core, ran late, or could not serve a connection, or when new connections per second exceed a set threshold. The design moved from v1 periodic logging (log_postmaster_stats) to threshold-based logging after Robert Haas's review.

**User-visible surface.**
- GUC log_postmaster_overloads (boolean, default off, postgresql.conf or command line only).
- GUC log_postmaster_excess_connections (integer connections/sec, default 0 = disabled, postgresql.conf or command line only).
- LOG messages "postmaster potentially overloaded, stats: ..." and "postmaster excessive connections, stats: ..." with conns, disconns, conn errors and parallel workers per second, plus CPU user/system/elapsed.
- New TAP test src/test/postmaster/t/005_log_postmaster.pl.

**What the author claims.**
- A storm of about 1800 connections/sec costs the postmaster about 5 s of system CPU per 10 s.
- Under 200 to 300 connections/sec can already drive the postmaster to 100% system CPU on some setups.
- getrusage() once per second costs nothing measurable next to thousands of fork() calls per second.
- A raw connection-rate threshold is hard to choose, so the CPU-based switch is the main feature.
- Postmaster bottlenecks occur in the field, and the author cites earlier threads and new cases.

**Thread state.** Robert Haas objected to the log-based design, then supported emergency logging but asked to drop the CPU checks and keep only a connection-rate threshold. The author kept both GUCs in v3 and v4, Robert has not answered, and the duplicate log lines when both GUCs are on are still open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-05-27; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. CPU-based trigger kept against reviewer advice and cannot be tested

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), new GUC · `maintenance-fit-1`

A later patch changes the postmaster wait loop or the timing of its wakeups. The log_postmaster_overloads path then stops firing, or fires on every disk stall, and no test fails because the CPU test is commented out. Users find out during an incident, when the expected message is missing or misleading.

Robert asked in message #7 to drop the CPU checks and keep only a connection-rate threshold, partly so the test could pass reliably. In message #8 the author kept both GUCs, citing the difficulty of choosing a rate. Message #6 says CI cannot overload the postmaster, and the commit message defines overload as "75% or getting late on CPU" without saying what "late" means. Robert has not replied since v3, so no committer has accepted owning an untested, scheduler-dependent trigger.

Thread: [#6 (Jakub Wartak, 2026-07-27)](https://www.postgresql.org/message-id/CAKZiRmy%2Bqqox8XSk%2Bcd%2Bc7XmbGZHHpnAHOg3D-Ho-atuQRP4fg%40mail.gmail.com), [#7 (Robert Haas, 2026-07-28)](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#8 (Jakub Wartak, 2026-07-30)](https://www.postgresql.org/message-id/CAKZiRmx996hSedQ9BW6v2qj092tUoZZiUHWHyD9YRyhxT8gvXg%40mail.gmail.com), [#9 (Jakub Wartak, 2026-11-09)](https://www.postgresql.org/message-id/CAKZiRmzTaWmFZWKuP-qLN5QU1p3LzYFPDtJK8%2BeONfT%2BwO3e7g%40mail.gmail.com)

**Question:** Can log_postmaster_overloads be split into a separate later patch, after log_postmaster_excess_connections is committed with its reliable test?

Counter-check: held. Message #6 confirms that CI cannot overload the postmaster and that the CPU test is mostly commented out. A trigger that depends on scheduling and has no test can break without any failure, which is a design-level maintainability concern.

### 2. Boolean overload GUC hides a fixed 75% CPU rule and an undefined lateness rule

fixable · level 2, rubric item 10 · new GUC, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A DBA enables log_postmaster_overloads and gets "potentially overloaded" lines while the postmaster waits on a slow disk or network write. On another machine a heavy storm uses only 60% of a fast core and nothing is logged. The DBA cannot tune either case because the trigger is hard-coded.

The commit message defines overload as "75% or getting late on CPU" but does not say how late counts as late. In message #7 Robert Haas predicted these false positives and false negatives and asked to drop the CPU checks. Message #8 keeps them as the "main feature" and gives no answer on misfires, so a committer would likely see the open point as unsettled. Once the GUC ships, its meaning becomes part of the contract, and changing the 75% rule or the lateness rule later changes behaviour for existing users.

Thread: [#7 (Robert Haas, 2026-07-28)](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#8 (Jakub Wartak, 2026-07-30)](https://www.postgresql.org/message-id/CAKZiRmx996hSedQ9BW6v2qj092tUoZZiUHWHyD9YRyhxT8gvXg%40mail.gmail.com), [#9 (Jakub Wartak, 2026-11-09)](https://www.postgresql.org/message-id/CAKZiRmzTaWmFZWKuP-qLN5QU1p3LzYFPDtJK8%2BeONfT%2BwO3e7g%40mail.gmail.com)

**Question:** What exact condition triggers log_postmaster_overloads, and why should 75% and the lateness rule be fixed and not settable?

Counter-check: held. The v4 docs state the 75% rule but do not define "getting late on CPU". Message #8 answers the cost of getrusage() but not the false positives and false negatives raised in message #7.

### 3. Two GUCs log the same stats twice per second during a storm

fixable · level 2, rubric item 12 · design, new GUC, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A site sets both log_postmaster_overloads = on and log_postmaster_excess_connections = 200. During a long connection storm the postmaster writes two lines with the same numbers every second, so the log doubles at the moment it is already under load. Log-based alert rules that count lines double their counts.

Message #9 shows the duplicate output and leaves the fix open. Both settings guard one diagnostic, with two triggers and two names, where Postgres usually has one parameter per diagnostic. The proposal also has no limit on lines per storm beyond one per second, so a storm that lasts an hour writes thousands of lines.

Thread: [#7 (Robert Haas, 2026-07-28)](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#9 (Jakub Wartak, 2026-11-09)](https://www.postgresql.org/message-id/CAKZiRmzTaWmFZWKuP-qLN5QU1p3LzYFPDtJK8%2BeONfT%2BwO3e7g%40mail.gmail.com)

**Question:** Why does one diagnostic need two independent GUCs, and what does the log contain when both trigger in the same second?

Counter-check: held. Message #9 shows the duplicate lines and offers a merged message only as an idea, so v4 still emits both. A new patch version can fix this.

### 4. The name log_postmaster_excess_connections does not say it is a rate

fixable · level 2, rubric item 10 · new GUC · `standards-compat-3`

A DBA reads the name next to max_connections, takes it as a count of open sessions, and sets it to 500. The setting means 500 new connections per second, so a real storm of 300 per second is never logged.

In message #7 Robert Haas proposed log_excessive_connection_rate because that name tells users what the setting means. The v4 name drops "rate", and the integer has no unit, while nearby connection settings count sessions. Message #8 argues that a raw rate is hard to choose but does not explain the name.

Thread: [#7 (Robert Haas, 2026-07-28)](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#8 (Jakub Wartak, 2026-07-30)](https://www.postgresql.org/message-id/CAKZiRmx996hSedQ9BW6v2qj092tUoZZiUHWHyD9YRyhxT8gvXg%40mail.gmail.com)

**Question:** Why is a name without "rate" or a per-second unit better than the log_excessive_connection_rate spelling suggested in message #7?

Counter-check: held. The docs say the value is connections per second, but the name does not, and message #7 asked for a name that explains itself. The thread gives no reason for dropping "rate".

### 5. Postmaster CPU checks duplicate per-process OS monitoring

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), new GUC, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A DBA turns on log_postmaster_overloads and also runs a host agent that samples per-process CPU, or pidstat on the PID from postmaster.pid. Both report the same saturation, but the core check uses a fixed 75% threshold and a one-second window the DBA cannot tune. The server log then carries a second, less flexible copy of data the OS already records.

Per-process CPU history of the postmaster is available today from standard OS tools, sampled at any interval, with the PID taken from postmaster.pid. Message #5 says the real problem is that users miss one busy core among many busy backends, which makes the OS route less visible but still possible. Robert Haas (message #7) supported a connection-rate threshold and asked to drop the CPU checks; log_postmaster_excess_connections covers what core alone can see. The 75% CPU switch mostly repeats OS accounting inside the postmaster loop.

Thread: [#5 (Jakub Wartak, 2026-07-23)](https://www.postgresql.org/message-id/CAKZiRmxEz9zQKHdb3VNiZVh9PT2JDBo8sivBEY0giecSAaFxrA%40mail.gmail.com), [#7 (Robert Haas, 2026-07-28)](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#8 (Jakub Wartak, 2026-07-30)](https://www.postgresql.org/message-id/CAKZiRmx996hSedQ9BW6v2qj092tUoZZiUHWHyD9YRyhxT8gvXg%40mail.gmail.com)

**Question:** Which postmaster overload cases does the CPU check catch that log_postmaster_excess_connections and per-process OS CPU sampling of the postmaster PID both miss?

Counter-check: held. OS per-process sampling can record postmaster CPU, and the thread's only answer is that users overlook it on large hosts. The material does not show a case that both the rate GUC and OS sampling miss.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds log_postmaster_overloads and log_postmaster_excess_connections. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Unauthenticated connection attempts can trigger the new LOG lines, at about one line per second per GUC. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The CPU trigger depends on postmaster loop timing and its test is mostly commented out. |
| performance on the hot path | **yes** | The postmaster main loop gains counters and a once-per-second getrusage() check when a GUC is on. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The 75% CPU check overlaps per-process CPU sampling that OS tools already provide. |

## 5. Questions the author must answer before anyone reviews code

1. Which postmaster overload cases does the CPU check catch that log_postmaster_excess_connections and OS sampling of the postmaster PID both miss?
2. What exact delay counts as "getting late on CPU", and why are that rule and the 75% threshold fixed?
3. Can log_postmaster_overloads move to a separate later patch, so log_postmaster_excess_connections can be committed first with its reliable test?
4. What does the log contain when both GUCs trigger in the same second, and is there any limit during an hour-long storm?
5. Why is log_postmaster_excess_connections preferred over the log_excessive_connection_rate name suggested in message #7?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Robert Haas reviewed up to message #7 but has not answered v3 or v4. The owner needs to know the postmaster main loop and its wakeup timing.
- **Liability:** Two GUC names, a log line format that alert rules will parse, and the fixed 75% and lateness rules, all for as long as the GUCs exist. The CPU trigger has no working test.
- **Author follow-up:** The author posted four versions between May and message #9 and moved the entry to the next CommitFest in October 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** Sites with connection storms gain from the rate threshold, and the patch is about 24 KiB, mostly in postmaster.c. The open disagreement about the CPU trigger blocks progress more than patch size does.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Per-process overload GUCs do not pass the accumulation test. Refuted (no failure scenario): Nobody in the thread proposes the same GUC for other processes, so the scenario is hypothetical. Per-process log settings such as log_checkpoints and log_startup_progress_interval are existing practice, and message #7 calls threshold logging "a good direction".
- `demand-1` Demand for the CPU-based trigger rests on the author's own cases. Refuted (not specific): Duplicate of already-exists-1, which asks the same question about cases only the CPU trigger catches. The title also overstates: the reports linked in messages #5 and #8 are other threads, which the author presents as field cases.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] changed which connection stages log_connections reports and was returned because its tests failed and the thread stalled, so neither its idea nor its reason applies here. Messages #5 and #8 link earlier threads, but only as field reports of postmaster bottlenecks, not as earlier attempts at this feature.
- standards-compat: 3 objection(s). Robert Haas raised the cumulative-statistics concern in message #4. He supported the emergency-logging direction in message #7, so it is not raised here.
- replication-ondisk: 0 objection(s). The proposal adds two postmaster-only GUCs, in-memory counters and LOG lines, with no on-disk format, WAL, catalog, protocol or replication state to check. The change to bgworker_internals.h is an internal header, not a documented extension API, so this angle finds no persistent or cross-node concern.
- security: 0 objection(s). No security objection. Both GUCs can be set only in postgresql.conf or on the command line, and both are off by default. The messages hold aggregate counts and postmaster CPU times, with no user, database or query data. Unauthenticated clients can trigger the lines, but output is capped at about one line per second per GUC, which is much less than log_connections writes today. Without the code I could not check that the counters and getrusage() add nothing that an unauthenticated client can exploit.
- maintenance-fit: 2 objection(s). Hot-path cost is not raised. Both GUCs default to off, and the once-per-second check falls only on users who turn them on.
- already-exists: 1 objection(s). The connection-rate threshold has no equivalent in core: pg_stat_database.sessions keeps no per-second history, and log_connections costs too much to leave on. Robert Haas endorsed that part in message #7, so this angle raises nothing against it.
- demand: 1 objection(s). The connection-rate GUC has credible demand. The author reports field cases, cites a thread with 2000 to 3000 connect requests per minute from parallel workers (message #5), and Robert Haas supported emergency logging in message #7. Both GUCs are off by default, so only users of the feature pay. The objection covers only the CPU-based GUC.

### Precedents retrieved

- P1: [Use SIGIO to detect postmaster death](https://commitfest.postgresql.org/patch/3066/) (cf, Withdrawn, 2021-07-13) - judged not relevant; reason summarised from its thread
- P2: [Create a new session for postmaster in pg_ctl](https://commitfest.postgresql.org/patch/1772/) (cf, Returned with feedback, 2018-09-30) - judged not relevant; reason summarised from its thread
- P3: [Enable fine-grained control over what gets logged on connection attempt (reduces log size)](https://commitfest.postgresql.org/patch/4005/) (cf, Returned with feedback, 2023-08-03) - used in the review; reason summarised from its thread
- P4: [Do not exit on postmaster death event inside CRIT sections.](https://commitfest.postgresql.org/patch/5964/) (cf, Returned with feedback, 2025-08-28) - judged not relevant; reason summarised from its thread
- P5: [Revert "Hack pg_ctl to report postmaster's exit status."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f38291e927fa8c04eb772e6a17a3dd44da2b69e8) (revert, Reverted, 2019-10-13) - judged not relevant
- P6: [Avoid possibly slow processing during postmaster abnormal or immediate shutdown](https://commitfest.postgresql.org/patch/816/) (cf, Rejected, 2016-11-18) - judged not relevant; reason summarised from its thread
- P7: [Revert "postmaster: Start syslogger earlier".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=833451552925d0175e1e15128e411ddef9a36996) (revert, Reverted, 2019-05-19) - judged not relevant
- P8: [Revert "Add log_statement_sample_rate parameter"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=75506195da81d75597a4025b72f8367e6c45f60d) (revert, Reverted, 2019-08-04) - judged not relevant
- P9: [allow to set log_statement_sample_limit - maximal duration for sampling](https://commitfest.postgresql.org/patch/2163/) (cf, Rejected, 2019-11-06) - judged not relevant; reason summarised from its thread
- P10: [Add sample rate to auto_explain](https://commitfest.postgresql.org/patch/261/) (cf, Returned with feedback, 2015-08-25) - judged not relevant; reason summarised from its thread
- P11: [Improve logicalrep_worker_launch() logic](https://commitfest.postgresql.org/patch/5734/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P12: [Revert "Replace PostmasterRandom() with a stronger way of generating randomness."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faae1c918e8aaae034eaf3ea103fcb6ba9adc5ab) (revert, Reverted, 2016-10-18) - judged not relevant
- P13: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P14: [Let pg_ctl check the result of a postmaster config reload](https://commitfest.postgresql.org/patch/185/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P15: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant
- P16: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - judged not relevant
- P17: [Improving connection scalability](https://commitfest.postgresql.org/patch/3651/) (cf, Rejected, 2022-08-01) - judged not relevant
- P18: [Improving the latch handling between logical replication launcher and worker processes.](https://commitfest.postgresql.org/patch/5095/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P19: [Add a GUC variable that control logical replication](https://commitfest.postgresql.org/patch/2296/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P20: [Revert "Add GUC backtrace_on_internal_error"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=592a2283721f7143999364ef487f2b4993f5161d) (revert, Reverted, 2024-04-29) - judged not relevant
- P21: [Built-in connection proxy](https://commitfest.postgresql.org/patch/2067/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P22: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P23: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant
- P24: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 140 s; cost reported by the CLI: $1.89
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 82 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:04:27.631Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
