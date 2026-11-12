# Design pre-review: Adding basic NUMA awareness

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6049](https://commitfest.postgresql.org/patch/6049/) (Needs review); authors: Tomas Vondra (fuzzycz), Greg Burd (gregburd)
- Thread: [Adding basic NUMA awareness](https://www.postgresql.org/message-id/flat/099b9433-2855-4f1b-b421-d078a5d82017%40vondra.me), started 2025-07-01
- Messages read: 111 of 111
- Patch prose read: v20260624-0001-Add-shmem_populate-and-shmem_interleave-GU.patch, v20260624-0002-Infrastructure-for-partitioning-of-shared-.patch, v20260624-0003-NUMA-shared-buffers-partitioning.patch, v20260624-0004-clock-sweep-basic-partitioning.patch, v20260624-0005-clock-sweep-balancing-of-allocations.patch, v20260624-0006-clock-sweep-scan-all-partitions.patch, v20260624-0007-Add-parttioned-clocksweep-and-NUMA-goodies.patch, v1-0001-NUMA-interleaving-buffers.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal of NUMA-aware buffer placement has sustained developer interest, so level 1b does not fail for the series as a whole. Level 2 fails. maintenance-fit-1 and demand-1 show that the partitioning patches now depend on a replacement-policy change whose own author measured about 1% gain. maintenance-fit-2 shows a single-client loss with NUMA off. standards-compat-3 and replication-ondisk-1 show SIGBUS cases with no startup design.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 111 messages and the commit messages were read, so claims about later messages rest on the panel's reading. The cost of the allocation path with NUMA off and the SIGBUS handling cannot be judged without the code.

## The proposal, restated

**Problem.** On multi-socket and multi-node machines, the kernel can place most of shared buffers on one NUMA node, for example after warmup by a single backend. This gives unpredictable performance and remote memory access, and the single clock-sweep hand is a contention point under buffer pressure. The target is large NUMA servers.

**Proposed design.** The series splits shared buffers into a small registry of partitions and binds each partition, with its buffer descriptors, to a NUMA node on a best-effort basis at page granularity. Clock-sweep gets one hand per partition, a home partition per backend, balancing of allocations between partitions, and a scan of all partitions before failing. Greg Burd's later patches replace the 0..5 usage count with one second-chance bit, admit new pages on probation, and bound the work of one allocation.

**User-visible surface.**
- GUCs shmem_populate and shmem_interleave (MAP_POPULATE, interleaving of the whole shared memory segment).
- GUC shared_buffers_numa (default off) to bind buffer partitions to NUMA nodes.
- pg_buffercache 1.8: new pg_buffercache_partitions view with clock-sweep and balancing counters.
- Development GUCs: three clocksweep GUCs (including clocksweep_balance_recalc), debug_numa_node, and function pg_buffercache_set_weights(int, int[]).
- pg_buffercache usagecount and pg_buffercache_usage_counts() now report only 0 or 1.

**What the author claims.**
- Balanced placement makes results predictable and independent of how the cache was warmed.
- Greg Burd measured +5-12% on seqconcurrscans with 2 nodes and +23-47% with 6 nodes; pgbench -S is flat on 2 nodes.
- Single-client sequential scans regress, -12% on 2 nodes and -49% on 6 nodes, because each partition is too small for the table.
- Partitioning alone makes the worst-case single allocation 4.1-4.6x worse on x86; the bound in 0006 removes this.
- Best-effort page mapping wastes about one memory page per partition, which the authors call negligible.

**Thread state.** Tomas Vondra, Jakub Wartak, Andres Freund and Greg Burd discussed it for over a year; Tomas reported regressions in June 2026, and Jakub's 0008 fixes and Greg's measurements followed. Open items include the c=1 regression, a partition size floor, crashes on CPU-less (CXL-like) nodes, and whether Greg's evictor changes belong in this series.

## 2. Prior attempts

### Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671)
- **Date:** decided 2017-03-20; started 2016-04-11
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect. Change was a part of commit 48354581a49c30f5757c203415aa8412d85b0f70 during 9.6 release cycle, so backpath to 9.6 https://commitfest.postgresql.org/13/1029/
- **Reason (quote):** "Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect."
- **What is different this time:** P4 reverted an unintended change to usage count increments on pin. This series replaces the usage count on purpose, so P4 only shows that such changes can disable buffer access strategies.


## 3. Objections a reviewer is likely to raise

### 1. Replacement-policy rewrite is bundled under the NUMA partitioning patches

fixable · level 2, rubric item 6 · design, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A committer who wants only buffer placement and clock-sweep partitioning cannot take 0008-0012 without first taking 0004-0007. Those patches replace usage_count with one bit, add probation admission and add bgwriter behaviour. If a later release shows a workload whose hit ratio drops, the project has to revert the evictor and the partitioning together.

In message #108 the author of 0004 measured a gain of about 1% and wrote that the change is "not worth the potential risks". Message #109 then merged it, plus two new policy changes, ahead of the partitioning. Message #111 justifies the order by the 4x worst-case allocation that 0008 introduces alone, so the series now gives no NUMA step that can be committed without the evictor. Tools that read pg_buffercache usagecount also see their values collapse to 0 or 1.

Thread: [#108 (Greg Burd, 2026-09-22)](https://www.postgresql.org/message-id/z5ln7bfyLp7MD3StPXOgrDAP4mnG4Aj1ZWWkmBENrn7lU43ZFWyGBfAzh3UYQFhYwm0Uj4hwDuc0BimjTVCHpc5RxUvthDWuH9pF44AYFaY%3D%40burd.me), [#109 (Greg Burd, 2026-10-12)](https://www.postgresql.org/message-id/NzRAGLGvd0CoKX6vm7BEWJBEVi-apddzrNoRSzV83nScunblHKAR0Wz0xAOz0M_yFdMtauzUIdCuFR7En_xMbapM3ECeqM34AIv3Z9uy32A%3D%40burd.me), [#111 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/8ylrXvoYwIDHkJUNzxTPpIFdkttaknqyUNpjOq9g05OQxCfgFE_Egkta_Z7rdso8MqQilfKRKtXHTIPg5hKfI2DXQPH1_HnZ4VG4VoC-zPg%3D%40burd.me)

**Question:** Can 0008-0012 stand on the existing 0..5 usage_count with their own fix for the worst-case allocation, so that the evictor change becomes a separate proposal?

Counter-check: held. Message #111 explains the order by the 4.1-4.6x worst case, but it does not say why the bound in 0006 needs the one-bit state. Message #108 judged the evictor not worth its risk, and the brief shows no agreement from Tomas.

### 2. Partitioned clock-sweep costs users who never enable NUMA

fixable · level 2, rubric item 4 · performance on the hot path, planner behaviour change · `maintenance-fit-2`

A user on a single-socket box upgrades. That user gets four clock-sweep partitions by default and a per-allocation node lookup. A single-client report that scans a table larger than one partition, but smaller than shared_buffers, gets slower. The user finds out after the upgrade.

Message #98 reports 0004 with shared_buffers_numa=off at 114% of master latency at c=1. The 0002 commit message hard-codes 4 partitions independent of NUMA. Messages #106 and #107 show the c=1 loss rising to 49% as partitions shrink, and no partition size floor exists yet. Message #95 raises sched_getcpu() cost on every allocation, and only Jakub's 0012 caches it.

Thread: [#95 (Jakub Wartak, 2026-06-16)](https://www.postgresql.org/message-id/CAKZiRmzo9xnJSgO4b26DTZqPuObcQ-6ncay%2BmOEKs9rzCkegUA%40mail.gmail.com), [#98 (Tomas Vondra, 2026-06-24)](https://www.postgresql.org/message-id/c3954e28-0c38-4df8-b76e-ec09f6c04021%40vondra.me), [#106 ("Greg Burd", 2026-09-15)](https://www.postgresql.org/message-id/b3c09a66-b2f6-49a6-a2fe-bbfe0cda3e6f%40app.fastmail.com), [#107 ("Greg Burd", 2026-09-15)](https://www.postgresql.org/message-id/dba6522b-7de9-403f-80f5-783cb90d4647%40app.fastmail.com)

**Question:** With every NUMA option at its default on a non-NUMA machine, does the series behave like master for a single client scanning a relation between NBuffers/nparts and NBuffers/4?

Counter-check: held. Message #98 says the numa=off regression mostly recovers with later patches, and 0012 caches the node lookup. The c=1 loss from small partitions remains, because the minimum of 4 partitions applies without NUMA and no partition size floor exists.

### 3. shared_buffers_numa does not follow NUMA limits set by numactl or systemd

fixable · level 2, rubric item 12 · design, new GUC, doesn't fit what Postgres is about (extension / external tool / already solved) · `standards-compat-3`

An operator limits the instance to node 0 with numactl or systemd NUMAMask, or runs on a host with a CPU-less memory node. With shared_buffers_numa=on, the postmaster calls mbind() for every configured node. Startup then fails with SIGBUS, or shared buffers land on slow memory the operator had excluded.

Message #104 reproduces the SIGBUS on a CPU-less node and notes that the code counts nodes as 0..numa_num_configured_nodes() and ignores sched_getaffinity() and systemd masks. Message #105 agrees that the server should respect a NUMA limit the operator already set. Today numactl and systemd are the standard way to place a Postgres instance on NUMA hardware, so a setting that overrides them breaks deployments that work now. Message #97 suggested a FATAL exit when huge pages run short on a node, but the proposal does not show one.

Thread: [#97 (Jakub Wartak, 2026-06-17)](https://www.postgresql.org/message-id/CAKZiRmzumFJ8%2BBqz5mfX%2BQGkrKEaH32zz7kPgEShHWXE4QzyQw%40mail.gmail.com), [#104 (Jakub Wartak, 2026-09-02)](https://www.postgresql.org/message-id/CAKZiRmzT1M1fEFf5fRq2Qg3Vz0wLbdY%2BuNdF-0ORhDTUOUVhcA%40mail.gmail.com), [#105 ("Greg Burd", 2026-09-10)](https://www.postgresql.org/message-id/f78dfe7d-4a37-40ec-8e60-c2cc47c48364%40app.fastmail.com)

**Question:** How does the series find the allowed node set, including non-contiguous, CPU-less, or masked nodes, and what does the server report when it cannot honour it?

Counter-check: held. Message #104 reproduces the SIGBUS on a CPU-less node and message #105 agrees that inherited masks should be respected. No later version in the material changes the node loop.

### 4. Huge page shortage on one node can surface as SIGBUS after startup

fixable · level 2, rubric item 7 · design, WAL / replication / standby · `replication-ondisk-1`

An operator sizes nr_hugepages from shared_memory_size_in_huge_pages, and the kernel gives one node fewer pages than its partition needs. The server starts. Later a backend first touches a buffer bound to that node and gets SIGBUS, and the postmaster resets every session and runs crash recovery.

Message #94 reports that earlier versions crashed "much later" when one node ran out of huge pages. Message #75 leaves MAP_POPULATE commented out and still expects a small chance of SIGBUS. Message #104 shows the same failure on a CPU-less node. A FATAL exit at startup was suggested in message #97, but the material does not show it. A standby or failover target with a different per-node huge page layout would meet this at the worst moment.

Thread: [#75 (Tomas Vondra, 2025-11-21)](https://www.postgresql.org/message-id/5bc6d309-61fe-4d8b-9e1f-a2961564a559%40vondra.me), [#94 (Tomas Vondra, 2026-06-05)](https://www.postgresql.org/message-id/334f7644-6716-4e63-a7c4-13c5cbcdd210%40vondra.me), [#97 (Jakub Wartak, 2026-06-17)](https://www.postgresql.org/message-id/CAKZiRmzumFJ8%2BBqz5mfX%2BQGkrKEaH32zz7kPgEShHWXE4QzyQw%40mail.gmail.com), [#104 (Jakub Wartak, 2026-09-02)](https://www.postgresql.org/message-id/CAKZiRmzT1M1fEFf5fRq2Qg3Vz0wLbdY%2BuNdF-0ORhDTUOUVhcA%40mail.gmail.com)

**Question:** How does the server detect at startup that every bound partition can be backed on its node, and fail cleanly instead of with a later SIGBUS?

Counter-check: held. Message #75 reduces the risk by interleaving first but still expects a small chance of SIGBUS, and the FATAL exit from message #97 is not shown. The standby detail adds little, but the startup check is still missing.

### 5. Replacing usage_count for all users lacks shown user benefit

fixable · level 1b, rubric item 4 · desirability, performance on the hot path, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `demand-1`

Every installation, NUMA or not, gets a one-bit second-chance evictor with probation admission and bgwriter changes. Users and tools that read pg_buffercache usagecount find that it now reports only 0 or 1. A workload tuned under the 0..5 policy loses hit ratio after upgrade, and the user finds out only from a production regression.

In message #108, the evictor's own author measured about 1% more index leaf pages resident and TPS inside the noise. He concluded it was "not worth the potential risks" and said he would drop it unless someone else supported it. No one in the visible thread did, yet message #109 merges a broader version (0004-0007) ahead of the partitioning work. The main stated reason in message #111 is to fix a worst-case allocation regression that partitioning itself introduces, so this does not show demand for the evictor change on its own merits.

Thread: [#107 ("Greg Burd", 2026-09-15)](https://www.postgresql.org/message-id/dba6522b-7de9-403f-80f5-783cb90d4647%40app.fastmail.com), [#108 (Greg Burd, 2026-09-22)](https://www.postgresql.org/message-id/z5ln7bfyLp7MD3StPXOgrDAP4mnG4Aj1ZWWkmBENrn7lU43ZFWyGBfAzh3UYQFhYwm0Uj4hwDuc0BimjTVCHpc5RxUvthDWuH9pF44AYFaY%3D%40burd.me), [#109 (Greg Burd, 2026-10-12)](https://www.postgresql.org/message-id/NzRAGLGvd0CoKX6vm7BEWJBEVi-apddzrNoRSzV83nScunblHKAR0Wz0xAOz0M_yFdMtauzUIdCuFR7En_xMbapM3ECeqM34AIv3Z9uy32A%3D%40burd.me), [#111 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/8ylrXvoYwIDHkJUNzxTPpIFdkttaknqyUNpjOq9g05OQxCfgFE_Egkta_Z7rdso8MqQilfKRKtXHTIPg5hKfI2DXQPH1_HnZ4VG4VoC-zPg%3D%40burd.me)

**Question:** Which workloads, outside fixing partitioning's own tail latency, gain measurably from the one-bit evictor, and who other than its author has asked for it?

Counter-check: held. Message #108 reports about 1% more leaf pages resident and TPS within run spread, and no other participant asked for the change. The benefit shown in message #111 is the removal of a regression that partitioning itself adds.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | see objection replication-ondisk-1: Huge page shortage on one node can surface as SIGBUS after startup |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The series changes buf_internals.h and bufmgr.h, and pg_buffercache usagecount now reports only 0 or 1. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The series adds shmem_populate, shmem_interleave, shared_buffers_numa, three clocksweep GUCs and debug_numa_node. |
| planner behaviour change | **yes** | see objection maintenance-fit-2: Partitioned clock-sweep costs users who never enable NUMA |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | pg_buffercache_set_weights() changes allocation weights for all backends, and the proposal does not say who may call it. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The series adds partitioned clock-sweep, balancing, mbind-based placement and a new eviction policy in the buffer manager. |
| performance on the hot path | **yes** | Buffer allocation in freelist.c and bufmgr.c changes for every installation, with a minimum of 4 partitions without NUMA. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection standards-compat-3: shared_buffers_numa does not follow NUMA limits set by numactl or systemd |

## 5. Questions the author must answer before anyone reviews code

1. Can the partitioning patches stand on the existing 0..5 usage count with their own bound on one allocation, so that the second-chance evictor becomes a separate proposal?
2. With every NUMA option at its default on a non-NUMA machine, does a single client scanning a relation larger than one partition perform like master?
3. How does the series find the allowed node set with CPU-less nodes, non-contiguous node IDs, or a mask set by numactl or systemd?
4. How does the server detect at startup that each bound partition has enough huge pages on its node, and fail with a FATAL message instead of a later SIGBUS?
5. Which of the new GUCs and pg_buffercache_set_weights() are proposed for commit as supported settings, and which are for development only?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tomas Vondra wrote the base series and takes part in the thread. The owner must know the buffer manager and Linux NUMA and huge page behaviour.
- **Liability:** The owner signs for buffer replacement behaviour on every installation, for startup failures tied to kernel NUMA and huge page layout, and for each shipped GUC, with no end date.
- **Author follow-up:** The authors have kept the series going for over a year, with rebases, regression reports and follow-up fixes through October 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The target is large multi-node servers, and the series changes buffer allocation for everyone and adds kernel-dependent startup code. A committer is engaged, but the evictor bundling, the single-client regression and the node set handling need answers before code review.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) usagecount in pg_buffercache keeps its name but changes meaning. Not listed: not among the strongest objections.
- `standards-compat-2` (fixable, level 2) No stated split between shipped GUCs and development-only knobs. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` Replacement-policy rewrite bundled as a prerequisite of NUMA partitioning. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same bundling point without a precedent. [P6] concerns LWLock acquisition and must not be used as support here.
- `replication-ondisk-2` usagecount semantics and buffer state macros change without notice. Refuted (not specific): Duplicate of standards-compat-1 for the usagecount column. The removed macros live in buf_internals.h, an internal header that carries no stability promise across major versions.
- `security-1` Privileges for pg_buffercache_set_weights and clocksweep runtime GUCs not stated. Refuted (no failure scenario): The scenario assumes grants and GUC contexts that the material does not show, and it needs clocksweep_balance_recalc turned off first. Message #98 presents these as development aids, so the open question is the one in standards-compat-2.
- `maintenance-fit-3` Startup depends on kernel NUMA and huge page layout with no design. Refuted (not specific): Duplicate of standards-compat-3 for the node set and of replication-ondisk-1 for the huge page shortage. The remark about future kernel or libnuma changes names no concrete case.
- `maintenance-fit-4` No plan for which of the new GUCs and functions ship. Refuted (not specific): Duplicate of standards-compat-2, which asks the same question about which settings ship and has the more concrete failure scenario.
- `already-exists-1` Balanced buffer placement repeats what numactl --interleave=all already gives. Refuted (already handled): Interleaving cannot tell the sweep which node a buffer is on. Messages #90 and #93 show the locality gain needs node-bound partitions so a backend allocates local memory first, and message #102 reports results against an interleaved master.
- `already-exists-2` shmem_interleave and debug_numa_node duplicate numactl options as GUCs. Refuted (already handled): Message #10 explains that numactl --interleave also interleaves backend private memory, which shmem_interleave avoids by acting on the shared segment only. debug_numa_node carries the debug_ prefix, and the rest duplicates standards-compat-2.
- `demand-2` Partitioned clock-sweep cost falls on everyone, gain on wide NUMA only. Refuted (not specific): Duplicate of maintenance-fit-2 for the cost on non-NUMA users. The demand part is weak, because several developers who know the buffer manager have worked on this problem for over a year.

### Review angles

- prior-attempts: 1 objection(s). No precedent shows NUMA buffer partitioning rejected before. Jakub Wartak's earlier NUMA entry, referenced in message #1, was returned in message #10 in favour of this approach, so it does not count against it.
- standards-compat: 3 objection(s). Three objections are listed. The SQL syntax and pg_dump angles do not apply: the series adds no SQL-level objects and no syntax.
- replication-ondisk: 2 objection(s). The series adds no WAL records, on-disk format or catalog state. Standbys and pg_upgrade are not affected beyond the two points above.
- security: 1 objection(s). The core design, which places buffers on NUMA nodes and partitions clock-sweep, does not expose data or cross a privilege boundary. The new pg_buffercache_partitions view shows only allocation counters.
- maintenance-fit: 4 objection(s). Not applicable. A committer (Tomas) and Andres are engaged, so ownership is less of a concern than design scope.
- already-exists: 2 objection(s). The partitioned clock-sweep itself has no existing equivalent in core or in an extension that I can name, so I raise nothing against it from this angle.
- demand: 2 objection(s). The NUMA placement part has independent developer interest: Jakub withdrew a competing patch, Andres has an earlier prototype, and two engineers ran measurements on 6-node hardware. These objections target only the parts whose cost lands on users who are not on NUMA hardware.

### Precedents retrieved

- P1: [\[BUG\] Assert fails for local buffers in BufferIsExclusiveLocked](https://commitfest.postgresql.org/patch/5427/) (cf, Withdrawn, 2024-12-09) - judged not relevant; reason summarised from its thread
- P2: [Asymmetric partition-wise JOIN](https://commitfest.postgresql.org/patch/2249/) (cf, Returned with feedback, 2020-11-30) - judged not relevant
- P3: [Default partition in list partitioning](https://commitfest.postgresql.org/patch/1066/) (cf, Returned with feedback, 2017-04-04) - judged not relevant; reason summarised from its thread
- P4: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - used in the review
- P5: [Asymmetric partition-wise JOIN](https://commitfest.postgresql.org/patch/3099/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P6: [Fix LWLock degradation on NUMA](https://commitfest.postgresql.org/patch/1166/) (cf, Returned with feedback, 2018-11-30) - used in the review; reason summarised from its thread
- P7: [Extend more usecase for planning time partition pruning and init partition pruning.](https://commitfest.postgresql.org/patch/2954/) (cf, Withdrawn, 2021-02-19) - judged not relevant; reason summarised from its thread
- P8: [Extend more usecase for planning time partition pruning and init partition pruning.](https://commitfest.postgresql.org/patch/2975/) (cf, Returned with feedback, 2021-03-27) - judged not relevant; reason summarised from its thread
- P9: [revert to showing buffer counts in explain (buffers)](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=27cb626f7a28abd18c0e8a34fc8d6674b0f6cdf3) (revert, Reverted, 2010-02-16) - judged not relevant
- P10: [Improved crc32c performance with AVX-512 algorithm.](https://commitfest.postgresql.org/patch/5100/) (cf, Returned with feedback, 2025-04-02) - judged not relevant; reason summarised from its thread
- P11: [Switch to FullTransactionId for PGPROC->xid and XLogRecord->xl_xid](https://commitfest.postgresql.org/patch/4741/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P12: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P13: [Identify huge pages accessibility using madvise](https://commitfest.postgresql.org/patch/4937/) (cf, Withdrawn, 2024-11-12) - judged not relevant
- P14: [Optimize numeric.c mul_var() using the Karatsuba algorithm](https://commitfest.postgresql.org/patch/4930/) (cf, Withdrawn, 2024-09-04) - judged not relevant
- P15: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P16: [Supporting huge pages on Windows](https://commitfest.postgresql.org/patch/813/) (cf, Returned with feedback, 2017-04-07) - judged not relevant
- P17: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant
- P18: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P19: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant
- P20: [Revert indexed and enlargable binary heap implementation.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=810f64a01567610af7b92b0de930f16f3e20064e) (revert, Reverted, 2024-04-11) - judged not relevant
- P21: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P22: [Support huge_pages on AIX](https://commitfest.postgresql.org/patch/1960/) (cf, Withdrawn, 2019-12-03) - judged not relevant
- P23: [Changing the autovacuum launcher scheduling; oldest table first algorithm](https://commitfest.postgresql.org/patch/1573/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P24: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0002-NUMA-localalloc.patch not read
- attachment limit (8) reached; v1-0003-freelist-Don-t-track-tail-of-a-freelist.patch not read
- attachment limit (8) reached; v1-0004-NUMA-partition-buffer-freelist.patch not read
- attachment limit (8) reached; v1-0005-NUMA-interleave-PGPROC-entries.patch not read
- attachment limit (8) reached; v1-0006-NUMA-pin-backends-to-NUMA-nodes.patch not read
- 67 long messages in the middle of the thread were left out of the prompts

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 218 s; cost reported by the CLI: $8.68
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 126 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:40:51.049Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
