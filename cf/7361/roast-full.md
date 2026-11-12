# Design pre-review: Protocol Compression (fourth attempt)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7361](https://commitfest.postgresql.org/patch/7361/) (Needs review); authors: Anthonin Bonnefoy (bonnefoa)
- Thread: [Protocol Compression (fourth attempt)](https://www.postgresql.org/message-id/flat/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), started 2026-09-29
- Messages read: 7 of 7
- Patch prose read: v1-0001-Print-cancel-key-as-a-hex-string-in-trace.patch, v1-0004-Make-pqtrace-write-lines-atomically.patch, v1-0005-Add-PQTRACE-env-to-enable-protocol-tracing-in-lib.patch, v1-0002-Add-more-regress-masks-in-pqtrace.patch, v1-0003-Add-mismatch-length-check-on-pqTraceOutputNoTypeB.patch, v1-0008-Add-a-protocol-trace-test-harness.patch, v1-0010-Add-backend-support-for-CompressedMessages.patch, v1-0006-Add-trace-of-startup-packets-in-libpq_pipeline-te.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has lasting demand and nothing survived at levels 1a or 1b. At level 2, v2 does not say how a client or server turns compression on or off (standards-compat-1), which is the kind of question that blocked [P5]. The default also decides the TLS length-leak exposure (security-1) and the replication use case (replication-ondisk-1). These points are fixable, but the list needs to agree on them before code review is useful.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only the first 3 of 7 messages and no code were read, so the v2 negotiation, the flush behaviour during replication and the decompression memory limits could not be checked. The cost on the libpq parse path for connections without compression is also unverified.

## The proposal, restated

**Problem.** Query results and WAL streams travel uncompressed between server and client, which costs network bandwidth. The thread names no specific user group, but the author expects replication to benefit through primary_conninfo. Four earlier attempts did not land.

**Proposed design.** The server wraps backend messages in a new CompressedMessages message. A compression layer stacks on top of PQcommMethods and uses streaming compression, and libpq decompresses through a new msg_buffer. The client lists its codecs in the _pq_.supported_compressions startup option. In v2, Borodin narrows this to zstd negotiated at startup, compressing only DataRow and CopyData, with frames that end at every ReadyForQuery.

**User-visible surface.**
- New protocol message CompressedMessages; v1 carries the algorithm and a list of nested message types, v2 compresses only DataRow and CopyData.
- New startup option _pq_.supported_compressions sent by libpq; v1 claims no protocol version bump is needed.
- v1 GUCs protocol_backend_compression, _allowed_algorithms, _threshold, _number_messages and _transaction_frame; v2 drops the public tuning knobs.
- New libpq connection parameters trace_file and trace_flags, plus PQTRACE and PQTRACEFLAGS environment variables (from a bundled patch set).
- v2 adds client-to-server COPY compression; protocol tracing marks nested messages with "B*".

**What the author claims.**
- A pgbouncer-style pooler can forward compressed messages without decompressing them, because the messages a pooler inspects stay uncompressed.
- Streaming caps backend memory at about 1.3 MB of compression context plus a 131 KB output buffer.
- With batches of 2000 or more messages, zstd shrinks a full pgbench_accounts scan from 1 GB to 18 MB at about uncompressed latency.
- v1 default batching raises select_all latency from 55 ms to 70 ms; v2 cuts first-row latency from 13 ms to about 0.5 ms.
- WAL streaming can be compressed by setting options in primary_conninfo.

**Thread state.** Andrey Borodin, author of an earlier attempt, proposed merging the two efforts, and Bonnefoy agreed to most of the scope cuts. Borodin posted a reduced v2 on 2026-10-23 and asked whether it works as a common base, and nobody has answered yet.

## 2. Prior attempts

### Libpq Compression (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/4746/](https://commitfest.postgresql.org/patch/4746/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com)
- **Date:** decided 2024-05-14; started 2023-12-19
- **Outcome:** Withdrawn (cf)
- **Who objected:** Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the patch after five months without updates, while waiting for the separate discussion on a protocol message to change GUCs. Robert Haas said the user interface was not clear enough to review, citing `libpq_compression=gzip:compress=off` as confusing. He called these UI questions blockers. A reworked syntax with `client_to_server`/`server_to_client` was being discussed when the entry closed.
- **Reason (quote):** "I would really like to see this patch go forward, but IMHO these UI questions are blockers." ([message](https://www.postgresql.org/message-id/CA%2BTgmoZ1eU35KwTZ6QPsfjMjpx4fkF8z5rVORfppc%3DY%2B0ZHVVQ%40mail.gmail.com))
- **What is different this time:** v2 removes most tuning knobs and fixes one codec at startup. It does not yet describe the user interface that replaces them, which is the point that blocked that attempt.

### libpq compression (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/3499/](https://commitfest.postgresql.org/patch/3499/)
- **Thread:** [https://www.postgresql.org/message-id/flat/ABAA09C6-BB95-47A5-890D-90353533F9AC@yandex-team.ru](https://www.postgresql.org/message-id/flat/ABAA09C6-BB95-47A5-890D-90353533F9AC@yandex-team.ru)
- **Date:** decided 2022-08-02; started 2021-12-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Justin Pryzby (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Jacob Champion marked the entry Returned with Feedback because it had waited on the author for longer than the two-week threshold. The last open review point came from Justin Pryzby in January 2022. He said compression still sat at the layer the 2019 thread had objected to, wrapping secure_read/write through function pointers instead of working inside them, where SSL is handled. The visible messages show no reply from the author to this point. Earlier review also asked for the server setting to default to off for security reasons, and reported an LZ4 assertion failure and CI test failures.
- **Reason (quote):** "This entry has been waiting on author input for a while (our current threshold is roughly two weeks), so I've marked it Returned with Feedback." ([message](https://www.postgresql.org/message-id/dcb059f2-f226-2aa3-6f34-2ec95ca45813%40timescale.com))
- **What is different this time:** That attempt wrapped secure_read/write through function pointers. This one works above the socket methods at message level. The security default raised in that review is not yet addressed here.

### libpq compression (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/1635/](https://commitfest.postgresql.org/patch/1635/)
- **Thread:** [https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru](https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru), [https://www.postgresql.org/message-id/flat/1553029215728-0.post@n3.nabble.com](https://www.postgresql.org/message-id/flat/1553029215728-0.post@n3.nabble.com)
- **Date:** decided 2019-04-09; started 2018-03-30
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier, Dmitry Dolgov (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers objected to the design: the compression option sent in the startup packet, and the compression stream bypassing secure_read()/pqsecure_read() rather than working as a layer under them, which they found invasive. The author and reviewers did not reach consensus on this layering, and Dmitry Dolgov posted an alternative experimental patch. In the last commitfest the patch had failing tests and no review, and the visible messages do not show a closing statement.
- **Reason (quote):** "I have not actually looked much at your patch, but I am wondering if it could not be possible to make the whole footprint less invasive which really worries me as now shaped." ([message](https://www.postgresql.org/message-id/20180606075330.GK1442%40paquier.xyz))
- **What is different this time:** Compression sits as a message-level layer in PQcommMethods and sends through the existing transport, so TLS stays underneath. The frontend still changes how all pqGet* callers read the input buffer.

### libpq compression (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/2773/](https://commitfest.postgresql.org/patch/2773/)
- **Thread:** [https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru](https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru)
- **Date:** decided 2021-12-03; started 2018-03-30
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier marked the entry Returned with feedback because it had been waiting on author for two months. The last author message, in October 2021, promised lz4 support, but no new version followed after Daniel Gustafsson's rebase with a Windows build fix. Earlier review asked for a cleaned-up patch set, a fix for make check-world hangs with zlib, and code cleanup in the shared compression API.
- **Reason (quote):** "This patch has been waiting on author for two months now, so I have marked it as RwF in the CF app." ([message](https://www.postgresql.org/message-id/YanH2NuKhZmm2D6w%40paquier.xyz))
- **What is different this time:** That attempt stalled waiting on its author. This one compresses only server-to-client DataRow and CopyData plus client COPY, and has two authors working together on it.


## 3. Objections a reviewer is likely to raise

### 1. v2 does not define how users or servers turn compression on or off

fixable · level 2, rubric item 10 · new GUC, wire protocol, WAL / replication / standby, security / privileges · `standards-compat-1`

A site upgrades libpq and the server, and existing applications and standbys start sending zstd-compressed DataRow and CopyData without anyone asking for it. The DBA wants it off for TLS connections or CPU-bound hosts and finds no server setting for that. If instead the default is off, the primary_conninfo recipe from message #1 no longer works.

v2 (message #7) negotiates zstd at startup and drops every public GUC, but it does not say whether libpq advertises zstd by default, which connection parameter controls it, or how the server refuses. In v1, protocol_backend_compression_allowed_algorithms='none' gave the server a veto, and v2 removes it without a replacement. The replication claim in message #1 rests on a GUC that v2 removed. In [P5], unclear compression UI was called a blocker.

**Precedents.**
- [Libpq Compression](https://commitfest.postgresql.org/patch/4746/) (Withdrawn, 2024-05-14); thread: [link](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com)

Thread: [#1 (Anthonin Bonnefoy, 2026-09-29)](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#5 (Andrey Borodin, 2026-09-30)](https://www.postgresql.org/message-id/845C3C95-E01F-49D1-AB2F-003356E76AD4%40yandex-team.ru), [#7 (Andrey Borodin, 2026-10-23)](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru)

**Question:** What is the default for a libpq client and a server after upgrade, and which parameter lets a client opt out and a server refuse?

Counter-check: held. Message #7 says only that v2 negotiates Zstandard at startup, and it drops the v1 veto protocol_backend_compression_allowed_algorithms='none' without naming a replacement. [P5] stalled on unclear compression UI, so the precedent applies.

### 2. Compression under TLS reopens CRIME-style leaks with no server-side default off

fixable · level 2, rubric item 7 · security / privileges, wire protocol · `security-1`

An application returns rows that mix a secret column with a column an attacker can write, such as a comment field. Both rows land in one zstd frame that runs until ReadyForQuery. An attacker who can watch TLS record sizes changes the input and reads the secret from the compressed length, and nobody notices.

Postgres disabled SSL compression because of this class of attack. The v2 frame covers every DataRow before ReadyForQuery, so attacker data and secret data share one compression context. The thread does not mention the attack, does not say how v2 turns compression on, and does not say whether the server can refuse it. Review of [P3] asked for the server setting to default to off for security reasons, which applies here.

**Precedents.**
- [libpq compression](https://commitfest.postgresql.org/patch/3499/) (Returned with feedback, 2022-08-02); thread: [link](https://www.postgresql.org/message-id/flat/ABAA09C6-BB95-47A5-890D-90353533F9AC@yandex-team.ru)

Thread: [#1 (Anthonin Bonnefoy, 2026-09-29)](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#7 (Andrey Borodin, 2026-10-23)](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru)

**Question:** Is compression off unless the server administrator enables it, and how do the docs tell users when mixing secret and attacker-controlled data under TLS is unsafe?

Counter-check: held. The backend disables SSL compression for this class of attack, and a v2 frame holds every DataRow up to ReadyForQuery. The thread does not mention the attack or state a default, and the [P3] review asked for default off.

### 3. Frame and flush rules do not cover COPY BOTH replication streams

fixable · level 2, rubric item 7 · WAL / replication / standby, wire protocol · `replication-ondisk-1`

A standby streams WAL through a compressed connection. The walsender treats WAL as sent while part of it still sits in the compression context, and the primary shuts down for a switchover. The standby lacks the last WAL, or a synchronous commit waits on a standby reply held in libpq's compression buffer until wal_sender_timeout.

v2 closes backend frames at ReadyForQuery and frontend frames at CopyDone/CopyFail (message #7). A START_REPLICATION session sends neither for its whole life, so only flush behaviour applies. The walsender uses non-blocking sends and pq_is_send_pending() to decide it has caught up and may exit, and the proposal does not say that a buffered 32 KiB input counts as pending. Message #1 offers replication as a main use, so this path needs a stated invariant.

Thread: [#1 (Anthonin Bonnefoy, 2026-09-29)](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#7 (Andrey Borodin, 2026-10-23)](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru)

**Question:** How does v2 guarantee that every pq_flush, pq_flush_if_writable and PQflush pushes all buffered input to the peer during START_REPLICATION, including walsender shutdown and standby replies?

Counter-check: held. Message #7 says existing flush points still apply, but a START_REPLICATION stream never reaches ReadyForQuery. The thread does not say whether input held in the compression context counts as pending for pq_is_send_pending(), and this cannot be checked without the code.

### 4. Startup option without a protocol version bump leaves negotiation rules open

fixable · level 2, rubric item 12 · wire protocol, ABI / extension API · `standards-compat-2`

A new libpq sends _pq_.supported_compressions to an older server or a proxy that does not know it. The client then needs a defined rule for learning that compression is off, and the proposal gives none. Drivers that implement the protocol themselves cannot tell from the protocol version whether CompressedMessages may appear.

Message #1 adds a new backend message type and says it does not think a version bump is needed, but adds "I'm not sure what are the rules here". v2 keeps the startup-option approach and does not return to the question. Recent protocol work moved toward minor version negotiation, so a committer would likely ask the patch to state its choice and how NegotiateProtocolVersion handles the unknown option.

Thread: [#1 (Anthonin Bonnefoy, 2026-09-29)](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#7 (Andrey Borodin, 2026-10-23)](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru)

**Question:** Why is a _pq_ option enough for a new message type, and how does the client learn that the server or proxy declined it?

Counter-check: held. One part is overstated: NegotiateProtocolVersion already lets a server report an unknown _pq_ option. But message #1 says the author is unsure of the rules, and v2 does not say how libpq treats that reply.

### 5. Client-to-server COPY decompression has no stated memory bound

fixable · level 2, rubric item 7 · security / privileges, wire protocol · `security-2`

A user with only COPY FROM STDIN on a temp table sends zstd frames that declare a very large window or expand at high ratio. Each backend allocates the decompression window and output, and a few connections exhaust server memory. The administrator sees this only when the OOM killer restarts the cluster.

Message #7 adds client-to-server COPY compression, so the server now decompresses data that any authenticated user controls. Message #1 gives a memory cap only for backend compression, not for decompression. The proposal says nothing about a maximum zstd window, a limit on decompressed size per message, or whether the server can turn off inbound compression. libpq has the same exposure when it connects to an untrusted server.

Thread: [#1 (Anthonin Bonnefoy, 2026-09-29)](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#7 (Andrey Borodin, 2026-10-23)](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru)

**Question:** What caps the memory a backend and libpq spend decompressing peer-supplied frames, including the zstd window size, and can the server disable inbound compression?

Counter-check: held. Streaming decompression bounds the output buffer and zstd rejects windows above 128 MiB by default, so the high-ratio part is weak. The window allocation per backend is still peer-controlled, and the thread states no cap or inbound off switch.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | Message #1 offers compression of the WAL stream through primary_conninfo, and CopyData compression covers START_REPLICATION. |
| wire protocol | **yes** | The proposal adds the CompressedMessages message and the _pq_.supported_compressions startup option. |
| catalog changes | no |  |
| ABI / extension API | **yes** | A compression layer stacks on PQcommMethods and changes libpq-be.h and libpq.h, and libpq gains connection parameters. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | v1 adds five protocol_backend_compression GUCs; v2 drops the public ones without naming a replacement control. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | Patch 0007 adds a HIDE_NOTIFICATIONS parameter to psql for regression tests. |
| security / privileges | **yes** | Compression under TLS allows length-leak attacks, and the server decompresses COPY data sent by any authenticated user. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new protocol message, a zstd dependency in the protocol path and a pooler-visible contract stay for ever. |
| performance on the hot path | **yes** | The backend putmessage and flush path and the libpq parse path (pqParseInput3, pqGet*) change for every connection. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. After upgrade, what is the default for a libpq client and for a server, and which parameter lets a client opt out and a server refuse compression?
2. Is compression off unless the server administrator enables it, and what is the stated position on CRIME/BREACH-style length leaks when it runs under TLS?
3. During START_REPLICATION, does every pq_flush, pq_flush_if_writable and PQflush push all input held in the compression context to the peer, including at walsender shutdown?
4. Why is a _pq_ option enough for a new message type without a minor version bump, and how does libpq learn that the server or a proxy declined it?
5. What caps the memory that a backend and libpq spend to decompress frames from the peer, including the zstd window size?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the libpq parse path and the frontend/backend protocol, with a view on TLS interaction and walsender flush behaviour. None has spoken in the visible thread.
- **Liability:** The owner signs for a new protocol message and negotiation rule that every driver and pooler must handle, with no end date. The owner also signs for the zstd dependency in the protocol path and any security report about compression under TLS.
- **Author follow-up:** Both authors are active: Bonnefoy agreed to most scope cuts, and Borodin posted v2 on 2026-10-23 and asked for agreement on a common base.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Five attempts since 2018 show lasting demand, and the two authors are converging on a smaller scope. The patch set has 16 parts across libpq, the backend and the protocol, so review is better spent once v2 states its enable and refuse controls and its security default.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-2` (fixable, level 2) Server has no control over compression CPU cost in v2. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` v2 does not say how users enable or refuse compression. Refuted (not specific): Duplicate of standards-compat-1, which covers the same missing enable and refuse controls and also names the removed server veto and the default after upgrade.
- `prior-attempts-2` Security default and TLS compression risk not addressed. Refuted (not specific): Duplicate of security-1, which states the same length-leak scenario and the same question about a server default of off.
- `standards-compat-3` Pooler forwarding assumes client and server negotiated the same codec. Refuted (no failure scenario): The scenario needs a pooler that requests zstd on its server connection and then forwards to a client that did not advertise it. The pooler sees both startup packets, and message #1 already states the forwarding condition.
- `replication-ondisk-2` v2 drops the only described way to compress replication. Refuted (not specific): Duplicate of standards-compat-1, which already notes that the primary_conninfo recipe from message #1 depends on a GUC that v2 removed.
- `maintenance-fit-1` Pooler-visible wire contract has nothing that enforces it. Refuted (no failure scenario): The scenario depends on a future patch that adds a message type to the compressed set, and v2 compresses only DataRow and CopyData. The missing documentation is already recorded on the CF entry and is outside levels 1-2.
- `maintenance-fit-3` No committer who knows the protocol code has engaged yet. Refuted (not specific): This is a committability note. The rubric says weak committability must not count against the design, and the thread is six weeks old with v2 posted three weeks ago.

### Review angles

- prior-attempts: 2 objection(s). Message #1 cites three earlier attempts, and message #2 reports a fourth by Borodin that the author had missed. That fourth attempt is now being merged into v2.
- standards-compat: 3 objection(s). Three fixable objections. The bundled PQTRACE parameters are left out because v2 uses the existing libpq testclient instead.
- replication-ondisk: 2 objection(s). There is no on-disk, WAL record or catalog change. The concerns are limited to the wire protocol as used by replication connections.
- security: 2 objection(s). Two security objections are raised. Startup and authentication stay uncompressed per message #1, so pre-auth exposure appears unchanged. That is a provisional reading, since v2 negotiation is not described.
- maintenance-fit: 3 objection(s). The objections above are fixable and do not reject the idea. Hot-path cost for clients that do not use compression could not be checked without the code.
- already-exists: 0 objection(s). Core has no way to compress protocol traffic since SSL compression was disabled ([P6]). wal_compression, pg_basebackup compression and pg_receivewal --compress do not cover query results or the streaming replication link. An SSH or VPN tunnel with compression works per link, but it cannot cover many application connections or pooled traffic, and the v1 design reuses the existing compression option syntax instead of adding a second one.
- demand: 0 objection(s). Demand has lasted: five attempts by at least five independent authors since 2018 ([P10], [P1], [P3], [P5], message #2), and the removal of SSL compression left no built-in alternative. The thread still lacks a field report with numbers, and the primary_conninfo replication benefit in message #1 has not been measured.

### Precedents retrieved

- P1: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - used in the review; reason summarised from its thread
- P2: [Add new protocol message to change GUCs to be able to change protocol extension parameters](https://commitfest.postgresql.org/patch/4736/) (cf, Withdrawn, 2025-02-22) - used in the review; reason summarised from its thread
- P3: [libpq compression](https://commitfest.postgresql.org/patch/3499/) (cf, Returned with feedback, 2022-08-02) - used in the review; reason summarised from its thread
- P4: [Use "protocol options" name instead of "protocol extensions" everywhere](https://commitfest.postgresql.org/patch/5341/) (cf, Rejected, 2025-03-11) - judged not relevant; reason summarised from its thread
- P5: [Libpq Compression](https://commitfest.postgresql.org/patch/4746/) (cf, Withdrawn, 2024-05-14) - used in the review; reason summarised from its thread
- P6: [Revert changes for SSL compression in libpq](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0ba71107efeeccde9158f47118f95043afdca0bb) (revert, Reverted, 2021-03-10) - used in the review
- P7: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant; reason summarised from its thread
- P8: [Hooks and API to allow loadable wire protocol extensions](https://commitfest.postgresql.org/patch/3018/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [Compression dictionaries](https://commitfest.postgresql.org/patch/3626/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P10: [libpq compression](https://commitfest.postgresql.org/patch/1635/) (cf, Returned with feedback, 2019-04-09) - used in the review; reason summarised from its thread
- P11: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P12: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P13: [WAL compression setting after PostgreSQL LZ4 default change](https://commitfest.postgresql.org/patch/6979/) (cf, Withdrawn, 2026-07-03) - judged not relevant
- P14: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P15: [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://commitfest.postgresql.org/patch/4166/) (cf, Withdrawn, 2024-01-30) - judged not relevant
- P16: [pg_basebackup: add test about zstd compress option](https://commitfest.postgresql.org/patch/3835/) (cf, Rejected, 2022-12-06) - judged not relevant
- P17: [Fix minor issues in astreamer_zstd.c](https://commitfest.postgresql.org/patch/6386/) (cf, Withdrawn, 2026-01-10) - judged not relevant
- P18: [Refactoring of connection with password prompt loop for frontends](https://commitfest.postgresql.org/patch/2248/) (cf, Returned with feedback, 2019-09-27) - judged not relevant
- P19: [Fix Windows network event handling](https://commitfest.postgresql.org/patch/3523/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P20: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant
- P21: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P22: [Reorder tablespaces for streaming basebackups](https://commitfest.postgresql.org/patch/1035/) (cf, Rejected, 2017-03-31) - judged not relevant
- P23: [libpq - improve PQexec documentation](https://commitfest.postgresql.org/patch/2094/) (cf, Returned with feedback, 2019-08-01) - judged not relevant
- P24: [logical streaming for large in-progress transactions](https://commitfest.postgresql.org/patch/1429/) (cf, Returned with feedback, 2018-10-02) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0009-Make-parse_compress_options-available-to-backend.patch not read
- attachment limit (8) reached; v1-0013-Add-tracing-for-PqMsg_CompressedMessages.patch not read
- attachment limit (8) reached; v1-0011-Create-a-new-msg_buffer-struct-for-PGconn-s-inBuf.patch not read
- attachment limit (8) reached; v1-0012-Add-Frontend-support-for-CompressedMessages.patch not read
- attachment limit (8) reached; v1-0014-Protocol-negotiation-for-compression-algorithms.patch not read
- attachment limit (8) reached; v1-0016-Add-trace-tests-for-compression.patch not read
- attachment limit (8) reached; v1-0015-Add-regress-tests-for-compression.patch not read
- attachment limit (8) reached; v1-0007-Add-HIDE_NOTIFICATIONS-parameter-to-psql-for-regr.patch not read
- attachment limit (8) reached; benchmarks.csv not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 185 s; cost reported by the CLI: $2.54
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 6 of 24 kept
- Tone check: 117 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:45:06.742Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
