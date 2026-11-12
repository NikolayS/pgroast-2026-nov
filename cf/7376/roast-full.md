# Design pre-review: Reduce LWLockWaitListLock() cache-line contention with adaptive spin reads

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7376](https://commitfest.postgresql.org/patch/7376/) (Needs review); authors: Andrew Kim (andrewk1), Baohong Min (minbaohong), Jun Jin (jun-intel)
- Thread: [\[PATCH\] Reduce LWLockWaitListLock() cache-line contention with adaptive spin reads](https://www.postgresql.org/message-id/flat/CH0PR11MB53947D1A122373D7CB2E22D6E9C02%40CH0PR11MB5394.namprd11.prod.outlook.com), started 2026-07-23
- Messages read: 11 of 11
- Patch prose read: LWLockWaitListLock-adaptive-spin-reads-v1.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the problem is stated, and three independent parties measured gains on large x86 and Arm machines. The one surviving objection, prior-attempts-1, is a fixable design point: the constants 5 and 256 are empirical, and no acquisition-latency numbers are posted. A committer who knows lwlock.c would have to judge whether the tested hardware is enough. The promised v2 is not posted, so this judgement covers v1 only.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v2 design (per-tranche interval, re-read after pg_usleep()) is not posted, and only 3 of 11 messages were shown. The code was not read, so the interaction with spins_per_delay and s_lock_stuck() could not be checked.

## The proposal, restated

**Problem.** On large multi-socket machines, backends spinning in LWLockWaitListLock() re-read lock->state on every spin, so the cache line bounces between cores and sockets. The authors report that pgbench TPC-B throughput drops at high client counts on a 2-socket Xeon. Perf shows LWLockWaitListLock() as a top symbol.

**Proposed design.** v1 re-reads lock->state only once every spins_per_lock_read spins, with a backend-local interval that grows by 5 up to 256 under heavy contention and shrinks by 1 otherwise. It also lowers MIN_DELAY_USEC in s_lock.c from 1000 to 100 microseconds, which affects every perform_spin_delay() caller. The locking primitive does not change.

**What the author claims.**
- pgbench TPC-B on a 2-socket, 240-vCPU Xeon gains 1.11x to 1.59x at 120 to 1000 clients.
- Throughput at 8 to 64 clients is unchanged within noise.
- The values 5 and 256 were chosen empirically, and nearby values such as 3, 6, 300 or 500 work as well.
- A backend-wide interval is tuned by the most contended lock, and its gain outweighs added latency on other LWLocks.
- A reviewer measured up to 59% higher peak throughput on 192-core Graviton V3, with no regressions on small systems.

**Thread state.** Jakub Wartak and wenhui qiu reviewed v1, and Haris Okanovic benchmarked it on Arm. The author promised v2: drop the MIN_DELAY_USEC change, make the interval per-tranche, re-read state after sleeps. No v2 is posted yet.

## 2. Prior attempts

### remove adaptive spins_per_delay code (related)

- **Link:** [https://commitfest.postgresql.org/patch/5210/](https://commitfest.postgresql.org/patch/5210/)
- **Thread:** [https://www.postgresql.org/message-id/flat/Zs37z9t4KD94H9BE@nathan](https://www.postgresql.org/message-id/flat/Zs37z9t4KD94H9BE@nathan)
- **Date:** decided 2024-11-03; started 2024-08-27
- **Outcome:** Withdrawn (cf)
- **Who objected:** Andres Freund (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Andres Freund reported that on multi-socket machines, some spins_per_delay values made contended performance much worse, and good values differ from those on smaller machines. Chiplet-based server CPUs behave the same way, so one fixed value is hard to choose. Nathan Bossart agreed and turned to other approaches, such as exponential backoff. No message states the reason for the withdrawal.
- **Reason (quote):** "So finding a good value probably isn't easy." ([message](https://www.postgresql.org/message-id/w272rwzgxf3s7n7ufobhqf5uce2hq5zvhtn2ku4unyfau6qjm2%40ds4n6lzgjvnm))
- **What is different this time:** P2 tried to remove the adaptive spins_per_delay and fix its value. This patch adds a separate adaptive read interval, but the bounds that drive it are fixed constants.

### Fix LWLock degradation on NUMA (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1166/](https://commitfest.postgresql.org/patch/1166/)
- **Thread:** [https://www.postgresql.org/message-id/flat/2968c0be065baab8865c4c95de3f435c@postgrespro.ru](https://www.postgresql.org/message-id/flat/2968c0be065baab8865c4c95de3f435c@postgrespro.ru)
- **Date:** decided 2018-11-30; started 2017-06-05
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** It targets the same problem, contended LWLock throughput on NUMA. The corpus does not record its design or why it was returned, so no lesson can be drawn from it.


## 3. Objections a reviewer is likely to raise

### 1. Fixed spin constants may not carry over across CPU types, as [P2] found

fixable · level 2, rubric item 8 · prior rejection, performance on the hot path, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A user on a chiplet-based x86 server, or a CPU with a different PAUSE cost, upgrades. With the interval near 256, waiters notice lock release later, so moderately contended LWLocks see higher acquisition latency. The user finds out only as a throughput or latency regression after the upgrade.

[P2] stopped hard-wiring spins_per_delay because good values on multi-socket machines differ from those on smaller ones. This patch adds new fixed constants (threshold 5, step +5/-1, cap 256), and message #9 calls them empirical. The tests cover two-socket Xeon, Graviton and old Sandy Bridge, but give throughput only, not acquisition latency, which message #7 asked for. Messages #3 and #6 report that a related LWLockWaitListLock() change regressed on x86, so it is now limited to aarch64.

**Precedents.**
- [remove adaptive spins_per_delay code](https://commitfest.postgresql.org/patch/5210/) (Withdrawn, 2024-11-03); thread: [link](https://www.postgresql.org/message-id/flat/Zs37z9t4KD94H9BE@nathan)

Thread: [#3 ("Okanovic, Haris", 2026-09-18)](https://www.postgresql.org/message-id/MN7PR18MB660231D56BF8FD8839F98F539DA8872%40MN7PR18MB660231.namprd18.prod.outlook.com), [#6 (Jakub Wartak, 2026-10-01)](https://www.postgresql.org/message-id/CAKZiRmy8nmL0urEER4tqOboUVwpa3gROwG2%3DE7PmZ6O9nZ6hqw%40mail.gmail.com), [#7 (wenhui qiu, 2026-10-02)](https://www.postgresql.org/message-id/CAGjGUAJAYx2T_GRizy4mSAnNfA4h88g3KZDQA3WFViU6UhGdQQ%40mail.gmail.com), [#9 ("Min, Baohong", 2026-10-10)](https://www.postgresql.org/message-id/CH0PR11MB539472114723348A411EC11AE9912%40CH0PR11MB5394.namprd11.prod.outlook.com)

**Question:** Do 5 and 256 hold up on single-socket chiplet x86 and on CPUs with different PAUSE latency, measured as acquisition latency as well as throughput?

Counter-check: held. Message #9 calls 5 and 256 empirical, and the acquisition-latency numbers that message #7 asked for are not posted. The futex-mutex regression in messages #3 and #6 concerns a different mechanism and must not be used as support.

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
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds an adaptive heuristic with fixed constants (5, 256) to lwlock.c that later hardware may need retuned. |
| performance on the hot path | **yes** | The patch changes the spin loop in LWLockWaitListLock(), and v1 lowers MIN_DELAY_USEC for every perform_spin_delay() caller. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do 5 and 256 hold up on single-socket chiplet x86 and on CPUs with a different PAUSE latency?
2. What are the LWLock acquisition-latency numbers, as opposed to throughput, with the interval near 256 on a moderately contended lock?
3. How many spins does the interval take to return to 1 after contention falls, and what does a waiter lose in that period?
4. What memory and code cost does the per-tranche interval planned for v2 add, and does it change any struct that extensions see?
5. Is per-architecture tuning of the constants planned, or is one pair of values meant to serve all platforms?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows lwlock.c and the spin-delay code in s_lock.c, with access to large multi-socket hardware. None has spoken in the visible thread.
- **Liability:** The owner signs for LWLock wait-list behaviour on every platform, including hardware nobody tested. A regression there would be silent, and the constants may need retuning as CPUs change.
- **Author follow-up:** The authors promised a v2 that drops the MIN_DELAY_USEC change, makes the interval per-tranche and re-reads state after sleeps. No v2 is posted yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Gains of 1.11x to 1.59x on a 240-vCPU Xeon and up to 59% on 192-core Graviton matter to users of large machines, and reviewers are active. The patch is small, but it changes a hot spin loop, so review needs benchmarks on more hardware and a posted v2.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Hard-coded constants 5 and 256 tuned on a few large machines. Refuted (not specific): Duplicate of prior-attempts-1, which makes the same point about the empirical constants, [P2] and the missing latency numbers.
- `maintenance-fit-2` Layered heuristic in a function with competing redesigns. Refuted (no failure scenario): The scenario depends on patches that are not committed: the futex-mutex variant is a separate aarch64-only proposal and message #8 is a draft idea. The lack of an engaged committer belongs to committability and does not show a design flaw.

### Review angles

- prior-attempts: 1 objection(s). One objection. [P8] (lockless wait queue) was returned for lack of interest, and this thread has active reviewers, so that reason does not carry over. [P10] and [P14] share only the topic area.
- standards-compat: 0 objection(s). The proposal has no user-facing contract: no syntax, no GUC, no catalog or pg_dump change, no psql or client-visible behaviour, and v1 touches only lwlock.c and s_lock.c. The global MIN_DELAY_USEC change is the only cross-caller behaviour change, and message #11 commits to dropping it, so nothing is left for the standards and compatibility angle.
- replication-ondisk: 0 objection(s). The v1 patch changes only backend-local spin behaviour in lwlock.c and a delay constant in s_lock.c, so it touches no persistent or cross-node state. It adds no on-disk format, WAL record, catalog change, protocol message or exported struct change. The promised per-tranche v2 has not been posted, so its shared-memory layout and extension ABI effects could not be checked.
- security: 0 objection(s). The proposal only changes how often a backend re-reads lock->state while spinning, plus the spin-delay floor in s_lock.c. It adds no privilege, no new user-visible object and no new information exposure. The risk of a waiter reaching s_lock_stuck() is a safety issue that the thread already raised (messages #7, #10) and that v2 is meant to fix, so it is not a privilege question.
- maintenance-fit: 2 objection(s). Two objections raised. The MIN_DELAY_USEC change and the missing re-read after pg_usleep() are left out because message #11 commits to fixing both in v2.
- already-exists: 0 objection(s). No existing feature, setting or extension changes how often LWLockWaitListLock() re-reads lock->state, and this spin loop cannot be changed from outside core. The adaptive spins_per_delay in s_lock.c controls when a waiter sleeps, not how often it reads the lock, and reviewers already raised how the two interact (messages #7 and #10).
- demand: 0 objection(s). The thread shows real demand from independent sources: the Intel authors (message #1), an AWS benchmark of 59% on 192-core Graviton (message #3), and an EnterpriseDB reviewer measuring +12% on plain pgbench -S (message #6). Large multi-socket and cloud machines with hundreds of clients are a common production setup, and reviewers report no loss on small systems, so the cost does not fall on non-users.

### Precedents retrieved

- P1: [Replace spinlock with lwlock in pg_stat_statements](https://commitfest.postgresql.org/patch/2634/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P2: [remove adaptive spins_per_delay code](https://commitfest.postgresql.org/patch/5210/) (cf, Withdrawn, 2024-11-03) - used in the review; reason summarised from its thread
- P3: [A infrastructure to detect misuse of spin lock easier](https://commitfest.postgresql.org/patch/4768/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P4: [Report tranche ID and LWLock pointer in LWLock static tracepoints](https://commitfest.postgresql.org/patch/2927/) (cf, Returned with feedback, 2021-04-07) - judged not relevant; reason summarised from its thread
- P5: [Add LWLock blocker(s) information](https://commitfest.postgresql.org/patch/2576/) (cf, Returned with feedback, 2020-11-18) - judged not relevant; reason summarised from its thread
- P6: [fix spinlock contention in LogwrtResult](https://commitfest.postgresql.org/patch/2716/) (cf, Withdrawn, 2022-09-23) - judged not relevant; reason summarised from its thread
- P7: [Additional LWLOCK_STATS statistics](https://commitfest.postgresql.org/patch/373/) (cf, Rejected, 2015-12-22) - judged not relevant; reason summarised from its thread
- P8: [Lockless queue of waiters based on atomic operations for LWLock](https://commitfest.postgresql.org/patch/3983/) (cf, Returned with feedback, 2024-01-26) - used in the review; reason summarised from its thread
- P9: [Revert: Implement pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=772faafca1b288c4dd66b7150a7831c27b768003) (revert, Reverted, 2024-04-11) - judged not relevant
- P10: [remove ISB in AArch64's spin_delay() implementation](https://commitfest.postgresql.org/patch/5763/) (cf, Returned with feedback, 2025-08-19) - used in the review
- P11: [Improve documentation for spinlocks file](https://commitfest.postgresql.org/patch/208/) (cf, Rejected, 2015-07-02) - judged not relevant
- P12: [Fix LWLock degradation on NUMA](https://commitfest.postgresql.org/patch/1166/) (cf, Returned with feedback, 2018-11-30) - used in the review
- P13: [minimizing pg_stat_statements.track='none' performance overhead](https://commitfest.postgresql.org/patch/2092/) (cf, Withdrawn, 2019-04-15) - judged not relevant
- P14: [LWLock optimization for Power processors](https://commitfest.postgresql.org/patch/984/) (cf, Returned with feedback, 2017-08-30) - used in the review
- P15: [Subtrans performance smoothing](https://commitfest.postgresql.org/patch/3792/) (cf, Withdrawn, 2022-09-06) - judged not relevant
- P16: [Improving connection scalability](https://commitfest.postgresql.org/patch/3651/) (cf, Rejected, 2022-08-01) - judged not relevant
- P17: [Backpatching injection point core facilities to REL_17_STABLE](https://commitfest.postgresql.org/patch/5865/) (cf, Withdrawn, 2025-08-17) - judged not relevant
- P18: [Exponential backoff for auth_delay](https://commitfest.postgresql.org/patch/4727/) (cf, Returned with feedback, 2024-04-07) - judged not relevant
- P19: [KnownAssignedXidsGetAndSetXmin performance](https://commitfest.postgresql.org/patch/3271/) (cf, Returned with feedback, 2022-11-15) - judged not relevant
- P20: [Precompute string lengths in PerformRadiusTransaction](https://commitfest.postgresql.org/patch/6377/) (cf, Withdrawn, 2026-03-25) - judged not relevant
- P21: [NUMA shared memory interleaving](https://commitfest.postgresql.org/patch/5703/) (cf, Returned with feedback, 2025-07-07) - judged not relevant
- P22: [Cache line alignment of PGXACT](https://commitfest.postgresql.org/patch/974/) (cf, Returned with feedback, 2017-09-22) - judged not relevant
- P23: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant
- P24: [Make pg_numa_init() cope with Docker](https://commitfest.postgresql.org/patch/6171/) (cf, Withdrawn, 2026-01-29) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 134 s; cost reported by the CLI: $1.76
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 1 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 61 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:47:20.116Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
