# Design pre-review: Adding basic NUMA awareness

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6049](https://commitfest.postgresql.org/patch/6049/); [thread](https://www.postgresql.org/message-id/flat/099b9433-2855-4f1b-b421-d078a5d82017%40vondra.me), 111 of 111 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal of NUMA-aware buffer placement has sustained developer interest, so level 1b does not fail for the series as a whole. Level 2 fails. maintenance-fit-1 and demand-1 show that the partitioning patches now depend on a replacement-policy change whose own author measured about 1% gain. maintenance-fit-2 shows a single-client loss with NUMA off. standards-compat-3 and replication-ondisk-1 show SIGBUS cases with no startup design.

Not checked: Only 3 of 111 messages and the commit messages were read, so claims about later messages rest on the panel's reading. The cost of the allocation path with NUMA off and the SIGBUS handling cannot be judged without the code.

## Prior attempts

- [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (related; Reverted, 2017-03-20). "Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect." Different now: P4 reverted an unintended change to usage count increments on pin. This series replaces the usage count on purpose, so P4 only shows that such changes can disable buffer access strategies.

## Objections a reviewer is likely to raise

### 1. Replacement-policy rewrite is bundled under the NUMA partitioning patches

A committer who wants only buffer placement and clock-sweep partitioning cannot take 0008-0012 without first taking 0004-0007. Those patches replace usage_count with one bit, add probation admission and add bgwriter behaviour. If a later release shows a workload whose hit ratio drops, the project has to revert the evictor and the partitioning together.

In message #108 the author of 0004 measured a gain of about 1% and wrote that the change is "not worth the potential risks". Message #109 then merged it, plus two new policy changes, ahead of the partitioning. Message #111 justifies the order by the 4x worst-case allocation that 0008 introduces alone, so the series now gives no NUMA step that can be committed without the evictor. Tools that read pg_buffercache usagecount also see their values collapse to 0 or 1.

**Question:** Can 0008-0012 stand on the existing 0..5 usage_count with their own fix for the worst-case allocation, so that the evictor change becomes a separate proposal?

<sub>fixable, level 2, design; messages [#108](https://www.postgresql.org/message-id/z5ln7bfyLp7MD3StPXOgrDAP4mnG4Aj1ZWWkmBENrn7lU43ZFWyGBfAzh3UYQFhYwm0Uj4hwDuc0BimjTVCHpc5RxUvthDWuH9pF44AYFaY%3D%40burd.me), [#109](https://www.postgresql.org/message-id/NzRAGLGvd0CoKX6vm7BEWJBEVi-apddzrNoRSzV83nScunblHKAR0Wz0xAOz0M_yFdMtauzUIdCuFR7En_xMbapM3ECeqM34AIv3Z9uy32A%3D%40burd.me), [#111](https://www.postgresql.org/message-id/8ylrXvoYwIDHkJUNzxTPpIFdkttaknqyUNpjOq9g05OQxCfgFE_Egkta_Z7rdso8MqQilfKRKtXHTIPg5hKfI2DXQPH1_HnZ4VG4VoC-zPg%3D%40burd.me); `maintenance-fit-1`</sub>

### 2. Partitioned clock-sweep costs users who never enable NUMA

A user on a single-socket box upgrades. That user gets four clock-sweep partitions by default and a per-allocation node lookup. A single-client report that scans a table larger than one partition, but smaller than shared_buffers, gets slower. The user finds out after the upgrade.

Message #98 reports 0004 with shared_buffers_numa=off at 114% of master latency at c=1. The 0002 commit message hard-codes 4 partitions independent of NUMA. Messages #106 and #107 show the c=1 loss rising to 49% as partitions shrink, and no partition size floor exists yet. Message #95 raises sched_getcpu() cost on every allocation, and only Jakub's 0012 caches it.

**Question:** With every NUMA option at its default on a non-NUMA machine, does the series behave like master for a single client scanning a relation between NBuffers/nparts and NBuffers/4?

<sub>fixable, level 2, performance on the hot path; messages [#95](https://www.postgresql.org/message-id/CAKZiRmzo9xnJSgO4b26DTZqPuObcQ-6ncay%2BmOEKs9rzCkegUA%40mail.gmail.com), [#98](https://www.postgresql.org/message-id/c3954e28-0c38-4df8-b76e-ec09f6c04021%40vondra.me), [#106](https://www.postgresql.org/message-id/b3c09a66-b2f6-49a6-a2fe-bbfe0cda3e6f%40app.fastmail.com), [#107](https://www.postgresql.org/message-id/dba6522b-7de9-403f-80f5-783cb90d4647%40app.fastmail.com); `maintenance-fit-2`</sub>

### 3. shared_buffers_numa does not follow NUMA limits set by numactl or systemd

An operator limits the instance to node 0 with numactl or systemd NUMAMask, or runs on a host with a CPU-less memory node. With shared_buffers_numa=on, the postmaster calls mbind() for every configured node. Startup then fails with SIGBUS, or shared buffers land on slow memory the operator had excluded.

Message #104 reproduces the SIGBUS on a CPU-less node and notes that the code counts nodes as 0..numa_num_configured_nodes() and ignores sched_getaffinity() and systemd masks. Message #105 agrees that the server should respect a NUMA limit the operator already set. Today numactl and systemd are the standard way to place a Postgres instance on NUMA hardware, so a setting that overrides them breaks deployments that work now. Message #97 suggested a FATAL exit when huge pages run short on a node, but the proposal does not show one.

**Question:** How does the series find the allowed node set, including non-contiguous, CPU-less, or masked nodes, and what does the server report when it cannot honour it?

<sub>fixable, level 2, design; messages [#97](https://www.postgresql.org/message-id/CAKZiRmzumFJ8%2BBqz5mfX%2BQGkrKEaH32zz7kPgEShHWXE4QzyQw%40mail.gmail.com), [#104](https://www.postgresql.org/message-id/CAKZiRmzT1M1fEFf5fRq2Qg3Vz0wLbdY%2BuNdF-0ORhDTUOUVhcA%40mail.gmail.com), [#105](https://www.postgresql.org/message-id/f78dfe7d-4a37-40ec-8e60-c2cc47c48364%40app.fastmail.com); `standards-compat-3`</sub>

### 4. Huge page shortage on one node can surface as SIGBUS after startup

An operator sizes nr_hugepages from shared_memory_size_in_huge_pages, and the kernel gives one node fewer pages than its partition needs. The server starts. Later a backend first touches a buffer bound to that node and gets SIGBUS, and the postmaster resets every session and runs crash recovery.

Message #94 reports that earlier versions crashed "much later" when one node ran out of huge pages. Message #75 leaves MAP_POPULATE commented out and still expects a small chance of SIGBUS. Message #104 shows the same failure on a CPU-less node. A FATAL exit at startup was suggested in message #97, but the material does not show it. A standby or failover target with a different per-node huge page layout would meet this at the worst moment.

**Question:** How does the server detect at startup that every bound partition can be backed on its node, and fail cleanly instead of with a later SIGBUS?

<sub>fixable, level 2, design; messages [#75](https://www.postgresql.org/message-id/5bc6d309-61fe-4d8b-9e1f-a2961564a559%40vondra.me), [#94](https://www.postgresql.org/message-id/334f7644-6716-4e63-a7c4-13c5cbcdd210%40vondra.me), [#97](https://www.postgresql.org/message-id/CAKZiRmzumFJ8%2BBqz5mfX%2BQGkrKEaH32zz7kPgEShHWXE4QzyQw%40mail.gmail.com), [#104](https://www.postgresql.org/message-id/CAKZiRmzT1M1fEFf5fRq2Qg3Vz0wLbdY%2BuNdF-0ORhDTUOUVhcA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 5. Replacing usage_count for all users lacks shown user benefit

Every installation, NUMA or not, gets a one-bit second-chance evictor with probation admission and bgwriter changes. Users and tools that read pg_buffercache usagecount find that it now reports only 0 or 1. A workload tuned under the 0..5 policy loses hit ratio after upgrade, and the user finds out only from a production regression.

In message #108, the evictor's own author measured about 1% more index leaf pages resident and TPS inside the noise. He concluded it was "not worth the potential risks" and said he would drop it unless someone else supported it. No one in the visible thread did, yet message #109 merges a broader version (0004-0007) ahead of the partitioning work. The main stated reason in message #111 is to fix a worst-case allocation regression that partitioning itself introduces, so this does not show demand for the evictor change on its own merits.

**Question:** Which workloads, outside fixing partitioning's own tail latency, gain measurably from the one-bit evictor, and who other than its author has asked for it?

<sub>fixable, level 1b, desirability; messages [#107](https://www.postgresql.org/message-id/dba6522b-7de9-403f-80f5-783cb90d4647%40app.fastmail.com), [#108](https://www.postgresql.org/message-id/z5ln7bfyLp7MD3StPXOgrDAP4mnG4Aj1ZWWkmBENrn7lU43ZFWyGBfAzh3UYQFhYwm0Uj4hwDuc0BimjTVCHpc5RxUvthDWuH9pF44AYFaY%3D%40burd.me), [#109](https://www.postgresql.org/message-id/NzRAGLGvd0CoKX6vm7BEWJBEVi-apddzrNoRSzV83nScunblHKAR0Wz0xAOz0M_yFdMtauzUIdCuFR7En_xMbapM3ECeqM34AIv3Z9uy32A%3D%40burd.me), [#111](https://www.postgresql.org/message-id/8ylrXvoYwIDHkJUNzxTPpIFdkttaknqyUNpjOq9g05OQxCfgFE_Egkta_Z7rdso8MqQilfKRKtXHTIPg5hKfI2DXQPH1_HnZ4VG4VoC-zPg%3D%40burd.me); `demand-1`</sub>

## Questions for the author

1. Can the partitioning patches stand on the existing 0..5 usage count with their own bound on one allocation, so that the second-chance evictor becomes a separate proposal?
2. With every NUMA option at its default on a non-NUMA machine, does a single client scanning a relation larger than one partition perform like master?
3. How does the series find the allowed node set with CPU-less nodes, non-contiguous node IDs, or a mask set by numactl or systemd?
4. How does the server detect at startup that each bound partition has enough huge pages on its node, and fail with a FATAL message instead of a later SIGBUS?
5. Which of the new GUCs and pg_buffercache_set_weights() are proposed for commit as supported settings, and which are for development only?

## Touch points

- **WAL / replication / standby:** see objection replication-ondisk-1: Huge page shortage on one node can surface as SIGBUS after startup
- **ABI / extension API:** The series changes buf_internals.h and bufmgr.h, and pg_buffercache usagecount now reports only 0 or 1.
- **new GUC:** The series adds shmem_populate, shmem_interleave, shared_buffers_numa, three clocksweep GUCs and debug_numa_node.
- **planner behaviour change:** see objection maintenance-fit-2: Partitioned clock-sweep costs users who never enable NUMA
- **security / privileges:** pg_buffercache_set_weights() changes allocation weights for all backends, and the proposal does not say who may call it.
- **maintenance burden ("who maintains this in 10 years"):** The series adds partitioned clock-sweep, balancing, mbind-based placement and a new eviction policy in the buffer manager.
- **performance on the hot path:** Buffer allocation in freelist.c and bufmgr.c changes for every installation, with a minimum of 4 partitions without NUMA.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection standards-compat-3: shared_buffers_numa does not follow NUMA limits set by numactl or systemd

## Who would commit it, and when

- Owner: qualified committer engaged. Tomas Vondra wrote the base series and takes part in the thread. The owner must know the buffer manager and Linux NUMA and huge page behaviour.
- Priority: fix design first; audience a sizeable minority; review cost high. The target is large multi-node servers, and the series changes buffer allocation for everyone and adds kernel-dependent startup code. A committer is engaged, but the evictor bundling, the single-client regression and the node set handling need answers before code review.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 8 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 6.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
