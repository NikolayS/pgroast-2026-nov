# Design pre-review: Support for kqueue in pgbench

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7525](https://commitfest.postgresql.org/patch/7525/) (Needs review); authors: Roman Khapov (rkhapov)
- Thread: [kqueue backend for pgbench](https://www.postgresql.org/message-id/flat/5EF180BD-3951-4E8C-903D-D91EF54A338D%40yandex-team.ru), started 2026-10-26
- Messages read: 8 of 8
- Patch prose read: v2-0001-pgbench-add-kqueue-implementation-of-the-socket-s.patch, 0001-pgbench-add-kqueue-implementation-of-the-socket-set-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is reasonable, and the cost stays inside pgbench on platforms without ppoll(). The thread has not settled whether a larger select() fd set would lift the limit with less code (maintenance-fit-2); message #7 answers that alternative without a prototype or a size. The kqueue path also replaces select() at every client count, with a -4% result on one laptop (prior-attempts-1), and no test guards its kernel state (maintenance-fit-1). Demand is thin but did not stop the review (demand-1).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the -C fd reuse handling, the v2 rule for skipping the disable step, and the kqueue build detection could not be checked. The numbers in message #8 come from one laptop and are taken from the reviewers' reports.

## The proposal, restated

**Problem.** pgbench falls back to select() on platforms without ppoll(), such as macOS, so one pgbench process cannot wait on more than 1024 sockets (FD_SETSIZE). The author needs many more clients than threads to test connection overhead and pool management in connection poolers.

**Proposed design.** The patch adds a third socket_set implementation in pgbench.c based on kqueue(), used where kqueue(2) exists and ppoll(2) does not. Because kqueue registrations persist in the kernel, wait_on_socket_set() disables stale registrations, enables the current set and maps events back to input slots through udata. v2 skips the disable step when the fd set has not changed since the previous wait.

**User-visible surface.**
- pgbench on macOS and similar platforms accepts more than about 1024 clients per thread; no new option or syntax.

**What the author claims.**
- Removes the FD_SETSIZE limit of 1024 clients for pgbench on macOS.
- v2 matches select() throughput wherever select() can run, within about -4% to +4% tps on the author's laptop.
- v1 lost about 5-8% tps against select() at small client counts because of an extra kevent() call per wait.
- Worst-case cost of stale-fd cleanup is O(nregistered * nclosed), mainly when all clients close at once.
- The design follows the existing kqueue use in the backend wait event code (waiteventset.c).

**Thread state.** Kirill Reshke supports it and tested it on macOS; Yura Sokolov says pgbench has wider O(n) per-wakeup costs. Jeff Anton argues that redefining FD_SETSIZE or a custom bit array is simpler, and the author replied with objections that nobody has answered yet.

## 2. Prior attempts

### pgbench - option to build using ppoll() for larger connection counts (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1307/](https://commitfest.postgresql.org/patch/1307/)
- **Thread:** [https://www.postgresql.org/message-id/flat/F6C5AD9B-0262-4C35-BE59-06C3EE6E7F87@amazon.com](https://www.postgresql.org/message-id/flat/F6C5AD9B-0262-4C35-BE59-06C3EE6E7F87@amazon.com)
- **Date:** decided 2017-11-28; started 2017-09-25
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P16 used ppoll() to lift the pgbench connection limit, and pgbench now uses ppoll() where it exists. This patch adds kqueue only for platforms that lack ppoll().

### kqueue (related)

- **Link:** [https://commitfest.postgresql.org/patch/1808/](https://commitfest.postgresql.org/patch/1808/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAEepm=37oF84-iXDTQ9MrGjENwVGds+5zTr38ca73kWR7ez_tA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAEepm=37oF84-iXDTQ9MrGjENwVGds+5zTr38ca73kWR7ez_tA@mail.gmail.com)
- **Date:** decided 2018-11-11; started 2016-03-29
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Matteo Beccati (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Benchmarks gave mixed results: kqueue helped on macOS and on large bare-metal FreeBSD servers with remote clients, but it was slower on NetBSD and on FreeBSD virtual machines when pgbench ran on the same machine. Neither the author nor the testers found the cause. The author called the slowdown for local clients a show stopper and was still asking kernel developers about it when the entry was returned.
- **Reason (quote):** "Degraded performance for clients on the same machine does seem to be a show stopper for this patch for now." ([message](https://www.postgresql.org/message-id/CAEepm%3D1YhBEH9FV_76k5GqzZcK4G%2BPF7_EqGc4eiMKswFtOYRg%40mail.gmail.com))
- **What is different this time:** P5 put kqueue in the server's WaitEventSet; this patch puts kqueue in pgbench only. The performance finding in P5 concerns the server and does not carry over to this patch.


## 3. Objections a reviewer is likely to raise

### 1. Lifting the select() limit may not need a third socket set

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-2`

The project adds and keeps a third wait implementation in pgbench for one platform. A heap-allocated fd_set, or _DARWIN_UNLIMITED_SELECT on macOS, could raise the same 1024 cap with a small change to the existing select() path. Every later socket_set change then has to be written and checked three times.

Message #6 proposes raising FD_SETSIZE or using a private bit array. Message #7 says that alternative would be fragile and about as large, but gives no size or prototype. The O(max_fd) cost it cites does not matter at the client counts pgbench reaches. A committer would likely want the smaller change ruled out with evidence before adding stateful, platform-specific code, and the stated need is one tester running connection-pooler tests on a laptop (messages #2 and #4).

Thread: [#2 (Kirill Reshke, 2026-10-28)](https://www.postgresql.org/message-id/CALdSSPg-maGunyKSaqOv-iUEhSfhMS-Kk3i%3DZKZ7dJAPnL4pZQ%40mail.gmail.com), [#4 (Roman Khapov, 2026-10-28)](https://www.postgresql.org/message-id/12B59C9C-2F60-4042-A5E1-CFE9E907B73D%40yandex-team.ru), [#6 (Jeff Anton, 2026-10-28)](https://www.postgresql.org/message-id/15e57b71-5c59-4cdd-bbde-cdc8f5a8de0b%40hesiod.org), [#7 (Roman Khapov, 2026-10-29)](https://www.postgresql.org/message-id/D6889C2B-0A6C-4732-A7D0-13C3FC214C3E%40yandex-team.ru)

**Question:** How large is a select()-based fix using a dynamically sized fd_set or _DARWIN_UNLIMITED_SELECT, and what does it fail to do that the kqueue set does?

Counter-check: held. Message #7 answers the select() alternative with questions and an O(max_fd) cost it calls not critical, without a prototype or size. The alternative can do the job on macOS, so the comparison is still open.

### 2. Default macOS pgbench runs may get slower to serve the over-1024 case

fixable · level 2, rubric item 8 · performance on the hot path · `prior-attempts-1`

A developer benchmarks a patch on macOS with pgbench -c 32 -j 1 against a local server. After this change pgbench uses kqueue for every run, so the client measures up to about 4% fewer tps. The developer may count the drop against the server patch, since nothing in the output shows the wait method changed.

In [P5], kqueue was returned partly because it was slower when pgbench ran on the same machine as the server. Message #8 shows v2 at -4% for -c 32 -j 1, from medians of 3-4 runs on one macOS laptop. The gain in this patch is functional and only matters above 1024 clients, but per message #1 the kqueue path replaces select() for all client counts. pgbench is itself a measuring tool, so any change in its own overhead shifts results for every macOS user.

**Precedents.**
- [kqueue](https://commitfest.postgresql.org/patch/1808/) (Returned with feedback, 2018-11-11); thread: [link](https://www.postgresql.org/message-id/flat/CAEepm=37oF84-iXDTQ9MrGjENwVGds+5zTr38ca73kWR7ez_tA@mail.gmail.com)

Thread: [#1 (Roman Khapov, 2026-10-26)](https://www.postgresql.org/message-id/5EF180BD-3951-4E8C-903D-D91EF54A338D%40yandex-team.ru), [#8 (Roman Khapov, 2026-10-30)](https://www.postgresql.org/message-id/59A71A20-634A-4A63-97B9-D81DAFF4E36E%40yandex-team.ru)

**Question:** Does kqueue stay within noise of select() on a second macOS machine and at low client counts, or should select() stay the default below FD_SETSIZE?

Counter-check: held. Message #8 shows -4% at -c 32 -j 1 from one laptop, and the kqueue path replaces select() for all client counts on macOS. P5 concerns the server's WaitEventSet and must not be used as support.

### 3. Stateful kqueue path keeps a pgbench invariant that only macOS exercises

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch changes how threadRun adds or drops client sockets, perhaps the O(n) rework suggested in messages #3 and #5. It passes on Linux with ppoll(). On macOS a stale or reused-fd knote then misses events or reports them to the wrong slot, and pgbench hangs or miscounts under -C. Only a macOS user running many clients finds it.

select() and ppoll() rebuild the set on every wait, so the rest of pgbench can ignore their state. The kqueue set keeps kernel registrations across calls, and v2 also skips the disable step when the fd set looks unchanged. Its correctness for -C rests on the kernel reusing fd numbers (message #5). Nothing fails on the main development platforms if a future change breaks that, and the thread shows testing only on two laptops, with a client count above 512 never measured.

Thread: [#3 (Yura Sokolov, 2026-10-28)](https://www.postgresql.org/message-id/1d251dc1-140c-468b-9a45-3918dd3e77ef%40postgrespro.ru), [#5 (Roman Khapov, 2026-10-28)](https://www.postgresql.org/message-id/0821A195-F22A-43CD-8B46-3A7C1F04D72B%40yandex-team.ru), [#8 (Roman Khapov, 2026-10-30)](https://www.postgresql.org/message-id/59A71A20-634A-4A63-97B9-D81DAFF4E36E%40yandex-team.ru)

**Question:** Which test, run on the buildfarm's macOS animals, would fail if a later pgbench change broke the kqueue set's view of added, closed or reused sockets?

Counter-check: held. The kqueue set keeps kernel registrations across waits, and message #5 rests the -C case on the kernel reusing fd numbers. The thread names no test that would fail on macOS if a later change broke this.

### 4. Demand for more than 1024 pgbench clients on macOS is not shown

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

The project adds a third socket_set path to pgbench that only macOS and BSD builds use. Few people run more than 1024 clients from a Mac, so bugs in the stateful kqueue path, such as fd reuse under -C, can go unnoticed for several releases. The pgbench maintainers then debug code that few developers can run or reproduce.

The only stated user is the author, who wants to stress-test connection poolers from a laptop (messages #1, #4). The only supporter calls the case "uncommon" (message #2). The other large-client user in the thread runs with -j equal to -c, which points to Linux hosts where ppoll() already removes the limit (message #3). Two workarounds exist: run pgbench on Linux, or start several pgbench processes. The proposal does not say why these fail for pooler testing.

**Precedents.**
- [kqueue support for WaitEventSet](https://commitfest.postgresql.org/patch/597/) (Returned with feedback, 2016-09-13); thread: [link](https://www.postgresql.org/message-id/flat/CAEepm=37oF84-iXDTQ9MrGjENwVGds+5zTr38ca73kWR7ez_tA@mail.gmail.com)

Thread: [#1 (Roman Khapov, 2026-10-26)](https://www.postgresql.org/message-id/5EF180BD-3951-4E8C-903D-D91EF54A338D%40yandex-team.ru), [#2 (Kirill Reshke, 2026-10-28)](https://www.postgresql.org/message-id/CALdSSPg-maGunyKSaqOv-iUEhSfhMS-Kk3i%3DZKZ7dJAPnL4pZQ%40mail.gmail.com), [#3 (Yura Sokolov, 2026-10-28)](https://www.postgresql.org/message-id/1d251dc1-140c-468b-9a45-3918dd3e77ef%40postgrespro.ru), [#4 (Roman Khapov, 2026-10-28)](https://www.postgresql.org/message-id/12B59C9C-2F60-4042-A5E1-CFE9E907B73D%40yandex-team.ru)

**Question:** Who besides the author drives more than 1024 clients from one pgbench process on macOS or BSD, and why do Linux hosts or several pgbench processes not work for them?

Counter-check: held. The thread shows one user with this need on macOS, and message #2 calls the case uncommon. P6 asked for a speed gain in the server and must not be used as support; FreeBSD has ppoll(), so the path is mostly macOS.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a third, stateful socket_set implementation in pgbench.c that only platforms without ppoll() use. |
| performance on the hot path | **yes** | The kqueue path replaces select() in the pgbench wait loop at every client count on macOS. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A larger select() fd set or several pgbench processes may already cover the stated need. |

## 5. Questions the author must answer before anyone reviews code

1. How large is a select()-based fix using a dynamically sized fd_set or _DARWIN_UNLIMITED_SELECT, and what does it fail to do that the kqueue set does?
2. Does kqueue stay within noise of select() on a second macOS machine at low client counts, or should select() stay the default below FD_SETSIZE?
3. Which test on the buildfarm's macOS animals would fail if a later pgbench change broke the kqueue set's view of added, closed or reused sockets?
4. Who besides the author drives more than 1024 clients from one pgbench process on macOS, and why do several pgbench processes not work for them?
5. How does the build detect kqueue when the patch touches only pgbench.c, and which platforms besides macOS take the kqueue path?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the pgbench threadRun loop and kqueue behaviour on macOS; nobody in the visible thread is identified as one.
- **Liability:** The owner signs for a stateful wait path that only macOS-like platforms exercise, including fd reuse under -C, for as long as pgbench supports platforms without ppoll().
- **Author follow-up:** The author answered the select() alternative in message #7 and posted v2 with measurements in message #8, both within two weeks of the first post.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is one 8.8 KB change to pgbench.c for users who run more than 1024 clients per pgbench process on macOS. The choice between kqueue and a larger select() fd set should be settled before code review.

## Appendix

### Refuted in the counter-check

- `already-exists-1` Several pgbench processes already exceed 1024 connections on macOS. Refuted (not specific): Duplicate of demand-1, which already asks why several pgbench processes do not work. P3 was closed because its author stopped responding, so it gives little support.
- `already-exists-2` A larger select() fd set may be smaller than a kqueue path. Refuted (not specific): Duplicate of maintenance-fit-2, which asks the same question about a select() variant with a larger fd set.

### Review angles

- prior-attempts: 1 objection(s). P6 was returned for lack of a measured gain, which does not apply because this patch's gain is functional. P3's fd_set-internals concern applies to the FD_SETSIZE alternative in message #6, not to this patch.
- standards-compat: 0 objection(s). From the standards and compatibility angle the proposal adds no syntax, option, GUC, catalog object or dump format, and existing pgbench scripts and option sets run unchanged on macOS. The only user-visible change is a higher client limit on that platform. I could not check from the material whether the pgbench documentation or the existing 'too many client connections for select()' error text names the limit and would need an update.
- replication-ondisk: 0 objection(s). The patch changes only how pgbench waits on its own client sockets inside one process. It adds no on-disk format, WAL record, catalog entry, wire protocol message or backend ABI change. There is no persistent or cross-node state for this angle to judge.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. pgbench is a client tool that runs as the invoking OS user and opens only its own connections. A kqueue socket set changes how that process waits on its own sockets, and grants no server-side privilege, file access or view of other sessions. A larger client count per thread is bounded by the user's own rlimits and by max_connections on the server, as with ppoll() on Linux today.
- maintenance-fit: 2 objection(s). Two fixable design and maintenance concerns. Costs stay inside pgbench on platforms without ppoll(), so nothing here is fatal.
- already-exists: 2 objection(s). Both objections are about desirability and simpler alternatives. The design mirrors kqueue use in waiteventset.c, which makes a fatal objection unlikely.
- demand: 1 objection(s). The cost is limited to pgbench on platforms without ppoll(), so this is a "not now, show demand" point rather than a rejection.

### Precedents retrieved

- P1: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant; reason summarised from its thread
- P2: [pgbench allow '=' in \set](https://commitfest.postgresql.org/patch/246/) (cf, Rejected, 2015-05-13) - judged not relevant; reason summarised from its thread
- P3: [pgbench stopped supporting large number of client connections on Windows](https://commitfest.postgresql.org/patch/2827/) (cf, Returned with feedback, 2021-04-08) - used in the review; reason summarised from its thread
- P4: [Revert "Cannot use WL_SOCKET_WRITEABLE without WL_SOCKET_READABLE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a58db3aa10e62e4228aa409ba006014fa07a8ca2) (revert, Reverted, 2020-12-15) - judged not relevant
- P5: [kqueue](https://commitfest.postgresql.org/patch/1808/) (cf, Returned with feedback, 2018-11-11) - used in the review; reason summarised from its thread
- P6: [kqueue support for WaitEventSet](https://commitfest.postgresql.org/patch/597/) (cf, Returned with feedback, 2016-09-13) - used in the review; reason summarised from its thread
- P7: [automatic client-side column-level encryption](https://commitfest.postgresql.org/patch/3718/) (cf, Returned with feedback, 2024-06-27) - judged not relevant; reason summarised from its thread
- P8: [Batched clock sweep to reduce cross-socket atomic contention](https://commitfest.postgresql.org/patch/7000/) (cf, Withdrawn, 2026-09-11) - judged not relevant; reason summarised from its thread
- P9: [socket_timeout in interfaces/libpq](https://commitfest.postgresql.org/patch/2175/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P10: [pgbench using COPY FREEZE](https://commitfest.postgresql.org/patch/3201/) (cf, Withdrawn, 2021-07-01) - judged not relevant
- P11: [merging pgbench logs](https://commitfest.postgresql.org/patch/193/) (cf, Returned with feedback, 2015-10-03) - judged not relevant
- P12: [pgbench log file headers](https://commitfest.postgresql.org/patch/4660/) (cf, Returned with feedback, 2024-04-07) - judged not relevant
- P13: [pgbench - more operators and functions](https://commitfest.postgresql.org/patch/596/) (cf, Returned with feedback, 2017-01-25) - judged not relevant
- P14: [Revert "Secure Unix-domain sockets of "make check" temporary clusters."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8f5578d0f9681ef81bc71a3762a191d66a29c8b1) (revert, Reverted, 2014-03-29) - judged not relevant
- P15: [Revert to our pre-7.4 behavior of identifying Unix-socket connections in ps status as '\[local\]', not as 'localhost' as the code has been doing recently.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b43e325ebfab423c937ce384bcdaf70ed0b462e) (revert, Reverted, 2003-09-12) - judged not relevant
- P16: [pgbench - option to build using ppoll() for larger connection counts](https://commitfest.postgresql.org/patch/1307/) (cf, Returned with feedback, 2017-11-28) - used in the review
- P17: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P18: [Revert misguided change to postgres_fdw FOR UPDATE/SHARE code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8ec8760fc87ecde0516e511f1c55aec627b01ea7) (revert, Reverted, 2014-12-12) - judged not relevant
- P19: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P20: [Parallel INSERT (INTO ... SELECT ...)](https://commitfest.postgresql.org/patch/2844/) (cf, Returned with feedback, 2021-04-01) - judged not relevant
- P21: [WIP: macOS SIP workaround](https://commitfest.postgresql.org/patch/2862/) (cf, Returned with feedback, 2021-03-08) - judged not relevant
- P22: [Revert "Allow --with-bonjour to work with non-macOS implementations of Bonjour."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d9adab60754ac71b0b500c91c45e12e940b3ce) (revert, Reverted, 2017-11-09) - judged not relevant
- P23: [Respect client-initiated CopyDone during logical streaming replication](https://commitfest.postgresql.org/patch/621/) (cf, Withdrawn, 2019-07-08) - judged not relevant
- P24: [INSERT ... ON CONFLICT DO SELECT \[FOR ...\]](https://commitfest.postgresql.org/patch/5413/) (cf, Returned with feedback, 2025-08-05) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 140 s; cost reported by the CLI: $1.51
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 77 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:04:20.294Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
