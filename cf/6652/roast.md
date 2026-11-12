# Design pre-review: Resizable shared memory structures

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6652](https://commitfest.postgresql.org/patch/6652/); [thread](https://www.postgresql.org/message-id/flat/CAExHW5vM1bneLYfg0wGeAa%3D52UiJ3z4vKd3AJ72X8Fw6k3KKrg%40mail.gmail.com), 93 of 93 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection to the goal survived, and the thread shows lasting demand for online shared_buffers resizing. The surviving objections are fixable design points at level 2. The decisive ones are prior-attempts-1 (what startup reserves when huge pages are in use) and replication-ondisk-1 (protections in backends forked after a resize), which the commit message itself lists as open. Confidence is medium because only 3 of 93 messages were read directly.

Not checked: The code was not read, so the real startup behaviour with huge pages and the placement of protection calls at backend start could not be checked. Most of the thread (90 of 93 messages) was seen only through the objections' citations.

## Prior attempts

- [Identify huge pages accessibility using madvise](https://www.postgresql.org/message-id/flat/20240413162255.56xzlbhoolw2vyqv@ddolgov.remote.csb) (related; Withdrawn, 2024-11-12). "Ah good, it looks like the issue was addressed properly in the kernel then, and we don't need the workaround your patch proposes anymore." ([message](https://www.postgresql.org/message-id/3fc1ea14-af2b-4b6e-9309-1557190d1591%40eisentraut.org)) Different now: P14 probed huge page availability with madvise at startup. This proposal maps with MAP_NORESERVE, so the reservation-time check that made P14 unnecessary no longer covers the growth range.
- [Detect shared buffer access violations using mprotect()](https://www.postgresql.org/message-id/flat/CANXE4Te4G0TGq6cr0-TvwP0H4BNiK_-hB5gHe8mF+nz0mcYfMQ@mail.gmail.com) (related; Returned with feedback, 2019-02-03). "It'd be better to set things up so that the patch adds support for catching bad accesses with either valgrind or mprotect, according to compile options." ([message](https://www.postgresql.org/message-id/22918.1547427152%40sss.pgh.pa.us)) Different now: P10 was a compile-time option that used mprotect on shared buffers to catch access-rule violations. Here ShmemProtectStruct() guards only the unused range of a resizable structure, and the thread treats it as optional.

## Objections a reviewer is likely to raise

### 1. MAP_NORESERVE removes the huge page reservation check at startup

A DBA runs with huge_pages = try and sizes the hugetlb pool for shared_memory_initial_size. With MAP_NORESERVE the mmap succeeds, so there is no fallback to normal pages. Buffer blocks are not touched at startup, so a backend that first faults one of them when the pool is short gets SIGBUS, and the postmaster restarts the cluster.

P14 was withdrawn because the kernel charges hugetlb at reservation time, so a short pool makes mmap fail up front and huge_pages = try falls back. MAP_NORESERVE gives up that reservation for the whole mapping. Message #5 notes that unpopulated huge pages can SIGBUS, but message #7 applies MADV_POPULATE_WRITE only during resize, not at startup. The runtime.sgml hunk says the maximum huge page count must be in the pool for startup, while message #88 says only the initial memory is needed, so the proposal gives two answers.

Precedent: [Identify huge pages accessibility using madvise](https://www.postgresql.org/message-id/flat/20240413162255.56xzlbhoolw2vyqv@ddolgov.remote.csb) (Withdrawn, 2024-11-12)

**Question:** With huge pages, what does startup reserve or populate, and how does a short pool fail at startup without a later SIGBUS in a backend?

<sub>fixable, level 2, design; messages [#5](https://www.postgresql.org/message-id/mlsruptoxgm2nqtdfyfsowjklzxl5zltsjb3y5bmywtigm474l%405tsonk4t3kia), [#7](https://www.postgresql.org/message-id/CAExHW5vz%2BPUHHUuzGRwtyx-mPLQk3nCZXxrFqnruRadEFrO5Xg%40mail.gmail.com), [#88](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Per-backend mprotect state has no defined sync with postmaster and new backends

A subsystem grows a resizable structure and uses a ProcSignalBarrier so running backends call ShmemProtectStruct(). The postmaster does not take part in the barrier and keeps its old protections, so a backend forked afterwards inherits them. When that backend touches the newly grown range it gets SIGSEGV, and the postmaster starts a crash restart of the whole cluster.

The commit message in message #88 says that in non-EXEC_BACKEND builds new backends inherit stale protections from the postmaster, and lists this as an open discussion point. The proposed answer is to reapply protections next to InitLocalDataChecksumState(), which ties shmem.c to ProcSignalBarrier and adds backend startup cost. Message #73 reports that Andres sees protection as optional while Matthias wants it in shmem.c. ShmemProtectStruct() is in the series now, so the cross-process rule has to be settled before an extension relies on it.

**Question:** How does a backend forked after a resize get protections that match the current size, with no window where a barrier is missed and no help from the postmaster?

<sub>fixable, level 2, design; messages [#73](https://www.postgresql.org/message-id/CAExHW5uLau-i0T-ybPa%3Dg5FLQo6OR1U9MR3GNijm6TDv12aLPw%40mail.gmail.com), [#77](https://www.postgresql.org/message-id/CAExHW5v5muT_SKV2NCxxVmvC%3D_38Rw0aiv-wU4CGzHaBCRYzqA%40mail.gmail.com), [#84](https://www.postgresql.org/message-id/CAExHW5uDJdmfUVX%3DmeyecbY2sGKq07Y33NQ68Wv2-ktaag%3D9qg%40mail.gmail.com), [#88](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 3. API lands without a core consumer to prove its shape

The patch commits ShmemResizeStruct(), new pg_shmem_allocations columns and new GUCs, and the shared_buffers work then needs hole punching or a different barrier hookup at startup. Extensions that adopted the API in the meantime must change twice. Callers can change, but the view columns and GUC names are hard to take back.

The only user in the patch is a test module, and shared_buffers resizing is in a separate thread (message #88). Heikki asks in message #79 whether resizing only at the end is enough. In message #82 the author suggests different calls, ShmemFreeStructPart() and ShmemAllocStructPart(). The Databricks patch in message #86 makes its own coordination choices. It is not yet clear the interface fits even its first intended user.

**Question:** Can the series include, or be committed together with, a converted core subsystem such as buffer descriptors that uses this API unchanged?

<sub>fixable, level 2, ABI / extension API; messages [#79](https://www.postgresql.org/message-id/b15a8fb8-5b59-4349-859c-07e7f070a21e%40iki.fi), [#82](https://www.postgresql.org/message-id/CAExHW5udO-ppz4Z6hrLPO0%2BovL8byPTH3zXAPjaOy4BH3RTqyQ%40mail.gmail.com), [#86](https://www.postgresql.org/message-id/CAM1e6U5XDwKYZo6Jj3yD3xpCB4qkhRSQn8upauHt%3DWhEbK9VZA%40mail.gmail.com), [#88](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com); `maintenance-fit-3`</sub>

### 4. Removing shared_memory_size and shared_memory_size_in_huge_pages breaks existing scripts

An operator's provisioning script or an operator image runs `postgres -D $PGDATA -C shared_memory_size_in_huge_pages` to size vm.nr_hugepages before startup. After the upgrade the parameter no longer exists, so `postgres -C` fails. The script aborts, or it sets zero huge pages and the server falls back to normal pages. The operator finds out at deploy time, or later from lower performance.

The patch renames both GUCs and the matching docs, and runtime.sgml has documented `-C shared_memory_size_in_huge_pages` as the way to size huge pages for years. The patch does not keep the old names as aliases and gives no deprecation period. No core subsystem uses resizable structures in this patch, so initial, minimum and maximum report the same value and users get no visible benefit yet. The old names could map to the maximum size, which is the figure the new docs tell users to provision.

**Question:** Why can shared_memory_size and shared_memory_size_in_huge_pages not stay as aliases for the maximum values, so that existing `postgres -C` users keep working?

<sub>fixable, level 2, new GUC; messages [#9](https://www.postgresql.org/message-id/CAExHW5so6VSxBC-1V%3D35229Z1%2Bdw5vhw8HxHg9ry7UzceKcXzA%40mail.gmail.com), [#88](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com); `standards-compat-1`</sub>

### 5. Access beyond current size is silent unless each caller opts in

A later patch loops over a resizable array using a stale size after a shrink. Reads return zeros and writes quietly allocate pages again, so memory is not returned and the data looks valid. Nobody notices until a user sees RSS that does not shrink, or wrong results.

Message #3 states that a write past the current size allocates memory silently and a read returns zeros. Protection depends on each subsystem calling ShmemProtectStruct() after every resize. Andres and Matthias disagree on whether that should be optional (message #73). The commit message leaves the non-EXEC_BACKEND startup case open, so the safety net is incomplete even for callers that opt in.

**Question:** Can protection be on by default, at least in assert-enabled builds, so that a stale size fails loudly in every backend?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/CAExHW5s9Vp%2B-vJi020UJ%2BotyccBBo7eT1g6bttdRKL6HAvscyQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/mlsruptoxgm2nqtdfyfsowjklzxl5zltsjb3y5bmywtigm474l%405tsonk4t3kia), [#73](https://www.postgresql.org/message-id/CAExHW5uLau-i0T-ybPa%3Dg5FLQo6OR1U9MR3GNijm6TDv12aLPw%40mail.gmail.com), [#88](https://www.postgresql.org/message-id/CAExHW5tRkt2c%2Bd4fi8USqyc7402J2q3B2ofFfZ_Nfe%2B8NyK_Zw%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. With huge pages, what does startup reserve or populate, and how does a short pool fail at startup instead of raising SIGBUS in a backend later?
2. Must the huge page pool hold the maximum size at startup, as the runtime.sgml hunk says, or only the initial size, as the commit message says?
3. How does a backend forked after a resize get protections that match the current size, with no window where it misses a ProcSignalBarrier?
4. Can a converted core subsystem, such as buffer descriptors, be committed together with this API and use it unchanged?
5. Why can shared_memory_size and shared_memory_size_in_huge_pages not stay as aliases for the maximum values, so that existing `postgres -C` scripts keep working?

## Touch points

- **catalog changes:** pg_shmem_allocations gets minimum_size, maximum_size and reserved_space columns, and the patch edits pg_proc.dat.
- **ABI / extension API:** ShmemStructOpts gets maximum_size and minimum_size, and extensions can call ShmemResizeStruct() and ShmemProtectStruct().
- **new GUC:** Six read-only GUCs replace shared_memory_size and shared_memory_size_in_huge_pages, and have_resizable_shmem is new.
- **maintenance burden ("who maintains this in 10 years"):** The feature works only on Linux with mmap shared memory and depends on MAP_NORESERVE, MADV_REMOVE and MADV_POPULATE_WRITE behaviour, including with huge pages.
- **performance on the hot path:** The open item on applying protections at backend start adds work to every backend startup, and the thread gives no cost figures.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas committed the shmem registration API this patch builds on and takes part in the thread, as does Andres Freund. The owner needs to know shmem.c, the Linux mmap and huge page code, and backend startup.
- Priority: fix design first; audience a sizeable minority; review cost high. Online shared_buffers resizing has shown demand, but this patch has only a test module as caller and leaves huge page startup and backend protections open. The patch is about 129 KiB and touches shmem.c, sysv_shmem.c, backend startup and the GUC tables.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
