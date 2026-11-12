# Design pre-review: Reduce LWLockWaitListLock() cache-line contention with adaptive spin reads

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7376](https://commitfest.postgresql.org/patch/7376/); [thread](https://www.postgresql.org/message-id/flat/CH0PR11MB53947D1A122373D7CB2E22D6E9C02%40CH0PR11MB5394.namprd11.prod.outlook.com), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the problem is stated, and three independent parties measured gains on large x86 and Arm machines. The one surviving objection, prior-attempts-1, is a fixable design point: the constants 5 and 256 are empirical, and no acquisition-latency numbers are posted. A committer who knows lwlock.c would have to judge whether the tested hardware is enough. The promised v2 is not posted, so this judgement covers v1 only.

Not checked: The v2 design (per-tranche interval, re-read after pg_usleep()) is not posted, and only 3 of 11 messages were shown. The code was not read, so the interaction with spins_per_delay and s_lock_stuck() could not be checked.

## Prior attempts

- [remove adaptive spins_per_delay code](https://www.postgresql.org/message-id/flat/Zs37z9t4KD94H9BE@nathan) (related; Withdrawn, 2024-11-03). "So finding a good value probably isn't easy." ([message](https://www.postgresql.org/message-id/w272rwzgxf3s7n7ufobhqf5uce2hq5zvhtn2ku4unyfau6qjm2%40ds4n6lzgjvnm)) Different now: P2 tried to remove the adaptive spins_per_delay and fix its value. This patch adds a separate adaptive read interval, but the bounds that drive it are fixed constants.
- [Fix LWLock degradation on NUMA](https://www.postgresql.org/message-id/flat/2968c0be065baab8865c4c95de3f435c@postgrespro.ru) (similar idea; Returned with feedback, 2018-11-30). reason not recorded Different now: It targets the same problem, contended LWLock throughput on NUMA. The corpus does not record its design or why it was returned, so no lesson can be drawn from it.

## Objections a reviewer is likely to raise

### 1. Fixed spin constants may not carry over across CPU types, as [P2] found

A user on a chiplet-based x86 server, or a CPU with a different PAUSE cost, upgrades. With the interval near 256, waiters notice lock release later, so moderately contended LWLocks see higher acquisition latency. The user finds out only as a throughput or latency regression after the upgrade.

[P2] stopped hard-wiring spins_per_delay because good values on multi-socket machines differ from those on smaller ones. This patch adds new fixed constants (threshold 5, step +5/-1, cap 256), and message #9 calls them empirical. The tests cover two-socket Xeon, Graviton and old Sandy Bridge, but give throughput only, not acquisition latency, which message #7 asked for. Messages #3 and #6 report that a related LWLockWaitListLock() change regressed on x86, so it is now limited to aarch64.

Precedent: [remove adaptive spins_per_delay code](https://www.postgresql.org/message-id/flat/Zs37z9t4KD94H9BE@nathan) (Withdrawn, 2024-11-03)

**Question:** Do 5 and 256 hold up on single-socket chiplet x86 and on CPUs with different PAUSE latency, measured as acquisition latency as well as throughput?

<sub>fixable, level 2, prior rejection; messages [#3](https://www.postgresql.org/message-id/MN7PR18MB660231D56BF8FD8839F98F539DA8872%40MN7PR18MB660231.namprd18.prod.outlook.com), [#6](https://www.postgresql.org/message-id/CAKZiRmy8nmL0urEER4tqOboUVwpa3gROwG2%3DE7PmZ6O9nZ6hqw%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAGjGUAJAYx2T_GRizy4mSAnNfA4h88g3KZDQA3WFViU6UhGdQQ%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CH0PR11MB539472114723348A411EC11AE9912%40CH0PR11MB5394.namprd11.prod.outlook.com); `prior-attempts-1`</sub>

## Questions for the author

1. Do 5 and 256 hold up on single-socket chiplet x86 and on CPUs with a different PAUSE latency?
2. What are the LWLock acquisition-latency numbers, as opposed to throughput, with the interval near 256 on a moderately contended lock?
3. How many spins does the interval take to return to 1 after contention falls, and what does a waiter lose in that period?
4. What memory and code cost does the per-tranche interval planned for v2 add, and does it change any struct that extensions see?
5. Is per-architecture tuning of the constants planned, or is one pair of values meant to serve all platforms?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch adds an adaptive heuristic with fixed constants (5, 256) to lwlock.c that later hardware may need retuned.
- **performance on the hot path:** The patch changes the spin loop in LWLockWaitListLock(), and v1 lowers MIN_DELAY_USEC for every perform_spin_delay() caller.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows lwlock.c and the spin-delay code in s_lock.c, with access to large multi-socket hardware. None has spoken in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost medium. Gains of 1.11x to 1.59x on a 240-vCPU Xeon and up to 59% on 192-core Graviton matter to users of large machines, and reviewers are active. The patch is small, but it changes a hot spin loop, so review needs benchmarks on more hardware and a posted v2.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
