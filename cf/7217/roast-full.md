# Design pre-review: Tighten SCRAM iteration parsing and bound libpq PBKDF2 work

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7217](https://commitfest.postgresql.org/patch/7217/) (Needs review); authors: Sehrope Sarkuni (sehrope)
- Thread: [Tighten SCRAM iteration parsing and bound libpq PBKDF2 work](https://www.postgresql.org/message-id/flat/CAH7T-aqV9KVuO2yCpzTM2E0Pz%3DyJOMhK1PyAGt%3Dod77_OztBVg%40mail.gmail.com), started 2026-05-24
- Messages read: 3 of 3
- Patch prose read: 0005-Add-test-for-connect_timeout-during-SCRAM-iteration.patch, 0004-libpq-honor-connect_timeout-during-SCRAM-iteration.patch, 0002-Add-test_scram-module-for-direct-unit-tests-of-SCRAM.patch, 0003-libpq-expose-connect-deadline-on-PGconn.patch, 0001-Fix-int-overflow-when-parsing-SCRAM-iteration-count.patch, 0006-libpq-add-scram_max_iterations-connection-parameter.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The parsing fix and the connect_timeout check have reviewer support in message #2, but three design points are open. The default cap of 100000 rejects servers that the scram_iterations setting allows (standards-compat-1), and the cap overlaps with connect_timeout once 0003-0005 land (already-exists-1). The shape of the interrupt hook in src/common is unsettled (maintenance-fit-2). All are fixable, and 0006 can be dropped or made opt-in without harm to the rest.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Not judged: what the backend does when the new parser rejects a stored secret, how connect_deadline is reset per host, and whether the 1-second TAP test is stable.

## The proposal, restated

**Problem.** The backend and libpq parse the SCRAM iteration count with strtol() into a long and store it in an int, so values above INT_MAX truncate and zero or negative counts pass. Separately, a server can advertise a huge iteration count and keep a libpq client in PBKDF2 past connect_timeout. This affects services that connect to user-supplied database hosts.

**Proposed design.** A new helper in src/common, scram_parse_iterations(), accepts only decimal digits in 1..INT_MAX, and both parsers use it. libpq copies the blocking connect deadline into PGconn, and the shared PBKDF2 loop calls an optional interrupt callback that aborts once that deadline passes. A new libpq option, scram_max_iterations (default 100000, 0 disables), rejects larger server counts before any PBKDF2 work starts.

**User-visible surface.**
- New libpq connection parameter scram_max_iterations and environment variable PGSCRAMMAXITERATIONS, default 100000.
- connect_timeout now also bounds SCRAM PBKDF2 work on the blocking path, with a new "connection timeout expired" error during authentication.
- New C function scram_SaltedPasswordExt() in src/common, which takes an interrupt callback; scram_SaltedPassword() keeps its signature.
- The backend rejects stored SCRAM secrets whose iteration count is zero, negative, non-decimal or above INT_MAX.
- New test module src/test/modules/test_scram with an SQL wrapper around scram_parse_iterations().

**What the author claims.**
- A stored secret with 2147483648 iterations, set through ALTER USER, overflows to -1 and skips the empty-password check.
- With a 10M iteration count and connect_timeout=1, psql took about 12.7 s before; it fails at once with the cap and after about 1 s with the cap disabled.
- pgjdbc and node-postgres ship a similar limit with a 100K default.
- The security list judged the client-side denial of service out of scope as a security issue.
- The truncation does not allow logins without the password.

**Thread state.** Daniel Gustafsson supports honoring connect_timeout during SCRAM but objects to frontend-only parameters in shared code and to passing PGconn to a callback. He is -1 on the new connection parameter and questions the 4096-iteration check interval; the author's reply has no response yet.

## 2. Prior attempts

### timeout parameters in libpq (related)

- **Link:** [https://commitfest.postgresql.org/patch/1843/](https://commitfest.postgresql.org/patch/1843/)
- **Thread:** [https://www.postgresql.org/message-id/flat/EDA4195584F5064680D8130B1CA91C45367328@G01JPEXMBYT04](https://www.postgresql.org/message-id/flat/EDA4195584F5064680D8130B1CA91C45367328@G01JPEXMBYT04)
- **Date:** decided 2018-11-30; started 2018-10-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Fabien COELHO, AYahorau(at)ibagroup(dot)eu (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Fabien COELHO's review asked why the server-side statement_timeout is not enough for socket_timeout. He also found that a timeout drops and resets the connection instead of cancelling the query, and that pqWait duplicates pqWaitTimed logic. The patch had no tests and no documentation. Andrei Yahorau warned that the pqWait change breaks callers that expect to wait forever; the author planned new patches after an absence, and the visible thread ends with the review.
- **Reason (quote):** "I'd like clarifications about the use case that needs this specific feature, especially to understand why the server-side "statement_timeout" setting is not right enough." ([message](https://www.postgresql.org/message-id/alpine.DEB.2.21.1811041235200.7070%40lancre))
- **What is different this time:** P5 added new libpq timeout parameters for query and network waits. This proposal extends the existing connect_timeout to PBKDF2 work and adds an iteration cap, and message #3 gives a use case.


## 3. Objections a reviewer is likely to raise

### 1. Default scram_max_iterations of 100000 breaks existing high-iteration servers

fixable · level 2, rubric item 10 · new GUC, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A site runs the server with scram_iterations = 500000, which the GUC allows. After a client-package upgrade, psql, pg_dump, postgres_fdw, dblink and a standby's walreceiver all fail with "exceeding scram_max_iterations". The site learns of it at the first reconnect after the upgrade, which may be a replication reconnect during an incident.

Message #1 says deployments with scram_iterations of 250K to 1M exist and would need the parameter set on libpq upgrade. Raising the iteration count is the documented way to harden stored secrets, so the users hit are the ones who followed that advice. Every libpq-based tool and each primary_conninfo or user mapping would need the new option. Gustafsson (message #2) is -1 on the parameter and warns that a wrong value rejects connections in error; message #3 answers with driver precedent, which does not cover this upgrade path.

Thread: [#1 (Sehrope Sarkuni, 2026-05-24)](https://www.postgresql.org/message-id/CAH7T-aqV9KVuO2yCpzTM2E0Pz%3DyJOMhK1PyAGt%3Dod77_OztBVg%40mail.gmail.com), [#2 (Daniel Gustafsson, 2026-05-25)](https://www.postgresql.org/message-id/836EAAF0-1765-4B0F-A145-123464A19255%40yesql.se), [#3 (Sehrope Sarkuni, 2026-05-25)](https://www.postgresql.org/message-id/CAH7T-areWmFFp2N58bx_fF-xJxZ5KPpPo8HThECoNJf7K_-z0w%40mail.gmail.com)

**Question:** Why should the default reject servers that use iteration counts the scram_iterations GUC accepts, instead of 0 with connect_timeout as the default protection?

Counter-check: held. Message #1 itself says deployments with scram_iterations of 250K to 1M exist and would need the parameter set on libpq upgrade. The thread has no answer to Gustafsson's point that a wrong value rejects valid connections, and a default of 0 would avoid the break.

### 2. The PBKDF2 interrupt hook is shaped for libpq, not shared code

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

src/common gains scram_SaltedPasswordExt(), a callback that receives a PGconn, and a frontend-only check interval. Later, another auth method or a frontend tool that derives SCRAM secrets needs a similar abort. It has to copy this pattern or change the signature again, and the old shim stays.

Message #2 objects to frontend-only parameters in code shared with the backend and to passing the whole PGconn, and it asks for something closer to CHECK_FOR_INTERRUPTS(). Message #3 asks whether a frontend example exists and offers to have the backend pass a CFI wrapper, but the design is not settled. Two entry points, scram_SaltedPassword() and scram_SaltedPasswordExt(), would then live in src/common indefinitely.

Thread: [#2 (Daniel Gustafsson, 2026-05-25)](https://www.postgresql.org/message-id/836EAAF0-1765-4B0F-A145-123464A19255%40yesql.se), [#3 (Sehrope Sarkuni, 2026-05-25)](https://www.postgresql.org/message-id/CAH7T-areWmFFp2N58bx_fF-xJxZ5KPpPo8HThECoNJf7K_-z0w%40mail.gmail.com)

**Question:** What frontend interrupt mechanism, used the same way by backend and frontend callers, would replace the PGconn callback and the Ext shim?

Counter-check: held. Gustafsson raised both points in message #2, and message #3 answers with a question and two alternatives. The thread settles neither the callback shape nor the Ext shim, so the internal API is still open.

### 3. Stricter secret parsing changes how existing role passwords are classified

fixable · level 2, rubric item 6 · pg_dump / restore round-trip, backpatchability, security / privileges · `standards-compat-2`

A role has a stored secret such as 'SCRAM-SHA-256$+4096:...' or one with an out-of-range count, which strtol() accepted. pg_dumpall emits it, and the patched server restores it. If the parse fails, the server may store the whole string as a new plaintext password and hash it, so the old password stops working without any error.

0001 makes the backend reject leading signs, whitespace, zero and values above INT_MAX in stored secrets. The server classifies a password string by whether it parses as a SCRAM secret, so a stricter parser can change what CREATE ROLE ... PASSWORD and ALTER ROLE do with existing dump output. Message #1 calls this a behavior change and still asks whether to backpatch it. The proposal does not say how existing pg_authid rows and dumped secrets are treated.

Thread: [#1 (Sehrope Sarkuni, 2026-05-24)](https://www.postgresql.org/message-id/CAH7T-aqV9KVuO2yCpzTM2E0Pz%3DyJOMhK1PyAGt%3Dod77_OztBVg%40mail.gmail.com), [#3 (Sehrope Sarkuni, 2026-05-25)](https://www.postgresql.org/message-id/CAH7T-areWmFFp2N58bx_fF-xJxZ5KPpPo8HThECoNJf7K_-z0w%40mail.gmail.com)

**Question:** After 0001, what does CREATE ROLE ... PASSWORD do with a previously accepted secret that the new parser rejects, and should it raise an error instead of storing a plaintext password?

Counter-check: held. The backend treats a string that fails parse_scram_secret() as a plaintext password, so create role or alter role hashes it on restore. A hand-written secret such as '$+4096:' works today and would stop working without an error. Such secrets are rare, so fixable.

### 4. scram_max_iterations overlaps with connect_timeout once 0003-0005 land

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), new GUC, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A site runs scram_iterations at 500000 and upgrades libpq. Every client that has not set scram_max_iterations now fails to connect with the new error, even though connect_timeout, after 0003-0005, already bounds the PBKDF2 work for blocking clients that set a timeout.

With patches 0003-0005, connect_timeout bounds PBKDF2 on the blocking path, so for those clients the cap adds a second limit on the same work. The author's answer in message #1 is that the cap also covers clients with no timeout and async PQconnectPoll() callers. Clients without a timeout can set connect_timeout today, and async callers might be served by a deadline taken at PQconnectStart(). Message #2 is -1 on another niche connection parameter, and message #3 does not answer that point.

Thread: [#1 (Sehrope Sarkuni, 2026-05-24)](https://www.postgresql.org/message-id/CAH7T-aqV9KVuO2yCpzTM2E0Pz%3DyJOMhK1PyAGt%3Dod77_OztBVg%40mail.gmail.com), [#2 (Daniel Gustafsson, 2026-05-25)](https://www.postgresql.org/message-id/836EAAF0-1765-4B0F-A145-123464A19255%40yesql.se), [#3 (Sehrope Sarkuni, 2026-05-25)](https://www.postgresql.org/message-id/CAH7T-areWmFFp2N58bx_fF-xJxZ5KPpPo8HThECoNJf7K_-z0w%40mail.gmail.com)

**Question:** Why can async PQconnectPoll() callers not be protected by extending connect_timeout or a connect-start deadline, instead of adding a separate parameter with a default that rejects existing high-iteration setups?

Counter-check: held. Once 0003-0005 land, the cap adds protection only for callers without a timeout and for async callers. The thread does not show that this gap justifies a permanent parameter, and Gustafsson's -1 on it has no reply on that point.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | 0001 changes which stored rolpassword values the backend accepts as SCRAM secrets. |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | libpq parses the server-first-message more strictly and, with 0006, rejects iteration counts above the cap. |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0004 adds scram_SaltedPasswordExt() with an interrupt callback to src/common; 0003 adds a field to PGconn. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | 0006 adds the libpq option scram_max_iterations and the environment variable PGSCRAMMAXITERATIONS. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | A dumped secret that strtol() accepted and the new parser rejects may restore differently. |
| psql UX conventions | no |  |
| security / privileges | **yes** | The series changes SCRAM secret validation and client behaviour against a hostile server. |
| backpatchability | **yes** | Message #1 asks whether 0001 and 0003-0005 should be backpatched; 0001 is a behavior change. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A permanent connection option, an environment variable and a second PBKDF2 entry point in src/common. |
| performance on the hot path | **yes** | 0004 adds a periodic callback check inside the PBKDF2 loop that every SCRAM login runs in libpq. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | After 0003-0005, scram_max_iterations bounds the same work that connect_timeout already bounds on the blocking path. |

## 5. Questions the author must answer before anyone reviews code

1. Why can async PQconnectPoll() callers not be protected by a deadline taken at connection start, instead of a separate scram_max_iterations parameter?
2. If scram_max_iterations stays, why should its default reject servers whose scram_iterations is within the allowed range, instead of defaulting to 0?
3. What frontend interrupt mechanism, used the same way by backend and frontend callers, would replace the PGconn callback and the scram_SaltedPasswordExt() shim?
4. After 0001, what does CREATE ROLE ... PASSWORD do with a previously accepted secret that the new parser rejects: raise an error, or store it as a plaintext password?
5. Which of 0001 and 0003-0005 do you propose to backpatch, given that 0001 rejects stored secrets that current branches accept?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson reviewed in message #2 and shows knowledge of the SCRAM iteration design. The owner must know libpq connection setup and the shared SCRAM code in src/common.
- **Liability:** The owner signs for a stricter stored-secret parser on every branch that gets 0001, a new interrupt API in src/common, and, if 0006 goes in, a connection option and default that libpq must keep for good.
- **Author follow-up:** The author replied within a day with a reproduction (message #3), moved the entry to the next CommitFest in October, and says he added similar limits to pgjdbc and node-postgres.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The timeout protection serves services that connect to user-supplied hosts (message #3), and the parsing fix is hygiene that the reviewer supports. Six small patches touch libpq authentication and src/common, and the reviewer's points from message #2 on the hook shape and the parameter are still open.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Stricter secret parsing may turn existing pg_authid entries into plaintext passwords. Refuted (not specific): Duplicate of standards-compat-2 for the dump and restore path. The login claim is incorrect: the server never accepts a stored value as a plaintext password, so an existing row that fails the parse makes authentication fail.
- `replication-ondisk-2` Default 100000 cap rejects valid server-first-messages from existing servers. Refuted (not specific): Duplicate of standards-compat-1. It describes the same upgrade break for a server with scram_iterations = 500000 and asks the same question about a default of 0.
- `security-1` Default cap of 100000 penalizes servers that chose stronger SCRAM hashing. Refuted (not specific): Duplicate of standards-compat-1. The added point that operators may lower scram_iterations or disable the cap everywhere is a consequence of the same default.
- `security-2` Attacker who supplies the connection string can disable both protections. Refuted (no failure scenario): The scenario ends in the behaviour libpq has today, so the patch does not make it worse. An application that passes a full user-written URI to libpq must already override options such as connect_timeout, and the missing documentation sentence is outside levels 1-2.
- `maintenance-fit-1` scram_max_iterations adds a permanent option with a breaking default. Refuted (not specific): Duplicate of standards-compat-1 for the breaking default and of already-exists-1 for the overlap with connect_timeout. It adds no separate failure.
- `maintenance-fit-3` Deadline on PGconn makes connect_timeout depend on the connection path. Refuted (wrong facts): libpq applies connect_timeout only in the blocking connect path today, and PQconnectPoll() callers run their own timeouts. The patch follows that existing split and states it, and the duty on future auth methods is speculative.
- `demand-1` Default scram_max_iterations of 100000 makes users who never opted in pay. Refuted (not specific): Duplicate of standards-compat-1. The who-pays argument is the same default-of-100000 break, with the same proposed remedy of a default of 0.
- `demand-2` Demand for scram_max_iterations rests on one class of application. Refuted (not specific): Duplicate of already-exists-1, which states the same narrow remaining gap after 0003-0005. The material also shows some demand, since pgjdbc and node-postgres ship the same limit.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposed the same idea. P5 only asked why an existing setting was not enough, and message #3 gives a use case and explains why connect_timeout alone does not cover clients without a timeout or async callers. P22 has no recorded reason. The thread mentions a security-list report and similar pgjdbc and node-postgres limits (message #3), but no earlier pgsql-hackers attempt.
- standards-compat: 2 objection(s). The connect_timeout change (0003-0005) adds no user-facing syntax and only makes an existing timeout cover more work, so it raises no compatibility objection from this angle.
- replication-ondisk: 2 objection(s). There are no WAL, replication, on-disk heap or index changes. The PGconn field is in libpq-int.h and scram_SaltedPassword() keeps its signature, so I see no ABI break.
- security: 2 objection(s). Patch 0001, the parsing fix, closes a real gap: per message #3, an overflowing iteration count skips the empty-password check. From the security angle, 0001 needs no objection.
- maintenance-fit: 3 objection(s). The parsing fix in 0001 and the test module in 0002 raise no maintenance or fit concern from this angle.
- already-exists: 1 objection(s). Nothing in Postgres caps the server-advertised iteration count today, and 0001 and 0003-0005 fix or extend existing mechanisms, so this angle raises nothing against them.
- demand: 2 objection(s). The parsing fix (0001) and honoring connect_timeout during SCRAM (0003-0005) have reviewer support in message #2, so this review raises no demand objection against them.

### Precedents retrieved

- P1: [Channel binding support for SCRAM-SHA-256](https://commitfest.postgresql.org/patch/1153/) (cf, Returned with feedback, 2017-09-26) - judged not relevant; reason summarised from its thread
- P2: [Support for SCRAM-SHA-256](https://commitfest.postgresql.org/patch/683/) (cf, Returned with feedback, 2016-09-29) - judged not relevant; reason summarised from its thread
- P3: [Authorization facility for multiple password verifiers + SCRAM-SHA1](https://commitfest.postgresql.org/patch/320/) (cf, Returned with feedback, 2015-12-01) - judged not relevant; reason summarised from its thread
- P4: [de-TOAST'ing using a iterator](https://commitfest.postgresql.org/patch/2194/) (cf, Returned with feedback, 2020-11-27) - judged not relevant; reason summarised from its thread
- P5: [timeout parameters in libpq](https://commitfest.postgresql.org/patch/1843/) (cf, Returned with feedback, 2018-11-30) - used in the review; reason summarised from its thread
- P6: [Support for SCRAM-SHA-256](https://commitfest.postgresql.org/patch/829/) (cf, Returned with feedback, 2017-01-23) - judged not relevant; reason summarised from its thread
- P7: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P8: [User functions for building SCRAM secrets](https://commitfest.postgresql.org/patch/3988/) (cf, Returned with feedback, 2024-01-27) - judged not relevant; reason summarised from its thread
- P9: [iteration over record in plpgsql](https://commitfest.postgresql.org/patch/124/) (cf, Returned with feedback, 2015-07-07) - judged not relevant
- P10: [automatic client-side column-level encryption](https://commitfest.postgresql.org/patch/3718/) (cf, Returned with feedback, 2024-06-27) - judged not relevant
- P11: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P12: [Allow specifying multiple hosts in libpq connect options](https://commitfest.postgresql.org/patch/222/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P13: [Revert changes for SSL compression in libpq](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0ba71107efeeccde9158f47118f95043afdca0bb) (revert, Reverted, 2021-03-10) - judged not relevant
- P14: [Respect client-initiated CopyDone during logical streaming replication](https://commitfest.postgresql.org/patch/621/) (cf, Withdrawn, 2019-07-08) - judged not relevant
- P15: [add DETAIL for wrong SCRAM password](https://commitfest.postgresql.org/patch/3022/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P16: [Revert "Add libpq function PQhostaddr()."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2cda889984a6f5ad405318f9e91202d258383b66) (revert, Reverted, 2014-11-29) - judged not relevant
- P17: [Client Connection redirection support for PostgreSQL](https://commitfest.postgresql.org/patch/1673/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P18: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant
- P19: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - judged not relevant
- P20: [Windows Service status is set to "SERVICE_RUNNING" when timeout is occurred (version 9.6 or earlier).](https://commitfest.postgresql.org/patch/1970/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P21: [Send numeric version to clients (server_version_num GUC_REPORT)](https://commitfest.postgresql.org/patch/752/) (cf, Rejected, 2016-08-30) - judged not relevant
- P22: [standardize integer parsing in front-end utilities](https://commitfest.postgresql.org/patch/2197/) (cf, Withdrawn, 2020-03-05) - used in the review
- P23: [Problem during Windows service start](https://commitfest.postgresql.org/patch/2087/) (cf, Withdrawn, 2019-09-05) - judged not relevant
- P24: [hostorder and failover_timeout for libpq](https://commitfest.postgresql.org/patch/1631/) (cf, Returned with feedback, 2019-02-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 233 s; cost reported by the CLI: $3.16
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 110 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:29:50.254Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
