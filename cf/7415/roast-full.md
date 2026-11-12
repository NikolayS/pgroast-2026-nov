# Design pre-review: Extensible ReadyForQuery wire protocol message and C hook, for connection pools and WAIT FOR LSN

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7415](https://commitfest.postgresql.org/patch/7415/) (Needs review); authors: Hannu Krosing (hannukrosing)
- Thread: [\[PATCH\] Extensible ReadyForQuery wire protocol message and C hook, for connection pools and WAIT FOR LSN](https://www.postgresql.org/message-id/flat/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), started 2026-10-08
- Messages read: 8 of 8
- Patch prose read: v3-0001-Extend-ReadyForQuery-wire-protocol-message-and-pr.patch, 0001-Extend-ReadyForQuery-wire-protocol-message-and-provi.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The proposal fails at level 1b. It does not compare rich 'Z' with ParameterStatus and GUC_REPORT, which already deliver session state before ReadyForQuery and which old libpq accepts (already-exists-1). The keys 'T', 'H' and 'P' do not cover enough state for a pooler to skip DISCARD ALL (maintenance-fit-2). No pooler maintainer or measured cost appears in the thread (demand-1, demand-2). The goal may be sound, but the list needs these answers before design review.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Level 2 was not evaluated because level 1b fails; surviving design objections on negotiation, key naming, the hook error loop and flag drift remain open. Without the code, the correctness and cost of the 'T', 'H' and 'P' tracking in v3 could not be judged.

## The proposal, restated

**Problem.** Connection poolers in transaction mode cannot tell cheaply whether a backend is clean enough to reuse, so they run DISCARD ALL or extra probe queries. Proxies that route read-your-writes traffic to standbys need the commit LSN for WAIT FOR LSN, which costs an extra round trip today.

**Proposed design.** A new GUC switches the ReadyForQuery ('Z') message between the standard 5-byte form and a 'rich' form that appends length-prefixed key-value pairs. The server fills built-in keys for temp objects, WITH HOLD cursors, prepared statements and the last commit LSN, and extensions can append their own pairs through a hook. libpq is changed to skip trailing bytes in 'Z' and to trace them.

**User-visible surface.**
- GUC ready_for_query_message (enum plain/rich, PGC_USERSET, default plain).
- Protocol: ReadyForQuery length becomes variable, followed by [uint8 key_len][key][uint8 val_len][val] pairs in rich mode.
- Built-in keys 'T', 'H', 'P' ('1'/'0') and 'l' (8-byte big-endian commit LSN).
- C API: ready_for_query_hook, ready_for_query_has_key(), ready_for_query_append_kv().
- libpq: fe-protocol3.c ignores extra 'Z' bytes and PQtrace() prints the pairs.

**What the author claims.**
- Plain mode is byte-identical to master, including startup, with no extra ParameterStatus since v3.
- Poolers and proxies avoid DISCARD ALL and extra LSN queries.
- 'H' and 'P' checks are O(1), and the 'T' catalog scan runs only when a transaction touched temp objects.
- Rich mode should be enabled only with clients that support it; protecting old clients is the user's job.
- check-world, the new module's 42 subtests and libpq_pipeline pass in both modes.

**Thread state.** Only Manu reviewed; he confirmed the v2 fixes in v3 but reported further false-negative 'T' and 'H' cases, a hook that throws ERROR leaving an unkillable backend, and mid-session SET to rich hanging every unpatched libpq. The author has not replied to messages #7 and #8.

## 2. Prior attempts

### Add new protocol message to change GUCs to be able to change protocol extension parameters (related)

- **Link:** [https://commitfest.postgresql.org/patch/4736/](https://commitfest.postgresql.org/patch/4736/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAGECzQTg2hcmb5GaU53uuWcdC7gCNJFLL6mnW0WNhWHgq9UTgw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAGECzQTg2hcmb5GaU53uuWcdC7gCNJFLL6mnW0WNhWHgq9UTgw@mail.gmail.com)
- **Date:** decided 2025-02-22; started 2023-12-29
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the entry and moved the remaining protocol patches to the thread about longer cancel keys. Some pieces were committed here, including a libpq function PQfullProtocolVersion added in place of redefining PQprotocolVersion. The visible messages show no objection that rejected the core design.
- **Reason (quote):** "This required another rebase, but I've decided that I think it'll be most fruitful to continue the discussion on protocol changes in the thread about longer cancel keys[1]." ([message](https://www.postgresql.org/message-id/CAGECzQTk8qAt3dfU7NuEv7nvAH%2Bckk4M28LFYpxs7W1Za%3D%2BeMg%40mail.gmail.com))
- **What is different this time:** P1 built protocol-level negotiation, with a minor version bump and protocol parameters, so poolers see session state changes. This proposal changes a message format through an ordinary GUC and does no negotiation.

### Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user (related)

- **Link:** [https://commitfest.postgresql.org/patch/4166/](https://commitfest.postgresql.org/patch/4166/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAFj8pRBFU-WzzQhNrwRHn67N0Ug8a9-0-9BOo69PPtcHiBDQMA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAFj8pRBFU-WzzQhNrwRHn67N0Ug8a9-0-9BOo69PPtcHiBDQMA@mail.gmail.com)
- **Date:** decided 2024-01-30; started 2023-02-03
- **Outcome:** Withdrawn (cf)
- **Who objected:** Peter Eisentraut, Jelte Fennema-Nio (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the patch and said he would resend it once Jelte Fennema-Nio's _pq_.report_parameters patch is committed. Peter Eisentraut suggested a GUC that lists the settings to report instead of a new protocol message. Jelte Fennema-Nio argued that proxies such as PgBouncer already re-sync GUCs, so a GUC-based design is less fragile for them than a new message type with its own connection state.
- **Reason (quote):** "I think this is probably one of the core reasons why I would very much prefer GUCs over new message types to configure protocol extensions like this: It means proxies would not to keep track of and re-sync a new kind of connection state every time a protocol extension is added." ([message](https://www.postgresql.org/message-id/CAGECzQSg0ND%3DgRmpZHBsiaZT5-a17y%3DVFf3AQfi5N7oECtM1Ug%40mail.gmail.com))
- **What is different this time:** P6 added a new message type to control reporting and was withdrawn in favor of a protocol extension parameter. This proposal uses a GUC, but an ordinary SET-able one, and changes an existing message.


## 3. Objections a reviewer is likely to raise

### 1. ParameterStatus with GUC_REPORT already pushes session state to clients

fixable (first rated fatal; changed in the counter-check) · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), wire protocol, ABI / extension API · `already-exists-1`

A pooler maintainer must parse ParameterStatus for in_hot_standby and other reported settings, and also a second key-value format inside 'Z'. An extension author picks one of two channels for the same kind of data. An application with an older libpq that meets the new channel hangs, where an unknown ParameterStatus would have been ignored.

The server already sends changed GUC_REPORT settings as ParameterStatus just before ReadyForQuery, so each change arrives at the same point in the cycle the patch targets. Extensions can define custom GUC_REPORT variables today, and libpq of every supported version accepts unknown parameter names. Messages #3 and #8 show that the new 'Z' form breaks every unpatched libpq, a cost the existing channel does not have. The thread does not say why 'T', 'H', 'P' and 'l' could not be read-only reported pseudo-variables.

**Precedents.**
- [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://commitfest.postgresql.org/patch/4166/) (Withdrawn, 2024-01-30); thread: [link](https://www.postgresql.org/message-id/flat/CAFj8pRBFU-WzzQhNrwRHn67N0Ug8a9-0-9BOo69PPtcHiBDQMA@mail.gmail.com)

Thread: [#1 (Hannu Krosing, 2026-10-08)](https://www.postgresql.org/message-id/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), [#3 (Manu, 2026-10-08)](https://www.postgresql.org/message-id/179146008286.393055.16733548853161134599%40gmail.com), [#6 (Hannu Krosing, 2026-10-09)](https://www.postgresql.org/message-id/CAMT0RQQ7GiNjmMzOP6rcfUnXPriqVjNNKxbiOpCgh4MGVAMvcQ%40mail.gmail.com), [#8 (Manu, 2026-10-17)](https://www.postgresql.org/message-id/179226954033.1033182.6926602894967135011%40gmail.com)

**Question:** What can the rich 'Z' carry that read-only GUC_REPORT pseudo-variables, sent as ParameterStatus before ReadyForQuery, cannot?

Counter-check: held. The server sends changed GUC_REPORT values just before ReadyForQuery and old libpq accepts unknown names, and the thread does not compare the two. P6 concerns how to configure reporting and is weak support; a later version can switch channels.

### 2. The indicators do not cover enough state to drop DISCARD ALL

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-2`

A pooler trusts T=0, H=0 and P=0 and skips DISCARD ALL. The previous client left a session-level SET, a LISTEN or an advisory lock. The next client gets wrong settings or a held lock, and finds out only when queries behave oddly or block.

DISCARD ALL also resets GUCs, LISTEN, advisory locks, sequence state and plan caches, and none of these has a key. The thread does not say which state a pooler may skip resetting on the basis of these three flags. No maintainer of PgBouncer, pgpool-II or Odyssey has asked for the format or commented on it, and only one reviewer has engaged (message #3 to message #8). Core would then carry a partial answer that its target users have not confirmed they can use.

Thread: [#1 (Hannu Krosing, 2026-10-08)](https://www.postgresql.org/message-id/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), [#6 (Hannu Krosing, 2026-10-09)](https://www.postgresql.org/message-id/CAMT0RQQ7GiNjmMzOP6rcfUnXPriqVjNNKxbiOpCgh4MGVAMvcQ%40mail.gmail.com), [#8 (Manu, 2026-10-17)](https://www.postgresql.org/message-id/179226954033.1033182.6926602894967135011%40gmail.com)

**Question:** Which pooler would skip DISCARD ALL based on T, H and P alone, and how would it handle GUC, LISTEN and advisory-lock state?

Counter-check: held. DISCARD ALL also resets GUCs, LISTEN, advisory locks, sequence state and cached plans, and no key covers these. The thread does not say which reset a pooler may skip using T, H and P alone.

### 3. No pooler or proxy maintainer has asked for rich ReadyForQuery

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), wire protocol, maintenance burden ("who maintains this in 10 years") · `demand-1`

Postgres ships the rich 'Z' format, a GUC and a hook. No pooler adds a parser for it, because none was asked about the format or its key set. The project then maintains a protocol variant and a hook API with no known consumer, and later poolers may want a different shape.

Message #1 and message #6 name PgBouncer, pgpool-II and Odyssey, but no maintainer or user of these tools appears in the thread. The only reviewer, in message #3 and message #7, tested behaviour and did not ask for the feature. Message #1 asks whether a finer setting such as "rich:THL" is wanted, and no consumer was there to answer. A change to a protocol message is hard to take back, so a committer would likely want a named consumer first.

Thread: [#1 (Hannu Krosing, 2026-10-08)](https://www.postgresql.org/message-id/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), [#3 (Manu, 2026-10-08)](https://www.postgresql.org/message-id/179146008286.393055.16733548853161134599%40gmail.com), [#6 (Hannu Krosing, 2026-10-09)](https://www.postgresql.org/message-id/CAMT0RQQ7GiNjmMzOP6rcfUnXPriqVjNNKxbiOpCgh4MGVAMvcQ%40mail.gmail.com), [#7 (Manu, 2026-10-09)](https://www.postgresql.org/message-id/179156812295.351250.10807709444689166669%40gmail.com)

**Question:** Which pooler or proxy maintainers have said they would parse rich 'Z', and which keys and failure behaviour do they need?

Counter-check: held. The thread names three poolers but shows no maintainer or user of them asking for this format. The only reviewer tested behaviour and did not state a need.

### 4. The problem statement does not match how transaction poolers reset today

fixable · level 1b, rubric item 2 · problem statement, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-2`

A PgBouncer user in transaction mode does not run DISCARD ALL between transactions by default, and gives up session state such as temp tables in exchange. Rich 'Z' saves this user no round trip. The users who do run a reset query per release are in session mode, where one reset per disconnect is cheap.

Message #1 and message #6 say transaction-mode poolers must run DISCARD ALL or probe queries to decide whether a backend is reusable, but give no measurement or field report of that cost. The proposal does not say which pooler setting or workload pays the cost today, or how large it is. The LSN case has a working path: a client can run a query for the commit LSN or carry the LSN in the application. The proposal does not show that the extra round trip is a measured bottleneck for anyone.

Thread: [#1 (Hannu Krosing, 2026-10-08)](https://www.postgresql.org/message-id/CAMT0RQSP2Qopgr%2B5Z5NH1wur17%3D2fs0VAn5v3F8TtyQEApCmAQ%40mail.gmail.com), [#6 (Hannu Krosing, 2026-10-09)](https://www.postgresql.org/message-id/CAMT0RQQ7GiNjmMzOP6rcfUnXPriqVjNNKxbiOpCgh4MGVAMvcQ%40mail.gmail.com)

**Question:** Which deployment runs DISCARD ALL or an LSN query per transaction today, and what share of latency or throughput does that round trip cost?

Counter-check: held. PgBouncer does not run its reset query in transaction mode by default, and the thread gives no measurement or field report of the round-trip cost. Messages #1 and #6 assert the cost without naming a deployment.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The 'l' key reports the last commit LSN from xact.c so that proxies can use WAIT FOR LSN on standbys. |
| wire protocol | **yes** | ReadyForQuery becomes variable-length with key-value pairs in rich mode, with no protocol version or _pq_ negotiation. |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds ready_for_query_hook, ready_for_query_has_key() and ready_for_query_append_kv() for extensions. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds ready_for_query_message (enum plain/rich, PGC_USERSET). |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Any user's SET, function or ALTER DATABASE SET can switch the 'Z' format, and a session can suppress its own 'T' flag. |
| backpatchability | **yes** | Released libpq versions 14 and later cannot parse rich 'Z'; only the patched libpq skips the extra bytes. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Core must keep the 'T' and 'H' bookkeeping in xact.c, namespace.c and portalmem.c correct, and keep the key format stable. |
| performance on the hot path | **yes** | Rich mode adds work to every ReadyForQuery; message #3 measured near-zero cost in plain mode. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | ParameterStatus with GUC_REPORT already sends session state to clients before ReadyForQuery. |

## 5. Questions the author must answer before anyone reviews code

1. What can rich 'Z' carry that read-only GUC_REPORT pseudo-variables, sent as ParameterStatus before ReadyForQuery, cannot?
2. Which pooler would skip DISCARD ALL based on 'T', 'H' and 'P' alone, and how would it handle GUC, LISTEN and advisory-lock state?
3. Which pooler or proxy maintainers have said they would parse rich 'Z', and which keys do they need?
4. Which deployment runs DISCARD ALL or an LSN query per transaction today, and what does that round trip cost in latency or throughput?
5. Why does a PGC_USERSET GUC choose the 'Z' format, when a _pq_ parameter or protocol minor version lets the client declare that it can parse it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the frontend/backend protocol and libpq would have to own it, with help on xact.c and portal code. Only one reviewer, Manu, appears in the visible thread.
- **Liability:** The owner signs for a permanent wire format and its key meanings, a hook that runs on every ReadyForQuery, and 'T' and 'H' bookkeeping that every later change to temp objects or portals must keep correct.
- **Author follow-up:** The author posted v2 and v3 in response to review, but has not replied to messages #7 and #8.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** unclear
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The thread names three poolers but shows no maintainer of them asking for the format, and the choice of channel and of opt-in mechanism is open. The patch changes a protocol message and touches xact.c, namespace.c, portalmem.c and libpq, so review is costly.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Wire format switched by a user-settable GUC, without protocol negotiation. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) Extension keys share a one-byte namespace with no collision rule. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-3` (fixable, level 2) Built-in key semantics are not defined precisely enough to freeze. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-3` (fixable, level 2) Hook contract runs extension code where an ERROR cannot be handled. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-1` (fixable, level 2) Session can suppress its own 'T' flag and plant state for the next client. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-2` (fixable, level 2) Any user's code can switch another session's protocol format. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) Mirrored session-state flags drift from the catalog as code changes. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Rich 'Z' changes the wire format without protocol negotiation. Refuted (not specific): Duplicate of standards-compat-1, which states the same missing negotiation and adds the shared-backend case through a transaction pooler.
- `replication-ondisk-1` Rich 'Z' format is switched by a user GUC, without protocol negotiation. Refuted (not specific): Duplicate of standards-compat-1.
- `replication-ondisk-2` Extension-defined keys become wire content with no namespace or limits. Refuted (not specific): Duplicate of standards-compat-2.
- `maintenance-fit-3` An unnegotiated 'Z' format change becomes a permanent wire commitment. Refuted (not specific): Duplicate of standards-compat-1. The permanence concern follows from the same missing negotiation, and a negotiated opt-in in a later version would answer it.
- `maintenance-fit-4` Hook API has no key namespace, error contract or size limit. Refuted (not specific): Duplicate of standards-compat-2 for the namespace and size limit, and of replication-ondisk-3 for the error contract. Keys also carry a uint8 length, so they are not one-byte keys.
- `already-exists-2` Opt-in by GUC bypasses existing protocol negotiation. Refuted (not specific): Duplicate of standards-compat-1.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent records a rejection reason for reporting session state in ReadyForQuery. P2, P9, P16, P10, P21 and P18 share the area, but their recorded reasons are absent or about rebases, so they do not apply.
- standards-compat: 3 objection(s). Objections listed above.
- replication-ondisk: 3 objection(s). The proposal touches no on-disk format, WAL record, catalog or replication protocol. Objections cover only the wire protocol and the extension API.
- security: 2 objection(s). The 'l', 'H' and 'P' values expose only session-local state or WAL positions that any user can already read. The hook error loop needs a superuser-installed extension, so it is out of scope for this angle.
- maintenance-fit: 4 objection(s). Not applicable; four objections are listed.
- already-exists: 2 objection(s). Objections raised. The LSN-without-extra-round-trip claim was not raised separately, because a client can already send COMMIT and an LSN query in one message but a transparent proxy cannot.
- demand: 2 objection(s). Not applicable. The plain-mode cost is negligible per message #3, so the demand objections concern what the project takes on, not a cost on non-users.

### Precedents retrieved

- P1: [Add new protocol message to change GUCs to be able to change protocol extension parameters](https://commitfest.postgresql.org/patch/4736/) (cf, Withdrawn, 2025-02-22) - used in the review; reason summarised from its thread
- P2: [Built-in connection pooler](https://commitfest.postgresql.org/patch/3043/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P3: [Move SLRU data into the regular buffer pool](https://commitfest.postgresql.org/patch/3514/) (cf, Returned with feedback, 2023-09-04) - judged not relevant; reason summarised from its thread
- P4: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P5: [Use "protocol options" name instead of "protocol extensions" everywhere](https://commitfest.postgresql.org/patch/5341/) (cf, Rejected, 2025-03-11) - judged not relevant; reason summarised from its thread
- P6: [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://commitfest.postgresql.org/patch/4166/) (cf, Withdrawn, 2024-01-30) - used in the review; reason summarised from its thread
- P7: [Hooks and API to allow loadable wire protocol extensions](https://commitfest.postgresql.org/patch/3018/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P8: [Send numeric version to clients (server_version_num GUC_REPORT)](https://commitfest.postgresql.org/patch/752/) (cf, Rejected, 2016-08-30) - used in the review; reason summarised from its thread
- P9: [Built-in connection proxy](https://commitfest.postgresql.org/patch/2067/) (cf, Returned with feedback, 2020-09-17) - used in the review; reason summarised from its thread
- P10: [Add and report the new "session_read_only" GUC pseudo-variable.](https://commitfest.postgresql.org/patch/1090/) (cf, Returned with feedback, 2019-02-04) - used in the review
- P11: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P12: [Re: MarkBufferDirtyHint() and LSN update](https://commitfest.postgresql.org/patch/2352/) (cf, Withdrawn, 2019-11-25) - judged not relevant
- P13: [Make async slave to wait for lsn to be replayed](https://commitfest.postgresql.org/patch/772/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P14: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant
- P15: [LSN Map](https://commitfest.postgresql.org/patch/89/) (cf, Returned with feedback, 2015-07-06) - judged not relevant
- P16: [Discarding DISCARD ALL](https://commitfest.postgresql.org/patch/2919/) (cf, Withdrawn, 2021-06-21) - used in the review
- P17: [ALTER SYSTEM READ { ONLY | WRITE }](https://commitfest.postgresql.org/patch/2602/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P18: [Make server_version_num GUC_REPORT](https://commitfest.postgresql.org/patch/749/) (cf, Returned with feedback, 2016-08-30) - used in the review
- P19: [Revise get_cheapest_parallel_safe_total_inner](https://commitfest.postgresql.org/patch/4285/) (cf, Returned with feedback, 2026-02-09) - judged not relevant
- P20: [Revert "Use WAIT FOR LSN in PostgreSQL::Test::Cluster::wait_for_catchup()"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e54ce0b2da62ac71e3b2a5212f32e82b07bf39a5) (revert, Reverted, 2026-01-07) - judged not relevant
- P21: [Add last commit LSN to pg_last_committed_xact()](https://commitfest.postgresql.org/patch/3515/) (cf, Returned with feedback, 2022-08-02) - used in the review
- P22: [quick and dirty fix: Quadratic planning time for ordered paths over partitioned tables](https://commitfest.postgresql.org/patch/5516/) (cf, Rejected, 2025-02-03) - judged not relevant
- P23: [Revert "Cannot use WL_SOCKET_WRITEABLE without WL_SOCKET_READABLE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a58db3aa10e62e4228aa409ba006014fa07a8ca2) (revert, Reverted, 2020-12-15) - judged not relevant
- P24: [Revert "Rename routines for write/read of pgstats file"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=381b3cfe2be02aa114ef7a28c4ec24feaf377c63) (revert, Reverted, 2026-07-14) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 178 s; cost reported by the CLI: $2.85
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 9 of 24 kept
- Tone check: 144 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:53:16.979Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
