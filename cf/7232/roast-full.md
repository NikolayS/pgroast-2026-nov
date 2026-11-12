# Design pre-review: Protocol compression: a fourth design

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7232](https://commitfest.postgresql.org/patch/7232/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Protocol compression: a fourth design](https://www.postgresql.org/message-id/flat/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru), started 2026-08-31
- Messages read: 3 of 3
- Patch prose read: v1-0001-Add-protocol-compression.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Demand is shown and no surviving objection is fatal, so levels 1a and 1b pass. Level 2 fails on fixable points that the list must settle before code review: replication-ondisk-1 (one compression history across all roles' WAL), maintenance-fit-2 and standards-compat-1 (first protocol extension and client option fixed before the planned codecs, levels and per-direction control have a shape), and replication-ondisk-2 (pooler contract). The author's three design questions in message #1 have no answer yet.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code I could not check where compression sits relative to secure_read and secure_write, how the walsender uses it, or the threshold for leaving small results uncompressed. The cost for connections that do not negotiate compression is also unchecked.

## The proposal, restated

**Problem.** Bulk data such as query results, COPY, pg_restore, base backup and WAL streaming uses a lot of network traffic, which slows clients on limited links. Three earlier protocol compression attempts stalled on negotiation, I/O layering, codec choice, poolers and compression before encryption. This proposal tries a design that answers those questions.

**Proposed design.** Client and server negotiate _pq_.compression=zstd, and compression starts after the first ReadyForQuery. Server DataRow and CopyData, and client CopyData during COPY FROM, travel as flushed segments of one persistent Zstandard stream per direction inside new CompressedData messages, with a 64 KiB history. Queries, bind parameters and control messages stay uncompressed, and each stream ends before ReadyForQuery or CopyDone/CopyFail so poolers keep their boundaries.

**User-visible surface.**
- New protocol extension _pq_.compression with value zstd, the first defined protocol extension.
- New protocol message CompressedData, Byte1('z'), sent by frontend and backend, with 17 MiB payload and 16 MiB output limits.
- New server GUC protocol_compression = off | zstd, default off; reload affects only new connections.
- New libpq connection option compression = off | prefer | zstd, and environment variable PGCOMPRESSION.
- libpq trace output changes (fe-trace.c is touched); the material does not show how.

**What the author claims.**
- pg_restore client-to-server traffic fell from 264 MB to 21 MB, with median local restore time 3.00 s versus 3.04 s.
- Earlier prototype runs cut mixed DataRow and COPY TO traffic to about 10% and 14% of the original size.
- One-row pgbench showed no loss (7034 versus 7018 TPS); random data only shrank to about 80% and cost about 0.5 s more CPU.
- A 64 KiB history keeps about 1 MiB extra memory per active backend, against about 8.6 MiB with an 8 MiB window.
- A pooler can route queries and relay CompressedData without understanding Zstandard; new libpq with prefer connects to an unmodified Postgres 14 server.

**Thread state.** Jochen Bandhauer shared a measurement script and results showing large traffic cuts and faster runs on LAN and rate-limited links; the author thanked him. Nobody has answered the author's three design questions on message scope, stream boundaries and the policy interface, and CFbot reports the patch needs a rebase.

## 2. Prior attempts

### Libpq Compression (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/4746/](https://commitfest.postgresql.org/patch/4746/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com)
- **Date:** decided 2024-05-14; started 2023-12-19
- **Outcome:** Withdrawn (cf)
- **Who objected:** Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the patch after five months without updates, while waiting for the separate discussion on a protocol message to change GUCs. Robert Haas said the user interface was not clear enough to review, citing `libpq_compression=gzip:compress=off` as confusing. He called these UI questions blockers. A reworked syntax with `client_to_server`/`server_to_client` was being discussed when the entry closed.
- **Reason (quote):** "I would really like to see this patch go forward, but IMHO these UI questions are blockers." ([message](https://www.postgresql.org/message-id/CA%2BTgmoZ1eU35KwTZ6QPsfjMjpx4fkF8z5rVORfppc%3DY%2B0ZHVVQ%40mail.gmail.com))
- **What is different this time:** The interface is smaller: off | zstd on the server and off | prefer | zstd in libpq, with no per-direction syntax. The new design adds no I/O stream abstraction in place of secure_read and secure_write.

### libpq compression (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/3499/](https://commitfest.postgresql.org/patch/3499/)
- **Thread:** [https://www.postgresql.org/message-id/flat/ABAA09C6-BB95-47A5-890D-90353533F9AC@yandex-team.ru](https://www.postgresql.org/message-id/flat/ABAA09C6-BB95-47A5-890D-90353533F9AC@yandex-team.ru)
- **Date:** decided 2022-08-02; started 2021-12-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Justin Pryzby (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Jacob Champion marked the entry Returned with Feedback because it had waited on the author for longer than the two-week threshold. The last open review point came from Justin Pryzby in January 2022. He said compression still sat at the layer the 2019 thread had objected to, wrapping secure_read/write through function pointers instead of working inside them, where SSL is handled. The visible messages show no reply from the author to this point. Earlier review also asked for the server setting to default to off for security reasons, and reported an LZ4 assertion failure and CI test failures.
- **Reason (quote):** "This entry has been waiting on author input for a while (our current threshold is roughly two weeks), so I've marked it Returned with Feedback." ([message](https://www.postgresql.org/message-id/dcb059f2-f226-2aa3-6f34-2ec95ca45813%40timescale.com))
- **What is different this time:** [P3] used the same _pq_.compression name and CompressedData message, with several codecs and full traffic in both directions. This proposal makes the server setting default to off and limits compression to bulk-data messages.

### libpq compression (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/2773/](https://commitfest.postgresql.org/patch/2773/)
- **Thread:** [https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru](https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru)
- **Date:** decided 2021-12-03; started 2018-03-30
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier marked the entry Returned with feedback because it had been waiting on author for two months. The last author message, in October 2021, promised lz4 support, but no new version followed after Daniel Gustafsson's rebase with a Windows build fix. Earlier review asked for a cleaned-up patch set, a fix for make check-world hangs with zlib, and code cleanup in the shared compression API.
- **Reason (quote):** "This patch has been waiting on author for two months now, so I have marked it as RwF in the CF app." ([message](https://www.postgresql.org/message-id/YanH2NuKhZmm2D6w%40paquier.xyz))
- **What is different this time:** Zstandard only, compression starts after the first ReadyForQuery, and it covers only DataRow and CopyData. Streams end at ReadyForQuery and COPY boundaries.

### libpq compression (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1635/](https://commitfest.postgresql.org/patch/1635/)
- **Thread:** [https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru](https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru), [https://www.postgresql.org/message-id/flat/1553029215728-0.post@n3.nabble.com](https://www.postgresql.org/message-id/flat/1553029215728-0.post@n3.nabble.com)
- **Date:** decided 2019-04-09; started 2018-03-30
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier, Dmitry Dolgov (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers objected to the design: the compression option sent in the startup packet, and the compression stream bypassing secure_read()/pqsecure_read() rather than working as a layer under them, which they found invasive. The author and reviewers did not reach consensus on this layering, and Dmitry Dolgov posted an alternative experimental patch. In the last commitfest the patch had failing tests and no review, and the visible messages do not show a closing statement.
- **Reason (quote):** "I have not actually looked much at your patch, but I am wondering if it could not be possible to make the whole footprint less invasive which really worries me as now shaped." ([message](https://www.postgresql.org/message-id/20180606075330.GK1442%40paquier.xyz))
- **What is different this time:** [P7] compressed the whole byte stream through a layer around socket reads and writes. This proposal wraps selected protocol messages in CompressedData and keeps queries and control messages ordinary.


## 3. Objections a reviewer is likely to raise

### 1. Replication streams share one compression history across all users' WAL

fixable · level 2, rubric item 7 · security / privileges, WAL / replication / standby · `replication-ondisk-1`

An administrator sets protocol_compression = zstd, and a standby uses compression=prefer in primary_conninfo over TLS. A low-privilege user on the primary inserts chosen values next to a secret column in another role's table, then watches the replication link. Compressed CopyData lengths over the long-lived walsender stream leak whether the guesses match recent WAL content.

Message #1 names WAL streaming and base backup as covered, and walsender.c is touched. The only reset point is ReadyForQuery, and a START_REPLICATION session can run for days without one. One 64 KiB history then mixes WAL from every database and role. The libpq warning speaks of values in "the same compressed response", and the per-server switch in message #1 does not separate replication connections from ordinary ones.

Thread: [#1 (Andrey Borodin, 2026-08-31)](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru)

**Question:** Do replication and base backup connections use compression, and if so, what bounds history sharing across roles in a walsender stream, or how is it disabled separately?

Counter-check: held. Message #1 lists WAL streaming as covered, and the protocol text allows one backend frame until ReadyForQuery. The 64 KiB history and the two opt-ins narrow the exposure, but the libpq warning does not cover data mixed across roles.

### 2. First protocol extension fixes scope while the scope question is open

fixable · level 2, rubric item 6 · wire protocol, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

Drivers and poolers implement _pq_.compression=zstd with the v1 rules: backend DataRow and CopyData, frontend CopyData only, frames that end at ReadyForQuery. A later release wants to compress Bind parameters, add levels, or allow per-direction control. It then needs a second extension or new values, and the project keeps both sets of rules on the server and in libpq indefinitely.

Message #1 asks whether the message scope is "future-proof" and defers levels, more codecs and per-direction control to follow-ups. The documentation hunk defines exact limits and frame rules that become part of the documented protocol. Message #1 says the framing need not change, but the scope and boundary rules are what external implementations will copy. Per-direction control was the UI point that blocked [P4].

**Precedents.**
- [Libpq Compression](https://commitfest.postgresql.org/patch/4746/) (Withdrawn, 2024-05-14); thread: [link](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com)

Thread: [#1 (Andrey Borodin, 2026-08-31)](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru)

**Question:** How would per-direction control, compression levels and a wider message scope be negotiated later without a second set of rules that servers and libpq must keep?

Counter-check: held. Message #1 says the framing stays, but it does not show how a wider scope, levels or per-direction control would be negotiated under _pq_.compression. [P4] stalled on option syntax, so it supports only the per-direction point.

### 3. Client option mixes policy and codec, with no room for more codecs

fixable · level 2, rubric item 10 · psql UX conventions, wire protocol · `standards-compat-1`

A later release adds lz4 and compression levels, as message #1 plans. A user who wants "lz4 or zstd, whichever the server allows", or wants zstd at a set level, cannot say so with off | prefer | zstd. libpq then needs a second syntax while connection strings written for the first one stay in use.

Message #1 says new codecs and levels need no framing change, but the client option does change: "prefer" silently means zstd, and a method name means "required". Postgres already separates policy from detail elsewhere: sslmode and sslnegotiation, and pg_dump and pg_basebackup use method[:detail] specs. A committer would likely ask for the full future syntax before the first release, because [P4] stalled on exactly this kind of unclear compression UI.

**Precedents.**
- [Libpq Compression](https://commitfest.postgresql.org/patch/4746/) (Withdrawn, 2024-05-14); thread: [link](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com)

Thread: [#1 (Andrey Borodin, 2026-08-31)](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru)

**Question:** What connection string will a user write to request lz4-or-zstd at a given level, and does today's syntax extend to it without changing meaning?

Counter-check: held. Message #1 plans more codecs and levels but shows no option syntax for them, and its question 3 on this interface is unanswered. [P4] concerned a different syntax, so it supports only the general point that compression UI gets close review.

### 4. Transaction poolers must match compression state per server connection

fixable · level 2, rubric item 6 · wire protocol · `replication-ondisk-2`

A transaction-mode pooler forwards a client's _pq_.compression=zstd when it opens a server connection. It later gives that server connection to a client that did not negotiate compression. That client receives a 'z' message in the middle of a result and drops the connection with a protocol error. The user sees this only under load, when the pool mixes clients.

Message #1 says a pooler that passes the negotiation through can relay CompressedData without understanding Zstandard. Negotiation is per server connection, but a transaction pooler shares server connections between clients. To relay safely, the pooler must understand the extension and keep separate pools, or it must decompress. As the first defined protocol extension, this sets the pattern for later ones, so the pooler contract matters beyond this feature.

Thread: [#1 (Andrey Borodin, 2026-08-31)](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru)

**Question:** What must a transaction-mode pooler do with _pq_.compression and CompressedData when its server connections are shared by clients with different settings?

Counter-check: held. The scenario needs a pooler that forwards the extension to a shared server connection, which a pooler that does not know the extension is unlikely to do. But message #1 does not say that relaying requires pooling by negotiated state, and question 2 asks about this.

### 5. prefer fallback is shown only against a server, not against poolers

fixable · level 2, rubric item 12 · wire protocol, backpatchability · `standards-compat-2`

An administrator sets PGCOMPRESSION=prefer for all clients. Some clients connect through an older pooler or proxy that answers an unknown _pq_ startup parameter with a FATAL error rather than NegotiateProtocolVersion. Those clients cannot connect at all, and the cause is not obvious.

_pq_.compression is the first protocol extension that is ever defined, so no middleware has seen one in production. Message #1 reports a fallback test only against an unmodified Postgres 14 server. It says a pooler can relay CompressedData, but it does not say how a pooler that does not know the extension handles the startup parameter. The brief lists this as unclear.

Thread: [#1 (Andrey Borodin, 2026-08-31)](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru)

**Question:** How do common poolers and proxies react today to an unknown _pq_ startup parameter, and does prefer fall back cleanly when one of them rejects it?

Counter-check: held. Message #1 reports the prefer fallback only against a Postgres 14 server. Some pooler versions reject unknown startup parameters with an error, and the proposal does not say whether prefer retries without the extension in that case.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | walsender.c is touched, and message #1 lists base backup and WAL streaming as covered by backend CopyData compression. |
| wire protocol | **yes** | The patch defines the first protocol extension, _pq_.compression, and a new CompressedData message, Byte1('z'). |
| catalog changes | no |  |
| ABI / extension API | **yes** | libpq gains a compression connection option and PGCOMPRESSION, and third-party drivers and poolers must implement the new message to take part. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | New server GUC protocol_compression = off \| zstd, default off. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | psql gets compression through the libpq option, but the proposal names no \conninfo output or libpq call for the negotiated state. |
| security / privileges | **yes** | Compression happens before SSL or GSS encryption, so compressed lengths can leak data; the only server control is one cluster-wide GUC. |
| backpatchability | **yes** | New libpq with prefer must connect to older servers and poolers; message #1 tests this only against a Postgres 14 server. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The documented frame rules, size limits and boundary rules become protocol that the server, libpq, drivers and poolers must keep. |
| performance on the hot path | **yes** | The patch touches pqcomm.c, postgres.c and fe-protocol3.c on the message send and receive paths; default is off. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do replication and base backup connections use compression, and if so, what bounds history sharing across roles in a walsender stream that never reaches ReadyForQuery?
2. How will more codecs, compression levels, per-direction control and a wider message scope be negotiated under _pq_.compression without a second set of rules?
3. What connection string will a user write for lz4-or-zstd at a given level, and do off | prefer | zstd keep their meaning then?
4. What must a transaction-mode pooler do with _pq_.compression and CompressedData when clients with different settings share its server connections?
5. Does prefer retry without the extension when a pooler or proxy rejects the unknown _pq_ startup parameter with a FATAL error?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the wire protocol, libpq and the walsender; the visible thread has only the author and one tester.
- **Liability:** The owner signs for the first protocol extension and a new message type that drivers and poolers will copy, plus the compress-before-encrypt exposure, for as long as protocol 3 lives.
- **Author follow-up:** The author replied to the tester within three days and moved the entry to the next CommitFest, but the rebase that CFbot asked for on 2026-09-02 is not posted.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Four earlier entries since 2018 and the measurements in message #2 show lasting demand from users on slow or metered links. An 88 KiB patch across backend, walsender and libpq adds permanent protocol rules, and the scope, pooler and interface questions are still open.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-3` (fixable, level 2) No way to see whether a connection negotiated compression. Not listed: not among the strongest objections.
- `security-2` (fixable, level 2) Server switch is all-or-nothing, with no per-role, database or pg_hba control. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` Client option repeats the codec-and-policy grammar that blocked [P4]. Refuted (not specific): Duplicate of standards-compat-1, which states the same client option concern with a more concrete case.
- `security-1` WAL streaming shares one compression history across all users' changes. Refuted (not specific): Duplicate of replication-ondisk-1, which describes the same walsender history sharing more precisely.
- `maintenance-fit-1` No committer engaged on a protocol change spanning backend and libpq. Refuted (not specific): A missing committer is a committability note that applies to any ten-week-old thread, and the rubric says it must not raise the verdict. [P1], [P3] and [P4] closed for author inactivity, and a committer was engaged in [P4].
- `maintenance-fit-3` Silent fallback in prefer with no way to see compression state. Refuted (not specific): Duplicate of standards-compat-3, which covers the same missing way to see whether a connection negotiated compression.
- `already-exists-1` Base backup compression duplicates pg_basebackup's existing server-side compression. Refuted (no failure scenario): Both settings are off by default, so the double compression happens only when a user enables both, and the result is extra CPU with correct output. Server-side backup compression does not cover queries, COPY or WAL streaming, so the mechanisms are not duplicates.

### Review angles

- prior-attempts: 1 objection(s). The other recorded reasons are inactivity in [P1] and [P3], layering in [P7], and default-off security in [P3]. Each is either answered by this design or does not bear on it. The patch has needed a rebase since 2026-09-02, which is a committability point, not an objection to the idea.
- standards-compat: 3 objection(s). Not applicable. Checks that need the code: the trace output format, how replication connections use the option, and whether 'z' collides with any type byte.
- replication-ondisk: 2 objection(s). The proposal has no on-disk, WAL format or catalog change, so pg_upgrade and standby replay are not affected; the objections cover only the wire protocol and replication connections.
- security: 2 objection(s). Decompression memory per segment is bounded and the default is off, so no denial-of-service objection is raised. Earlier requests for an off default appear to be met.
- maintenance-fit: 3 objection(s). Not applicable. Hot-path cost for non-users looks small with the default off, but this cannot be checked without the code.
- already-exists: 1 objection(s). No existing feature covers query results, COPY or pg_restore traffic on the wire. An ssh -C tunnel does, but it needs shell access that managed services do not give, and sslcompression was removed, so I raise no general already-exists objection.
- demand: 0 objection(s). The material shows real and lasting demand. Separate authors have tried protocol compression since 2018 ([P7], [P1], [P3], [P4]), and none of those entries ended because nobody wanted it; the recorded reasons are stalled authors, layering and the user interface. In message #2 an independent tester measured large traffic cuts and faster runs on LAN and rate-limited links. The feature is off by default, so its runtime cost falls mainly on the users who turn it on. Design, security and maintenance concerns belong to other angles.

### Precedents retrieved

- P1: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - used in the review; reason summarised from its thread
- P2: [Revert changes for SSL compression in libpq](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0ba71107efeeccde9158f47118f95043afdca0bb) (revert, Reverted, 2021-03-10) - judged not relevant
- P3: [libpq compression](https://commitfest.postgresql.org/patch/3499/) (cf, Returned with feedback, 2022-08-02) - used in the review; reason summarised from its thread
- P4: [Libpq Compression](https://commitfest.postgresql.org/patch/4746/) (cf, Withdrawn, 2024-05-14) - used in the review; reason summarised from its thread
- P5: [Use "protocol options" name instead of "protocol extensions" everywhere](https://commitfest.postgresql.org/patch/5341/) (cf, Rejected, 2025-03-11) - judged not relevant; reason summarised from its thread
- P6: [Add new protocol message to change GUCs to be able to change protocol extension parameters](https://commitfest.postgresql.org/patch/4736/) (cf, Withdrawn, 2025-02-22) - used in the review; reason summarised from its thread
- P7: [libpq compression](https://commitfest.postgresql.org/patch/1635/) (cf, Returned with feedback, 2019-04-09) - used in the review; reason summarised from its thread
- P8: [Hooks and API to allow loadable wire protocol extensions](https://commitfest.postgresql.org/patch/3018/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [automatic client-side column-level encryption](https://commitfest.postgresql.org/patch/3718/) (cf, Returned with feedback, 2024-06-27) - judged not relevant; reason summarised from its thread
- P10: [Add max_wal_replay_size connection parameter to libpq](https://commitfest.postgresql.org/patch/6630/) (cf, Withdrawn, 2026-07-23) - judged not relevant
- P11: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P12: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant
- P13: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P14: [Allow specifying multiple hosts in libpq connect options](https://commitfest.postgresql.org/patch/222/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P15: [pg_basebackup: add test about zstd compress option](https://commitfest.postgresql.org/patch/3835/) (cf, Rejected, 2022-12-06) - judged not relevant
- P16: [Fix minor issues in astreamer_zstd.c](https://commitfest.postgresql.org/patch/6386/) (cf, Withdrawn, 2026-01-10) - judged not relevant
- P17: [Data at rest encryption](https://commitfest.postgresql.org/patch/2104/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P18: [Mention column name in error messages](https://commitfest.postgresql.org/patch/324/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P19: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4739/) (cf, Rejected, 2024-04-08) - judged not relevant
- P20: [Mention column name in error messages](https://commitfest.postgresql.org/patch/818/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P21: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant
- P22: [Send numeric version to clients (server_version_num GUC_REPORT)](https://commitfest.postgresql.org/patch/752/) (cf, Rejected, 2016-08-30) - judged not relevant
- P23: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4750/) (cf, Withdrawn, 2024-10-12) - judged not relevant
- P24: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 227 s; cost reported by the CLI: $2.69
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 118 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:31:00.707Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
