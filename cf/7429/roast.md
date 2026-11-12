# Design pre-review: LWLock granular partition lock memory layout

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7429](https://commitfest.postgresql.org/patch/7429/); [thread](https://www.postgresql.org/message-id/flat/CAE8JnxOGU4pT19z6JRVW9W0inS1cXt_pRPpCN0dAdTueo_GVkg%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The proposal fails at level 1b on evidence. demand-1: message #3 asked for numbers from realistic workloads, and message #4 says the effect is not visible on one on the author's machine. already-exists-1: the thread never compares the patch with unpatched Postgres that has S times more padded partitions, which costs a few KiB. The idea may be sound, but the list needs both answers before design review.

Not checked: Without the code, the change in pg_config_manual.h, the value of S and the fate of the old offset macros are unknown. The level 2 objections on exclusive-heavy false sharing and on the second lock stride were not evaluated.

## Prior attempts

- [Cache line alignment of PGXACT](https://www.postgresql.org/message-id/flat/CAPpHfdtJY4zOEDsjad6J5AyZMqZcv6gSY9AkKpA7qN3jyQ2+1Q@mail.gmail.com) (related; Returned with feedback, 2017-09-22). "It became clear that this patch causes regression." ([message](https://www.postgresql.org/message-id/CAPpHfdtrbEFfT9BQ5D762QSdKhkFTewJ_%2BvEC%3DL6oHCu5JcE8w%40mail.gmail.com)) Different now: P6 added cache-line alignment to PGXACT entries; this proposal reduces padding of partition LWLocks. P6 supports only the request for read-write runs and a layout control.

## Objections a reviewer is likely to raise

### 1. No user or workload shown to suffer from padded partition locks

A committer changes the layout of shared lock arrays, the buffer mapping and lock manager locks among them, based on one laptop pgbench run. No user has reported contention on these locks, so nothing shows that anyone in the field gains. If the multi-socket gain never appears, the project keeps the extra layout complexity for nothing.

In message #3 Nathan Bossart asked for numbers from realistic workloads. In message #4 the author says the effect cannot be seen on a realistic workload on the test machine and only expects it on multi-socket hosts. Message #5 gives one run at 1 and 8 clients, without the script, scale, mode or which partitioned locks changed. The thread has no field report, no wait-event data and no second person asking for this.

**Question:** On which production-like workload and hardware do partition LWLock waits show up in pg_stat_activity wait events, and how much does the patch reduce them?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/asaVvxCLAu0t372f%40nathan), [#4](https://www.postgresql.org/message-id/CAE8JnxPhhbyajhri9X6YRXX0WuR%2BtHt1D0AOea-Rk3j_i5YJKA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAE8JnxO699xhiRx%3DQjBdO3V8wYz32yTkFdhq0aiqa4iXTXwV6w%40mail.gmail.com); `demand-1`</sub>

### 2. Raising the existing partition-count constants may give the same gain

A committer compares the patch with a one-line change to LOG2_NUM_LOCK_PARTITIONS or NUM_BUFFER_PARTITIONS, which keeps every lock on its own padded cache line. If that matches the 26% gain, the project has added a second LWLock layout type and a new struct layout for a few KiB of shared memory.

Partition counts are already compile-time constants in lwlock.h. The author's model compares S packed locks against one padded lock, but the plain alternative is S padded locks on S cache lines, which has no false sharing. With 128 partitions that alternative costs about 8 KiB more per S step. Message #5 compares patched against base at the same count and never against base with S times more partitions.

Precedent: [Cache line alignment of PGXACT](https://www.postgresql.org/message-id/flat/CAPpHfdtJY4zOEDsjad6J5AyZMqZcv6gSY9AkKpA7qN3jyQ2+1Q@mail.gmail.com) (Returned with feedback, 2017-09-22)

**Question:** With the same pgbench run, how does the patch compare with unpatched Postgres where the partition count is raised by a factor of S, and what is the memory difference?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAE8JnxOGU4pT19z6JRVW9W0inS1cXt_pRPpCN0dAdTueo_GVkg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAE8JnxPhhbyajhri9X6YRXX0WuR%2BtHt1D0AOea-Rk3j_i5YJKA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAE8JnxO699xhiRx%3DQjBdO3V8wYz32yTkFdhq0aiqa4iXTXwV6w%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. On which realistic workload and hardware do partition LWLock waits show up in wait events, and how much does the patch reduce them?
2. With the same pgbench run, how does the patch compare with unpatched Postgres whose partition count is raised by a factor of S, and what is the memory difference?
3. What were the pgbench script, scale, mode and number of runs behind the 26% result at 8 clients?
4. Which partitioned locks change (buffer mapping, lock manager, predicate lock), and does the number of partitions change or only the memory layout?
5. What do read-write and exclusive-heavy runs show on a multi-socket machine, where locks on one cache line may cause false sharing?

## Touch points

- **ABI / extension API:** Patch 0002 adds LWLockAligned to lwlock.h, and 0001 changes partition lock declarations in lock.h, buf_internals.h and lwlocklist.h.
- **maintenance burden ("who maintains this in 10 years"):** A second lock stride sits next to LWLockPadded, and each future partitioned lock group must choose one.
- **performance on the hot path:** Partition LWLocks, which sit on the lock acquire and release path, would share cache lines.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Raising the existing compile-time partition counts with padded locks may give the same gain.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the LWLock code and the layout of the main lock array. The one reviewer in the thread only asked for numbers.
- Not now, show demand first: Repeatable results on a many-core or multi-socket machine, with script, scale, mode and run count stated, for read-only and read-write loads. They must show a gain over both unpatched Postgres and unpatched Postgres with S times more padded partitions.
- Priority: park; audience unclear; review cost medium. The patches are small, but they change the layout of core lock arrays, and judging them needs benchmark work on large machines. The thread shows one laptop pgbench result and no workload that suffers today, so review time is better spent once the measurements exist.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
