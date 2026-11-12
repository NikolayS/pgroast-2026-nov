# Design pre-review: PROXY protocol support

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6916](https://commitfest.postgresql.org/patch/6916/); [thread](https://www.postgresql.org/message-id/flat/CABUevExJ0ifpUEiX4uOREy0s2kHBrBrb%3DpXLEHhpMTR1vVR1XA%40mail.gmail.com), 53 of 53 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has shown demand and no objection at level 1 survived. The listener design is not settled: message #52 questions the single-port proxy_networks shape and message #53 only plans a rework (prior-attempts-1, standards-compat-1). Trust for Unix-domain sockets (security-1) and the header-supplied raddr (maintenance-fit-2) also need answers in the design. Code review makes sense once the list agrees on the configuration shape.

Not checked: Only patch prose for v1 and v11 was read; no v12 or v13 prose is shown. The masking of the new pg_stat_activity columns, the full set of raddr users and the connection-time cost can only be checked in the code.

## Prior attempts

None found before 2021-03-02.

## Objections a reviewer is likely to raise

### 1. Trusting Unix sockets lets any local OS user claim any client IP

An admin trusts Unix sockets so that a sidecar Envoy can forward connections. Another OS user on the same host, such as an application account, connects to the same socket with a forged header and claims an address that matches a `host ... trust` or weaker pg_hba line. pg_hba accepts the forged address, and the logs record it as the real client.

Message #34 notes that the PROXY socket cannot get its own permissions, so local access to the main socket becomes the power to set the source address. Message #52 notes that all Unix-domain sockets are trusted or none are. Both points were deferred to a later version in messages #41 and #42, and v12 and v13 do not change them. Under this design the trust boundary for a pg_hba bypass is whoever may open the socket file, which nothing in the documentation hunks warns about.

**Question:** How does the next version make sure that only the intended proxy, and not every OS user who can open the socket, can send a trusted PROXY header over a Unix socket?

<sub>fixable, level 2, security / privileges; messages [#34](https://www.postgresql.org/message-id/0c4b20fd6b33e21f3776cac88219920cac73a883.camel%40vmware.com), [#41](https://www.postgresql.org/message-id/ef8f4431460b96f8edad29fd624f850d4e26f679.camel%40vmware.com), [#42](https://www.postgresql.org/message-id/CABUevEzoaXRaOXocOUmXVENZ4N6fR%2Bag%3DUr574r6QDHGgSq8YQ%40mail.gmail.com), [#52](https://www.postgresql.org/message-id/525eb6a7-fef7-425e-a9b9-c4ccdebe5a11%40eisentraut.org); `security-1`</sub>

### 2. Single-port design reopens the port-sharing issue the spec warns about

An operator puts a load balancer subnet into proxy_networks on a host that also has a monitoring agent or a second, non-PROXY proxy on that subnet. Every header-less connection from that subnet is rejected. The operator finds out at the next deploy, when health checks or tools fail, and the only fix is another IP address.

Message #3 quotes the spec text that "explicitly prevents port sharing", and messages #4, #9 and #12 moved Magnus's patches to a separate proxy_port for that reason. The v12 rewrite returns to one port and makes trust depend on source address only. Message #52 says this ties a host to the PROXY protocol, invites IPv4/IPv6 confusion, and does not handle more than one Unix-domain socket. Message #53 says the author is only considering a return to a per-listener design, so the current patch still carries the issue.

**Question:** Will v14 mark individual listeners, Unix-domain sockets included, as PROXY-only, and how does it then handle mixed PROXY and plain clients from one trusted network?

<sub>fixable, level 2, design; messages [#3](https://www.postgresql.org/message-id/b097096a23f0c9ee5b4c8d97fcc4aa64ec3fc97c.camel%40vmware.com), [#4](https://www.postgresql.org/message-id/CABUevEwpMNDB7UGAS596wEu_OrvAdfzZ1tJJHNXsNWvofGOqrQ%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/955367547640a1e24a72d0e79236f49ec7f0dd21.camel%40vmware.com), [#12](https://www.postgresql.org/message-id/CABUevEy%2BaeHCCtKQMi38tvAjLfUq1Jf7fCpOB2z8V4g3y7Zrhg%40mail.gmail.com), [#52](https://www.postgresql.org/message-id/525eb6a7-fef7-425e-a9b9-c4ccdebe5a11%40eisentraut.org), [#53](https://www.postgresql.org/message-id/b3614036-bab2-4db8-9344-0d6bd54098d0%40riou.xyz); `prior-attempts-1`</sub>

### 3. Listener configuration is unsettled and any new GUC may become permanent

Version 20 ships proxy_port or proxy_networks, and operators build configurations and automation around it. A later release adds the endpoint list from message #52 for direct SSL and other listener properties. The project then keeps two overlapping ways to mark a listener as PROXY, with rules for which one wins, for many years.

The thread has used three shapes: proxy_servers only, proxy_port plus proxy_servers, and proxy_networks on the shared port. In message #52 a committer prefers a general endpoint facility because each per-feature GUC covers only one slice, and message #53 plans to return to a separate port. Message #52 also notes that several Unix-domain sockets cannot be marked one by one under the single literal unix. A GUC cannot be removed without breaking configurations, so the shape should be chosen before the first one ships.

**Question:** How would the chosen PROXY setting fit a later general endpoint list, including separate Unix-domain sockets, without leaving a duplicate GUC to deprecate?

<sub>fixable, level 2, new GUC; messages [#4](https://www.postgresql.org/message-id/CABUevEwpMNDB7UGAS596wEu_OrvAdfzZ1tJJHNXsNWvofGOqrQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CABUevEy%2BaeHCCtKQMi38tvAjLfUq1Jf7fCpOB2z8V4g3y7Zrhg%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/fafc3f393dd968dfbe71ca98a034472f4fbeeb15.camel%40vmware.com), [#52](https://www.postgresql.org/message-id/525eb6a7-fef7-425e-a9b9-c4ccdebe5a11%40eisentraut.org), [#53](https://www.postgresql.org/message-id/b3614036-bab2-4db8-9344-0d6bd54098d0%40riou.xyz); `standards-compat-1`</sub>

### 4. Proxied raddr becomes an invariant that future auth code must remember

A later patch adds an authentication method or a check that trusts port->raddr as the real TCP peer, as ident does. On proxied connections the address comes from the header instead, and the new code makes a trust decision on it. No test or assertion catches this, so the problem shows up in production.

Message #25 shows that raddr no longer matches a real connection to laddr. It names ident as one case that breaks. Message #26 adds SSL and GSSAPI checks that also had to change. Message #41 says only ident was found, after a manual search of the auth code. The proposal does not say what tells a future author that raddr may be synthetic.

**Question:** What makes a future auth method or connection check fail loudly, or at least warn the author, if it treats a header-supplied raddr as the real peer?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#25](https://www.postgresql.org/message-id/fafc3f393dd968dfbe71ca98a034472f4fbeeb15.camel%40vmware.com), [#26](https://www.postgresql.org/message-id/CABUevEygBkZWyw0qYt8pMZ4jp3DOjMYKoj-68T_qFjeYqoi%2B6g%40mail.gmail.com), [#41](https://www.postgresql.org/message-id/ef8f4431460b96f8edad29fd624f850d4e26f679.camel%40vmware.com); `maintenance-fit-2`</sub>

### 5. Log lines show two different client addresses for one connection

An operator turns on PROXY support and keeps log_connections enabled. The 'connection received' line logs the proxy address, while the authentication line and pg_stat_activity use the client address from the header. Log parsers and ban tools that read source addresses from 'connection received' then attribute every connection to the proxy, and nobody notices until an audit.

Message #51 shows this behaviour in v12 and asks for one of two fixes, but the thread has not chosen. The stated purpose of the feature is correct addresses in logs for audit, per messages #1 and #6, so a first log line with the proxy address works against the purpose. The meaning of an existing log line for proxied connections is part of the user-facing contract and should be fixed in the design.

**Question:** For a proxied connection, which address will 'connection received' report, and where is the other address logged?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CABUevExJ0ifpUEiX4uOREy0s2kHBrBrb%3DpXLEHhpMTR1vVR1XA%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAD%2BGXYOMrNH%3D%3Dn3%3DTuqBqvuo800MGV6SdFvPb88vSwYTH4YUvg%40mail.gmail.com), [#51](https://www.postgresql.org/message-id/179012327876.1567269.13282166184972783880%40gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Which configuration will v14 use: a per-listener PROXY port with a trust list, the shared port with proxy_networks, or a general endpoint list as suggested in message #52?
2. How does the next version stop every OS user who can open a trusted Unix-domain socket from sending a PROXY header with a forged client address?
3. How would the chosen PROXY setting fit a later general endpoint list, including separate Unix-domain sockets, without leaving a duplicate GUC to deprecate?
4. What makes a future authentication method or connection check fail loudly if it treats a header-supplied raddr as the real TCP peer?
5. For a proxied connection, which address does the "connection received" log line report, and where is the other address logged?

## Touch points

- **wire protocol:** The server reads a PROXY header before the normal startup packet on a connection.
- **catalog changes:** v12 adds proxy address, host name and port columns to the pg_stat_activity system view.
- **ABI / extension API:** The patch changes libpq-be.h and libpq.h, and makes raddr a header-supplied address that extensions may read.
- **new GUC:** The versions add proxy_servers, proxy_port or proxy_networks, and the final shape is not chosen.
- **security / privileges:** pg_hba.conf matching uses an address taken from a header sent by a trusted proxy; ident is disabled for proxied connections.
- **maintenance burden ("who maintains this in 10 years"):** New listener settings, new log and view fields, and a rule that later auth code must not treat raddr as the real peer.
- **performance on the hot path:** The patch changes connection startup in postmaster.c and pqcomm.c; no benchmark backs the no-regression claim.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Eisentraut commented on the listener design in message #52. The owner needs to know postmaster connection startup, pg_hba.conf matching and the authentication code.
- Priority: fix design first; audience a sizeable minority; review cost medium. Independent users asked for this from 2021 to 2026, and the feature is off by default. The listener configuration and the Unix-domain socket trust rule are open after message #52, so a v14 that settles them should come before detailed review of postmaster and auth changes.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
