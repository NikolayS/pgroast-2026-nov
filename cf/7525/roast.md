# Design pre-review: Support for kqueue in pgbench

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7525](https://commitfest.postgresql.org/patch/7525/); [thread](https://www.postgresql.org/message-id/flat/5EF180BD-3951-4E8C-903D-D91EF54A338D%40yandex-team.ru), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is reasonable, and the cost stays inside pgbench on platforms without ppoll(). The thread has not settled whether a larger select() fd set would lift the limit with less code (maintenance-fit-2); message #7 answers that alternative without a prototype or a size. The kqueue path also replaces select() at every client count, with a -4% result on one laptop (prior-attempts-1), and no test guards its kernel state (maintenance-fit-1). Demand is thin but did not stop the review (demand-1).

Not checked: Without the code, the -C fd reuse handling, the v2 rule for skipping the disable step, and the kqueue build detection could not be checked. The numbers in message #8 come from one laptop and are taken from the reviewers' reports.

## Prior attempts

- [pgbench - option to build using ppoll() for larger connection counts](https://www.postgresql.org/message-id/flat/F6C5AD9B-0262-4C35-BE59-06C3EE6E7F87@amazon.com) (similar idea; Returned with feedback, 2017-11-28). reason not recorded Different now: P16 used ppoll() to lift the pgbench connection limit, and pgbench now uses ppoll() where it exists. This patch adds kqueue only for platforms that lack ppoll().
- [kqueue](https://www.postgresql.org/message-id/flat/CAEepm=37oF84-iXDTQ9MrGjENwVGds+5zTr38ca73kWR7ez_tA@mail.gmail.com) (related; Returned with feedback, 2018-11-11). "Degraded performance for clients on the same machine does seem to be a show stopper for this patch for now." ([message](https://www.postgresql.org/message-id/CAEepm%3D1YhBEH9FV_76k5GqzZcK4G%2BPF7_EqGc4eiMKswFtOYRg%40mail.gmail.com)) Different now: P5 put kqueue in the server's WaitEventSet; this patch puts kqueue in pgbench only. The performance finding in P5 concerns the server and does not carry over to this patch.

## Objections a reviewer is likely to raise

### 1. Lifting the select() limit may not need a third socket set

The project adds and keeps a third wait implementation in pgbench for one platform. A heap-allocated fd_set, or _DARWIN_UNLIMITED_SELECT on macOS, could raise the same 1024 cap with a small change to the existing select() path. Every later socket_set change then has to be written and checked three times.

Message #6 proposes raising FD_SETSIZE or using a private bit array. Message #7 says that alternative would be fragile and about as large, but gives no size or prototype. The O(max_fd) cost it cites does not matter at the client counts pgbench reaches. A committer would likely want the smaller change ruled out with evidence before adding stateful, platform-specific code, and the stated need is one tester running connection-pooler tests on a laptop (messages #2 and #4).

**Question:** How large is a select()-based fix using a dynamically sized fd_set or _DARWIN_UNLIMITED_SELECT, and what does it fail to do that the kqueue set does?

<sub>fixable, level 2, design; messages [#2](https://www.postgresql.org/message-id/CALdSSPg-maGunyKSaqOv-iUEhSfhMS-Kk3i%3DZKZ7dJAPnL4pZQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/12B59C9C-2F60-4042-A5E1-CFE9E907B73D%40yandex-team.ru), [#6](https://www.postgresql.org/message-id/15e57b71-5c59-4cdd-bbde-cdc8f5a8de0b%40hesiod.org), [#7](https://www.postgresql.org/message-id/D6889C2B-0A6C-4732-A7D0-13C3FC214C3E%40yandex-team.ru); `maintenance-fit-2`</sub>

### 2. Default macOS pgbench runs may get slower to serve the over-1024 case

A developer benchmarks a patch on macOS with pgbench -c 32 -j 1 against a local server. After this change pgbench uses kqueue for every run, so the client measures up to about 4% fewer tps. The developer may count the drop against the server patch, since nothing in the output shows the wait method changed.

In [P5], kqueue was returned partly because it was slower when pgbench ran on the same machine as the server. Message #8 shows v2 at -4% for -c 32 -j 1, from medians of 3-4 runs on one macOS laptop. The gain in this patch is functional and only matters above 1024 clients, but per message #1 the kqueue path replaces select() for all client counts. pgbench is itself a measuring tool, so any change in its own overhead shifts results for every macOS user.

Precedent: [kqueue](https://www.postgresql.org/message-id/flat/CAEepm=37oF84-iXDTQ9MrGjENwVGds+5zTr38ca73kWR7ez_tA@mail.gmail.com) (Returned with feedback, 2018-11-11)

**Question:** Does kqueue stay within noise of select() on a second macOS machine and at low client counts, or should select() stay the default below FD_SETSIZE?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/5EF180BD-3951-4E8C-903D-D91EF54A338D%40yandex-team.ru), [#8](https://www.postgresql.org/message-id/59A71A20-634A-4A63-97B9-D81DAFF4E36E%40yandex-team.ru); `prior-attempts-1`</sub>

### 3. Stateful kqueue path keeps a pgbench invariant that only macOS exercises

A later patch changes how threadRun adds or drops client sockets, perhaps the O(n) rework suggested in messages #3 and #5. It passes on Linux with ppoll(). On macOS a stale or reused-fd knote then misses events or reports them to the wrong slot, and pgbench hangs or miscounts under -C. Only a macOS user running many clients finds it.

select() and ppoll() rebuild the set on every wait, so the rest of pgbench can ignore their state. The kqueue set keeps kernel registrations across calls, and v2 also skips the disable step when the fd set looks unchanged. Its correctness for -C rests on the kernel reusing fd numbers (message #5). Nothing fails on the main development platforms if a future change breaks that, and the thread shows testing only on two laptops, with a client count above 512 never measured.

**Question:** Which test, run on the buildfarm's macOS animals, would fail if a later pgbench change broke the kqueue set's view of added, closed or reused sockets?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/1d251dc1-140c-468b-9a45-3918dd3e77ef%40postgrespro.ru), [#5](https://www.postgresql.org/message-id/0821A195-F22A-43CD-8B46-3A7C1F04D72B%40yandex-team.ru), [#8](https://www.postgresql.org/message-id/59A71A20-634A-4A63-97B9-D81DAFF4E36E%40yandex-team.ru); `maintenance-fit-1`</sub>

### 4. Demand for more than 1024 pgbench clients on macOS is not shown

The project adds a third socket_set path to pgbench that only macOS and BSD builds use. Few people run more than 1024 clients from a Mac, so bugs in the stateful kqueue path, such as fd reuse under -C, can go unnoticed for several releases. The pgbench maintainers then debug code that few developers can run or reproduce.

The only stated user is the author, who wants to stress-test connection poolers from a laptop (messages #1, #4). The only supporter calls the case "uncommon" (message #2). The other large-client user in the thread runs with -j equal to -c, which points to Linux hosts where ppoll() already removes the limit (message #3). Two workarounds exist: run pgbench on Linux, or start several pgbench processes. The proposal does not say why these fail for pooler testing.

Precedent: [kqueue support for WaitEventSet](https://www.postgresql.org/message-id/flat/CAEepm=37oF84-iXDTQ9MrGjENwVGds+5zTr38ca73kWR7ez_tA@mail.gmail.com) (Returned with feedback, 2016-09-13)

**Question:** Who besides the author drives more than 1024 clients from one pgbench process on macOS or BSD, and why do Linux hosts or several pgbench processes not work for them?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/5EF180BD-3951-4E8C-903D-D91EF54A338D%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/CALdSSPg-maGunyKSaqOv-iUEhSfhMS-Kk3i%3DZKZ7dJAPnL4pZQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/1d251dc1-140c-468b-9a45-3918dd3e77ef%40postgrespro.ru), [#4](https://www.postgresql.org/message-id/12B59C9C-2F60-4042-A5E1-CFE9E907B73D%40yandex-team.ru); `demand-1`</sub>

## Questions for the author

1. How large is a select()-based fix using a dynamically sized fd_set or _DARWIN_UNLIMITED_SELECT, and what does it fail to do that the kqueue set does?
2. Does kqueue stay within noise of select() on a second macOS machine at low client counts, or should select() stay the default below FD_SETSIZE?
3. Which test on the buildfarm's macOS animals would fail if a later pgbench change broke the kqueue set's view of added, closed or reused sockets?
4. Who besides the author drives more than 1024 clients from one pgbench process on macOS, and why do several pgbench processes not work for them?
5. How does the build detect kqueue when the patch touches only pgbench.c, and which platforms besides macOS take the kqueue path?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch adds a third, stateful socket_set implementation in pgbench.c that only platforms without ppoll() use.
- **performance on the hot path:** The kqueue path replaces select() in the pgbench wait loop at every client count on macOS.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A larger select() fd set or several pgbench processes may already cover the stated need.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the pgbench threadRun loop and kqueue behaviour on macOS; nobody in the visible thread is identified as one.
- Priority: fix design first; audience niche; review cost low. The patch is one 8.8 KB change to pgbench.c for users who run more than 1024 clients per pgbench process on macOS. The choice between kqueue and a larger select() fd set should be settled before code review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
