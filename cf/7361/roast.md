# Design pre-review: Protocol Compression (fourth attempt)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7361](https://commitfest.postgresql.org/patch/7361/); [thread](https://www.postgresql.org/message-id/flat/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has lasting demand and nothing survived at levels 1a or 1b. At level 2, v2 does not say how a client or server turns compression on or off (standards-compat-1), which is the kind of question that blocked [P5]. The default also decides the TLS length-leak exposure (security-1) and the replication use case (replication-ondisk-1). These points are fixable, but the list needs to agree on them before code review is useful.

Not checked: Only the first 3 of 7 messages and no code were read, so the v2 negotiation, the flush behaviour during replication and the decompression memory limits could not be checked. The cost on the libpq parse path for connections without compression is also unverified.

## Prior attempts

- [Libpq Compression](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com) (same idea; Withdrawn, 2024-05-14). "I would really like to see this patch go forward, but IMHO these UI questions are blockers." ([message](https://www.postgresql.org/message-id/CA%2BTgmoZ1eU35KwTZ6QPsfjMjpx4fkF8z5rVORfppc%3DY%2B0ZHVVQ%40mail.gmail.com)) Different now: v2 removes most tuning knobs and fixes one codec at startup. It does not yet describe the user interface that replaces them, which is the point that blocked that attempt.
- [libpq compression](https://www.postgresql.org/message-id/flat/ABAA09C6-BB95-47A5-890D-90353533F9AC@yandex-team.ru) (same idea; Returned with feedback, 2022-08-02). "This entry has been waiting on author input for a while (our current threshold is roughly two weeks), so I've marked it Returned with Feedback." ([message](https://www.postgresql.org/message-id/dcb059f2-f226-2aa3-6f34-2ec95ca45813%40timescale.com)) Different now: That attempt wrapped secure_read/write through function pointers. This one works above the socket methods at message level. The security default raised in that review is not yet addressed here.
- [libpq compression](https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru) (same idea; Returned with feedback, 2019-04-09). "I have not actually looked much at your patch, but I am wondering if it could not be possible to make the whole footprint less invasive which really worries me as now shaped." ([message](https://www.postgresql.org/message-id/20180606075330.GK1442%40paquier.xyz)) Different now: Compression sits as a message-level layer in PQcommMethods and sends through the existing transport, so TLS stays underneath. The frontend still changes how all pqGet* callers read the input buffer.
- [libpq compression](https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru) (same idea; Returned with feedback, 2021-12-03). "This patch has been waiting on author for two months now, so I have marked it as RwF in the CF app." ([message](https://www.postgresql.org/message-id/YanH2NuKhZmm2D6w%40paquier.xyz)) Different now: That attempt stalled waiting on its author. This one compresses only server-to-client DataRow and CopyData plus client COPY, and has two authors working together on it.

## Objections a reviewer is likely to raise

### 1. v2 does not define how users or servers turn compression on or off

A site upgrades libpq and the server, and existing applications and standbys start sending zstd-compressed DataRow and CopyData without anyone asking for it. The DBA wants it off for TLS connections or CPU-bound hosts and finds no server setting for that. If instead the default is off, the primary_conninfo recipe from message #1 no longer works.

v2 (message #7) negotiates zstd at startup and drops every public GUC, but it does not say whether libpq advertises zstd by default, which connection parameter controls it, or how the server refuses. In v1, protocol_backend_compression_allowed_algorithms='none' gave the server a veto, and v2 removes it without a replacement. The replication claim in message #1 rests on a GUC that v2 removed. In [P5], unclear compression UI was called a blocker.

Precedent: [Libpq Compression](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com) (Withdrawn, 2024-05-14)

**Question:** What is the default for a libpq client and a server after upgrade, and which parameter lets a client opt out and a server refuse?

<sub>fixable, level 2, new GUC; messages [#1](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/845C3C95-E01F-49D1-AB2F-003356E76AD4%40yandex-team.ru), [#7](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru); `standards-compat-1`</sub>

### 2. Compression under TLS reopens CRIME-style leaks with no server-side default off

An application returns rows that mix a secret column with a column an attacker can write, such as a comment field. Both rows land in one zstd frame that runs until ReadyForQuery. An attacker who can watch TLS record sizes changes the input and reads the secret from the compressed length, and nobody notices.

Postgres disabled SSL compression because of this class of attack. The v2 frame covers every DataRow before ReadyForQuery, so attacker data and secret data share one compression context. The thread does not mention the attack, does not say how v2 turns compression on, and does not say whether the server can refuse it. Review of [P3] asked for the server setting to default to off for security reasons, which applies here.

Precedent: [libpq compression](https://www.postgresql.org/message-id/flat/ABAA09C6-BB95-47A5-890D-90353533F9AC@yandex-team.ru) (Returned with feedback, 2022-08-02)

**Question:** Is compression off unless the server administrator enables it, and how do the docs tell users when mixing secret and attacker-controlled data under TLS is unsafe?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru); `security-1`</sub>

### 3. Frame and flush rules do not cover COPY BOTH replication streams

A standby streams WAL through a compressed connection. The walsender treats WAL as sent while part of it still sits in the compression context, and the primary shuts down for a switchover. The standby lacks the last WAL, or a synchronous commit waits on a standby reply held in libpq's compression buffer until wal_sender_timeout.

v2 closes backend frames at ReadyForQuery and frontend frames at CopyDone/CopyFail (message #7). A START_REPLICATION session sends neither for its whole life, so only flush behaviour applies. The walsender uses non-blocking sends and pq_is_send_pending() to decide it has caught up and may exit, and the proposal does not say that a buffered 32 KiB input counts as pending. Message #1 offers replication as a main use, so this path needs a stated invariant.

**Question:** How does v2 guarantee that every pq_flush, pq_flush_if_writable and PQflush pushes all buffered input to the peer during START_REPLICATION, including walsender shutdown and standby replies?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru); `replication-ondisk-1`</sub>

### 4. Startup option without a protocol version bump leaves negotiation rules open

A new libpq sends _pq_.supported_compressions to an older server or a proxy that does not know it. The client then needs a defined rule for learning that compression is off, and the proposal gives none. Drivers that implement the protocol themselves cannot tell from the protocol version whether CompressedMessages may appear.

Message #1 adds a new backend message type and says it does not think a version bump is needed, but adds "I'm not sure what are the rules here". v2 keeps the startup-option approach and does not return to the question. Recent protocol work moved toward minor version negotiation, so a committer would likely ask the patch to state its choice and how NegotiateProtocolVersion handles the unknown option.

**Question:** Why is a _pq_ option enough for a new message type, and how does the client learn that the server or proxy declined it?

<sub>fixable, level 2, wire protocol; messages [#1](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru); `standards-compat-2`</sub>

### 5. Client-to-server COPY decompression has no stated memory bound

A user with only COPY FROM STDIN on a temp table sends zstd frames that declare a very large window or expand at high ratio. Each backend allocates the decompression window and output, and a few connections exhaust server memory. The administrator sees this only when the OOM killer restarts the cluster.

Message #7 adds client-to-server COPY compression, so the server now decompresses data that any authenticated user controls. Message #1 gives a memory cap only for backend compression, not for decompression. The proposal says nothing about a maximum zstd window, a limit on decompressed size per message, or whether the server can turn off inbound compression. libpq has the same exposure when it connects to an untrusted server.

**Question:** What caps the memory a backend and libpq spend decompressing peer-supplied frames, including the zstd window size, and can the server disable inbound compression?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/CAO6_Xqr-kr5%3D9nh_PvyKHzw5f82M8C-2_NJ84P%2Bs4uLDgyKZQQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/9CCDE409-7813-4A2E-A15C-4D755EF54178%40yandex-team.ru); `security-2`</sub>

## Questions for the author

1. After upgrade, what is the default for a libpq client and for a server, and which parameter lets a client opt out and a server refuse compression?
2. Is compression off unless the server administrator enables it, and what is the stated position on CRIME/BREACH-style length leaks when it runs under TLS?
3. During START_REPLICATION, does every pq_flush, pq_flush_if_writable and PQflush push all input held in the compression context to the peer, including at walsender shutdown?
4. Why is a _pq_ option enough for a new message type without a minor version bump, and how does libpq learn that the server or a proxy declined it?
5. What caps the memory that a backend and libpq spend to decompress frames from the peer, including the zstd window size?

## Touch points

- **WAL / replication / standby:** Message #1 offers compression of the WAL stream through primary_conninfo, and CopyData compression covers START_REPLICATION.
- **wire protocol:** The proposal adds the CompressedMessages message and the _pq_.supported_compressions startup option.
- **ABI / extension API:** A compression layer stacks on PQcommMethods and changes libpq-be.h and libpq.h, and libpq gains connection parameters.
- **new GUC:** v1 adds five protocol_backend_compression GUCs; v2 drops the public ones without naming a replacement control.
- **psql UX conventions:** Patch 0007 adds a HIDE_NOTIFICATIONS parameter to psql for regression tests.
- **security / privileges:** Compression under TLS allows length-leak attacks, and the server decompresses COPY data sent by any authenticated user.
- **maintenance burden ("who maintains this in 10 years"):** A new protocol message, a zstd dependency in the protocol path and a pooler-visible contract stay for ever.
- **performance on the hot path:** The backend putmessage and flush path and the libpq parse path (pqParseInput3, pqGet*) change for every connection.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the libpq parse path and the frontend/backend protocol, with a view on TLS interaction and walsender flush behaviour. None has spoken in the visible thread.
- Priority: fix design first; audience a sizeable minority; review cost high. Five attempts since 2018 show lasting demand, and the two authors are converging on a smaller scope. The patch set has 16 parts across libpq, the backend and the protocol, so review is better spent once v2 states its enable and refuse controls and its security default.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 9.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
