# Design pre-review: Tighten SCRAM iteration parsing and bound libpq PBKDF2 work

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7217](https://commitfest.postgresql.org/patch/7217/); [thread](https://www.postgresql.org/message-id/flat/CAH7T-aqV9KVuO2yCpzTM2E0Pz%3DyJOMhK1PyAGt%3Dod77_OztBVg%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The parsing fix and the connect_timeout check have reviewer support in message #2, but three design points are open. The default cap of 100000 rejects servers that the scram_iterations setting allows (standards-compat-1), and the cap overlaps with connect_timeout once 0003-0005 land (already-exists-1). The shape of the interrupt hook in src/common is unsettled (maintenance-fit-2). All are fixable, and 0006 can be dropped or made opt-in without harm to the rest.

Not checked: The code was not read. Not judged: what the backend does when the new parser rejects a stored secret, how connect_deadline is reset per host, and whether the 1-second TAP test is stable.

## Prior attempts

- [timeout parameters in libpq](https://www.postgresql.org/message-id/flat/EDA4195584F5064680D8130B1CA91C45367328@G01JPEXMBYT04) (related; Returned with feedback, 2018-11-30). "I'd like clarifications about the use case that needs this specific feature, especially to understand why the server-side "statement_timeout" setting is not right enough." ([message](https://www.postgresql.org/message-id/alpine.DEB.2.21.1811041235200.7070%40lancre)) Different now: P5 added new libpq timeout parameters for query and network waits. This proposal extends the existing connect_timeout to PBKDF2 work and adds an iteration cap, and message #3 gives a use case.

## Objections a reviewer is likely to raise

### 1. Default scram_max_iterations of 100000 breaks existing high-iteration servers

A site runs the server with scram_iterations = 500000, which the GUC allows. After a client-package upgrade, psql, pg_dump, postgres_fdw, dblink and a standby's walreceiver all fail with "exceeding scram_max_iterations". The site learns of it at the first reconnect after the upgrade, which may be a replication reconnect during an incident.

Message #1 says deployments with scram_iterations of 250K to 1M exist and would need the parameter set on libpq upgrade. Raising the iteration count is the documented way to harden stored secrets, so the users hit are the ones who followed that advice. Every libpq-based tool and each primary_conninfo or user mapping would need the new option. Gustafsson (message #2) is -1 on the parameter and warns that a wrong value rejects connections in error; message #3 answers with driver precedent, which does not cover this upgrade path.

**Question:** Why should the default reject servers that use iteration counts the scram_iterations GUC accepts, instead of 0 with connect_timeout as the default protection?

<sub>fixable, level 2, new GUC; messages [#1](https://www.postgresql.org/message-id/CAH7T-aqV9KVuO2yCpzTM2E0Pz%3DyJOMhK1PyAGt%3Dod77_OztBVg%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/836EAAF0-1765-4B0F-A145-123464A19255%40yesql.se), [#3](https://www.postgresql.org/message-id/CAH7T-areWmFFp2N58bx_fF-xJxZ5KPpPo8HThECoNJf7K_-z0w%40mail.gmail.com); `standards-compat-1`</sub>

### 2. The PBKDF2 interrupt hook is shaped for libpq, not shared code

src/common gains scram_SaltedPasswordExt(), a callback that receives a PGconn, and a frontend-only check interval. Later, another auth method or a frontend tool that derives SCRAM secrets needs a similar abort. It has to copy this pattern or change the signature again, and the old shim stays.

Message #2 objects to frontend-only parameters in code shared with the backend and to passing the whole PGconn, and it asks for something closer to CHECK_FOR_INTERRUPTS(). Message #3 asks whether a frontend example exists and offers to have the backend pass a CFI wrapper, but the design is not settled. Two entry points, scram_SaltedPassword() and scram_SaltedPasswordExt(), would then live in src/common indefinitely.

**Question:** What frontend interrupt mechanism, used the same way by backend and frontend callers, would replace the PGconn callback and the Ext shim?

<sub>fixable, level 2, ABI / extension API; messages [#2](https://www.postgresql.org/message-id/836EAAF0-1765-4B0F-A145-123464A19255%40yesql.se), [#3](https://www.postgresql.org/message-id/CAH7T-areWmFFp2N58bx_fF-xJxZ5KPpPo8HThECoNJf7K_-z0w%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Stricter secret parsing changes how existing role passwords are classified

A role has a stored secret such as 'SCRAM-SHA-256$+4096:...' or one with an out-of-range count, which strtol() accepted. pg_dumpall emits it, and the patched server restores it. If the parse fails, the server may store the whole string as a new plaintext password and hash it, so the old password stops working without any error.

0001 makes the backend reject leading signs, whitespace, zero and values above INT_MAX in stored secrets. The server classifies a password string by whether it parses as a SCRAM secret, so a stricter parser can change what CREATE ROLE ... PASSWORD and ALTER ROLE do with existing dump output. Message #1 calls this a behavior change and still asks whether to backpatch it. The proposal does not say how existing pg_authid rows and dumped secrets are treated.

**Question:** After 0001, what does CREATE ROLE ... PASSWORD do with a previously accepted secret that the new parser rejects, and should it raise an error instead of storing a plaintext password?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#1](https://www.postgresql.org/message-id/CAH7T-aqV9KVuO2yCpzTM2E0Pz%3DyJOMhK1PyAGt%3Dod77_OztBVg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAH7T-areWmFFp2N58bx_fF-xJxZ5KPpPo8HThECoNJf7K_-z0w%40mail.gmail.com); `standards-compat-2`</sub>

### 4. scram_max_iterations overlaps with connect_timeout once 0003-0005 land

A site runs scram_iterations at 500000 and upgrades libpq. Every client that has not set scram_max_iterations now fails to connect with the new error, even though connect_timeout, after 0003-0005, already bounds the PBKDF2 work for blocking clients that set a timeout.

With patches 0003-0005, connect_timeout bounds PBKDF2 on the blocking path, so for those clients the cap adds a second limit on the same work. The author's answer in message #1 is that the cap also covers clients with no timeout and async PQconnectPoll() callers. Clients without a timeout can set connect_timeout today, and async callers might be served by a deadline taken at PQconnectStart(). Message #2 is -1 on another niche connection parameter, and message #3 does not answer that point.

**Question:** Why can async PQconnectPoll() callers not be protected by extending connect_timeout or a connect-start deadline, instead of adding a separate parameter with a default that rejects existing high-iteration setups?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAH7T-aqV9KVuO2yCpzTM2E0Pz%3DyJOMhK1PyAGt%3Dod77_OztBVg%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/836EAAF0-1765-4B0F-A145-123464A19255%40yesql.se), [#3](https://www.postgresql.org/message-id/CAH7T-areWmFFp2N58bx_fF-xJxZ5KPpPo8HThECoNJf7K_-z0w%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Why can async PQconnectPoll() callers not be protected by a deadline taken at connection start, instead of a separate scram_max_iterations parameter?
2. If scram_max_iterations stays, why should its default reject servers whose scram_iterations is within the allowed range, instead of defaulting to 0?
3. What frontend interrupt mechanism, used the same way by backend and frontend callers, would replace the PGconn callback and the scram_SaltedPasswordExt() shim?
4. After 0001, what does CREATE ROLE ... PASSWORD do with a previously accepted secret that the new parser rejects: raise an error, or store it as a plaintext password?
5. Which of 0001 and 0003-0005 do you propose to backpatch, given that 0001 rejects stored secrets that current branches accept?

## Touch points

- **on-disk format / pg_upgrade:** 0001 changes which stored rolpassword values the backend accepts as SCRAM secrets.
- **wire protocol:** libpq parses the server-first-message more strictly and, with 0006, rejects iteration counts above the cap.
- **ABI / extension API:** 0004 adds scram_SaltedPasswordExt() with an interrupt callback to src/common; 0003 adds a field to PGconn.
- **new GUC:** 0006 adds the libpq option scram_max_iterations and the environment variable PGSCRAMMAXITERATIONS.
- **pg_dump / restore round-trip:** A dumped secret that strtol() accepted and the new parser rejects may restore differently.
- **security / privileges:** The series changes SCRAM secret validation and client behaviour against a hostile server.
- **backpatchability:** Message #1 asks whether 0001 and 0003-0005 should be backpatched; 0001 is a behavior change.
- **maintenance burden ("who maintains this in 10 years"):** A permanent connection option, an environment variable and a second PBKDF2 entry point in src/common.
- **performance on the hot path:** 0004 adds a periodic callback check inside the PBKDF2 loop that every SCRAM login runs in libpq.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** After 0003-0005, scram_max_iterations bounds the same work that connect_timeout already bounds on the blocking path.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson reviewed in message #2 and shows knowledge of the SCRAM iteration design. The owner must know libpq connection setup and the shared SCRAM code in src/common.
- Priority: fix design first; audience niche; review cost medium. The timeout protection serves services that connect to user-supplied hosts (message #3), and the parsing fix is hygiene that the reviewer supports. Six small patches touch libpq authentication and src/common, and the reviewer's points from message #2 on the hook shape and the parameter are still open.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 8 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
