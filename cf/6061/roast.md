# Design pre-review: MPTCP - multiplexing many TCP connections through one socket to get better bandwidth

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6061](https://commitfest.postgresql.org/patch/6061/); [thread](https://www.postgresql.org/message-id/flat/CAKZiRmy6j9PBzDHZwdgwHavwKDzv5GWtRSWOTj6-jv6SCOZ%3DYA%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The surviving level 1b objections, already-exists-1 and demand-1, decide this. Message #1 shows mptcpize giving the same multipath gain to unmodified binaries, and the thread names no setup that the wrapper cannot serve and no second user. The goal may be fine and the cost falls only on users who turn the feature on. So the result is "not now, show demand", resting on absence of evidence.

Not checked: Without the code, I could not check whether keepalive and tcp_user_timeout socket options behave the same on MPTCP sockets. I also could not check whether the test in 003_start_stop.pl covers more than socket creation.

## Prior attempts

None found before 2025-09-04.

## Objections a reviewer is likely to raise

### 1. mptcpize already gives MPTCP to unmodified Postgres and libpq clients

An operator who needs more than one flow can run `mptcpize run postgres ...` and `mptcpize run pg_basebackup ...` today and get the same subflows. Core then carries a GUC, a libpq option and an environment variable for a choice the kernel tooling already offers. That cost lasts for the life of the code.

Message #1 names mptcpize and dismisses it as "not elegant" because it needs edits to systemd startup scripts. That makes the existing way less convenient, but it does not show the job cannot be done. Message #1 also shows mptcpize running iperf3 with multipath gains, and the patch changes only the protocol argument to socket(), which is exactly what the wrapper overrides. A committer would likely ask what in-core placement adds beyond saving one wrapper line per service, and whether that gain is worth a permanent server and client setting.

**Question:** Which setup cannot be served by running the postmaster and clients under mptcpize, for example per-connection control in primary_conninfo or packaged clients you cannot wrap?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAKZiRmy6j9PBzDHZwdgwHavwKDzv5GWtRSWOTj6-jv6SCOZ%3DYA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAKZiRmyEhdJ0%3D7yj3KOq5m0kft-H6EayRSKBx0sG6Yex-PKF6w%40mail.gmail.com); `already-exists-1`</sub>

### 2. Demand rests on the author's own setup; mptcpize already covers it

An operator on a cloud VM sets PGMPTCP=1 and listen_mptcp=on and gets one subflow and no speedup. The kernel path manager needs `ip mptcp endpoint` and limits configured on both hosts, plus open firewall ports. Postgres would then own a GUC, a libpq option and an environment variable whose effect depends on host networking that it cannot see.

The demand evidence is the author's tc-limited demo, an iperf3 run and one production use from years ago (message #1, #2). One reviewer likes the idea but reports no testing (message #3, #6). The users who benefit already do root-level kernel MPTCP setup, so `mptcpize run` is a small extra step for them. Message #1 calls that wrapper inelegant, but it works today without any change to Postgres.

**Question:** Which users, besides the author, run or want MPTCP for pg_basebackup or replication, and what blocks them from using mptcpize today?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAKZiRmy6j9PBzDHZwdgwHavwKDzv5GWtRSWOTj6-jv6SCOZ%3DYA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAKZiRmyEhdJ0%3D7yj3KOq5m0kft-H6EayRSKBx0sG6Yex-PKF6w%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAKAnmmLnfQ%3D7RjPbuBMgYs-L7LW_CpaRPCsD%3DXXVMt5vqm3X-A%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAKAnmmLszGvRhr-ffk4q%2BW734SQhun%3D18euQ2Wgwdzf%3DVfP0zA%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which setup cannot be served by running the postmaster and clients under mptcpize, for example per-connection control in primary_conninfo or packaged clients that cannot be wrapped?
2. Which users, besides the author, run or want MPTCP for pg_basebackup or replication, and has anyone else tested or benchmarked the patch?
3. What happens to a connection when one side asks for MPTCP and the other side has it disabled or lacks kernel support?
4. How does a user find out, from the server or from psql, that a given connection runs over MPTCP and not plain TCP?
5. Why does mptcp=1 fail the connection where MPTCP is unavailable, when libpq ignores keepalive settings on platforms that do not support them?

## Touch points

- **WAL / replication / standby:** Streaming and logical replication connections are named targets, so walsender and walreceiver sockets would run over MPTCP.
- **new GUC:** The patch adds the boolean server GUC listen_mptcp, default off, with a check hook.
- **security / privileges:** Later MPTCP subflows can use addresses and ports that listen_addresses and pg_hba.conf do not check.
- **maintenance burden ("who maintains this in 10 years"):** The mptcp option, PGMPTCP and listen_mptcp become permanent names that depend on Linux kernel MPTCP behavior.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The mptcpize wrapper already gives MPTCP sockets to an unmodified postmaster and unmodified libpq clients.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the postmaster listen-socket code in pqcomm.c and the libpq connection code. No committer has posted in the visible thread.
- Not now, show demand first: Reports from independent users who need more than one flow for pg_basebackup or replication, plus a concrete setup where mptcpize does not work. An independent benchmark of the patch on a real cloud link would also help.
- Priority: needs champion; audience niche; review cost low. The patch is about 12 KiB, off by default and Linux-only, so cost falls on users who turn it on. It has been in 5 CommitFests with one reviewer and no committer, and the open point is demand beyond the author.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
