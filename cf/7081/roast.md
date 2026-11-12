# Design pre-review: Add PQTRACE env to enable protocol tracing in libpq

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7081](https://commitfest.postgresql.org/patch/7081/); [thread](https://www.postgresql.org/message-id/flat/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is stated and has support: message #2 gives a +1 and P1 shows an earlier attempt at the same need. No surviving objection questions the idea. Four fixable design points remain at level 2. The decisive one is security-1: trace_file opens a file, and dblink and postgres_fdw pass user-supplied options to libpq inside the server. standards-compat-2 and standards-compat-3 concern names and the stdout target, which are hard to change after release.

Not checked: The code was not read, so it is unknown whether trace_file is marked as a debug option, which dblink and postgres_fdw already hide. The file open mode and the handling of connection retries could not be checked either.

## Prior attempts

- [libpq trace log](https://www.postgresql.org/message-id/flat/71E660EB361DF14299875B198D4CE5423DE3FBA4@g01jpexmbkw25) (similar idea; Returned with feedback, 2019-09-30). "This seems to make pg_conn much larger than we'd like." ([message](https://www.postgresql.org/message-id/20190905222122.GA29713%40alvherre.pgsql)) Different now: P1 started as a query log enabled by connection parameters or environment variables, then narrowed to reworking PQtrace() output and deferred the no-recompile part. This proposal does that deferred part on top of the one-line-per-message format.

## Objections a reviewer is likely to raise

### 1. trace_file lets dblink and postgres_fdw users write server files

A non-superuser with EXECUTE on dblink_connect, or a postgres_fdw user mapping, passes trace_file=$PGDATA/postgresql.auto.conf. The backend opens that path as the postgres OS user, truncating or overwriting it. The admin finds out at the next restart or reload.

dblink and postgres_fdw run libpq inside the server and pass user-supplied connection options to it, so a libpq option that opens a file becomes a server-side file write. Today these modules hide only debug options and a short exclusion list. The patch prose does not say whether trace_file is marked as a debug option, nor does it mention dblink, postgres_fdw or other in-server libpq callers. The same issue applies to applications that accept connection URIs from end users.

**Question:** How is trace_file kept out of connection strings that dblink, postgres_fdw and other in-server or multi-tenant libpq callers accept from less privileged users?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com); `security-1`</sub>

### 2. Trace to stdout by default mixes protocol lines into program output

A developer exports PQTRACE=- in a shell to debug psql and later runs pg_dump > db.sql or psql -At -c ... | script in the same shell. Trace lines go into the dump or the piped data, and the user finds out when restore or the script fails.

An environment variable is inherited by every libpq program started afterwards, and many of them write their real output to stdout. Postgres client tools send diagnostics to stderr. The only special value the proposal offers is "-" for stdout, and message #1 shows it as the main usage.

**Question:** Why should the special value send traces to stdout and not to stderr, given pg_dump and psql scripts write data to stdout?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com); `standards-compat-3`</sub>

### 3. PQTRACE breaks the PG prefix and parameter-name pairing of libpq environment variables

A user who knows PGSSLMODE and PGAPPNAME sets PGTRACE or PGTRACEFILE, and nothing happens. Tools that clear or list libpq settings by matching PG* environment variables miss PQTRACE, so tracing leaks into their child processes.

Every libpq environment variable in the docs section the patch edits uses the PG prefix and mostly mirrors its parameter name, for example sslmode and PGSSLMODE. The patch pairs trace_file with PQTRACE and trace_flags with PQTRACEFLAGS. Message #1 and the commit message call the feature env vars only, while the doc hunk also adds connection parameters, so the thread has not settled the names. Once released, they cannot be renamed without breaking scripts.

**Question:** Why not use PGTRACEFILE and PGTRACEFLAGS, matching the parameter names and the existing PG prefix?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com); `standards-compat-2`</sub>

### 4. Startup packets in expected traces couple tests to build and server settings

A later patch marks a new GUC as GUC_REPORT, or changes SSL or GSS negotiation, and nine libpq_pipeline trace files fail. A buildfarm animal built with GSSAPI, or without SSL, sends a different startup sequence and goes red although nothing is broken.

Patch 0006 adds startup messages to nine expected trace files. Patch 0002 masks parameter values and packet length, but the sequence of messages stays in the expected output. That sequence depends on build options, sslmode, gssencmode, the protocol version and the set of reported GUCs. Every later patch that adds a reported parameter or a negotiation step must update these files, and the proposal does not say how build-dependent messages are kept out of the expected output.

**Question:** How do the expected traces stay stable across SSL and GSSAPI builds, and what must a patch that adds a GUC_REPORT setting update?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAO6_XqrOuYe8J1vOuErx0XarkF3e96fCO4fAHWOK-6amJ_UCew%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. How is trace_file kept out of connection strings that dblink, postgres_fdw and other in-server libpq callers accept from less privileged users?
2. Why does the special value "-" send the trace to stdout and not to stderr, given that pg_dump and psql scripts write data to stdout?
3. Why are the variables named PQTRACE and PQTRACEFLAGS, and not PGTRACEFILE and PGTRACEFLAGS to match the parameter names and the PG prefix?
4. How do the expected traces in libpq_pipeline stay the same across builds with and without SSL or GSSAPI, and when a new GUC_REPORT setting is added?
5. Does libpq truncate or append to the trace file, when does it close the file, and what happens on a retry with multiple hosts?

## Touch points

- **WAL / replication / standby:** walreceiver and logical replication workers connect through libpq, so they also process trace_file and PQTRACE.
- **wire protocol:** see objection maintenance-fit-1: Startup packets in expected traces couple tests to build and server settings
- **psql UX conventions:** PQTRACE="-" writes trace lines to stdout of psql and of every other libpq client that inherits the variable.
- **security / privileges:** trace_file makes libpq open a file named in a connection string, including strings that dblink and postgres_fdw accept from users.
- **maintenance burden ("who maintains this in 10 years"):** Two connection parameters and two environment variables become permanent, and nine expected trace files now include startup messages.

## Who would commit it, and when

- Owner: silent. A committer who knows libpq connection option handling and the in-server libpq callers (dblink, postgres_fdw, walreceiver) would have to own it. Nobody has reviewed the patches in the visible thread.
- Priority: fix design first; audience niche; review cost medium. The feature serves developers who debug the protocol or write tests, and it has a +1 in message #2. The series is six patches limited to libpq and one test module, but the file-write question in security-1 and the names should be settled before code review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
