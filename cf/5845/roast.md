# Design pre-review: Carefully exposing information without authentication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5845](https://commitfest.postgresql.org/patch/5845/); [thread](https://www.postgresql.org/message-id/flat/CAKAnmm%2BT-CEDLmRezWfH-7ZEsFfD_kU2KY1TgB288K%2BwprB_4Q%40mail.gmail.com), 18 of 18 messages read; no code read.

## Verdict

**LIKELY-REJECT** (confidence: medium). The proposal fails at level 1b on already-exists-1. Message #13 itself describes a working script on a separate port, and message #18 recommends that route, so the thread shows the external approach takes setup but does the job. Demand for core placement rests on one F5 site (demand-1), and no operator asks for the version or sysid endpoints (demand-2). Meeting the objection means a separate listener or service, which is a different proposal.

Not checked: Only 3 of 18 messages and no code were read. The exact-match claim, the socket state restore, the send timeout and the Windows failure after reload could not be checked.

## Prior attempts

None found before 2025-05-29.

## Objections a reviewer is likely to raise

### 1. A small separate service already answers this without core changes

A site with a load balancer and no Patroni enables expose_information to avoid writing a health-check script. Core now carries an unauthenticated HTTP responder on the Postgres port permanently. A side-by-side service would have given the same 200/503 answer and could be bound to a management interface.

The need is met today by the Patroni REST API, or by a short script that runs pg_is_in_recovery() and serves the result on another port, as message #13 describes and message #18 recommends. Libpq clients can already choose the primary with target_session_attrs. Message #13 calls the script approach "a whole lot of trouble", which is a convenience argument. The proposal does not show that the external approach is impossible, only that it takes setup.

**Question:** What does a load balancer gain from core placement that a separate health-check service on its own port and interface cannot give?

<sub>fatal, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#13](https://www.postgresql.org/message-id/CAKAnmm%2BeGs%2B%2BamVMuq-dGXzUOHRNytHgMcaX_5N%2BjvitmBn6MA%40mail.gmail.com), [#18](https://www.postgresql.org/message-id/CAOYmi%2Bkfpag%2BSVDr%2B6yQgzkD0ws%2BUaamkG2bbpVQDr6AbeL-kg%40mail.gmail.com); `already-exists-1`</sub>

### 2. Demand rests on one consulting client's load balancer

The project ships unauthenticated pre-startup handling of client bytes for a small group of sites that run no Patroni and use a load balancer that cannot speak the Postgres protocol. Every later change to backend startup, direct TLS or the protocol must keep this path safe, even though few installations turn it on.

The only concrete field report is one F5 BIG-IP client in message #13. No other user in the 18 messages asks for the feature, and the thread cites no -general or -bugs reports. The workaround already works, as message #13 itself describes: a small script on a separate port that runs pg_is_in_recovery(). Message #18 suggests the same separate service. The author finds that workaround inconvenient, but that alone does not show the group is large.

**Question:** Which independent users or HA and load balancer vendors need an unauthenticated role check on the Postgres port, and why can a separate small service not serve them?

<sub>fixable, level 1b, demand; messages [#13](https://www.postgresql.org/message-id/CAKAnmm%2BeGs%2B%2BamVMuq-dGXzUOHRNytHgMcaX_5N%2BjvitmBn6MA%40mail.gmail.com), [#18](https://www.postgresql.org/message-id/CAOYmi%2Bkfpag%2BSVDr%2B6yQgzkD0ws%2BUaamkG2bbpVQDr6AbeL-kg%40mail.gmail.com); `demand-1`</sub>

### 3. Version and sysid endpoints have no stated user who needs them

A DBA enables role for a load balancer and adds version as well, because the documentation presents it as harmless. Anyone who can reach the port can then read the exact minor version without logging in, and match it against known CVEs.

The reason given for GET /version is that scanners guess the version from error output today (messages #1 and #14), so the people who gain are the scanners, not the operators of the database. No operator in the thread asks for the exact version to be readable without a login. GET /sysid is described as useful for checking that a cluster is the same one (message #1), but no tool or user is named who needs this without authentication. Message #13 says people want the version or the role, but the only workflow shown uses the role.

**Question:** Which operators, not scanners, need the exact version or the system identifier without authentication, and what do they use today?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAKAnmm%2BT-CEDLmRezWfH-7ZEsFfD_kU2KY1TgB288K%2BwprB_4Q%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAKAnmm%2BeGs%2B%2BamVMuq-dGXzUOHRNytHgMcaX_5N%2BjvitmBn6MA%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAKAnmmJBhwNK-vBgRJ0EgBVe_2dLomKOGtt%3DOLFbkA1Kd1eFjA%40mail.gmail.com); `demand-2`</sub>

## Questions for the author

1. What does a load balancer gain from an endpoint on the Postgres port that a separate health-check service on its own port and interface cannot give?
2. Which independent users or load balancer vendors, beyond the one F5 site in message #13, ask for an unauthenticated role check on the Postgres port?
3. Which operators, as opposed to scanners, need the exact version or the system identifier without a login, and what do they use today?
4. What happens to a direct TLS connection when expose_information is set, and how does the design prevent cross-protocol attacks between HTTP and the Postgres protocol?
5. What does one fork and exit per probe cost, and how does a flood of such probes affect connection slots?

## Touch points

- **WAL / replication / standby:** The role endpoints report RecoveryInProgress() on primaries and standbys, and load balancers would route on that answer.
- **wire protocol:** The backend accepts HTTP request lines as first bytes on the Postgres port, next to the startup packet.
- **new GUC:** The patch adds the string GUC expose_information with the options role, version and sysid.
- **security / privileges:** The server answers before authentication and before pg_hba.conf is consulted, in plain text.
- **maintenance burden ("who maintains this in 10 years"):** Documented HTTP endpoints, described as Patroni drop-in replacements, must stay stable and be checked against every later startup change.
- **performance on the hot path:** When the GUC is set, every new connection gets a non-blocking peek in backend_startup.c, and each probe costs a fork.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A separate health-check service or the Patroni REST API gives the same answer outside core.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane, Daniel Gustafsson and Jacob Champion take part, but all three object to input handling before authentication. The owner would need to know backend startup and protocol security, and nobody has offered.
- Priority: withdraw and resubmit; audience niche; review cost medium. The shown demand is one site that has a load balancer and no cluster manager, while the cost is a permanent HTTP interface before authentication on the Postgres port. The 25 KiB patch is small, but it changes backend_startup.c, and the thread points to a separate listener or service.

---

[roast-full.md](roast-full.md) has 6 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
