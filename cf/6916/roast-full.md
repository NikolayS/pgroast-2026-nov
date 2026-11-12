# Design pre-review: PROXY protocol support

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6916](https://commitfest.postgresql.org/patch/6916/) (Needs review); authors: Julien Riou (riouj)
- Thread: [PROXY protocol support](https://www.postgresql.org/message-id/flat/CABUevExJ0ifpUEiX4uOREy0s2kHBrBrb%3DpXLEHhpMTR1vVR1XA%40mail.gmail.com), started 2021-03-02
- Messages read: 53 of 53
- Patch prose read: proxy_protocol_11.patch, proxy_protocol.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has shown demand and no objection at level 1 survived. The listener design is not settled: message #52 questions the single-port proxy_networks shape and message #53 only plans a rework (prior-attempts-1, standards-compat-1). Trust for Unix-domain sockets (security-1) and the header-supplied raddr (maintenance-fit-2) also need answers in the design. Code review makes sense once the list agrees on the configuration shape.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only patch prose for v1 and v11 was read; no v12 or v13 prose is shown. The masking of the new pg_stat_activity columns, the full set of raddr users and the connection-time cost can only be checked in the code.

## The proposal, restated

**Problem.** When Postgres runs behind a load balancer or proxy such as HAProxy, Envoy or a cloud load balancer, the server sees the proxy's address instead of the client's. So pg_hba.conf rules, log lines and pg_stat_activity use the wrong address, which matters for audit and conformance requirements.

**Proposed design.** The server accepts the PROXY protocol header, a one-way prefix before the normal startup packet, from trusted proxies and uses the address in it as the client address. Magnus Hagander's patches (v1 to v11) used a separate proxy_port plus a proxy_servers trust list. Julien Riou's rewrite (v12, v13) uses one port and a proxy_networks list, supports protocol v1 and v2, and in message #53 he considers going back to a per-listener port with a trust list.

**User-visible surface.**
- GUC proxy_servers (v1 to v11): list of trusted proxy IPs, CIDRs or the literal unix.
- GUC proxy_port (v3 to v11): separate TCP port that accepts only PROXY connections.
- GUC proxy_networks (v12, v13): trusted networks on the regular port; a connection from them without a header is rejected.
- pg_stat_activity columns for the proxy address, hostname and port, plus proxy fields in stderr, csvlog and jsonlog (v12).
- inet_server_addr() and inet_server_port() return the address and port the client used to reach the proxy; ident auth is disabled for proxied connections.

**What the author claims.**
- The PROXY protocol is a light, widely used standard, supported by HAProxy, Envoy and cloud load balancers.
- A single port is simpler for operators because existing network rules keep working (Riou).
- Late parsing means no performance regression unless proxy_networks is set; no numbers are given.
- An untrusted client that omits the header gets the generic incomplete startup packet error, so scanners cannot detect the feature.
- The TAP suite in src/test/protocol covers the changes; the reviewer reports 71 passing tests.

**Thread state.** Peter Eisentraut questioned the single-port proxy_networks design and suggested a general endpoint list setting; Riou surveyed other products and plans to rework toward a per-listener port with a trust list. Manu reported v12 testing results and asked whether the connection received log line should show the client address.

## 2. Prior attempts

No precedent found (corpus searched for items before 2021-03-02; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Trusting Unix sockets lets any local OS user claim any client IP

fixable · level 2, rubric item 7 · security / privileges · `security-1`

An admin trusts Unix sockets so that a sidecar Envoy can forward connections. Another OS user on the same host, such as an application account, connects to the same socket with a forged header and claims an address that matches a `host ... trust` or weaker pg_hba line. pg_hba accepts the forged address, and the logs record it as the real client.

Message #34 notes that the PROXY socket cannot get its own permissions, so local access to the main socket becomes the power to set the source address. Message #52 notes that all Unix-domain sockets are trusted or none are. Both points were deferred to a later version in messages #41 and #42, and v12 and v13 do not change them. Under this design the trust boundary for a pg_hba bypass is whoever may open the socket file, which nothing in the documentation hunks warns about.

Thread: [#34 (Jacob Champion, 2021-07-14)](https://www.postgresql.org/message-id/0c4b20fd6b33e21f3776cac88219920cac73a883.camel%40vmware.com), [#41 (Jacob Champion, 2021-11-15)](https://www.postgresql.org/message-id/ef8f4431460b96f8edad29fd624f850d4e26f679.camel%40vmware.com), [#42 (Magnus Hagander, 2022-02-25)](https://www.postgresql.org/message-id/CABUevEzoaXRaOXocOUmXVENZ4N6fR%2Bag%3DUr574r6QDHGgSq8YQ%40mail.gmail.com), [#52 (Peter Eisentraut, 2026-10-19)](https://www.postgresql.org/message-id/525eb6a7-fef7-425e-a9b9-c4ccdebe5a11%40eisentraut.org)

**Question:** How does the next version make sure that only the intended proxy, and not every OS user who can open the socket, can send a trusted PROXY header over a Unix socket?

Counter-check: held. Message #34 names unix_socket_group and unix_socket_permissions as the only control, and they cannot be set separately for the PROXY socket. Messages #41 and #42 deferred this, and message #52 confirms v12 treats all Unix-domain sockets alike.

### 2. Single-port design reopens the port-sharing issue the spec warns about

fixable · level 2, rubric item 12 · design, security / privileges, new GUC · `prior-attempts-1`

An operator puts a load balancer subnet into proxy_networks on a host that also has a monitoring agent or a second, non-PROXY proxy on that subnet. Every header-less connection from that subnet is rejected. The operator finds out at the next deploy, when health checks or tools fail, and the only fix is another IP address.

Message #3 quotes the spec text that "explicitly prevents port sharing", and messages #4, #9 and #12 moved Magnus's patches to a separate proxy_port for that reason. The v12 rewrite returns to one port and makes trust depend on source address only. Message #52 says this ties a host to the PROXY protocol, invites IPv4/IPv6 confusion, and does not handle more than one Unix-domain socket. Message #53 says the author is only considering a return to a per-listener design, so the current patch still carries the issue.

Thread: [#3 (Jacob Champion, 2021-03-03)](https://www.postgresql.org/message-id/b097096a23f0c9ee5b4c8d97fcc4aa64ec3fc97c.camel%40vmware.com), [#4 (Magnus Hagander, 2021-03-03)](https://www.postgresql.org/message-id/CABUevEwpMNDB7UGAS596wEu_OrvAdfzZ1tJJHNXsNWvofGOqrQ%40mail.gmail.com), [#9 (Jacob Champion, 2021-03-04)](https://www.postgresql.org/message-id/955367547640a1e24a72d0e79236f49ec7f0dd21.camel%40vmware.com), [#12 (Magnus Hagander, 2021-03-04)](https://www.postgresql.org/message-id/CABUevEy%2BaeHCCtKQMi38tvAjLfUq1Jf7fCpOB2z8V4g3y7Zrhg%40mail.gmail.com), [#52 (Peter Eisentraut, 2026-10-19)](https://www.postgresql.org/message-id/525eb6a7-fef7-425e-a9b9-c4ccdebe5a11%40eisentraut.org), [#53 (Julien Riou, 2026-10-28)](https://www.postgresql.org/message-id/b3614036-bab2-4db8-9344-0d6bd54098d0%40riou.xyz)

**Question:** Will v14 mark individual listeners, Unix-domain sockets included, as PROXY-only, and how does it then handle mixed PROXY and plain clients from one trusted network?

Counter-check: held. Message #52 raises the same points against v12 and v13, and message #53 only says the author is considering a per-listener rework. No posted version answers it yet, so it stands as fixable.

### 3. Listener configuration is unsettled and any new GUC may become permanent

fixable · level 2, rubric item 12 · new GUC, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

Version 20 ships proxy_port or proxy_networks, and operators build configurations and automation around it. A later release adds the endpoint list from message #52 for direct SSL and other listener properties. The project then keeps two overlapping ways to mark a listener as PROXY, with rules for which one wins, for many years.

The thread has used three shapes: proxy_servers only, proxy_port plus proxy_servers, and proxy_networks on the shared port. In message #52 a committer prefers a general endpoint facility because each per-feature GUC covers only one slice, and message #53 plans to return to a separate port. Message #52 also notes that several Unix-domain sockets cannot be marked one by one under the single literal unix. A GUC cannot be removed without breaking configurations, so the shape should be chosen before the first one ships.

Thread: [#4 (Magnus Hagander, 2021-03-03)](https://www.postgresql.org/message-id/CABUevEwpMNDB7UGAS596wEu_OrvAdfzZ1tJJHNXsNWvofGOqrQ%40mail.gmail.com), [#12 (Magnus Hagander, 2021-03-04)](https://www.postgresql.org/message-id/CABUevEy%2BaeHCCtKQMi38tvAjLfUq1Jf7fCpOB2z8V4g3y7Zrhg%40mail.gmail.com), [#25 (Jacob Champion, 2021-03-05)](https://www.postgresql.org/message-id/fafc3f393dd968dfbe71ca98a034472f4fbeeb15.camel%40vmware.com), [#52 (Peter Eisentraut, 2026-10-19)](https://www.postgresql.org/message-id/525eb6a7-fef7-425e-a9b9-c4ccdebe5a11%40eisentraut.org), [#53 (Julien Riou, 2026-10-28)](https://www.postgresql.org/message-id/b3614036-bab2-4db8-9344-0d6bd54098d0%40riou.xyz)

**Question:** How would the chosen PROXY setting fit a later general endpoint list, including separate Unix-domain sockets, without leaving a duplicate GUC to deprecate?

Counter-check: held. A committer asked in message #52 how the setting relates to a general endpoint list. Message #53 picks a narrower shape without saying how it would fit one later, so the question is open.

### 4. Proxied raddr becomes an invariant that future auth code must remember

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), security / privileges · `maintenance-fit-2`

A later patch adds an authentication method or a check that trusts port->raddr as the real TCP peer, as ident does. On proxied connections the address comes from the header instead, and the new code makes a trust decision on it. No test or assertion catches this, so the problem shows up in production.

Message #25 shows that raddr no longer matches a real connection to laddr. It names ident as one case that breaks. Message #26 adds SSL and GSSAPI checks that also had to change. Message #41 says only ident was found, after a manual search of the auth code. The proposal does not say what tells a future author that raddr may be synthetic.

Thread: [#25 (Jacob Champion, 2021-03-05)](https://www.postgresql.org/message-id/fafc3f393dd968dfbe71ca98a034472f4fbeeb15.camel%40vmware.com), [#26 (Magnus Hagander, 2021-03-06)](https://www.postgresql.org/message-id/CABUevEygBkZWyw0qYt8pMZ4jp3DOjMYKoj-68T_qFjeYqoi%2B6g%40mail.gmail.com), [#41 (Jacob Champion, 2021-11-15)](https://www.postgresql.org/message-id/ef8f4431460b96f8edad29fd624f850d4e26f679.camel%40vmware.com)

**Question:** What makes a future auth method or connection check fail loudly, or at least warn the author, if it treats a header-supplied raddr as the real peer?

Counter-check: held. Message #41 says the search for raddr users besides ident was manual and incomplete, and message #42 agrees it needs more review. The v12 rewrite does not say what marks a header-supplied address for later auth code.

### 5. Log lines show two different client addresses for one connection

fixable · level 2, rubric item 10 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

An operator turns on PROXY support and keeps log_connections enabled. The 'connection received' line logs the proxy address, while the authentication line and pg_stat_activity use the client address from the header. Log parsers and ban tools that read source addresses from 'connection received' then attribute every connection to the proxy, and nobody notices until an audit.

Message #51 shows this behaviour in v12 and asks for one of two fixes, but the thread has not chosen. The stated purpose of the feature is correct addresses in logs for audit, per messages #1 and #6, so a first log line with the proxy address works against the purpose. The meaning of an existing log line for proxied connections is part of the user-facing contract and should be fixed in the design.

Thread: [#1 (Magnus Hagander, 2021-03-02)](https://www.postgresql.org/message-id/CABUevExJ0ifpUEiX4uOREy0s2kHBrBrb%3DpXLEHhpMTR1vVR1XA%40mail.gmail.com), [#6 (Bruno Lavoie, 2021-03-03)](https://www.postgresql.org/message-id/CAD%2BGXYOMrNH%3D%3Dn3%3DTuqBqvuo800MGV6SdFvPb88vSwYTH4YUvg%40mail.gmail.com), [#51 (Manu, 2026-09-23)](https://www.postgresql.org/message-id/179012327876.1567269.13282166184972783880%40gmail.com)

**Question:** For a proxied connection, which address will 'connection received' report, and where is the other address logged?

Counter-check: held. Message #51 shows the two addresses in consecutive log lines and offers two fixes. Message #53 does not pick one, and correct addresses in logs are the stated purpose in messages #1 and #6.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | The server reads a PROXY header before the normal startup packet on a connection. |
| catalog changes | **yes** | v12 adds proxy address, host name and port columns to the pg_stat_activity system view. |
| ABI / extension API | **yes** | The patch changes libpq-be.h and libpq.h, and makes raddr a header-supplied address that extensions may read. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The versions add proxy_servers, proxy_port or proxy_networks, and the final shape is not chosen. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | pg_hba.conf matching uses an address taken from a header sent by a trusted proxy; ident is disabled for proxied connections. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | New listener settings, new log and view fields, and a rule that later auth code must not treat raddr as the real peer. |
| performance on the hot path | **yes** | The patch changes connection startup in postmaster.c and pqcomm.c; no benchmark backs the no-regression claim. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which configuration will v14 use: a per-listener PROXY port with a trust list, the shared port with proxy_networks, or a general endpoint list as suggested in message #52?
2. How does the next version stop every OS user who can open a trusted Unix-domain socket from sending a PROXY header with a forged client address?
3. How would the chosen PROXY setting fit a later general endpoint list, including separate Unix-domain sockets, without leaving a duplicate GUC to deprecate?
4. What makes a future authentication method or connection check fail loudly if it treats a header-supplied raddr as the real TCP peer?
5. For a proxied connection, which address does the "connection received" log line report, and where is the other address logged?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Eisentraut commented on the listener design in message #52. The owner needs to know postmaster connection startup, pg_hba.conf matching and the authentication code.
- **Liability:** The owner signs for a pg_hba.conf trust decision based on a proxy-supplied address, for new listener GUCs that cannot be removed later, and for every later auth change that reads raddr. This lasts as long as the feature exists.
- **Author follow-up:** The current author posted v12 and v13, and in message #53 answered the design review with a survey and a rework plan. The entry is tagged as a first patch, and the original author's versions stopped at v11.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Independent users asked for this from 2021 to 2026, and the feature is off by default. The listener configuration and the Unix-domain socket trust rule are open after message #52, so a v14 that settles them should come before detailed review of postmaster and auth changes.

## Appendix

### Refuted in the counter-check

- `security-2` New proxy columns in pg_stat_activity may expose internal addresses. Refuted (no failure scenario): The scenario assumes the new columns are visible to unprivileged roles, which the material does not show. pg_stat_activity already hides client_addr of other sessions from roles without pg_read_all_stats, and only the code can show whether the new columns differ.
- `maintenance-fit-1` Single-purpose listener settings before a general endpoint syntax exists. Refuted (not specific): Duplicate of standards-compat-1, which makes the same point about a later endpoint list and also covers separate Unix-domain sockets.

### Review angles

- prior-attempts: 1 objection(s). No precedent found in the list. The earlier attempt is Magnus Hagander's v1 to v11 in this thread. It stalled on rebase, not on rejection (messages #46 to #48), and its separate-port design is what the author plans to return to in message #53.
- standards-compat: 2 objection(s). Behaviour changes for existing users happen only when the feature is configured. The retrieval found no relevant precedent.
- replication-ondisk: 0 objection(s). From this angle the proposal has no on-disk, WAL, replication-slot or catalog-table state, and adds no new protocol message: the PROXY header is a pre-startup prefix, and existing clients and libpq do not change. The new pg_stat_activity columns and csvlog/jsonlog fields follow how earlier releases added columns there, and the next version's listener design is not yet written, so it could not be checked.
- security: 2 objection(s). Message #51 settled DoS from truncated headers, since authentication_timeout covers it, and scanner detection. A broad proxy_networks CIDR is a trust risk, but message #53 moves back to a per-listener port that a firewall can restrict.
- maintenance-fit: 2 objection(s). The connection-time cost is limited to connections that start up, and the feature is opt-in, so I raise no hot-path objection. A benchmark is still missing. Ownership is a committability matter. The original committer-author has been inactive since 2022, and the current author is new.
- already-exists: 0 objection(s). Postgres has no existing feature, GUC, hook or contrib module that recovers the client address from a proxy, and the header has to be read in the postmaster before authentication, which an extension cannot reach today. Network-level transparent proxying keeps the client IP, but it needs kernel and routing changes and is not available with many load balancers, so it does not cover the same need.
- demand: 0 objection(s). The material shows real and lasting demand from independent users, including audit-driven HAProxy use (message #6), Envoy sidecar production use (message #24) and requests from 2021 to 2026 (messages #48, #49, #51). Message #53 lists more than fifteen other products that support the protocol, and the feature is off by default, so its cost falls mostly on its users.

### Precedents retrieved

- P1: [Built-in connection proxy](https://commitfest.postgresql.org/patch/2067/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P2: [Respect client-initiated CopyDone during logical streaming replication](https://commitfest.postgresql.org/patch/621/) (cf, Withdrawn, 2019-07-08) - judged not relevant; reason summarised from its thread
- P3: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant; reason summarised from its thread
- P4: [Pattern based listeners for asynchronous messaging (LISTEN/NOTIFY)](https://commitfest.postgresql.org/patch/1218/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P5: [Improve handling of dash and plus signs in email address user names, and perhaps improve URL parsing](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Text_Search) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P6: [Revert "Move portal pinning from PL/pgSQL to SPI"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b48b2f8793ef256d19274b4ef6ff587fd47ab553) (revert, Reverted, 2018-01-10) - judged not relevant
- P7: [postgres_fdw: evaluate placeholdervars on remote server](https://commitfest.postgresql.org/patch/1055/) (cf, Returned with feedback, 2017-09-27) - judged not relevant; reason summarised from its thread
- P8: [Revert "Hopefully-portable regression tests for CREATE/ALTER/DROP COLLATION."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f33c53ec5b27a90a0f00ac27d4e5178fcc33168f) (revert, Reverted, 2013-07-03) - judged not relevant
- P9: [Revert my bad decision of about a year ago to make PortalDefineQuery responsible for copying the query string into the new Portal.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1591fcbec77f6544a0f665758bae912c68e6fa42) (revert, Reverted, 2008-04-02) - judged not relevant
- P10: [Embedded server](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P11: [pg_hba_file_settings](https://commitfest.postgresql.org/patch/723/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P12: ["Causal reads" mode for load balancing reads without stale data](https://commitfest.postgresql.org/patch/487/) (cf, Returned with feedback, 2016-04-06) - judged not relevant; reason summarised from its thread
- P13: [SERIALIZABLE on standby servers](https://commitfest.postgresql.org/patch/1799/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P14: ["synchronous_replay" mode for load balancing reads without stale data](https://commitfest.postgresql.org/patch/951/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P15: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P16: [pg_hba_lookup function to get all matching pg_hba.conf entries](https://commitfest.postgresql.org/patch/356/) (cf, Returned with feedback, 2016-04-08) - judged not relevant
- P17: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant
- P18: [Revert setting client_min_messages to 'debug1' in new tests.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5655565c077c53b6e9b4b9bfcdf96439cf3af065) (revert, Reverted, 2019-03-13) - judged not relevant
- P19: [Separate log files for extensions or PG subsystems](https://commitfest.postgresql.org/patch/1427/) (cf, Returned with feedback, 2018-02-07) - judged not relevant
- P20: [Replace polyphase merge with a simple k-way balanced merge](https://commitfest.postgresql.org/patch/959/) (cf, Returned with feedback, 2017-02-27) - judged not relevant
- P21: [server side variables](https://commitfest.postgresql.org/patch/916/) (cf, Returned with feedback, 2017-02-01) - judged not relevant
- P22: [A separate table level option to control compression](https://commitfest.postgresql.org/patch/2001/) (cf, Returned with feedback, 2019-04-08) - judged not relevant
- P23: [base backup client as auxiliary backend process](https://commitfest.postgresql.org/patch/2192/) (cf, Returned with feedback, 2020-02-28) - judged not relevant
- P24: [Add radiustimeout parameter for RADIUS HBA](https://commitfest.postgresql.org/patch/846/) (cf, Returned with feedback, 2016-12-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 142 s; cost reported by the CLI: $3.96
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 83 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:08:20.336Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
