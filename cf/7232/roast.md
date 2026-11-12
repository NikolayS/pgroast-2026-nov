# Design pre-review: Protocol compression: a fourth design

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7232](https://commitfest.postgresql.org/patch/7232/); [thread](https://www.postgresql.org/message-id/flat/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Demand is shown and no surviving objection is fatal, so levels 1a and 1b pass. Level 2 fails on fixable points that the list must settle before code review: replication-ondisk-1 (one compression history across all roles' WAL), maintenance-fit-2 and standards-compat-1 (first protocol extension and client option fixed before the planned codecs, levels and per-direction control have a shape), and replication-ondisk-2 (pooler contract). The author's three design questions in message #1 have no answer yet.

Not checked: Without the code I could not check where compression sits relative to secure_read and secure_write, how the walsender uses it, or the threshold for leaving small results uncompressed. The cost for connections that do not negotiate compression is also unchecked.

## Prior attempts

- [Libpq Compression](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com) (same idea; Withdrawn, 2024-05-14). "I would really like to see this patch go forward, but IMHO these UI questions are blockers." ([message](https://www.postgresql.org/message-id/CA%2BTgmoZ1eU35KwTZ6QPsfjMjpx4fkF8z5rVORfppc%3DY%2B0ZHVVQ%40mail.gmail.com)) Different now: The interface is smaller: off | zstd on the server and off | prefer | zstd in libpq, with no per-direction syntax. The new design adds no I/O stream abstraction in place of secure_read and secure_write.
- [libpq compression](https://www.postgresql.org/message-id/flat/ABAA09C6-BB95-47A5-890D-90353533F9AC@yandex-team.ru) (same idea; Returned with feedback, 2022-08-02). "This entry has been waiting on author input for a while (our current threshold is roughly two weeks), so I've marked it Returned with Feedback." ([message](https://www.postgresql.org/message-id/dcb059f2-f226-2aa3-6f34-2ec95ca45813%40timescale.com)) Different now: [P3] used the same _pq_.compression name and CompressedData message, with several codecs and full traffic in both directions. This proposal makes the server setting default to off and limits compression to bulk-data messages.
- [libpq compression](https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru) (same idea; Returned with feedback, 2021-12-03). "This patch has been waiting on author for two months now, so I have marked it as RwF in the CF app." ([message](https://www.postgresql.org/message-id/YanH2NuKhZmm2D6w%40paquier.xyz)) Different now: Zstandard only, compression starts after the first ReadyForQuery, and it covers only DataRow and CopyData. Streams end at ReadyForQuery and COPY boundaries.
- [libpq compression](https://www.postgresql.org/message-id/flat/aad16e41-b3f9-e89d-fa57-fb4c694bec25@postgrespro.ru) (similar idea; Returned with feedback, 2019-04-09). "I have not actually looked much at your patch, but I am wondering if it could not be possible to make the whole footprint less invasive which really worries me as now shaped." ([message](https://www.postgresql.org/message-id/20180606075330.GK1442%40paquier.xyz)) Different now: [P7] compressed the whole byte stream through a layer around socket reads and writes. This proposal wraps selected protocol messages in CompressedData and keeps queries and control messages ordinary.

## Objections a reviewer is likely to raise

### 1. Replication streams share one compression history across all users' WAL

An administrator sets protocol_compression = zstd, and a standby uses compression=prefer in primary_conninfo over TLS. A low-privilege user on the primary inserts chosen values next to a secret column in another role's table, then watches the replication link. Compressed CopyData lengths over the long-lived walsender stream leak whether the guesses match recent WAL content.

Message #1 names WAL streaming and base backup as covered, and walsender.c is touched. The only reset point is ReadyForQuery, and a START_REPLICATION session can run for days without one. One 64 KiB history then mixes WAL from every database and role. The libpq warning speaks of values in "the same compressed response", and the per-server switch in message #1 does not separate replication connections from ordinary ones.

**Question:** Do replication and base backup connections use compression, and if so, what bounds history sharing across roles in a walsender stream, or how is it disabled separately?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru); `replication-ondisk-1`</sub>

### 2. First protocol extension fixes scope while the scope question is open

Drivers and poolers implement _pq_.compression=zstd with the v1 rules: backend DataRow and CopyData, frontend CopyData only, frames that end at ReadyForQuery. A later release wants to compress Bind parameters, add levels, or allow per-direction control. It then needs a second extension or new values, and the project keeps both sets of rules on the server and in libpq indefinitely.

Message #1 asks whether the message scope is "future-proof" and defers levels, more codecs and per-direction control to follow-ups. The documentation hunk defines exact limits and frame rules that become part of the documented protocol. Message #1 says the framing need not change, but the scope and boundary rules are what external implementations will copy. Per-direction control was the UI point that blocked [P4].

Precedent: [Libpq Compression](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com) (Withdrawn, 2024-05-14)

**Question:** How would per-direction control, compression levels and a wider message scope be negotiated later without a second set of rules that servers and libpq must keep?

<sub>fixable, level 2, wire protocol; messages [#1](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru); `maintenance-fit-2`</sub>

### 3. Client option mixes policy and codec, with no room for more codecs

A later release adds lz4 and compression levels, as message #1 plans. A user who wants "lz4 or zstd, whichever the server allows", or wants zstd at a set level, cannot say so with off | prefer | zstd. libpq then needs a second syntax while connection strings written for the first one stay in use.

Message #1 says new codecs and levels need no framing change, but the client option does change: "prefer" silently means zstd, and a method name means "required". Postgres already separates policy from detail elsewhere: sslmode and sslnegotiation, and pg_dump and pg_basebackup use method[:detail] specs. A committer would likely ask for the full future syntax before the first release, because [P4] stalled on exactly this kind of unclear compression UI.

Precedent: [Libpq Compression](https://www.postgresql.org/message-id/flat/CACzsqT4cJG0kaCbz24Sd=GAEgiQDpzU8yuD6vF25zo870+3M6g@mail.gmail.com) (Withdrawn, 2024-05-14)

**Question:** What connection string will a user write to request lz4-or-zstd at a given level, and does today's syntax extend to it without changing meaning?

<sub>fixable, level 2, psql UX conventions; messages [#1](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru); `standards-compat-1`</sub>

### 4. Transaction poolers must match compression state per server connection

A transaction-mode pooler forwards a client's _pq_.compression=zstd when it opens a server connection. It later gives that server connection to a client that did not negotiate compression. That client receives a 'z' message in the middle of a result and drops the connection with a protocol error. The user sees this only under load, when the pool mixes clients.

Message #1 says a pooler that passes the negotiation through can relay CompressedData without understanding Zstandard. Negotiation is per server connection, but a transaction pooler shares server connections between clients. To relay safely, the pooler must understand the extension and keep separate pools, or it must decompress. As the first defined protocol extension, this sets the pattern for later ones, so the pooler contract matters beyond this feature.

**Question:** What must a transaction-mode pooler do with _pq_.compression and CompressedData when its server connections are shared by clients with different settings?

<sub>fixable, level 2, wire protocol; messages [#1](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru); `replication-ondisk-2`</sub>

### 5. prefer fallback is shown only against a server, not against poolers

An administrator sets PGCOMPRESSION=prefer for all clients. Some clients connect through an older pooler or proxy that answers an unknown _pq_ startup parameter with a FATAL error rather than NegotiateProtocolVersion. Those clients cannot connect at all, and the cause is not obvious.

_pq_.compression is the first protocol extension that is ever defined, so no middleware has seen one in production. Message #1 reports a fallback test only against an unmodified Postgres 14 server. It says a pooler can relay CompressedData, but it does not say how a pooler that does not know the extension handles the startup parameter. The brief lists this as unclear.

**Question:** How do common poolers and proxies react today to an unknown _pq_ startup parameter, and does prefer fall back cleanly when one of them rejects it?

<sub>fixable, level 2, wire protocol; messages [#1](https://www.postgresql.org/message-id/CF7DB21D-F9CF-4FD1-8966-47C5B560471F%40yandex-team.ru); `standards-compat-2`</sub>

## Questions for the author

1. Do replication and base backup connections use compression, and if so, what bounds history sharing across roles in a walsender stream that never reaches ReadyForQuery?
2. How will more codecs, compression levels, per-direction control and a wider message scope be negotiated under _pq_.compression without a second set of rules?
3. What connection string will a user write for lz4-or-zstd at a given level, and do off | prefer | zstd keep their meaning then?
4. What must a transaction-mode pooler do with _pq_.compression and CompressedData when clients with different settings share its server connections?
5. Does prefer retry without the extension when a pooler or proxy rejects the unknown _pq_ startup parameter with a FATAL error?

## Touch points

- **WAL / replication / standby:** walsender.c is touched, and message #1 lists base backup and WAL streaming as covered by backend CopyData compression.
- **wire protocol:** The patch defines the first protocol extension, _pq_.compression, and a new CompressedData message, Byte1('z').
- **ABI / extension API:** libpq gains a compression connection option and PGCOMPRESSION, and third-party drivers and poolers must implement the new message to take part.
- **new GUC:** New server GUC protocol_compression = off | zstd, default off.
- **psql UX conventions:** psql gets compression through the libpq option, but the proposal names no \conninfo output or libpq call for the negotiated state.
- **security / privileges:** Compression happens before SSL or GSS encryption, so compressed lengths can leak data; the only server control is one cluster-wide GUC.
- **backpatchability:** New libpq with prefer must connect to older servers and poolers; message #1 tests this only against a Postgres 14 server.
- **maintenance burden ("who maintains this in 10 years"):** The documented frame rules, size limits and boundary rules become protocol that the server, libpq, drivers and poolers must keep.
- **performance on the hot path:** The patch touches pqcomm.c, postgres.c and fe-protocol3.c on the message send and receive paths; default is off.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the wire protocol, libpq and the walsender; the visible thread has only the author and one tester.
- Priority: fix design first; audience a sizeable minority; review cost high. Four earlier entries since 2018 and the measurements in message #2 show lasting demand from users on slow or metered links. An 88 KiB patch across backend, walsender and libpq adds permanent protocol rules, and the scope, pooler and interface questions are still open.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
