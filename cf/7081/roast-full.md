# Design pre-review: Add PQTRACE env to enable protocol tracing in libpq

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7081](https://commitfest.postgresql.org/patch/7081/) (Needs review); authors: Anthonin Bonnefoy (bonnefoa)
- Thread: [Add PQTRACE env to enable protocol tracing in libpq](https://www.postgresql.org/message-id/flat/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com), started 2026-07-31
- Messages read: 2 of 2
- Patch prose read: v1-0003-Add-mismatch-length-check-on-pqTraceOutputNoTypeB.patch, v1-0001-Print-cancel-key-as-a-hex-string-in-trace.patch, v1-0002-Add-more-regress-masks-in-pqtrace.patch, v1-0004-Make-pqtrace-write-lines-atomically.patch, v1-0005-Add-PQTRACE-env-to-enable-protocol-tracing-in-lib.patch, v1-0006-Add-trace-of-startup-packets-in-libpq_pipeline-te.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is stated and has support: message #2 gives a +1 and P1 shows an earlier attempt at the same need. No surviving objection questions the idea. Four fixable design points remain at level 2. The decisive one is security-1: trace_file opens a file, and dblink and postgres_fdw pass user-supplied options to libpq inside the server. standards-compat-2 and standards-compat-3 concern names and the stdout target, which are hard to change after release.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so it is unknown whether trace_file is marked as a debug option, which dblink and postgres_fdw already hide. The file open mode and the handling of connection retries could not be checked either.

## The proposal, restated

**Problem.** Protocol tracing in libpq starts only when the application calls PQtrace(), so psql and other existing clients cannot use it without code changes. The libpq_pipeline test also misses startup packets because it calls PQtrace() after the connection exists. The target users are developers who debug protocol behavior and write tests.

**Proposed design.** libpq gains the connection parameters trace_file and trace_flags and the environment variables PQTRACE and PQTRACEFLAGS, and calls PQtrace() while it processes connection options, before the first packet. Supporting patches make fe-trace.c write each line with one fwrite, add regress masks for StartupMessage and ParameterStatus, and print the cancel key as hex. libpq_pipeline switches to trace_file.

**User-visible surface.**
- Connection parameter trace_file: a file path, or "-" for stdout.
- Connection parameter trace_flags: integer bitmask, 1 for PQTRACE_SUPPRESS_TIMESTAMPS, 2 for PQTRACE_REGRESS_MODE.
- Environment variables PQTRACE and PQTRACEFLAGS, equal to trace_file and trace_flags.
- Trace output change: the cancel key in BackendKeyData and CancelRequest prints as hex only.
- Regress mode now masks parameter values and packet length in StartupMessage and ParameterStatus.

**What the author claims.**
- Tracing becomes usable from psql and any libpq application without code changes.
- libpq_pipeline gets simpler and gains test coverage of startup packets.
- Writing each trace line with one fwrite stops interleaving when several connections trace to stdout.
- The limits are acceptable for a debugging tool: no shared trace file, and no tracing of cancel connections when the target is a file.
- The feature allows a future regress-style harness that compares SQL scripts against expected traces.

**Thread state.** Jelte Fennema-Nio gave a strong +1 for the idea but has not reviewed the patches. No technical review is visible, and the entry has moved to a second CommitFest.

## 2. Prior attempts

### libpq trace log (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1854/](https://commitfest.postgresql.org/patch/1854/)
- **Thread:** [https://www.postgresql.org/message-id/flat/71E660EB361DF14299875B198D4CE5423DE3FBA4@g01jpexmbkw25](https://www.postgresql.org/message-id/flat/71E660EB361DF14299875B198D4CE5423DE3FBA4@g01jpexmbkw25)
- **Date:** decided 2019-09-30; started 2018-08-24
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Alvaro Herrera from 2ndQuadrant (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Earlier, David Steele moved the patch to PG13 because, in agreement with Andres, it was not clear where the patch was going. The last message before the decision is a review by Alvaro Herrera, who agreed with the one-line-per-message direction but asked for changes. He objected to a frontend_entry[MAXPGPATH] array in pg_conn and listed smaller fixes. The thread shows no new patch or author reply after that review before the entry was returned.
- **Reason (quote):** "This seems to make pg_conn much larger than we'd like." ([message](https://www.postgresql.org/message-id/20190905222122.GA29713%40alvherre.pgsql))
- **What is different this time:** P1 started as a query log enabled by connection parameters or environment variables, then narrowed to reworking PQtrace() output and deferred the no-recompile part. This proposal does that deferred part on top of the one-line-per-message format.


## 3. Objections a reviewer is likely to raise

### 1. trace_file lets dblink and postgres_fdw users write server files

fixable · level 2, rubric item 7 · security / privileges · `security-1`

A non-superuser with EXECUTE on dblink_connect, or a postgres_fdw user mapping, passes trace_file=$PGDATA/postgresql.auto.conf. The backend opens that path as the postgres OS user, truncating or overwriting it. The admin finds out at the next restart or reload.

dblink and postgres_fdw run libpq inside the server and pass user-supplied connection options to it, so a libpq option that opens a file becomes a server-side file write. Today these modules hide only debug options and a short exclusion list. The patch prose does not say whether trace_file is marked as a debug option, nor does it mention dblink, postgres_fdw or other in-server libpq callers. The same issue applies to applications that accept connection URIs from end users.

Thread: [#1 (Anthonin Bonnefoy, 2026-07-31)](https://www.postgresql.org/message-id/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com)

**Question:** How is trace_file kept out of connection strings that dblink, postgres_fdw and other in-server or multi-tenant libpq callers accept from less privileged users?

Counter-check: held. dblink and postgres_fdw accept libpq options from users and filter only debug options and a short list. The patch prose does not say that trace_file is a debug option or that in-server callers reject it.

### 2. Trace to stdout by default mixes protocol lines into program output

fixable · level 2, rubric item 10 · design, psql UX conventions · `standards-compat-3`

A developer exports PQTRACE=- in a shell to debug psql and later runs pg_dump > db.sql or psql -At -c ... | script in the same shell. Trace lines go into the dump or the piped data, and the user finds out when restore or the script fails.

An environment variable is inherited by every libpq program started afterwards, and many of them write their real output to stdout. Postgres client tools send diagnostics to stderr. The only special value the proposal offers is "-" for stdout, and message #1 shows it as the main usage.

Thread: [#1 (Anthonin Bonnefoy, 2026-07-31)](https://www.postgresql.org/message-id/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com)

**Question:** Why should the special value send traces to stdout and not to stderr, given pg_dump and psql scripts write data to stdout?

Counter-check: held. The user sets the variable, but child programs such as pg_dump inherit it and write their data to stdout. The proposal offers no stderr target and does not say why stdout is the special value.

### 3. PQTRACE breaks the PG prefix and parameter-name pairing of libpq environment variables

fixable · level 2, rubric item 12 · design, psql UX conventions · `standards-compat-2`

A user who knows PGSSLMODE and PGAPPNAME sets PGTRACE or PGTRACEFILE, and nothing happens. Tools that clear or list libpq settings by matching PG* environment variables miss PQTRACE, so tracing leaks into their child processes.

Every libpq environment variable in the docs section the patch edits uses the PG prefix and mostly mirrors its parameter name, for example sslmode and PGSSLMODE. The patch pairs trace_file with PQTRACE and trace_flags with PQTRACEFLAGS. Message #1 and the commit message call the feature env vars only, while the doc hunk also adds connection parameters, so the thread has not settled the names. Once released, they cannot be renamed without breaking scripts.

Thread: [#1 (Anthonin Bonnefoy, 2026-07-31)](https://www.postgresql.org/message-id/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com)

**Question:** Why not use PGTRACEFILE and PGTRACEFLAGS, matching the parameter names and the existing PG prefix?

Counter-check: held. The doc hunk adds PQTRACE to a list where every other variable uses the PG prefix, and message #1 gives no reason. The thread has no reply on naming, so the consistency question stands.

### 4. Startup packets in expected traces couple tests to build and server settings

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), wire protocol · `maintenance-fit-1`

A later patch marks a new GUC as GUC_REPORT, or changes SSL or GSS negotiation, and nine libpq_pipeline trace files fail. A buildfarm animal built with GSSAPI, or without SSL, sends a different startup sequence and goes red although nothing is broken.

Patch 0006 adds startup messages to nine expected trace files. Patch 0002 masks parameter values and packet length, but the sequence of messages stays in the expected output. That sequence depends on build options, sslmode, gssencmode, the protocol version and the set of reported GUCs. Every later patch that adds a reported parameter or a negotiation step must update these files, and the proposal does not say how build-dependent messages are kept out of the expected output.

Thread: [#1 (Anthonin Bonnefoy, 2026-07-31)](https://www.postgresql.org/message-id/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com)

**Question:** How do the expected traces stay stable across SSL and GSSAPI builds, and what must a patch that adds a GUC_REPORT setting update?

Counter-check: held. Message #1 shows SSLRequest in the trace, and that line depends on the build and on TCP or Unix socket use. Patch 0002 masks values and lengths only, so the message sequence stays in nine expected files.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | walreceiver and logical replication workers connect through libpq, so they also process trace_file and PQTRACE. |
| wire protocol | **yes** | see objection maintenance-fit-1: Startup packets in expected traces couple tests to build and server settings |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | PQTRACE="-" writes trace lines to stdout of psql and of every other libpq client that inherits the variable. |
| security / privileges | **yes** | trace_file makes libpq open a file named in a connection string, including strings that dblink and postgres_fdw accept from users. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Two connection parameters and two environment variables become permanent, and nine expected trace files now include startup messages. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How is trace_file kept out of connection strings that dblink, postgres_fdw and other in-server libpq callers accept from less privileged users?
2. Why does the special value "-" send the trace to stdout and not to stderr, given that pg_dump and psql scripts write data to stdout?
3. Why are the variables named PQTRACE and PQTRACEFLAGS, and not PGTRACEFILE and PGTRACEFLAGS to match the parameter names and the PG prefix?
4. How do the expected traces in libpq_pipeline stay the same across builds with and without SSL or GSSAPI, and when a new GUC_REPORT setting is added?
5. Does libpq truncate or append to the trace file, when does it close the file, and what happens on a retry with multiple hosts?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows libpq connection option handling and the in-server libpq callers (dblink, postgres_fdw, walreceiver) would have to own it. Nobody has reviewed the patches in the visible thread.
- **Liability:** The owner signs for a connection parameter that opens files, for as long as libpq keeps it. The owner also signs for the names of two environment variables and for trace output that expected test files now depend on.
- **Author follow-up:** The author moved the entry to the next CommitFest on 2026-10-02. The thread has no author reply after message #1.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The feature serves developers who debug the protocol or write tests, and it has a +1 in message #2. The series is six patches limited to libpq and one test module, but the file-write question in security-1 and the names should be settled before code review.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` trace_file in server-side connection strings lets users write server files. Refuted (not specific): Duplicate of security-1, which states the same server-side file write with a sharper scenario. The subscription and pg_dump points fall under its question about other in-server libpq callers.
- `replication-ondisk-1` Server-side libpq users such as walreceiver would also start tracing. Refuted (not specific): Duplicate of security-1 for the subscription owner case. The rest is an operator who sets a debug variable for the server and gets the trace that was asked for, which is not a failure.
- `security-2` Inherited PQTRACE writes passwords and row data to files. Refuted (no failure scenario): The user who sets PQTRACE owns the traced processes and asks for the trace, and umask applies as it does to pg_dump -f output. The trace content is what PQtrace() already prints today.
- `maintenance-fit-2` PQTRACE names break the PG prefix convention and become permanent. Refuted (not specific): Duplicate of standards-compat-2, which makes the same PG prefix point with a more concrete question.

### Review angles

- prior-attempts: 0 objection(s). P1's recorded reason concerns a fixed MAXPGPATH array in pg_conn. That is an implementation detail, and nothing in this proposal's prose shows the same choice. Message #1 points to an earlier thread about BackendKeyData tracing as the first posting of this patchset; the thread reports no rejection there.
- standards-compat: 3 objection(s). Three objections raised. The hex cancel key and new regress masks change trace output, but tracing is a debugging aid, so I did not count that as a compatibility break.
- replication-ondisk: 1 objection(s). The proposal changes no on-disk format, WAL record, catalog or wire message. The pg_conn change sits in libpq-int.h and is not public ABI. The only point in this angle is server processes that use libpq.
- security: 2 objection(s). Two security objections are raised. Both depend on code that was not shown, such as the option's dispchar and the file open mode, so they are provisional.
- maintenance-fit: 2 objection(s). No hot-path objection: the extra work runs only when tracing is enabled. No committer has reviewed the patches yet, which affects committability but not the verdict.
- already-exists: 0 objection(s). No existing feature does this job. Postgres has no way to start PQtrace() in an unmodified client such as psql. psql -L logs queries and results, not protocol messages. Wireshark with the PostgreSQL dissector needs capture privileges, does not see Unix-socket traffic, and needs TLS keys (sslkeylogfile). Its output also cannot serve as expected test output. For the libpq_pipeline part only, calling PQtrace() between PQconnectStartParams() and PQconnectPoll() could capture startup packets without a new option, but that does not help psql. No extension or hook covers this, so from this angle the gap is real.
- demand: 0 objection(s). Demand looks sufficient for a small developer tool: message #2 adds a second voice, and [P1] shows an earlier attempt to enable libpq tracing without recompiling. The in-tree libpq_pipeline test is a concrete consumer that gains startup-packet coverage. Non-users pay little, mostly two connection parameters and two environment variables, and the one design issue for this angle (the PQ prefix) belongs to other angles.

### Precedents retrieved

- P1: [libpq trace log](https://commitfest.postgresql.org/patch/1854/) (cf, Returned with feedback, 2019-09-30) - used in the review; reason summarised from its thread
- P2: [pg_tracing](https://commitfest.postgresql.org/patch/4456/) (cf, Withdrawn, 2024-07-10) - judged not relevant; reason summarised from its thread
- P3: [meson/testwrap: support EXTRA_REGRESS_OPTS environment variable](https://commitfest.postgresql.org/patch/6711/) (cf, Withdrawn, 2026-05-04) - judged not relevant; reason summarised from its thread
- P4: [Report tranche ID and LWLock pointer in LWLock static tracepoints](https://commitfest.postgresql.org/patch/2927/) (cf, Returned with feedback, 2021-04-07) - judged not relevant; reason summarised from its thread
- P5: [Make installcheck-world in a clean environment](https://commitfest.postgresql.org/patch/1672/) (cf, Withdrawn, 2019-07-30) - judged not relevant; reason summarised from its thread
- P6: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P7: [Transaction traceability - txid_convert_if_recent(bigint)](https://commitfest.postgresql.org/patch/748/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P8: [Add max_wal_replay_size connection parameter to libpq](https://commitfest.postgresql.org/patch/6630/) (cf, Withdrawn, 2026-07-23) - judged not relevant; reason summarised from its thread
- P9: [Revert removal of trace_userlocks, because userlocks aren't gone.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71b2b657c034743f81ae7906d7b0fcc8e9a0a6a3) (revert, Reverted, 2011-11-10) - judged not relevant
- P10: [Revert non-text output formats for pg_dumpall](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7ca548f23a60887541493b2da47f2b2720c35ee3) (revert, Reverted, 2026-06-18) - judged not relevant
- P11: [Revert "Don't truncate database and user names in startup packets."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0ff56e2d3ff1db3de727b33b2dac985ccc43ef8) (revert, Reverted, 2024-12-12) - judged not relevant
- P12: [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://commitfest.postgresql.org/patch/4166/) (cf, Withdrawn, 2024-01-30) - judged not relevant; reason summarised from its thread
- P13: [improve DEBUG1 logging of parallel workers for CREATE INDEX](https://commitfest.postgresql.org/patch/5490/) (cf, Withdrawn, 2025-02-03) - judged not relevant; reason summarised from its thread
- P14: [psql JSON output format](https://commitfest.postgresql.org/patch/4707/) (cf, Returned with feedback, 2024-05-15) - judged not relevant
- P15: [Add description about using geqo to EXPLAIN or debug log.](https://commitfest.postgresql.org/patch/3680/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P16: [Fix formatting of Interval output](https://commitfest.postgresql.org/patch/3548/) (cf, Withdrawn, 2022-02-19) - judged not relevant
- P17: [Batch (pipelined) mode for libpq](https://commitfest.postgresql.org/patch/634/) (cf, Returned with feedback, 2016-11-22) - judged not relevant
- P18: [TAP format output for pg_regress](https://commitfest.postgresql.org/patch/3559/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P19: [timeout parameters in libpq](https://commitfest.postgresql.org/patch/1843/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P20: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1024/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P21: [Revert 019_replslot_limit.pl related debugging aids.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3f8148c256e067dc2e8929ed174671ba7dc3339c) (revert, Reverted, 2022-07-05) - judged not relevant
- P22: [Revert setting client_min_messages to 'debug1' in new tests.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5655565c077c53b6e9b4b9bfcdf96439cf3af065) (revert, Reverted, 2019-03-13) - judged not relevant
- P23: [format_datum debugging function](https://commitfest.postgresql.org/patch/5165/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P24: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1317/) (cf, Returned with feedback, 2018-11-29) - judged not relevant

### Warnings

- review angle "maintenance-fit" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 149 s; cost reported by the CLI: $1.48
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 90 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:18:28.158Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
