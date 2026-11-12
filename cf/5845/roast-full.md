# Design pre-review: Carefully exposing information without authentication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5845](https://commitfest.postgresql.org/patch/5845/) (Needs review); authors: Greg Sabino Mullane (greg)
- Thread: [POC: Carefully exposing information without authentication](https://www.postgresql.org/message-id/flat/CAKAnmm%2BT-CEDLmRezWfH-7ZEsFfD_kU2KY1TgB288K%2BwprB_4Q%40mail.gmail.com), started 2025-05-29
- Messages read: 18 of 18
- Patch prose read: 0008-Allow-specific-information-to-be-output-directly-by-Postgres.patch, 0001-Allow-specific-information-to-be-output-directly-by-Postgres.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**LIKELY-REJECT** (confidence: medium)

The proposal fails at level 1b on already-exists-1. Message #13 itself describes a working script on a separate port, and message #18 recommends that route, so the thread shows the external approach takes setup but does the job. Demand for core placement rests on one F5 site (demand-1), and no operator asks for the version or sysid endpoints (demand-2). Meeting the objection means a separate listener or service, which is a different proposal.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 18 messages and no code were read. The exact-match claim, the socket state restore, the send timeout and the Windows failure after reload could not be checked.

## The proposal, restated

**Problem.** HA systems, load balancers and security scanners need to know whether a server is primary or replica, or which version it runs. Today they must log in and run pg_is_in_recovery(), or guess the version from error output after a bad login. Without Patroni, users write wrapper scripts and create a database user just to answer "is this the primary?".

**Proposed design.** A new GUC expose_information lists which facts the server may reveal. Right after fork, before the startup packet, the backend peeks at the first bytes in non-blocking mode; on an exact match of a known HTTP request it sends a minimal HTTP/1.1 response and exits. Bytes that do not match stay in the socket buffer and normal startup and authentication continue.

**User-visible surface.**
- GUC expose_information (string, comma-separated: role, version, sysid; default empty; changes with a reload).
- HTTP endpoints on the Postgres port: HEAD /replica and HEAD /primary return status 200 or 503 with no body.
- GET /replica and GET /primary return 1 or 0; GET /version returns server_version_num; GET /sysid returns the system identifier.
- New log message "failed to send information to client" when send() fails.

**What the author claims.**
- The facts exposed are low-impact and not a security leak.
- HEAD /replica and HEAD /primary are drop-in replacements for the same Patroni REST API calls.
- GET /version gives scanners an exact answer instead of a guess from source line numbers in error messages.
- Skipping authentication and backend setup makes the check lightweight; early tests showed barely measurable postmaster impact.
- Normal connections are unaffected even when expose_information is set.

**Thread state.** Tom Lane, Daniel Gustafsson and Jacob Champion object to any work on client input before authentication; Tom doubts it will ever be committed, and Jacob calls mixing HTTP and the Postgres protocol on one port a nonstarter. Andres Freund's points on direct TLS, man-in-the-middle trust and a separate interface are answered only partly.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-05-29; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. A small separate service already answers this without core changes

fatal · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), security / privileges · `already-exists-1`

A site with a load balancer and no Patroni enables expose_information to avoid writing a health-check script. Core now carries an unauthenticated HTTP responder on the Postgres port permanently. A side-by-side service would have given the same 200/503 answer and could be bound to a management interface.

The need is met today by the Patroni REST API, or by a short script that runs pg_is_in_recovery() and serves the result on another port, as message #13 describes and message #18 recommends. Libpq clients can already choose the primary with target_session_attrs. Message #13 calls the script approach "a whole lot of trouble", which is a convenience argument. The proposal does not show that the external approach is impossible, only that it takes setup.

Thread: [#13 (Greg Sabino Mullane, 2026-09-07)](https://www.postgresql.org/message-id/CAKAnmm%2BeGs%2B%2BamVMuq-dGXzUOHRNytHgMcaX_5N%2BjvitmBn6MA%40mail.gmail.com), [#18 (Jacob Champion, 2026-09-16)](https://www.postgresql.org/message-id/CAOYmi%2Bkfpag%2BSVDr%2B6yQgzkD0ws%2BUaamkG2bbpVQDr6AbeL-kg%40mail.gmail.com)

**Question:** What does a load balancer gain from core placement that a separate health-check service on its own port and interface cannot give?

Counter-check: held. Message #13 itself describes a working script on a separate port, and message #18 recommends that route. The thread shows that this takes setup, but not that it cannot do the job.

### 2. Demand rests on one consulting client's load balancer

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), security / privileges · `demand-1`

The project ships unauthenticated pre-startup handling of client bytes for a small group of sites that run no Patroni and use a load balancer that cannot speak the Postgres protocol. Every later change to backend startup, direct TLS or the protocol must keep this path safe, even though few installations turn it on.

The only concrete field report is one F5 BIG-IP client in message #13. No other user in the 18 messages asks for the feature, and the thread cites no -general or -bugs reports. The workaround already works, as message #13 itself describes: a small script on a separate port that runs pg_is_in_recovery(). Message #18 suggests the same separate service. The author finds that workaround inconvenient, but that alone does not show the group is large.

Thread: [#13 (Greg Sabino Mullane, 2026-09-07)](https://www.postgresql.org/message-id/CAKAnmm%2BeGs%2B%2BamVMuq-dGXzUOHRNytHgMcaX_5N%2BjvitmBn6MA%40mail.gmail.com), [#18 (Jacob Champion, 2026-09-16)](https://www.postgresql.org/message-id/CAOYmi%2Bkfpag%2BSVDr%2B6yQgzkD0ws%2BUaamkG2bbpVQDr6AbeL-kg%40mail.gmail.com)

**Question:** Which independent users or HA and load balancer vendors need an unauthenticated role check on the Postgres port, and why can a separate small service not serve them?

Counter-check: held. The only field report is the one F5 client in message #13, and no other user in the thread asks for the feature. Patroni's REST API shows the need for HTTP role checks, but not for one on the Postgres port.

### 3. Version and sysid endpoints have no stated user who needs them

fixable · level 1b, rubric item 2 · desirability, security / privileges, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-2`

A DBA enables role for a load balancer and adds version as well, because the documentation presents it as harmless. Anyone who can reach the port can then read the exact minor version without logging in, and match it against known CVEs.

The reason given for GET /version is that scanners guess the version from error output today (messages #1 and #14), so the people who gain are the scanners, not the operators of the database. No operator in the thread asks for the exact version to be readable without a login. GET /sysid is described as useful for checking that a cluster is the same one (message #1), but no tool or user is named who needs this without authentication. Message #13 says people want the version or the role, but the only workflow shown uses the role.

Thread: [#1 (Greg Sabino Mullane, 2025-05-29)](https://www.postgresql.org/message-id/CAKAnmm%2BT-CEDLmRezWfH-7ZEsFfD_kU2KY1TgB288K%2BwprB_4Q%40mail.gmail.com), [#13 (Greg Sabino Mullane, 2026-09-07)](https://www.postgresql.org/message-id/CAKAnmm%2BeGs%2B%2BamVMuq-dGXzUOHRNytHgMcaX_5N%2BjvitmBn6MA%40mail.gmail.com), [#14 (Greg Sabino Mullane, 2026-09-15)](https://www.postgresql.org/message-id/CAKAnmmJBhwNK-vBgRJ0EgBVe_2dLomKOGtt%3DOLFbkA1Kd1eFjA%40mail.gmail.com)

**Question:** Which operators, not scanners, need the exact version or the system identifier without authentication, and what do they use today?

Counter-check: held. The thread names scanners as the users of GET /version and no tool that needs GET /sysid without a login. Message #13 shows a workflow only for role.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The role endpoints report RecoveryInProgress() on primaries and standbys, and load balancers would route on that answer. |
| wire protocol | **yes** | The backend accepts HTTP request lines as first bytes on the Postgres port, next to the startup packet. |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds the string GUC expose_information with the options role, version and sysid. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The server answers before authentication and before pg_hba.conf is consulted, in plain text. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Documented HTTP endpoints, described as Patroni drop-in replacements, must stay stable and be checked against every later startup change. |
| performance on the hot path | **yes** | When the GUC is set, every new connection gets a non-blocking peek in backend_startup.c, and each probe costs a fork. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A separate health-check service or the Patroni REST API gives the same answer outside core. |

## 5. Questions the author must answer before anyone reviews code

1. What does a load balancer gain from an endpoint on the Postgres port that a separate health-check service on its own port and interface cannot give?
2. Which independent users or load balancer vendors, beyond the one F5 site in message #13, ask for an unauthenticated role check on the Postgres port?
3. Which operators, as opposed to scanners, need the exact version or the system identifier without a login, and what do they use today?
4. What happens to a direct TLS connection when expose_information is set, and how does the design prevent cross-protocol attacks between HTTP and the Postgres protocol?
5. What does one fork and exit per probe cost, and how does a flood of such probes affect connection slots?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane, Daniel Gustafsson and Jacob Champion take part, but all three object to input handling before authentication. The owner would need to know backend startup and protocol security, and nobody has offered.
- **Liability:** The owner signs for unauthenticated input handling in backend startup and for HTTP endpoint names that monitoring setups depend on, in every release that ships them. Each later change to connection startup or direct TLS must be checked against the HTTP path.
- **Author follow-up:** The author posted revised patches through at least message #14, answered reviews, and moved the entry through six CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** withdraw and resubmit
- **Audience:** niche
- **Maintenance burden:** high
- **Review cost:** medium
- **Why:** The shown demand is one site that has a load balancer and no cluster manager, while the cost is a permanent HTTP interface before authentication on the Postgres port. The 25 KiB patch is small, but it changes backend_startup.c, and the thread points to a separate listener or service.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fatal, level 2) HTTP on the Postgres port adds a permanent second wire contract. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Local recovery state is not the leader answer Patroni gives. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-2` (fixable, level 2) Unauthenticated role answer can be spoofed to misroute writes. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-3` (fixable, level 2) No address or pg_hba control over who receives version and sysid. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Core would follow Patroni's REST API semantics indefinitely. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-3` (fixable, level 2) No committer in the thread is willing to own pre-authentication code. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-2` Pre-auth endpoints bypass pg_hba.conf and per-address limits. Refuted (not specific): Duplicate of security-3. This version also says the setting works outside listen_addresses, but the endpoints answer only on addresses the server listens on.
- `standards-compat-3` No TLS path, so the check works only as plaintext HTTP. Refuted (not specific): Duplicate of security-2, which states the integrity problem of a plaintext role answer more directly. The missing TLS form is the same gap seen from the monitor's side.
- `standards-compat-4` Claimed Patroni drop-in compatibility does not match the check performed. Refuted (not specific): Duplicate of replication-ondisk-1. Its scenario is also weaker, because Patroni's plain /replica check does not test replication lag unless the caller asks for it.
- `replication-ondisk-2` Second protocol on the Postgres port constrains future wire changes. Refuted (not specific): Duplicate of standards-compat-1 for the wire contract and of security-2 for the missing TLS path. Poolers that do not pass the request through are a limit of the feature and break nothing.
- `security-1` HTTP and the Postgres protocol share one port before authentication. Refuted (no failure scenario): The patch matches exact strings and exits, so the same bytes never reach both parsers; "GET " read as a startup packet length is rejected as too large. The duty to check future startup changes is covered by standards-compat-1.
- `maintenance-fit-1` Health-check service belongs outside the backend startup path. Refuted (not specific): Duplicate of already-exists-1 for the external service argument and of standards-compat-1 for the burden on future startup changes.
- `already-exists-2` A second protocol on the port instead of extending the startup packet. Refuted (already handled): Message #9 answers this: a startup-packet option cannot serve an HTTP-only monitor without a libpq client or a sidecar. The sidecar alternative is already-exists-1, and the burden on later startup changes is standards-compat-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P14] and [P21] would report server_version_num to clients that are already connected through the Postgres protocol, which is a different idea from unauthenticated HTTP replies, and neither row records a reason. The thread cites no earlier hackers discussion; message #1 links only to a blog post about guessing the version.
- standards-compat: 4 objection(s). Precedents P14 and P21 cover reporting the server version to authenticated clients and have no recorded reason, so none were cited.
- replication-ondisk: 2 objection(s). The patch adds no on-disk, WAL or catalog state beyond one GUC. Objections are limited to role semantics across nodes and the wire protocol.
- security: 3 objection(s). Pre-authentication DoS is not raised. Each request forks and is bounded by a send timeout, much like a normal pre-authentication connection, and the material gives no numbers either way.
- maintenance-fit: 3 objection(s). Precedents P14 and P21 have no recorded reason and concern GUC_REPORT for the version number, so they do not support these objections.
- already-exists: 2 objection(s). Precedents P14 and P21 concern sending server_version_num to clients, but no reason is recorded, so they are not used.
- demand: 2 objection(s). The precedents [P14] and [P21] are about reporting server_version_num to clients, but no reason is recorded for either, so neither is cited here.

### Precedents retrieved

- P1: [Revert "Hack pg_ctl to report postmaster's exit status."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f38291e927fa8c04eb772e6a17a3dd44da2b69e8) (revert, Reverted, 2019-10-13) - judged not relevant
- P2: [Revert "Don't truncate database and user names in startup packets."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0ff56e2d3ff1db3de727b33b2dac985ccc43ef8) (revert, Reverted, 2024-12-12) - judged not relevant
- P3: [Implementing OCSP Stapling for Server certificate status verification](https://commitfest.postgresql.org/patch/4849/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P4: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P5: [Revert "Disallow infinite endpoints in generate_series() for timestamps."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29904f5f2fdafbbb96ef3685fd361053b061aeb1) (revert, Reverted, 2022-05-09) - judged not relevant
- P6: [Built-in connection pooler](https://commitfest.postgresql.org/patch/3043/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P7: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P8: [Redacting information from logs](https://commitfest.postgresql.org/patch/2222/) (cf, Withdrawn, 2019-09-05) - judged not relevant; reason summarised from its thread
- P9: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant; reason summarised from its thread
- P10: [Use SIGIO to detect postmaster death](https://commitfest.postgresql.org/patch/3066/) (cf, Withdrawn, 2021-07-13) - judged not relevant; reason summarised from its thread
- P11: [Add LWLock blocker(s) information](https://commitfest.postgresql.org/patch/2576/) (cf, Returned with feedback, 2020-11-18) - judged not relevant; reason summarised from its thread
- P12: [Built-in connection proxy](https://commitfest.postgresql.org/patch/2067/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P13: [add timing information to pg_upgrade](https://commitfest.postgresql.org/patch/4464/) (cf, Withdrawn, 2023-12-02) - judged not relevant; reason summarised from its thread
- P14: [Send numeric version to clients (server_version_num GUC_REPORT)](https://commitfest.postgresql.org/patch/752/) (cf, Rejected, 2016-08-30) - used in the review
- P15: [Allow cluster owner to bypass authentication](https://commitfest.postgresql.org/patch/2244/) (cf, Returned with feedback, 2020-04-06) - judged not relevant
- P16: [Making WAL receiver startup rely on GUC context for primary_conninfo and primary_slot_name](https://commitfest.postgresql.org/patch/1916/) (cf, Rejected, 2019-02-16) - judged not relevant
- P17: [pg_stat_activity: make slow/hanging authentication more visible](https://commitfest.postgresql.org/patch/4974/) (cf, Withdrawn, 2025-03-21) - judged not relevant
- P18: [Addition of authenticated ID to pg_stat_activity](https://commitfest.postgresql.org/patch/3095/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P19: [Allow subfield references without parentheses](https://commitfest.postgresql.org/patch/5449/) (cf, Returned with feedback, 2025-02-14) - judged not relevant
- P20: [deferred primary key and logical replication](https://commitfest.postgresql.org/patch/2631/) (cf, Returned with feedback, 2020-11-29) - judged not relevant
- P21: [Make server_version_num GUC_REPORT](https://commitfest.postgresql.org/patch/749/) (cf, Returned with feedback, 2016-08-30) - used in the review
- P22: [Create a new session for postmaster in pg_ctl](https://commitfest.postgresql.org/patch/1772/) (cf, Returned with feedback, 2018-09-30) - judged not relevant
- P23: [Allow EXPLAIN to Output Page Fault Information](https://commitfest.postgresql.org/patch/5477/) (cf, Withdrawn, 2025-02-08) - judged not relevant
- P24: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 210 s; cost reported by the CLI: $3.05
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 133 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:35:58.532Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
