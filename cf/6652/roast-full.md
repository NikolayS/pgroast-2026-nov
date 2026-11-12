# Design pre-review: Resizable shared memory structures

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6652](https://commitfest.postgresql.org/patch/6652/) (Needs review); authors: Ashutosh Bapat (ashutoshbapat)
- Thread: [Better shared data structure management and resizable shared data structures](https://www.postgresql.org/message-id/flat/CAExHW5vM1bneLYfg0wGeAa%3D52UiJ3z4vKd3AJ72X8Fw6k3KKrg%40mail.gmail.com), started 2026-02-13
- Messages read: 93 of 93
- Patch prose read: v20260724-0006-Resizable-shared-memory-structures.patch, 0002-Get-rid-of-global-shared-memory-pointer-mac-20260213.patch, 0001-wip-Introduce-a-new-way-of-registering-shar-20260213.patch, 0003-WIP-Resizable-shared-memory-structures-20260213.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection to the goal survived, and the thread shows lasting demand for online shared_buffers resizing. The surviving objections are fixable design points at level 2. The decisive ones are prior-attempts-1 (what startup reserves when huge pages are in use) and replication-ondisk-1 (protections in backends forked after a resize), which the commit message itself lists as open. Confidence is medium because only 3 of 93 messages were read directly.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the real startup behaviour with huge pages and the placement of protection calls at backend start could not be checked. Most of the thread (90 of 93 messages) was seen only through the objections' citations.

## The proposal, restated

**Problem.** Shared memory structures have a fixed size set at postmaster startup, so changing them needs a restart. The work started from resizing shared_buffers online, and the author wants a general facility for core subsystems and extensions. The thread names buffer blocks, buffer descriptors and checkpointer data as likely users.

**Proposed design.** A structure requested with ShmemStructOpts.maximum_size gets address space for its maximum size in the main shared memory segment, which is mapped with MAP_NORESERVE, but only its initial size is backed by memory. ShmemResizeStruct() frees pages with MADV_REMOVE and allocates them with MADV_POPULATE_WRITE, so the start address never changes. The caller must coordinate access across backends, for example with a ProcSignalBarrier; optional ShmemProtectStruct() uses mprotect on the unused range.

**User-visible surface.**
- C API: ShmemStructOpts.maximum_size and .minimum_size, ShmemResizeStruct(), ShmemProtectStruct().
- pg_shmem_allocations gets new columns minimum_size, maximum_size and reserved_space; allocated_size changes meaning for resizable structures.
- GUC shared_memory_size is replaced by shared_memory_initial_size, shared_memory_minimum_size and shared_memory_maximum_size.
- GUC shared_memory_size_in_huge_pages is replaced by the initial, minimum and maximum _in_huge_pages variants.
- New read-only GUC have_resizable_shmem (Message 73 calls it have_resizable_shared_memory) reports whether the server supports the feature.

**What the author claims.**
- One mapping with madvise needs much less code than one segment per resizable structure.
- Most users run Linux with mmap shared memory, so a Linux-only feature helps most of them.
- Struct layout and functions are the same with or without HAVE_RESIZABLE_SHMEM, so extension builds stay ABI compatible.
- Memory for future growth is not needed at startup, because the mapping uses MAP_NORESERVE.
- The design is close to what a threaded server would need, so it is not a dead end (Andres Freund).

**Thread state.** Heikki Linnakangas committed the new shmem registration API in April, and the resizable patch now sits alone in its third CommitFest. In August Palak Chaturvedi sent test portability fixes that Ashutosh partly accepted; mprotect handling, hole punching in the middle of a structure, and Windows and BSD support are still open.

## 2. Prior attempts

### Identify huge pages accessibility using madvise (related)

- **Link:** [https://commitfest.postgresql.org/patch/4937/](https://commitfest.postgresql.org/patch/4937/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20240413162255.56xzlbhoolw2vyqv@ddolgov.remote.csb](https://www.postgresql.org/message-id/flat/20240413162255.56xzlbhoolw2vyqv@ddolgov.remote.csb)
- **Date:** decided 2024-11-12; started 2024-04-13
- **Outcome:** Withdrawn (cf)
- **Who objected:** Dmitry Dolgov, Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author found that since Linux 5.7 cgroups also limit hugetlb at reservation time (hugetlb.<size>.rsvd.limit_in_bytes), so a correct setup makes the mmap fail up front. Peter Eisentraut agreed that the kernel had addressed the issue and that the workaround was no longer needed. The entry was withdrawn, and users on older kernels or with orchestration tools that do not set the rsvd limit can apply the patch themselves.
- **Reason (quote):** "Ah good, it looks like the issue was addressed properly in the kernel then, and we don't need the workaround your patch proposes anymore." ([message](https://www.postgresql.org/message-id/3fc1ea14-af2b-4b6e-9309-1557190d1591%40eisentraut.org))
- **What is different this time:** P14 probed huge page availability with madvise at startup. This proposal maps with MAP_NORESERVE, so the reservation-time check that made P14 unnecessary no longer covers the growth range.

### Detect shared buffer access violations using mprotect() (related)

- **Link:** [https://commitfest.postgresql.org/patch/1765/](https://commitfest.postgresql.org/patch/1765/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CANXE4Te4G0TGq6cr0-TvwP0H4BNiK_-hB5gHe8mF+nz0mcYfMQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CANXE4Te4G0TGq6cr0-TvwP0H4BNiK_-hB5gHe8mF+nz0mcYfMQ@mail.gmail.com)
- **Date:** decided 2019-02-03; started 2018-07-11
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane asked that the patch be restructured so that bad accesses can be caught with either Valgrind or mprotect depending on compile options, using macros such as ProtectBuffer() that expand to nothing when the feature is disabled instead of inline #ifdef blocks. He also raised the concern that buffers are not reliably aligned on system page boundaries. The author did not address this feedback before the commitfest ended, so Andres Freund marked the entry returned with feedback.
- **Reason (quote):** "It'd be better to set things up so that the patch adds support for catching bad accesses with either valgrind or mprotect, according to compile options." ([message](https://www.postgresql.org/message-id/22918.1547427152%40sss.pgh.pa.us))
- **What is different this time:** P10 was a compile-time option that used mprotect on shared buffers to catch access-rule violations. Here ShmemProtectStruct() guards only the unused range of a resizable structure, and the thread treats it as optional.


## 3. Objections a reviewer is likely to raise

### 1. MAP_NORESERVE removes the huge page reservation check at startup

fixable · level 2, rubric item 7 · design, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A DBA runs with huge_pages = try and sizes the hugetlb pool for shared_memory_initial_size. With MAP_NORESERVE the mmap succeeds, so there is no fallback to normal pages. Buffer blocks are not touched at startup, so a backend that first faults one of them when the pool is short gets SIGBUS, and the postmaster restarts the cluster.

P14 was withdrawn because the kernel charges hugetlb at reservation time, so a short pool makes mmap fail up front and huge_pages = try falls back. MAP_NORESERVE gives up that reservation for the whole mapping. Message #5 notes that unpopulated huge pages can SIGBUS, but message #7 applies MADV_POPULATE_WRITE only during resize, not at startup. The runtime.sgml hunk says the maximum huge page count must be in the pool for startup, while message #88 says only the initial memory is needed, so the proposal gives two answers.

**Precedents.**
- [Identify huge pages accessibility using madvise](https://commitfest.postgresql.org/patch/4937/) (Withdrawn, 2024-11-12); thread: [link](https://www.postgresql.org/message-id/flat/20240413162255.56xzlbhoolw2vyqv@ddolgov.remote.csb)

Thread: [#5 (Andres Freund, 2026-02-16)](https://www.postgresql.org/message-id/mlsruptoxgm2nqtdfyfsowjklzxl5zltsjb3y5bmywtigm474l%405tsonk4t3kia), [#7 (Ashutosh Bapat, 2026-02-18)](https://www.postgresql.org/message-id/CAExHW5vz%2BPUHHUuzGRwtyx-mPLQk3nCZXxrFqnruRadEFrO5Xg%40mail.gmail.com), [#88 (Ashutosh Bapat, 2026-07-24)](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com)

**Question:** With huge pages, what does startup reserve or populate, and how does a short pool fail at startup without a later SIGBUS in a backend?

Counter-check: held. Message #5 says unpopulated huge pages can raise SIGBUS, and message #7 populates only on resize. The runtime.sgml hunk and message #88 disagree on what the pool must hold at startup.

### 2. Per-backend mprotect state has no defined sync with postmaster and new backends

fixable · level 2, rubric item 14 · design, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A subsystem grows a resizable structure and uses a ProcSignalBarrier so running backends call ShmemProtectStruct(). The postmaster does not take part in the barrier and keeps its old protections, so a backend forked afterwards inherits them. When that backend touches the newly grown range it gets SIGSEGV, and the postmaster starts a crash restart of the whole cluster.

The commit message in message #88 says that in non-EXEC_BACKEND builds new backends inherit stale protections from the postmaster, and lists this as an open discussion point. The proposed answer is to reapply protections next to InitLocalDataChecksumState(), which ties shmem.c to ProcSignalBarrier and adds backend startup cost. Message #73 reports that Andres sees protection as optional while Matthias wants it in shmem.c. ShmemProtectStruct() is in the series now, so the cross-process rule has to be settled before an extension relies on it.

Thread: [#73 (Ashutosh Bapat, 2026-04-07)](https://www.postgresql.org/message-id/CAExHW5uLau-i0T-ybPa%3Dg5FLQo6OR1U9MR3GNijm6TDv12aLPw%40mail.gmail.com), [#77 (Ashutosh Bapat, 2026-04-07)](https://www.postgresql.org/message-id/CAExHW5v5muT_SKV2NCxxVmvC%3D_38Rw0aiv-wU4CGzHaBCRYzqA%40mail.gmail.com), [#84 (Ashutosh Bapat, 2026-04-21)](https://www.postgresql.org/message-id/CAExHW5uDJdmfUVX%3DmeyecbY2sGKq07Y33NQ68Wv2-ktaag%3D9qg%40mail.gmail.com), [#88 (Ashutosh Bapat, 2026-07-24)](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com)

**Question:** How does a backend forked after a resize get protections that match the current size, with no window where a barrier is missed and no help from the postmaster?

Counter-check: held. The commit message in message #88 lists initial protections in non-EXEC_BACKEND backends as an open discussion point. It proposes reapplying them at backend start but leaves the startup cost and the barrier coupling unsettled.

### 3. API lands without a core consumer to prove its shape

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

The patch commits ShmemResizeStruct(), new pg_shmem_allocations columns and new GUCs, and the shared_buffers work then needs hole punching or a different barrier hookup at startup. Extensions that adopted the API in the meantime must change twice. Callers can change, but the view columns and GUC names are hard to take back.

The only user in the patch is a test module, and shared_buffers resizing is in a separate thread (message #88). Heikki asks in message #79 whether resizing only at the end is enough. In message #82 the author suggests different calls, ShmemFreeStructPart() and ShmemAllocStructPart(). The Databricks patch in message #86 makes its own coordination choices. It is not yet clear the interface fits even its first intended user.

Thread: [#79 (Heikki Linnakangas, 2026-04-07)](https://www.postgresql.org/message-id/b15a8fb8-5b59-4349-859c-07e7f070a21e%40iki.fi), [#82 (Ashutosh Bapat, 2026-04-08)](https://www.postgresql.org/message-id/CAExHW5udO-ppz4Z6hrLPO0%2BovL8byPTH3zXAPjaOy4BH3RTqyQ%40mail.gmail.com), [#86 (Haoyu Huang, 2026-06-12)](https://www.postgresql.org/message-id/CAM1e6U5XDwKYZo6Jj3yD3xpCB4qkhRSQn8upauHt%3DWhEbK9VZA%40mail.gmail.com), [#88 (Ashutosh Bapat, 2026-07-24)](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com)

**Question:** Can the series include, or be committed together with, a converted core subsystem such as buffer descriptors that uses this API unchanged?

Counter-check: held. Message #88 says the patch comes from the shared_buffers series, and message #82 says buffer structures resize only at the end, so a consumer exists. This entry still has only a test module as caller, so the request to commit them together stands.

### 4. Removing shared_memory_size and shared_memory_size_in_huge_pages breaks existing scripts

fixable · level 2, rubric item 12 · new GUC, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An operator's provisioning script or an operator image runs `postgres -D $PGDATA -C shared_memory_size_in_huge_pages` to size vm.nr_hugepages before startup. After the upgrade the parameter no longer exists, so `postgres -C` fails. The script aborts, or it sets zero huge pages and the server falls back to normal pages. The operator finds out at deploy time, or later from lower performance.

The patch renames both GUCs and the matching docs, and runtime.sgml has documented `-C shared_memory_size_in_huge_pages` as the way to size huge pages for years. The patch does not keep the old names as aliases and gives no deprecation period. No core subsystem uses resizable structures in this patch, so initial, minimum and maximum report the same value and users get no visible benefit yet. The old names could map to the maximum size, which is the figure the new docs tell users to provision.

Thread: [#9 (Ashutosh Bapat, 2026-02-23)](https://www.postgresql.org/message-id/CAExHW5so6VSxBC-1V%3D35229Z1%2Bdw5vhw8HxHg9ry7UzceKcXzA%40mail.gmail.com), [#88 (Ashutosh Bapat, 2026-07-24)](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com)

**Question:** Why can shared_memory_size and shared_memory_size_in_huge_pages not stay as aliases for the maximum values, so that existing `postgres -C` users keep working?

Counter-check: held. Message #9 explains why one value no longer describes the segment, but the proposal does not say why the old names cannot stay as aliases. runtime.sgml documents `postgres -C shared_memory_size_in_huge_pages` today, so the break is concrete.

### 5. Access beyond current size is silent unless each caller opts in

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A later patch loops over a resizable array using a stale size after a shrink. Reads return zeros and writes quietly allocate pages again, so memory is not returned and the data looks valid. Nobody notices until a user sees RSS that does not shrink, or wrong results.

Message #3 states that a write past the current size allocates memory silently and a read returns zeros. Protection depends on each subsystem calling ShmemProtectStruct() after every resize. Andres and Matthias disagree on whether that should be optional (message #73). The commit message leaves the non-EXEC_BACKEND startup case open, so the safety net is incomplete even for callers that opt in.

Thread: [#3 (Ashutosh Bapat, 2026-02-16)](https://www.postgresql.org/message-id/CAExHW5s9Vp%2B-vJi020UJ%2BotyccBBo7eT1g6bttdRKL6HAvscyQ%40mail.gmail.com), [#5 (Andres Freund, 2026-02-16)](https://www.postgresql.org/message-id/mlsruptoxgm2nqtdfyfsowjklzxl5zltsjb3y5bmywtigm474l%405tsonk4t3kia), [#73 (Ashutosh Bapat, 2026-04-07)](https://www.postgresql.org/message-id/CAExHW5uLau-i0T-ybPa%3Dg5FLQo6OR1U9MR3GNijm6TDv12aLPw%40mail.gmail.com), [#88 (Ashutosh Bapat, 2026-07-24)](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com)

**Question:** Can protection be on by default, at least in assert-enabled builds, so that a stale size fails loudly in every backend?

Counter-check: held. Message #3 confirms that reads past the current size return zeros and writes allocate silently. The thread weighed this (messages #5, #73) without agreeing, and the xfunc.sgml text leaves protection after a resize to each caller.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | pg_shmem_allocations gets minimum_size, maximum_size and reserved_space columns, and the patch edits pg_proc.dat. |
| ABI / extension API | **yes** | ShmemStructOpts gets maximum_size and minimum_size, and extensions can call ShmemResizeStruct() and ShmemProtectStruct(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Six read-only GUCs replace shared_memory_size and shared_memory_size_in_huge_pages, and have_resizable_shmem is new. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The feature works only on Linux with mmap shared memory and depends on MAP_NORESERVE, MADV_REMOVE and MADV_POPULATE_WRITE behaviour, including with huge pages. |
| performance on the hot path | **yes** | The open item on applying protections at backend start adds work to every backend startup, and the thread gives no cost figures. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. With huge pages, what does startup reserve or populate, and how does a short pool fail at startup instead of raising SIGBUS in a backend later?
2. Must the huge page pool hold the maximum size at startup, as the runtime.sgml hunk says, or only the initial size, as the commit message says?
3. How does a backend forked after a resize get protections that match the current size, with no window where it misses a ProcSignalBarrier?
4. Can a converted core subsystem, such as buffer descriptors, be committed together with this API and use it unchanged?
5. Why can shared_memory_size and shared_memory_size_in_huge_pages not stay as aliases for the maximum values, so that existing `postgres -C` scripts keep working?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas committed the shmem registration API this patch builds on and takes part in the thread, as does Andres Freund. The owner needs to know shmem.c, the Linux mmap and huge page code, and backend startup.
- **Liability:** The owner signs for Linux-specific madvise and MAP_NORESERVE behaviour with huge pages, for a C API that extensions will adopt, and for renamed GUCs and new view columns. The names and columns are hard to change after release.
- **Author follow-up:** The author has kept the entry alive across three CommitFests, rebased it, and partly accepted test portability fixes from another contributor in August.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Online shared_buffers resizing has shown demand, but this patch has only a test module as caller and leaves huge page startup and backend protections open. The patch is about 129 KiB and touches shmem.c, sysv_shmem.c, backend startup and the GUC tables.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Removing shared_memory_size_in_huge_pages breaks huge page provisioning scripts. Refuted (not specific): Duplicate of standards-compat-1, which states the same script breakage and the alias question more fully.
- `maintenance-fit-4` Every resizable subsystem keeps a fixed-size path for other platforms. Refuted (no failure scenario): The fallback is the existing fixed-size code, and message #72 keeps structs and functions the same with or without HAVE_RESIZABLE_SHMEM. CFbot and the buildfarm cover the other platforms, and SysV shared memory turns the feature off on Linux for local tests.
- `already-exists-1` Extensions get a second, Linux-only path for growable shared memory. Refuted (already handled): Messages #1 and #54 give the reason for a new mechanism: structures in the main segment need a stable start address, which DSA and dshash do not provide. The remaining request is a documentation pointer to DSA, which is outside levels 1-2.
- `demand-1` No in-tree user of the facility ships with this patch. Refuted (not specific): Duplicate of maintenance-fit-3. As a demand objection it does not hold, because the thread shows lasting demand for shared_buffers resizing from several parties (messages #1, #86).
- `demand-2` Renaming shared_memory_size GUCs costs all users for a Linux-only feature. Refuted (not specific): Duplicate of standards-compat-1, which covers the same GUC rename and the same alias question.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent is the same idea. P14 is the only one whose recorded reason bears on this design.
- standards-compat: 1 objection(s). Only one objection from this angle. The new pg_shmem_allocations columns are additive, and allocated_size changes meaning only for resizable structures, which no existing structure is, so existing queries keep their results.
- replication-ondisk: 1 objection(s). The proposal has no on-disk, WAL, replication or wire-protocol state. The ABI concern about HAVE_RESIZABLE_SHMEM was settled in messages #72-73, and the struct is new in this release, so backpatching does not apply.
- security: 0 objection(s). The proposal adds only a C API for trusted server code and read-only GUCs, and no SQL-callable resize function outside the test module, so an unprivileged user gains no new action. The new pg_shmem_allocations columns keep the existing superuser and pg_read_all_stats restriction, and MADV_REMOVE returns zeroed pages, so freed contents do not leak. A later patch that exposes resizing through SQL, such as the pg_resize_shared_buffers() function in message #86, would need its own privilege review.
- maintenance-fit: 4 objection(s). Not applicable: four objections raised. Heikki and Andres engaged on the design, which lowers the concern about who owns the result.
- already-exists: 1 objection(s). For the core use case of resizing buffer blocks and descriptors in place, no existing feature works. DSM and DSA segments do not keep one stable address in every backend, so the only objection from this angle concerns extensions.
- demand: 2 objection(s). Demand for the end goal, online shared_buffers resizing, is shown: a production patch at Databricks (message #86) and several engaged committers. The objections cover the standalone facility and the cost of the GUC rename, not the goal itself.

### Precedents retrieved

- P1: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P2: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P3: [NUMA shared memory interleaving](https://commitfest.postgresql.org/patch/5703/) (cf, Returned with feedback, 2025-07-07) - used in the review; reason summarised from its thread
- P4: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P5: [WIP: Track relation size in shared memory](https://commitfest.postgresql.org/patch/2933/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [Initial mach based shared memory support.](https://commitfest.postgresql.org/patch/2843/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P7: [Revert recent change to RequestNamedLWLockTranche().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d814d7fc3d5257ae258b502229fc7ca97c97270a) (revert, Reverted, 2025-09-04) - judged not relevant
- P8: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P9: ["could not reattach to shared memory" on buildfarm member dory (design review sought)](https://commitfest.postgresql.org/patch/1777/) (cf, Rejected, 2018-09-29) - judged not relevant; reason summarised from its thread
- P10: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - used in the review; reason summarised from its thread
- P11: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - judged not relevant; reason summarised from its thread
- P12: [Remove investigative code for can't-reattach-to-shared-memory errors.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bcbf2346d69f6006f126044864dd9383d50d87b4) (revert, Partially reverted, 2018-05-01) - judged not relevant
- P13: [Allow shared buffer cache contents to affect index cost computations](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P14: [Identify huge pages accessibility using madvise](https://commitfest.postgresql.org/patch/4937/) (cf, Withdrawn, 2024-11-12) - used in the review; reason summarised from its thread
- P15: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant
- P16: [GUC-ify WALSender's MAX_SEND_SIZE constant](https://commitfest.postgresql.org/patch/4781/) (cf, Rejected, 2024-05-14) - judged not relevant
- P17: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant
- P18: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P19: [Reduce size of postgres.bki](https://commitfest.postgresql.org/patch/4544/) (cf, Returned with feedback, 2024-05-15) - judged not relevant
- P20: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/2496/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P21: [Revert previous patch to put the shared memory segment on win32 in the Global\ namespace, because it caused permission errors on a lot of platforms.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cc0222680229c4572354cf2b3907c7ff18ad685) (revert, Reverted, 2008-10-30) - judged not relevant
- P22: [Online checksums verification](https://commitfest.postgresql.org/patch/2369/) (cf, Returned with feedback, 2020-11-05) - judged not relevant
- P23: [guc-ify the formerly hard-coded MAX_SEND_SIZE to max_wal_send](https://commitfest.postgresql.org/patch/958/) (cf, Returned with feedback, 2017-03-28) - judged not relevant
- P24: [Revert "Add missing padding from MemoryChunk struct"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5495796ad12aae971137d184ff25add8f6b03968) (revert, Reverted, 2022-08-30) - judged not relevant

### Warnings

- 21 long messages in the middle of the thread were left out of the prompts

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 188 s; cost reported by the CLI: $8.16
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 102 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:58:58.706Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
