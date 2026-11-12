# Design pre-review: MPTCP - multiplexing many TCP connections through one socket to get better bandwidth

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6061](https://commitfest.postgresql.org/patch/6061/) (Needs review); authors: Jakub Wartak (jakub.wartak.tomtom)
- Thread: [MPTCP - multiplexing many TCP connections through one socket to get better bandwidth](https://www.postgresql.org/message-id/flat/CAKZiRmy6j9PBzDHZwdgwHavwKDzv5GWtRSWOTj6-jv6SCOZ%3DYA%40mail.gmail.com), started 2025-09-04
- Messages read: 7 of 7
- Patch prose read: v5-0001-Add-MPTCP-protocol-support-to-server-and-libpq-on.patch, v1-0001-Add-MPTCP-protocol-support-to-server-and-libpq-on.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The surviving level 1b objections, already-exists-1 and demand-1, decide this. Message #1 shows mptcpize giving the same multipath gain to unmodified binaries, and the thread names no setup that the wrapper cannot serve and no second user. The goal may be fine and the cost falls only on users who turn the feature on. So the result is "not now, show demand", resting on absence of evidence.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check whether keepalive and tcp_user_timeout socket options behave the same on MPTCP sockets. I also could not check whether the test in 003_start_stop.pl covers more than socket creation.

## The proposal, restated

**Problem.** A single TCP connection is often capped below the available bandwidth, by per-flow limits in clouds (about 5 Gbps on many AWS and Azure VMs) or by WAN chokepoints. Every libpq connection uses one socket, so pg_basebackup, streaming and logical replication, COPY and postgres_fdw cannot use more than one flow. The workaround today is the mptcpize LD_PRELOAD wrapper, which the author calls inelegant.

**Proposed design.** The postmaster opens its TCP listen sockets with IPPROTO_MPTCP when listen_mptcp is on, and libpq creates MPTCP client sockets when the mptcp option is set. The Linux kernel then spreads one application connection over several TCP subflows, and Postgres needs no other change. The support is Linux-only behind #ifdef IPPROTO_MPTCP, a GUC check hook rejects the setting where it is unsupported, and Unix-domain sockets ignore it.

**User-visible surface.**
- Server GUC listen_mptcp (boolean, default off), placed in postgresql.conf.sample near client_connection_check_interval.
- libpq connection option mptcp, default 0; only the value 1 turns it on in v5.
- libpq environment variable PGMPTCP, same as the mptcp option.
- libpq error message when MPTCP is requested on a TCP connection but not available.

**What the author claims.**
- About 3 times faster \copy and pg_basebackup in a tc-limited test with 4 server MPTCP endpoints.
- On AWS, iperf3 reached about 41 Gbps over one MPTCP stream, against 15-23 Gbps over one TCP stream.
- Speeds up any libpq-based use with no other rework, and works with SSL/TLS (tested).
- MPTCP is an IETF standard (RFC 8684) and is enabled by default in modern Linux distributions.
- Built-in support is simpler than solving the bandwidth problem separately for each tool.

**Thread state.** Greg Sabino Mullane, the only other participant, likes the idea and reviewed v4 for style, freeing, error messages, test lists and tests. v5 addresses his points, but the value syntax and a more generic listen_protocols / PGSOCKETPROTOCOL naming are still open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-09-04; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. mptcpize already gives MPTCP to unmodified Postgres and libpq clients

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), new GUC, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An operator who needs more than one flow can run `mptcpize run postgres ...` and `mptcpize run pg_basebackup ...` today and get the same subflows. Core then carries a GUC, a libpq option and an environment variable for a choice the kernel tooling already offers. That cost lasts for the life of the code.

Message #1 names mptcpize and dismisses it as "not elegant" because it needs edits to systemd startup scripts. That makes the existing way less convenient, but it does not show the job cannot be done. Message #1 also shows mptcpize running iperf3 with multipath gains, and the patch changes only the protocol argument to socket(), which is exactly what the wrapper overrides. A committer would likely ask what in-core placement adds beyond saving one wrapper line per service, and whether that gain is worth a permanent server and client setting.

Thread: [#1 (Jakub Wartak, 2025-09-04)](https://www.postgresql.org/message-id/CAKZiRmy6j9PBzDHZwdgwHavwKDzv5GWtRSWOTj6-jv6SCOZ%3DYA%40mail.gmail.com), [#2 (Jakub Wartak, 2026-08-13)](https://www.postgresql.org/message-id/CAKZiRmyEhdJ0%3D7yj3KOq5m0kft-H6EayRSKBx0sG6Yex-PKF6w%40mail.gmail.com)

**Question:** Which setup cannot be served by running the postmaster and clients under mptcpize, for example per-connection control in primary_conninfo or packaged clients you cannot wrap?

Counter-check: held. Message #1 itself shows mptcpize giving multipath gains, and its only stated drawback is editing systemd startup scripts. The thread names no setup that the wrapper cannot serve, so the inconvenience alone does not answer the objection.

### 2. Demand rests on the author's own setup; mptcpize already covers it

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

An operator on a cloud VM sets PGMPTCP=1 and listen_mptcp=on and gets one subflow and no speedup. The kernel path manager needs `ip mptcp endpoint` and limits configured on both hosts, plus open firewall ports. Postgres would then own a GUC, a libpq option and an environment variable whose effect depends on host networking that it cannot see.

The demand evidence is the author's tc-limited demo, an iperf3 run and one production use from years ago (message #1, #2). One reviewer likes the idea but reports no testing (message #3, #6). The users who benefit already do root-level kernel MPTCP setup, so `mptcpize run` is a small extra step for them. Message #1 calls that wrapper inelegant, but it works today without any change to Postgres.

Thread: [#1 (Jakub Wartak, 2025-09-04)](https://www.postgresql.org/message-id/CAKZiRmy6j9PBzDHZwdgwHavwKDzv5GWtRSWOTj6-jv6SCOZ%3DYA%40mail.gmail.com), [#2 (Jakub Wartak, 2026-08-13)](https://www.postgresql.org/message-id/CAKZiRmyEhdJ0%3D7yj3KOq5m0kft-H6EayRSKBx0sG6Yex-PKF6w%40mail.gmail.com), [#3 (Greg Sabino Mullane, 2026-08-13)](https://www.postgresql.org/message-id/CAKAnmmLnfQ%3D7RjPbuBMgYs-L7LW_CpaRPCsD%3DXXVMt5vqm3X-A%40mail.gmail.com), [#6 (Greg Sabino Mullane, 2026-08-18)](https://www.postgresql.org/message-id/CAKAnmmLszGvRhr-ffk4q%2BW734SQhun%3D18euQ2Wgwdzf%3DVfP0zA%40mail.gmail.com)

**Question:** Which users, besides the author, run or want MPTCP for pg_basebackup or replication, and what blocks them from using mptcpize today?

Counter-check: held. The cloud per-flow table in message #2 shows the limit is common, but the thread has no second user who wants MPTCP or has tested the patch. The mptcpize part repeats already-exists-1 and its no-speedup scenario does not bear on demand.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | Streaming and logical replication connections are named targets, so walsender and walreceiver sockets would run over MPTCP. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds the boolean server GUC listen_mptcp, default off, with a check hook. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Later MPTCP subflows can use addresses and ports that listen_addresses and pg_hba.conf do not check. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The mptcp option, PGMPTCP and listen_mptcp become permanent names that depend on Linux kernel MPTCP behavior. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The mptcpize wrapper already gives MPTCP sockets to an unmodified postmaster and unmodified libpq clients. |

## 5. Questions the author must answer before anyone reviews code

1. Which setup cannot be served by running the postmaster and clients under mptcpize, for example per-connection control in primary_conninfo or packaged clients that cannot be wrapped?
2. Which users, besides the author, run or want MPTCP for pg_basebackup or replication, and has anyone else tested or benchmarked the patch?
3. What happens to a connection when one side asks for MPTCP and the other side has it disabled or lacks kernel support?
4. How does a user find out, from the server or from psql, that a given connection runs over MPTCP and not plain TCP?
5. Why does mptcp=1 fail the connection where MPTCP is unavailable, when libpq ignores keepalive settings on platforms that do not support them?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the postmaster listen-socket code in pqcomm.c and the libpq connection code. No committer has posted in the visible thread.
- **Liability:** The owner signs for listen_mptcp, the mptcp option and PGMPTCP for as long as libpq exists. The owner also answers reports that depend on Linux kernel path-manager behavior, which the buildfarm cannot reproduce.
- **Author follow-up:** Good sign: the author revived the thread after a year, posted versions up to v5 that address review, and moved the entry through five CommitFests.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: Reports from independent users who need more than one flow for pg_basebackup or replication, plus a concrete setup where mptcpize does not work. An independent benchmark of the patch on a real cloud link would also help.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 12 KiB, off by default and Linux-only, so cost falls on users who turn it on. It has been in 5 CommitFests with one reviewer and no committer, and the open point is demand beyond the author.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Boolean mptcp option with hard error breaks the libpq mode convention. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) No way for users to see whether a connection uses MPTCP. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-1` (fixable, level 2) Network controls apply only to the first subflow of an MPTCP connection. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-1` Transport-specific GUC and libpq option do not scale to later transports. Refuted (no failure scenario): The scenario needs a hypothetical later transport, and one option per feature matches existing libpq practice (`sslmode`, `gssencmode`, `keepalives`). Message #6 weighed the generic name and dropped it because it cannot set a protocol per address family.
- `maintenance-fit-2` Behavior depends on kernel path-manager setup the project cannot test. Refuted (no failure scenario): A no-speedup report caused by missing `ip mptcp` endpoints or a firewall is a host setup problem, like routing, and the patch changes only the protocol argument to socket(). The visibility part duplicates standards-compat-2, and committer ownership belongs to committability.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P6] and [P7] are libpq compression patches, a different idea, and both were returned because the author did not follow up, a reason that does not apply here. The thread cites no earlier MPTCP discussion. Message #2 links a separate pg_basebackup patch only for benchmark numbers.
- standards-compat: 2 objection(s). Greg's generic listen_protocols naming point was withdrawn by him in message #6, so it is not raised. The check hook that rejects listen_mptcp on unsupported platforms follows existing practice (check_bonjour), so it is not raised either.
- replication-ondisk: 0 objection(s). The proposal adds no on-disk, WAL, catalog or protocol-message state; MPTCP changes only the socket protocol, and the kernel falls back to plain TCP. Not checked without code: whether keepalive and tcp_user_timeout socket options behave the same on MPTCP sockets on walsender and walreceiver connections.
- security: 1 objection(s). The defaults are safe: listen_mptcp is off and can be set only by an admin. The client option only changes the client's own socket, so I see no path for privilege escalation or denial of service beyond the kernel's subflow limits.
- maintenance-fit: 2 objection(s). The cost falls only on users who turn it on, since both settings default to off. I found no hot-path objection.
- already-exists: 1 objection(s). One objection only; the listed precedents concern libpq compression and do not apply to this angle.
- demand: 1 objection(s). The cost falls mostly on users of the feature: it is off by default and Linux-only, so the only demand objection is not now, show demand.

### Precedents retrieved

- P1: [Revert to our pre-7.4 behavior of identifying Unix-socket connections in ps status as '\[local\]', not as 'localhost' as the code has been doing recently.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b43e325ebfab423c937ce384bcdaf70ed0b462e) (revert, Reverted, 2003-09-12) - judged not relevant
- P2: [Revert "Cannot use WL_SOCKET_WRITEABLE without WL_SOCKET_READABLE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a58db3aa10e62e4228aa409ba006014fa07a8ca2) (revert, Reverted, 2020-12-15) - judged not relevant
- P3: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P4: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P5: [socket_timeout in interfaces/libpq](https://commitfest.postgresql.org/patch/2175/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P6: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - used in the review; reason summarised from its thread
- P7: [libpq compression](https://commitfest.postgresql.org/patch/3499/) (cf, Returned with feedback, 2022-08-02) - used in the review; reason summarised from its thread
- P8: [Add new protocol message to change GUCs to be able to change protocol extension parameters](https://commitfest.postgresql.org/patch/4736/) (cf, Withdrawn, 2025-02-22) - judged not relevant; reason summarised from its thread
- P9: [Use "protocol options" name instead of "protocol extensions" everywhere](https://commitfest.postgresql.org/patch/5341/) (cf, Rejected, 2025-03-11) - judged not relevant; reason summarised from its thread
- P10: [Revert "Secure Unix-domain sockets of "make check" temporary clusters."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8f5578d0f9681ef81bc71a3762a191d66a29c8b1) (revert, Reverted, 2014-03-29) - judged not relevant
- P11: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant; reason summarised from its thread
- P12: [Move libpq's write_failed mechanism down to pqsecure_raw_write().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa189c932d51945b2285e277128b0f26b96afdd) (revert, Partially reverted, 2022-02-12) - judged not relevant
- P13: [Optimize LISTEN/NOTIFY](https://commitfest.postgresql.org/patch/5912/) (cf, Withdrawn, 2025-08-07) - judged not relevant; reason summarised from its thread
- P14: [Prevent pg_basebackup running as root](https://commitfest.postgresql.org/patch/2431/) (cf, Rejected, 2020-02-27) - judged not relevant
- P15: [macOS Secure Transport SSL Support](https://commitfest.postgresql.org/patch/1490/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P16: [Pattern based listeners for asynchronous messaging (LISTEN/NOTIFY)](https://commitfest.postgresql.org/patch/1218/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P17: [Allow LISTEN on patterns](https://commitfest.postgresql.org/patch/5630/) (cf, Withdrawn, 2025-03-07) - judged not relevant
- P18: [Application name for pg_basebackup, pg_receivewal and pg_recvlogical](https://commitfest.postgresql.org/patch/2335/) (cf, Rejected, 2019-11-07) - judged not relevant
- P19: [pgbench: add multiconnect support](https://commitfest.postgresql.org/patch/3227/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P20: [pg_basebackup: add test about zstd compress option](https://commitfest.postgresql.org/patch/3835/) (cf, Rejected, 2022-12-06) - judged not relevant
- P21: [Allow specifying multiple hosts in libpq connect options](https://commitfest.postgresql.org/patch/222/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P22: [Hooks and API to allow loadable wire protocol extensions](https://commitfest.postgresql.org/patch/3018/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P23: [Support wildcards in LISTEN command](https://commitfest.postgresql.org/patch/4896/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P24: [libpq: Support for Apple Secure Transport SSL library](https://commitfest.postgresql.org/patch/1228/) (cf, Returned with feedback, 2017-11-28) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 176 s; cost reported by the CLI: $2.75
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 85 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:41:23.771Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
