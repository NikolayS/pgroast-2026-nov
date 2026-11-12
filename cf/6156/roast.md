# Design pre-review: Add read-only disconnect_requested GUC for graceful client disconnect (reviously goaway)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6156](https://commitfest.postgresql.org/patch/6156/); [thread](https://www.postgresql.org/message-id/flat/DDPQ1RV5FE9U.I2WW34NGRD8Z%40jeltef.nl), 22 of 22 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The goal has support in the thread, but two design points of the GUC version are open. standards-compat-1: the author named an opt-in as necessary in message #19 and weighed dropping it in message #21, so old poolers may forward the signal. replication-ondisk-2: message #21 does not say how a ParameterStatus reaches an idle connection, which is the main target. Confidence is low because the GUC design is in a thread this review cannot see.

Not checked: The GUC version and its thread are not visible, so its opt-in, its report timing and the fate of pg_goaway_backend() and PQgoAwayReceived() could not be checked. The backend idle-path cost also needs the code.

## Prior attempts

- [Add and report the new "session_read_only" GUC pseudo-variable.](https://www.postgresql.org/message-id/flat/1700970.cRWpxnom9y@hammer.magicstack.net) (similar idea; Returned with feedback, 2019-02-04). "Looking through the thread, it seems like there's a pretty fundamental design issue that hasn't been resolved, namely whether and how this ought to interact with default_transaction_read_only." ([message](https://www.postgresql.org/message-id/26312.1542046117%40sss.pgh.pa.us)) Different now: Both add a server-reported read-only GUC sent by ParameterStatus and set by a cross-backend signal. P11 stalled on how its GUC combined with default_transaction_read_only, which has no counterpart in a disconnect_requested flag.
- [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://www.postgresql.org/message-id/flat/CAFj8pRBFU-WzzQhNrwRHn67N0Ug8a9-0-9BOo69PPtcHiBDQMA@mail.gmail.com) (related; Withdrawn, 2024-01-30). "I think this is probably one of the core reasons why I would very much prefer GUCs over new message types to configure protocol extensions like this: It means proxies would not to keep track of and re-sync a new kind of connection state every time a protocol extension is added." ([message](https://www.postgresql.org/message-id/CAGECzQSg0ND%3DgRmpZHBsiaZT5-a17y%3DVFf3AQfi5N7oECtM1Ug%40mail.gmail.com)) Different now: P3 added a new protocol message for GUC_REPORT control. Its key argument prefers GUCs over new message types for proxies, which is the direction this proposal later took.
- [Client Connection redirection support for PostgreSQL](https://www.postgresql.org/message-id/flat/CY1PR21MB00246DE1F9E9C58455A78A37915C0@CY1PR21MB0024.namprd21.prod.outlook.com) (similar idea; Returned with feedback, 2018-10-02). reason not recorded Different now: P14 told clients where to reconnect. This proposal only asks them to disconnect when convenient, without a target. No reason is recorded for P14.

## Objections a reviewer is likely to raise

### 1. Reported GUC drops the opt-in that kept old poolers from forwarding

A site runs an existing PgBouncer or Pgpool-II in front of a server with this change, and its application driver learns to act on disconnect_requested. The pooler forwards the ParameterStatus, so the application closes its pooler connection while the pooler keeps the server connection. The replica drain does not speed up, and the extra client reconnects show up during the switchover.

In message #19 the author says a connection-level opt-in is needed so an old PgBouncer does not forward the signal by accident. In message #21 the author accepts losing that opt-in as "maybe not too bad" in exchange for using ParameterStatus. Once drivers react to a reported parameter, the project cannot add an opt-in later without breaking them. The material does not say whether the GUC version keeps any opt-in, for example through report_parameters or a protocol extension.

**Question:** Does the disconnect_requested version report the parameter only to clients that ask for it, and if not, what do clients behind current poolers see?

<sub>fixable, level 2, wire protocol; messages [#19](https://www.postgresql.org/message-id/CAGECzQTu%3DSXZxg5ODf3xzQ5_pZiY5mPBO30ZL_2-0LNfM%3DwHyA%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CAGECzQSbr%3DWDJDZ-fFTDRjY7udC6H9Sb3VHWk9BxWx9uTG27tg%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/20260331.184019.1086246580212079929.ishii%40postgresql.org); `standards-compat-1`</sub>

### 2. ParameterStatus sent while idle changes when this message can arrive

A pool keeps idle connections that see no query during the drain. If the server reports disconnect_requested only at the end of a query, those connections never learn of the request and the drain waits for the pool's maximum lifetime. If the server sends it while the backend is idle, it uses a timing that clients and proxies may not expect.

In message #21 the author noted that ParameterStatus is normally sent at the end of a query and said this "can be worked around somehow", without saying how. Sending it from an idle backend means a new send path in the idle loop, plus rules for messages that arrive between ReadyForQuery and the next command. Those rules matter to every client and proxy, not only to users of this feature. The material does not show either choice or its effect on idle pooled connections.

**Question:** When does the server send the disconnect_requested ParameterStatus to an idle backend's client, and which existing clients or proxies were checked against that timing?

<sub>fixable, level 2, wire protocol; messages [#21](https://www.postgresql.org/message-id/CAGECzQSbr%3DWDJDZ-fFTDRjY7udC6H9Sb3VHWk9BxWx9uTG27tg%40mail.gmail.com); `replication-ondisk-2`</sub>

### 3. No pooler or driver shown that would act on the signal

An operator starts a smart shutdown on a read replica to switch over. PgBouncer, JDBC pools and other drivers do not yet watch disconnect_requested, so they keep connections until the 1-hour lifetime. The drain takes as long as today, while Postgres still carries a new reported GUC and a libpq function.

The feature helps only when clients react to it, and the material shows no client that does. Kirill Reshke (message #2) wanted this for Odyssey in 2020. Message #4 points to one related patch, and message #14 suggests another trigger. Tomas Vondra asked for a pooler or client proof of concept (message #12), and the author offered to write one (message #15), but none appears in the visible thread. Pgpool-II's maintainer called it no problem for Pgpool-II (message #20), which is not a plan to use it.

**Question:** Which pooler or driver maintainers have committed to handling disconnect_requested, and can a working proof of concept show switchover drain time before and after?

<sub>fixable, level 1b, demand; messages [#2](https://www.postgresql.org/message-id/CALdSSPiORRiJ892J07tT3_xcAm%3DO9JCW-qHUQvFc6axHoDW_Og%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAER375Nugod9uuowwEUAai9PpoXP2xD-G54MW6hhjQw-6LtSLQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/f97528e4-5766-415e-b998-678da925ce05%40vondra.me), [#14](https://www.postgresql.org/message-id/CAMFBP2rSvyj6DYAw0EEBGR2UUP85ByKAC_D0djJm7rdypuMvdQ%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/DHB2ZT3ZN8L5.21CRG9GA9317G%40jeltef.nl), [#20](https://www.postgresql.org/message-id/20260331.081724.15339448107258985.ishii%40postgresql.org); `demand-1`</sub>

## Questions for the author

1. Does the disconnect_requested version report the parameter only to connections that ask for it, and what do clients behind a current PgBouncer or Pgpool-II see?
2. When does the server send the disconnect_requested ParameterStatus to a client whose backend is idle, and which clients and proxies were tested with that timing?
3. Which pooler or driver has a working change that reacts to disconnect_requested, and how much faster does a replica switchover drain with it?
4. Does pg_goaway_backend() remain in the GUC version, and does it use the same role checks as pg_terminate_backend()?
5. Can the disconnect_requested value later carry the deadline or priority that Tomas Vondra and Zsolt Parragi asked for, without breaking clients that read it as a boolean?

## Touch points

- **wire protocol:** The server sends a new signal to clients, first as a GoAway message and now probably as a ParameterStatus for disconnect_requested.
- **catalog changes:** v8 adds the SQL function pg_goaway_backend(), which needs a pg_proc entry.
- **ABI / extension API:** The v1 patch adds PQgoAwayReceived() to libpq's exports.txt.
- **new GUC:** The CommitFest title names a new read-only disconnect_requested GUC.
- **psql UX conventions:** psql logs an info line when the server asks the client to disconnect.
- **security / privileges:** pg_goaway_backend() acts on another session, and the visible material does not state its privilege rule.
- **backpatchability:** see objection standards-compat-1: Reported GUC drops the opt-in that kept old poolers from forwarding
- **maintenance burden ("who maintains this in 10 years"):** A client-visible parameter and a libpq function stay for good once drivers and poolers depend on them.
- **performance on the hot path:** The patch touches tcop/postgres.c and procsignal.c; the cost looks small but the code was not read.

## Who would commit it, and when

- Owner: silent. Someone who knows the frontend/backend protocol, libpq and ParameterStatus reporting would have to own it. Several reviewers replied, but the visible material shows no committer taking it on.
- Priority: fix design first; audience a sizeable minority; review cost medium. Sites that drain replicas or restart poolers gain, and the patch is small, but it adds a lasting client-visible contract. The opt-in and idle-timing questions need an answer first; the GUC version of the patch was not read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
