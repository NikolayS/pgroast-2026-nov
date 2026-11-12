# Design pre-review: Add storage I/O tracking to EXPLAIN IO option

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5526](https://commitfest.postgresql.org/patch/5526/); [thread](https://www.postgresql.org/message-id/flat/c20f6340eb26f3b736abc59471bfada8%40oss.nttdata.com), 50 of 50 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread, but v13 shows no Storage I/O output under io_method = worker, the default (maintenance-fit-1). Andres said in message #26 that the approach then fails where most users run, and nothing after v12 shows that he or Tom accepts hiding the output (prior-attempts-1). The thread also does not weigh an extension plus a small parallel-worker hook (already-exists-2). The level 2 objections were not evaluated.

Not checked: Without the code, nobody can check the getrusage() overhead in parallel workers or whether the io_method gating covers every path. The level 2 objections on platform units, zero versus not measured, and future I/O changes were not evaluated.

## Prior attempts

None found before 2024-12-24.

## Objections a reviewer is likely to raise

### 1. Counters vanish under the default io_method, where most users run

A user on a managed service, where only I/O workers are offered, runs EXPLAIN (ANALYZE, IO) to check whether a slow query is disk-bound. No Storage I/O line appears. The core feature gives that user nothing, but the project still carries its parallel plumbing, platform quirks and tests.

Message #39 says worker is now the default io_method. Message #46 reports that managed providers offer only I/O workers and that containers often lack io_uring. The author answers that sync and io_uring users still benefit and that io_uring adoption is growing (message #48, #49). A committer would likely ask whether a core EXPLAIN feature should rest on that forecast, since the cost lands on every release while the benefit reaches only a minority of installations.

**Question:** What share of Postgres 19+ installations can use sync or io_uring today, and what evidence shows those users ask for this in EXPLAIN?

<sub>fixable, level 1b, desirability; messages [#26](https://www.postgresql.org/message-id/3gz6ymrgnua75aviagsl4d4traoqxo2g2rzzykqa3yl4jyts3y%40gj6lcc6aziil), [#39](https://www.postgresql.org/message-id/a8fb87e49a5a547519bb916e375ab563%40oss.nttdata.com), [#46](https://www.postgresql.org/message-id/CAP53PkyM8uRn9uPrEAmv6UaO%2B-O0f5VtQFGDWGWC3BD-Zo1UWQ%40mail.gmail.com), [#48](https://www.postgresql.org/message-id/CAM6-o%3DCATBQpDQAa3akuKqw9gWfpN-N7nVr9%3D8qSejKV%2BQyHXQ%40mail.gmail.com), [#49](https://www.postgresql.org/message-id/95ff66682bec0d10e18ea9c6a3dd3f93%40oss.nttdata.com); `maintenance-fit-1`</sub>

### 2. Earlier in-thread objection about I/O workers still stands for v13

An operator on a managed service that offers only io_method = worker runs EXPLAIN (ANALYZE, IO) on a slow query and gets no Storage I/O line. Others compare numbers taken under sync with numbers taken under worker and misread the difference. The bug reports go to the AIO maintainers.

In message #17 and message #26 Andres objected that the approach cannot work with worker-based AIO, the expected default, and that he would be on the hook for the bug reports. Hiding the output (message #35) answers the undercount in message #23 but not the demand question. Message #46 reports that worker is now the default and that some providers offer only workers. No reply from Andres or Tom follows v12 or v13.

**Question:** What share of Postgres 19 and later installations can use sync or io_uring, and do the AIO maintainers accept hiding the output under worker?

<sub>fixable, level 1b, prior rejection; messages [#17](https://www.postgresql.org/message-id/myw76agwrlmisvamwbfl6ibxgwh5glzitydiwnfmtb5aui232i%40274yxxtnbnsp), [#23](https://www.postgresql.org/message-id/mzjytadtjqv4knvwukppaol5zx2qzt2bkuqvjlo4rjawvhn4ql%40qmdf64qkmxjo), [#26](https://www.postgresql.org/message-id/3gz6ymrgnua75aviagsl4d4traoqxo2g2rzzykqa3yl4jyts3y%40gj6lcc6aziil), [#35](https://www.postgresql.org/message-id/0310f2ff46d990485f96624ac1744c01%40oss.nttdata.com), [#46](https://www.postgresql.org/message-id/CAP53PkyM8uRn9uPrEAmv6UaO%2B-O0f5VtQFGDWGWC3BD-Zo1UWQ%40mail.gmail.com), [#49](https://www.postgresql.org/message-id/95ff66682bec0d10e18ea9c6a3dd3f93%40oss.nttdata.com); `prior-attempts-1`</sub>

### 3. An extension could add this through existing EXPLAIN and executor hooks

Core takes on a counter that is empty under the default io_method = worker, always zero on Windows, and counted in different units on each platform. Andres expects to receive the resulting bug reports (message #26). An extension owner could carry the same cost alone, as pg_stat_kcache has done for years.

pg_stat_kcache already reads these counters and splits shared-buffer hits, OS-cache hits and disk reads per query (messages #3, #11). Extensions can already wrap the executor, and recent releases let them add EXPLAIN options and output. The only piece that may need core is passing worker counts to the leader. A small hook for that would cost less than a core option whose behaviour depends on io_method.

**Question:** Which part of the feature, other than parallel-worker aggregation, cannot be built in an extension with today's executor and EXPLAIN extension hooks?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#3](https://www.postgresql.org/message-id/6023c8ac6b1823e2126a001f668ef245%40oss.nttdata.com), [#11](https://www.postgresql.org/message-id/Z3upZkA5Bi6lYS6a%40jrouhaud), [#26](https://www.postgresql.org/message-id/3gz6ymrgnua75aviagsl4d4traoqxo2g2rzzykqa3yl4jyts3y%40gj6lcc6aziil), [#46](https://www.postgresql.org/message-id/CAP53PkyM8uRn9uPrEAmv6UaO%2B-O0f5VtQFGDWGWC3BD-Zo1UWQ%40mail.gmail.com); `already-exists-2`</sub>

## Questions for the author

1. Do the AIO maintainers accept hiding Storage I/O output under io_method = worker, given the objection in message #26?
2. What share of Postgres 19 and later installations can use io_method = sync or io_uring, and what evidence shows that share?
3. Which part of the feature, other than passing parallel-worker counts to the leader, cannot be built as an extension EXPLAIN option?
4. How does a text reader or a JSON consumer tell "not measured" (Windows, io_method = worker) apart from a measured zero?
5. Which unit and counting rule do ru_inblock and ru_oublock follow on each supported platform other than Linux?

## Touch points

- **ABI / extension API:** The patch changes instrument.h and execParallel.h to pass storage I/O counts from parallel workers to the leader.
- **psql UX conventions:** EXPLAIN (ANALYZE, IO) gains "Storage I/O" lines in text output and two new properties in JSON, XML and YAML.
- **maintenance burden ("who maintains this in 10 years"):** Output depends on io_method and on getrusage() semantics per platform, and the patch changes the Windows getrusage() port.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** pg_stat_kcache already reads the same getrusage() counters, and extensions can add EXPLAIN options.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund and Tom Lane took part, but as objectors, and neither has replied since v12. The owner would need to know the AIO subsystem and EXPLAIN instrumentation.
- Priority: needs design rethink; audience a sizeable minority; review cost medium. Only io_method = sync and io_uring users see the output, and the thread gives no numbers for their share. The 75 KiB patch touches EXPLAIN, parallel executor instrumentation, parallel index builds and the Windows port, and the hiding approach has no visible agreement from the AIO maintainers.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
