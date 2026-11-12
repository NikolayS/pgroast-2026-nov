# Design pre-review: Add read-only disconnect_requested GUC for graceful client disconnect (reviously goaway)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6156](https://commitfest.postgresql.org/patch/6156/) (Needs review); authors: Jelte Fennema-Nio (jeltef)
- Thread: [Add GoAway protocol message for graceful but fast server shutdown/switchover](https://www.postgresql.org/message-id/flat/DDPQ1RV5FE9U.I2WW34NGRD8Z%40jeltef.nl), started 2025-10-23
- Messages read: 22 of 22
- Patch prose read: nocfbot.v1-0003-Add-pytest-based-tests-for-GoAway-message.patch, v1-0001-Bump-protocol-version-to-3.3.patch, v1-0002-Add-GoAway-protocol-message-for-graceful-but-fast.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The goal has support in the thread, but two design points of the GUC version are open. standards-compat-1: the author named an opt-in as necessary in message #19 and weighed dropping it in message #21, so old poolers may forward the signal. replication-ondisk-2: message #21 does not say how a ParameterStatus reaches an idle connection, which is the main target. Confidence is low because the GUC design is in a thread this review cannot see.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The GUC version and its thread are not visible, so its opt-in, its report timing and the fate of pg_goaway_backend() and PQgoAwayReceived() could not be checked. The backend idle-path cost also needs the code.

## The proposal, restated

**Problem.** Draining connections from a server in smart shutdown can take as long as the pool's maximum connection lifetime, often 1 hour, even when every transaction is short. This slows read replica switchovers and pooler restarts. Connection poolers, proxies and drivers that keep pools are the main audience.

**Proposed design.** The server tells each client that it should disconnect when convenient, and the connection keeps working. Postgres sends this signal to all regular backends' clients when smart shutdown starts, and v8 also adds a function to signal a single backend. The design moved from a new protocol message under protocol 3.3, to a protocol extension, and per the CommitFest title now to a read-only GUC.

**User-visible surface.**
- GoAway backend message (byte 'g', no payload), at first under protocol 3.3, later behind an opt-in protocol extension (goaway or _pq_ name).
- libpq function PQgoAwayReceived(const PGconn *conn), returning 1 after a GoAway message arrives.
- SQL function pg_goaway_backend() to send the request to one backend (added in v8).
- psql logs an info line when the server sent GoAway.
- Read-only disconnect_requested GUC, named only in the CommitFest title; probably reported through ParameterStatus.

**What the author claims.**
- Clients that react to the signal let smart shutdown drain in much less time than the pool's maximum lifetime.
- Smart shutdown followed by fast shutdown after about 1 second lets most clients disconnect outside a query, so fewer errors and only added latency.
- No payload is needed, because clients have nothing to do between closing now and closing when idle.
- A NOTICE does not work, because client_min_messages can suppress it.
- An opt-in at connection level stops old poolers from forwarding the signal and dropping clients without need.

**Thread state.** Kirill Reshke, Zsolt Parragi, Tomas Vondra, Jacob Champion, Jim Nasby and Tatsuo Ishii replied; the last message is from 2026-03-31, and the author was weighing ParameterStatus. The GUC version lives in a separate attached thread (DJVLCL7PV7U6...) that is not visible, and Andrey Borodin set Needs review in September.

## 2. Prior attempts

### Add and report the new "session_read_only" GUC pseudo-variable. (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1090/](https://commitfest.postgresql.org/patch/1090/)
- **Thread:** [https://www.postgresql.org/message-id/flat/1700970.cRWpxnom9y@hammer.magicstack.net](https://www.postgresql.org/message-id/flat/1700970.cRWpxnom9y@hammer.magicstack.net)
- **Date:** decided 2019-02-04; started 2017-03-17
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane found an unresolved design issue: session_read_only combines default_transaction_read_only and RecoveryInProgress(), and the patch does not track intra-session changes to DefaultXactReadOnly. He asked for two separate GUC_REPORT variables, a new name and SIGHUP in place of the new signaling mechanism. The author agreed but sent no new version, and the entry was returned with feedback after that.
- **Reason (quote):** "Looking through the thread, it seems like there's a pretty fundamental design issue that hasn't been resolved, namely whether and how this ought to interact with default_transaction_read_only." ([message](https://www.postgresql.org/message-id/26312.1542046117%40sss.pgh.pa.us))
- **What is different this time:** Both add a server-reported read-only GUC sent by ParameterStatus and set by a cross-backend signal. P11 stalled on how its GUC combined with default_transaction_read_only, which has no counterpart in a disconnect_requested flag.

### Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user (related)

- **Link:** [https://commitfest.postgresql.org/patch/4166/](https://commitfest.postgresql.org/patch/4166/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAFj8pRBFU-WzzQhNrwRHn67N0Ug8a9-0-9BOo69PPtcHiBDQMA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAFj8pRBFU-WzzQhNrwRHn67N0Ug8a9-0-9BOo69PPtcHiBDQMA@mail.gmail.com)
- **Date:** decided 2024-01-30; started 2023-02-03
- **Outcome:** Withdrawn (cf)
- **Who objected:** Peter Eisentraut, Jelte Fennema-Nio (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the patch and said he would resend it once Jelte Fennema-Nio's _pq_.report_parameters patch is committed. Peter Eisentraut suggested a GUC that lists the settings to report instead of a new protocol message. Jelte Fennema-Nio argued that proxies such as PgBouncer already re-sync GUCs, so a GUC-based design is less fragile for them than a new message type with its own connection state.
- **Reason (quote):** "I think this is probably one of the core reasons why I would very much prefer GUCs over new message types to configure protocol extensions like this: It means proxies would not to keep track of and re-sync a new kind of connection state every time a protocol extension is added." ([message](https://www.postgresql.org/message-id/CAGECzQSg0ND%3DgRmpZHBsiaZT5-a17y%3DVFf3AQfi5N7oECtM1Ug%40mail.gmail.com))
- **What is different this time:** P3 added a new protocol message for GUC_REPORT control. Its key argument prefers GUCs over new message types for proxies, which is the direction this proposal later took.

### Client Connection redirection support for PostgreSQL (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1673/](https://commitfest.postgresql.org/patch/1673/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CY1PR21MB00246DE1F9E9C58455A78A37915C0@CY1PR21MB0024.namprd21.prod.outlook.com](https://www.postgresql.org/message-id/flat/CY1PR21MB00246DE1F9E9C58455A78A37915C0@CY1PR21MB0024.namprd21.prod.outlook.com)
- **Date:** decided 2018-10-02; started 2017-11-02
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P14 told clients where to reconnect. This proposal only asks them to disconnect when convenient, without a target. No reason is recorded for P14.


## 3. Objections a reviewer is likely to raise

### 1. Reported GUC drops the opt-in that kept old poolers from forwarding

fixable · level 2, rubric item 6 · wire protocol, new GUC, backpatchability · `standards-compat-1`

A site runs an existing PgBouncer or Pgpool-II in front of a server with this change, and its application driver learns to act on disconnect_requested. The pooler forwards the ParameterStatus, so the application closes its pooler connection while the pooler keeps the server connection. The replica drain does not speed up, and the extra client reconnects show up during the switchover.

In message #19 the author says a connection-level opt-in is needed so an old PgBouncer does not forward the signal by accident. In message #21 the author accepts losing that opt-in as "maybe not too bad" in exchange for using ParameterStatus. Once drivers react to a reported parameter, the project cannot add an opt-in later without breaking them. The material does not say whether the GUC version keeps any opt-in, for example through report_parameters or a protocol extension.

Thread: [#19 (Jelte Fennema-Nio, 2026-03-30)](https://www.postgresql.org/message-id/CAGECzQTu%3DSXZxg5ODf3xzQ5_pZiY5mPBO30ZL_2-0LNfM%3DwHyA%40mail.gmail.com), [#21 (Jelte Fennema-Nio, 2026-03-31)](https://www.postgresql.org/message-id/CAGECzQSbr%3DWDJDZ-fFTDRjY7udC6H9Sb3VHWk9BxWx9uTG27tg%40mail.gmail.com), [#22 (Tatsuo Ishii, 2026-03-31)](https://www.postgresql.org/message-id/20260331.184019.1086246580212079929.ishii%40postgresql.org)

**Question:** Does the disconnect_requested version report the parameter only to clients that ask for it, and if not, what do clients behind current poolers see?

Counter-check: held. The author raised this forwarding problem in message #19 and left it open in message #21. The visible material does not say whether the GUC version reports only to clients that ask.

### 2. ParameterStatus sent while idle changes when this message can arrive

fixable · level 2, rubric item 12 · wire protocol, new GUC · `replication-ondisk-2`

A pool keeps idle connections that see no query during the drain. If the server reports disconnect_requested only at the end of a query, those connections never learn of the request and the drain waits for the pool's maximum lifetime. If the server sends it while the backend is idle, it uses a timing that clients and proxies may not expect.

In message #21 the author noted that ParameterStatus is normally sent at the end of a query and said this "can be worked around somehow", without saying how. Sending it from an idle backend means a new send path in the idle loop, plus rules for messages that arrive between ReadyForQuery and the next command. Those rules matter to every client and proxy, not only to users of this feature. The material does not show either choice or its effect on idle pooled connections.

Thread: [#21 (Jelte Fennema-Nio, 2026-03-31)](https://www.postgresql.org/message-id/CAGECzQSbr%3DWDJDZ-fFTDRjY7udC6H9Sb3VHWk9BxWx9uTG27tg%40mail.gmail.com)

**Question:** When does the server send the disconnect_requested ParameterStatus to an idle backend's client, and which existing clients or proxies were checked against that timing?

Counter-check: held. Message #21 names the end-of-query timing problem and gives no solution, and idle pooled connections are the main target. The "unexpected timing" half is weaker, because the protocol documentation already tells frontends to accept ParameterStatus at any time.

### 3. No pooler or driver shown that would act on the signal

fixable · level 1b, rubric item 3 · demand, wire protocol, new GUC, maintenance burden ("who maintains this in 10 years") · `demand-1`

An operator starts a smart shutdown on a read replica to switch over. PgBouncer, JDBC pools and other drivers do not yet watch disconnect_requested, so they keep connections until the 1-hour lifetime. The drain takes as long as today, while Postgres still carries a new reported GUC and a libpq function.

The feature helps only when clients react to it, and the material shows no client that does. Kirill Reshke (message #2) wanted this for Odyssey in 2020. Message #4 points to one related patch, and message #14 suggests another trigger. Tomas Vondra asked for a pooler or client proof of concept (message #12), and the author offered to write one (message #15), but none appears in the visible thread. Pgpool-II's maintainer called it no problem for Pgpool-II (message #20), which is not a plan to use it.

Thread: [#2 (Kirill Reshke, 2025-10-24)](https://www.postgresql.org/message-id/CALdSSPiORRiJ892J07tT3_xcAm%3DO9JCW-qHUQvFc6axHoDW_Og%40mail.gmail.com), [#4 (Ajit Awekar, 2025-12-01)](https://www.postgresql.org/message-id/CAER375Nugod9uuowwEUAai9PpoXP2xD-G54MW6hhjQw-6LtSLQ%40mail.gmail.com), [#12 (Tomas Vondra, 2026-03-20)](https://www.postgresql.org/message-id/f97528e4-5766-415e-b998-678da925ce05%40vondra.me), [#14 (Jim Nasby, 2026-03-23)](https://www.postgresql.org/message-id/CAMFBP2rSvyj6DYAw0EEBGR2UUP85ByKAC_D0djJm7rdypuMvdQ%40mail.gmail.com), [#15 ("Jelte Fennema-Nio", 2026-03-24)](https://www.postgresql.org/message-id/DHB2ZT3ZN8L5.21CRG9GA9317G%40jeltef.nl), [#20 (Tatsuo Ishii, 2026-03-30)](https://www.postgresql.org/message-id/20260331.081724.15339448107258985.ishii%40postgresql.org)

**Question:** Which pooler or driver maintainers have committed to handling disconnect_requested, and can a working proof of concept show switchover drain time before and after?

Counter-check: held. Tomas Vondra asked for a pooler or client proof of concept in message #12, and the visible thread shows none after the offer in message #15. Messages #2, #4 and #14 show interest, so this is a request for evidence and no reason to reject.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | The server sends a new signal to clients, first as a GoAway message and now probably as a ParameterStatus for disconnect_requested. |
| catalog changes | **yes** | v8 adds the SQL function pg_goaway_backend(), which needs a pg_proc entry. |
| ABI / extension API | **yes** | The v1 patch adds PQgoAwayReceived() to libpq's exports.txt. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The CommitFest title names a new read-only disconnect_requested GUC. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | psql logs an info line when the server asks the client to disconnect. |
| security / privileges | **yes** | pg_goaway_backend() acts on another session, and the visible material does not state its privilege rule. |
| backpatchability | **yes** | see objection standards-compat-1: Reported GUC drops the opt-in that kept old poolers from forwarding |
| maintenance burden ("who maintains this in 10 years") | **yes** | A client-visible parameter and a libpq function stay for good once drivers and poolers depend on them. |
| performance on the hot path | **yes** | The patch touches tcop/postgres.c and procsignal.c; the cost looks small but the code was not read. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the disconnect_requested version report the parameter only to connections that ask for it, and what do clients behind a current PgBouncer or Pgpool-II see?
2. When does the server send the disconnect_requested ParameterStatus to a client whose backend is idle, and which clients and proxies were tested with that timing?
3. Which pooler or driver has a working change that reacts to disconnect_requested, and how much faster does a replica switchover drain with it?
4. Does pg_goaway_backend() remain in the GUC version, and does it use the same role checks as pg_terminate_backend()?
5. Can the disconnect_requested value later carry the deadline or priority that Tomas Vondra and Zsolt Parragi asked for, without breaking clients that read it as a boolean?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the frontend/backend protocol, libpq and ParameterStatus reporting would have to own it. Several reviewers replied, but the visible material shows no committer taking it on.
- **Liability:** The owner signs for a client-visible parameter and its timing rules for as long as drivers and poolers read it. Any exported libpq function also stays for good.
- **Author follow-up:** Good signs: the author sent v2 one day after the first review, reworked the design twice, and moved the entry to the next CommitFest on 2026-10-01.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Sites that drain replicas or restart poolers gain, and the patch is small, but it adds a lasting client-visible contract. The opt-in and idle-timing questions need an answer first; the GUC version of the patch was not read.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` The user-facing contract under review is not in the visible material. Refuted (no failure scenario): A driver author who builds against an uncommitted patch version is not a failure of the proposal. The GUC design is in an attached thread that this review cannot see, so this is a limit of the input.
- `replication-ondisk-1` Without an opt-in, old poolers forward the disconnect request to their clients. Refuted (not specific): Duplicate of standards-compat-1.
- `security-1` Privilege rule for pg_goaway_backend() is not stated. Refuted (no failure scenario): The scenario assumes pg_goaway_backend() skips the role checks, but the proposal does not say that it does. The v8 code and docs are not shown, so this is an item that could not be checked.
- `maintenance-fit-1` A reported disconnect_requested GUC reaches poolers that replay ParameterStatus. Refuted (not specific): Duplicate of standards-compat-1 for the opt-in and of replication-ondisk-2 for the idle timing.
- `maintenance-fit-2` Permanent libpq and protocol surface with no shown client consumer. Refuted (not specific): Duplicate of demand-1. It also misreads message #20, where "No problem for Pgpool-II" answers the opt-in question and says nothing about need.
- `already-exists-1` Proposal does not compare against idle_session_timeout for draining. Refuted (no failure scenario): The objection itself says idle_session_timeout ends sessions with a FATAL error that the client sees on next use, which is the error the proposal avoids. It also does nothing for pooler restarts, a main use case in message #3.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent records a rejection reason that applies here: P11 died on a read-only interaction this flag lacks, and P3 supports the GUC direction. The thread points to an earlier discussion only for the protocol-version pushback (message #5), which the proposal already answered by dropping the minor version bump.
- standards-compat: 2 objection(s). The read-only reported GUC itself follows existing practice such as in_hot_standby, so I raise no objection to the mechanism.
- replication-ondisk: 2 objection(s). The proposal has no on-disk, WAL, catalog or replication state. Walsenders were removed from the signal in message #8. The objections above are about the wire protocol only.
- security: 1 objection(s). The read-only GUC and the shutdown-time signal expose only the fact that a smart shutdown has started, and the material shows no setting a user can change. The visible privilege question concerns pg_goaway_backend() alone.
- maintenance-fit: 2 objection(s). Hot-path cost looks small from the prose: one procsignal at smart shutdown and an idle check. Without the GUC thread and its code, this cannot be confirmed.
- already-exists: 1 objection(s). The thread settled NOTICE as an alternative in message #19 because client_min_messages can suppress it. The move to a reported GUC uses the existing ParameterStatus mechanism and does not add a second one. I found no extension or contrib module that does this.
- demand: 1 objection(s). Demand exists but is thin: one pooler author in favour and no client adoption shown, so the honest outcome is not now, show demand.

### Precedents retrieved

- P1: [Add new protocol message to change GUCs to be able to change protocol extension parameters](https://commitfest.postgresql.org/patch/4736/) (cf, Withdrawn, 2025-02-22) - used in the review; reason summarised from its thread
- P2: [Use "protocol options" name instead of "protocol extensions" everywhere](https://commitfest.postgresql.org/patch/5341/) (cf, Rejected, 2025-03-11) - judged not relevant; reason summarised from its thread
- P3: [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://commitfest.postgresql.org/patch/4166/) (cf, Withdrawn, 2024-01-30) - used in the review; reason summarised from its thread
- P4: [Optional message to signalled process in pg_terminate_backend()/pg_cancel_backend()](https://commitfest.postgresql.org/patch/1174/) (cf, Returned with feedback, 2017-10-02) - used in the review
- P5: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P6: [Get rid of an extra error for client disconnection on Windows](https://commitfest.postgresql.org/patch/659/) (cf, Rejected, 2016-09-13) - judged not relevant; reason summarised from its thread
- P7: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant; reason summarised from its thread
- P8: [Revert "graceful shutdown" changes for Windows.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29992a6a509b256efc4ac560a1586b51a64b2637) (revert, Reverted, 2022-03-22) - judged not relevant
- P9: [Built-in connection pooler](https://commitfest.postgresql.org/patch/3043/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P10: [propagating replica identity to partitions](https://commitfest.postgresql.org/patch/2044/) (cf, Returned with feedback, 2019-09-11) - judged not relevant; reason summarised from its thread
- P11: [Add and report the new "session_read_only" GUC pseudo-variable.](https://commitfest.postgresql.org/patch/1090/) (cf, Returned with feedback, 2019-02-04) - used in the review; reason summarised from its thread
- P12: [Extend comments on signal handling and cross-reference comments on signal handling functions](https://commitfest.postgresql.org/patch/2951/) (cf, Returned with feedback, 2021-09-27) - judged not relevant
- P13: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P14: [Client Connection redirection support for PostgreSQL](https://commitfest.postgresql.org/patch/1673/) (cf, Returned with feedback, 2018-10-02) - used in the review
- P15: [Move tablespace path re-creation from the makefiles to pg_regress (test for mkdir function in the backend)](https://commitfest.postgresql.org/patch/3067/) (cf, Withdrawn, 2021-04-16) - judged not relevant
- P16: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1148/) (cf, Returned with feedback, 2017-11-28) - used in the review
- P17: [Add client connection check during the execution of the query](https://commitfest.postgresql.org/patch/1882/) (cf, Returned with feedback, 2019-08-01) - judged not relevant
- P18: [Set default transactions to read-only at servers start in pg_upgrade](https://commitfest.postgresql.org/patch/2970/) (cf, Withdrawn, 2021-12-01) - judged not relevant
- P19: [Hooks and API to allow loadable wire protocol extensions](https://commitfest.postgresql.org/patch/3018/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [Support optional message in backend cancellation/termination](https://commitfest.postgresql.org/patch/1494/) (cf, Returned with feedback, 2018-04-09) - used in the review
- P21: [TRUNCATE behavior when session_replication_role = replica](https://commitfest.postgresql.org/patch/1447/) (cf, Rejected, 2018-04-07) - judged not relevant
- P22: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1568/) (cf, Returned with feedback, 2018-03-01) - used in the review
- P23: [pgbench stopped supporting large number of client connections on Windows](https://commitfest.postgresql.org/patch/2827/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P24: [get_extension_version](https://commitfest.postgresql.org/patch/4242/) (cf, Returned with feedback, 2023-12-02) - judged not relevant

### Warnings

- CommitFest entry has 2 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/DDPQ1RV5FE9U.I2WW34NGRD8Z%40jeltef.nl). Others: https://www.postgresql.org/message-id/flat/DJVLCL7PV7U6.35IJF87CZZ61Y%40jeltef.nl

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 172 s; cost reported by the CLI: $2.76
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 8 of 24 kept
- Tone check: 98 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:44:02.536Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
