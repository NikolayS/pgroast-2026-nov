# Design pre-review: Extensible ReadyForQuery wire protocol message and C hook, for connection pools and WAIT FOR LSN

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7415](https://commitfest.postgresql.org/patch/7415/); [thread](https://www.postgresql.org/message-id/flat/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The proposal fails at level 1b. It does not compare rich 'Z' with ParameterStatus and GUC_REPORT, which already deliver session state before ReadyForQuery and which old libpq accepts (already-exists-1). The keys 'T', 'H' and 'P' do not cover enough state for a pooler to skip DISCARD ALL (maintenance-fit-2). No pooler maintainer or measured cost appears in the thread (demand-1, demand-2). The goal may be sound, but the list needs these answers before design review.

Not checked: Level 2 was not evaluated because level 1b fails; surviving design objections on negotiation, key naming, the hook error loop and flag drift remain open. Without the code, the correctness and cost of the 'T', 'H' and 'P' tracking in v3 could not be judged.

## Prior attempts

- [Add new protocol message to change GUCs to be able to change protocol extension parameters](https://www.postgresql.org/message-id/flat/CAGECzQTg2hcmb5GaU53uuWcdC7gCNJFLL6mnW0WNhWHgq9UTgw@mail.gmail.com) (related; Withdrawn, 2025-02-22). "This required another rebase, but I've decided that I think it'll be most fruitful to continue the discussion on protocol changes in the thread about longer cancel keys[1]." ([message](https://www.postgresql.org/message-id/CAGECzQTk8qAt3dfU7NuEv7nvAH%2Bckk4M28LFYpxs7W1Za%3D%2BeMg%40mail.gmail.com)) Different now: P1 built protocol-level negotiation, with a minor version bump and protocol parameters, so poolers see session state changes. This proposal changes a message format through an ordinary GUC and does no negotiation.
- [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://www.postgresql.org/message-id/flat/CAFj8pRBFU-WzzQhNrwRHn67N0Ug8a9-0-9BOo69PPtcHiBDQMA@mail.gmail.com) (related; Withdrawn, 2024-01-30). "I think this is probably one of the core reasons why I would very much prefer GUCs over new message types to configure protocol extensions like this: It means proxies would not to keep track of and re-sync a new kind of connection state every time a protocol extension is added." ([message](https://www.postgresql.org/message-id/CAGECzQSg0ND%3DgRmpZHBsiaZT5-a17y%3DVFf3AQfi5N7oECtM1Ug%40mail.gmail.com)) Different now: P6 added a new message type to control reporting and was withdrawn in favor of a protocol extension parameter. This proposal uses a GUC, but an ordinary SET-able one, and changes an existing message.

## Objections a reviewer is likely to raise

### 1. ParameterStatus with GUC_REPORT already pushes session state to clients

A pooler maintainer must parse ParameterStatus for in_hot_standby and other reported settings, and also a second key-value format inside 'Z'. An extension author picks one of two channels for the same kind of data. An application with an older libpq that meets the new channel hangs, where an unknown ParameterStatus would have been ignored.

The server already sends changed GUC_REPORT settings as ParameterStatus just before ReadyForQuery, so each change arrives at the same point in the cycle the patch targets. Extensions can define custom GUC_REPORT variables today, and libpq of every supported version accepts unknown parameter names. Messages #3 and #8 show that the new 'Z' form breaks every unpatched libpq, a cost the existing channel does not have. The thread does not say why 'T', 'H', 'P' and 'l' could not be read-only reported pseudo-variables.

Precedent: [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://www.postgresql.org/message-id/flat/CAFj8pRBFU-WzzQhNrwRHn67N0Ug8a9-0-9BOo69PPtcHiBDQMA@mail.gmail.com) (Withdrawn, 2024-01-30)

**Question:** What can the rich 'Z' carry that read-only GUC_REPORT pseudo-variables, sent as ParameterStatus before ReadyForQuery, cannot?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/179146008286.393055.16733548853161134599%40gmail.com), [#6](https://www.postgresql.org/message-id/CAMT0RQQ7GiNjmMzOP6rcfUnXPriqVjNNKxbiOpCgh4MGVAMvcQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/179226954033.1033182.6926602894967135011%40gmail.com); `already-exists-1`</sub>

### 2. The indicators do not cover enough state to drop DISCARD ALL

A pooler trusts T=0, H=0 and P=0 and skips DISCARD ALL. The previous client left a session-level SET, a LISTEN or an advisory lock. The next client gets wrong settings or a held lock, and finds out only when queries behave oddly or block.

DISCARD ALL also resets GUCs, LISTEN, advisory locks, sequence state and plan caches, and none of these has a key. The thread does not say which state a pooler may skip resetting on the basis of these three flags. No maintainer of PgBouncer, pgpool-II or Odyssey has asked for the format or commented on it, and only one reviewer has engaged (message #3 to message #8). Core would then carry a partial answer that its target users have not confirmed they can use.

**Question:** Which pooler would skip DISCARD ALL based on T, H and P alone, and how would it handle GUC, LISTEN and advisory-lock state?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAMT0RQQ7GiNjmMzOP6rcfUnXPriqVjNNKxbiOpCgh4MGVAMvcQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/179226954033.1033182.6926602894967135011%40gmail.com); `maintenance-fit-2`</sub>

### 3. No pooler or proxy maintainer has asked for rich ReadyForQuery

Postgres ships the rich 'Z' format, a GUC and a hook. No pooler adds a parser for it, because none was asked about the format or its key set. The project then maintains a protocol variant and a hook API with no known consumer, and later poolers may want a different shape.

Message #1 and message #6 name PgBouncer, pgpool-II and Odyssey, but no maintainer or user of these tools appears in the thread. The only reviewer, in message #3 and message #7, tested behaviour and did not ask for the feature. Message #1 asks whether a finer setting such as "rich:THL" is wanted, and no consumer was there to answer. A change to a protocol message is hard to take back, so a committer would likely want a named consumer first.

**Question:** Which pooler or proxy maintainers have said they would parse rich 'Z', and which keys and failure behaviour do they need?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/179146008286.393055.16733548853161134599%40gmail.com), [#6](https://www.postgresql.org/message-id/CAMT0RQQ7GiNjmMzOP6rcfUnXPriqVjNNKxbiOpCgh4MGVAMvcQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/179156812295.351250.10807709444689166669%40gmail.com); `demand-1`</sub>

### 4. The problem statement does not match how transaction poolers reset today

A PgBouncer user in transaction mode does not run DISCARD ALL between transactions by default, and gives up session state such as temp tables in exchange. Rich 'Z' saves this user no round trip. The users who do run a reset query per release are in session mode, where one reset per disconnect is cheap.

Message #1 and message #6 say transaction-mode poolers must run DISCARD ALL or probe queries to decide whether a backend is reusable, but give no measurement or field report of that cost. The proposal does not say which pooler setting or workload pays the cost today, or how large it is. The LSN case has a working path: a client can run a query for the commit LSN or carry the LSN in the application. The proposal does not show that the extra round trip is a measured bottleneck for anyone.

**Question:** Which deployment runs DISCARD ALL or an LSN query per transaction today, and what share of latency or throughput does that round trip cost?

<sub>fixable, level 1b, problem statement; messages [#1](https://www.postgresql.org/message-id/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAMT0RQQ7GiNjmMzOP6rcfUnXPriqVjNNKxbiOpCgh4MGVAMvcQ%40mail.gmail.com); `demand-2`</sub>

## Questions for the author

1. What can rich 'Z' carry that read-only GUC_REPORT pseudo-variables, sent as ParameterStatus before ReadyForQuery, cannot?
2. Which pooler would skip DISCARD ALL based on 'T', 'H' and 'P' alone, and how would it handle GUC, LISTEN and advisory-lock state?
3. Which pooler or proxy maintainers have said they would parse rich 'Z', and which keys do they need?
4. Which deployment runs DISCARD ALL or an LSN query per transaction today, and what does that round trip cost in latency or throughput?
5. Why does a PGC_USERSET GUC choose the 'Z' format, when a _pq_ parameter or protocol minor version lets the client declare that it can parse it?

## Touch points

- **WAL / replication / standby:** The 'l' key reports the last commit LSN from xact.c so that proxies can use WAIT FOR LSN on standbys.
- **wire protocol:** ReadyForQuery becomes variable-length with key-value pairs in rich mode, with no protocol version or _pq_ negotiation.
- **ABI / extension API:** The patch adds ready_for_query_hook, ready_for_query_has_key() and ready_for_query_append_kv() for extensions.
- **new GUC:** The patch adds ready_for_query_message (enum plain/rich, PGC_USERSET).
- **security / privileges:** Any user's SET, function or ALTER DATABASE SET can switch the 'Z' format, and a session can suppress its own 'T' flag.
- **backpatchability:** Released libpq versions 14 and later cannot parse rich 'Z'; only the patched libpq skips the extra bytes.
- **maintenance burden ("who maintains this in 10 years"):** Core must keep the 'T' and 'H' bookkeeping in xact.c, namespace.c and portalmem.c correct, and keep the key format stable.
- **performance on the hot path:** Rich mode adds work to every ReadyForQuery; message #3 measured near-zero cost in plain mode.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** ParameterStatus with GUC_REPORT already sends session state to clients before ReadyForQuery.

## Who would commit it, and when

- Owner: silent. A committer who knows the frontend/backend protocol and libpq would have to own it, with help on xact.c and portal code. Only one reviewer, Manu, appears in the visible thread.
- Priority: needs design rethink; audience unclear; review cost high. The thread names three poolers but shows no maintainer of them asking for the format, and the choice of channel and of opt-in mechanism is open. The patch changes a protocol message and touches xact.c, namespace.c, portalmem.c and libpq, so review is costly.

---

[roast-full.md](roast-full.md) has 7 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
