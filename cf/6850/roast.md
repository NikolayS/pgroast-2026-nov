# Design pre-review: Don't consider FATAL errors "abnormal server termination"

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6850](https://commitfest.postgresql.org/patch/6850/); [thread](https://www.postgresql.org/message-id/flat/up7whuxn263agr7whdzzoak7mo7ywryc7z7z3ejdvgbbke4vji%403puckkgm4ide), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal: four participants agreed in 2025 that the appended text misleads, and demand is shown in message #1. Two fixable points remain. standards-compat-1 asks what clients that match the appended string should use instead. replication-ondisk-1 asks which libpq callers depend on the old PQstatus() timing, since v2 had to change postgres_fdw. Both need an answer from someone who knows libpq connection state, but neither requires a different proposal.

Not checked: The v2 commit message and the postgres_fdw change are not in the material, so the effect on other libpq callers could not be judged. Behaviour during startup, COPY and pipeline mode needs the code.

## Prior attempts

- [Revert commits 290acac92b and 8a7e9e9dad.](https://www.postgresql.org/message-id/flat/E1iaD8h-0004us-K9@gemulon.postgresql.org) (related; Reverted, 2019-11-30). "We found that on Windows, the client end of the socket is not able to read the data once we close the socket in the server which leads to loss of error message which is not what we expect." Different now: P1 removed tests that needed the client to read the FATAL after pg_terminate_backend() or a SIGTERM, because Windows can drop that message. This proposal changes libpq only when the FATAL does arrive, so where Windows drops it the old text stays.

## Objections a reviewer is likely to raise

### 1. Clients that match the appended string lose reconnect detection

An application uses a framework that detects a lost connection by matching "server closed the connection unexpectedly" and then reconnects. After its libpq is upgraded, a session ended by pg_terminate_backend() or a restart returns only the FATAL text. The framework treats it as an ordinary query error and does not retry, and operators find out from failed requests after a routine failover.

Message #3 lists several packages that parse this exact string to detect lost connections, and warns against changing it. The patch removes the string in the FATAL case, which is the usual case after an administrator command or a shutdown. The commit message argues the text is wrong, but it does not weigh what existing parsers lose or offer them a stable replacement. A committer would likely ask for this to be called out as a compatibility change and for the replacement to be documented.

**Question:** How should applications that match this string today detect an admin-terminated connection after the change, and will the release notes and docs say so?

<sub>fixable, level 1b, desirability; messages [#3](https://www.postgresql.org/message-id/Z-V2v_yqewKY51BS%40msg.df7cb.de), [#5](https://www.postgresql.org/message-id/DISFEF23Y30A.3H6YMBMPYVF8I%40jeltef.nl), [#7](https://www.postgresql.org/message-id/DJ60NPKYB9L9.3I3SS9I2MBL00%40jeltef.nl); `standards-compat-1`</sub>

### 2. Earlier CONNECTION_BAD changes libpq API behavior for existing callers

An extension, or libpqwalreceiver in a logical apply worker, receives a FATAL and then checks PQstatus() or keeps calling PQgetResult() as it did before. With the patch, PQstatus() reports CONNECTION_BAD before the socket closes, so the old code path changes. The extension author finds out when the extension's tests fail or its error reporting changes on PG20.

v2 had to change postgres_fdw (message #7), so at least one in-tree libpq consumer depended on the old ordering. The material does not say what that change is, or whether dblink, libpqwalreceiver and the logical replication workers were checked. Out-of-tree code built on libpq would need the same change, with no compile-time signal. The commit message, the only place where the postgres_fdw change is explained, is not in the material.

**Question:** Which caller pattern did postgres_fdw rely on, and have dblink, libpqwalreceiver and the logical replication workers been checked for the same pattern?

<sub>fixable, level 2, ABI / extension API; messages [#5](https://www.postgresql.org/message-id/DISFEF23Y30A.3H6YMBMPYVF8I%40jeltef.nl), [#7](https://www.postgresql.org/message-id/DJ60NPKYB9L9.3I3SS9I2MBL00%40jeltef.nl); `replication-ondisk-1`</sub>

## Questions for the author

1. How should applications that match "server closed the connection unexpectedly" detect an admin-terminated connection after the change, and will the docs and release notes say so?
2. What did postgres_fdw assume about PQstatus() after a FATAL, and have dblink, libpqwalreceiver and the logical replication workers been checked for the same assumption?
3. How does libpq behave when the FATAL arrives during connection startup, inside COPY, or in pipeline mode?
4. Does v2 pass the full test suite on Windows, where the client can lose the server's final FATAL message?
5. Is a change to the message and hint for real unexpected closes left for a separate patch?

## Touch points

- **WAL / replication / standby:** libpqwalreceiver and logical replication workers are libpq callers that see CONNECTION_BAD earlier after a FATAL.
- **wire protocol:** libpq reacts differently to an ErrorResponse with FATAL or PANIC severity; no message format changes.
- **ABI / extension API:** PQstatus() returns CONNECTION_BAD before the socket closes, and v2 had to change postgres_fdw for it.
- **backpatchability:** The entry is tagged Bugfix but changes error text and PQstatus() timing that deployed clients depend on; the author targets PG20 only.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund started the thread and reported the v1 test failures, but nobody has replied to v2. The owner needs to know libpq connection state handling and its in-tree callers.
- Priority: needs champion; audience a sizeable minority; review cost low. The goal has agreement and v1 touches one libpq file, but v2 has had no reply across three CommitFests. The v2 patch, including its postgres_fdw change, was not read, so the review cost is an estimate.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
