# Design pre-review: enhancing pg_basebackup speeds up to ~23Gbps (small fixes + io_uring/Direct I/O)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7142](https://commitfest.postgresql.org/patch/7142/) (Needs review); authors: Jakub Wartak (jakub.wartak.tomtom)
- Thread: [enhancing pg_basebackup speeds up to ~23Gbps (small fixes + io_uring/Direct I/O)](https://www.postgresql.org/message-id/flat/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com), started 2026-08-13
- Messages read: 15 of 15
- Patch prose read: v06112026-0001-pg_basebackup-rename-the-blackhole-backup-.patch, v06112026-0002-pg_basebackup-add-new-client-blackhole-ben.patch, v06112026-0004-basebackup-bump-SINK_BUFFER_LENGTH-to-256k.patch, v06112026-0005-Fix-re-reading-and-short-reads-races-in-ba.patch, v06112026-0003-pg_basebackup-report-average-data-transfer.patch, v06112026-0008-pg_basebackup-preallocate-extracted-files-.patch, v06112026-0006-pg_basebackup-issue-posix_fadvise-for-more.patch, v06112026-0009-libpq-pg_basebackup-add-PQgetCopyDataInter.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal, and a committer is already reviewing the server-side buffer and fadvise patches. The surviving objections are fixable design points on the client side: the exported libpq function (replication-ondisk-1), the `blackhole` rename (standards-compat-1), and Direct I/O on by default with no stated fallback (replication-ondisk-3, standards-compat-3). demand-1 asks for an io_uring measurement without MPTCP. Confidence is medium because only 3 of 15 messages are visible.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the short-read and truncation handling in basebackup.c, the buffer lifetime of the new libpq call, and the Direct I/O alignment and error paths could not be checked. Only 3 of 15 messages were visible, so later answers may exist.

## The proposal, restated

**Problem.** pg_basebackup tops out at about 10-15 Gbps because one core does all the work on each side, with CPU spent on small pread() calls, an extra libpq memcpy and buffered writes. Incremental backups read changed blocks one at a time, so kernel readahead over-reads scattered blocks. The users are operators backing up large clusters on fast networks and storage.

**Proposed design.** The series keeps the single-threaded, single-connection design. It adds benchmarking targets and a throughput message, raises SINK_BUFFER_LENGTH to 256 kB with short-read retry and a sendFile()/sendRelFile() split, and adds posix_fadvise hints (SEQUENTIAL for full, WILLNEED windows for incremental). On the client side it drops stdio, preallocates files, adds a no-copy libpq receive call, and adds io_uring Direct I/O writes.

**User-visible surface.**
- pg_basebackup --target value blackhole renamed to server-blackhole, also in the BASE_BACKUP protocol TARGET option docs.
- New pg_basebackup --target value client-blackhole that receives and discards the backup.
- Verbose "base backup completed" message now reports average MB/s.
- New exported libpq function PQgetCopyDataInternalBuf() (exports.txt entry 212).
- Environment variable PG_BASEBACKUP_NODIO to turn off Direct I/O; liburing becomes a client build dependency.

**What the author claims.**
- The full series with MPTCP gives about 1.85x on AWS (about 1610 to 2975 MB/s).
- A 256 kB sink buffer raises loopback throughput from about 3.1 to 5.5 GB/s.
- Removing the libpq copy takes pg_basebackup from 2.4 to about 5.5 GB/s on localhost without disk writes.
- Incremental fadvise prefetch is about 9.8x faster at 14% delta, falling to 1.2x at 57%, by avoiding wasted kernel readahead.
- Tar output with Direct I/O needs posix_fallocate(); without it throughput drops to 410 MB/s versus about 2400 MB/s.

**Thread state.** Heikki Linnakangas reviewed the buffer increase, found short-read and truncation races, and posted a refactor that Jakub merged into 0005; Gustavo William co-authors the incremental fadvise patch. Patches 0007-0011 (stdio, fallocate, libpq API, io_uring) have no review in the thread.

## 2. Prior attempts

### parallel backup (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2472/](https://commitfest.postgresql.org/patch/2472/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CADM=JehKgobEknb+_nab9179HzGj=9EiTzWMOd2mpqr_rifm0Q@mail.gmail.com](https://www.postgresql.org/message-id/flat/CADM=JehKgobEknb+_nab9179HzGj=9EiTzWMOd2mpqr_rifm0Q@mail.gmail.com)
- **Date:** decided 2020-07-06; started 2019-08-21
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P17 sped up base backup with parallel workers and connections. This proposal keeps the single-threaded, single-connection design and cuts per-byte CPU and I/O cost instead.


## 3. Objections a reviewer is likely to raise

### 1. New public libpq symbol for a single in-tree consumer

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A driver author finds PQgetCopyDataInternalBuf() in libpq-fe.h, keeps the returned pointer across a later libpq call, and gets COPY data overwritten when libpq refills or moves its input buffer. The corruption is silent. Once exports.txt entry 212 ships, the project cannot remove or change the function without an ABI break.

Patch 0009 adds the function to exports.txt and libpq-fe.h, with no libpq.sgml hunk. The returned buffer is valid only until the next call, which ties callers to libpq's internal buffer handling. The commit message names pg_basebackup as the only consumer, and patches 0007-0011 have no review in the thread. A committer would likely ask for a frontend-private route, or for a documented API that other COPY consumers would use.

Thread: [#1 (Jakub Wartak, 2026-08-13)](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com)

**Question:** Why must this be an exported libpq function instead of a frontend-private interface, and which buffer lifetime rules will the documentation promise?

Counter-check: held. The exports.txt hunk adds entry 212 and the patch has no libpq.sgml hunk. The commit message ties the returned data to libpq's internal buffer until the next call, and nobody in the thread reviewed this patch.

### 2. io_uring Direct I/O benefit shown only above single-flow TCP limits

fixable · level 1b, rubric item 4 · demand, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A packager adds liburing as a client build dependency and users get the Direct I/O path, but on links under about 15 Gbps they see no speedup. In run i2, without MPTCP, throughput fell from 1870 to 1695 MB/s. Every build carries the dependency and the PG_BASEBACKUP_NODIO switch, while only users on very fast links gain.

Message #1 measures the 0009 gain only with MPTCP, which is not part of this entry and belongs in its own thread. Without MPTCP, run f already saturates the link at 1818 MB/s before 0009. The author's own table [3] puts single-flow TCP at 1-10 Gbps for most environments. That leaves a small group who can use the io_uring path, while the build dependency and the extra code path land on everyone.

Thread: [#1 (Jakub Wartak, 2026-08-13)](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com)

**Question:** Which users today get more than about 15 Gbps on one TCP flow without MPTCP, and what does 0009 give them on that path alone?

Counter-check: held. Message #1 shows the io_uring gain only with MPTCP (i1), and run f already saturates the link without it. Table [3] lists LAN flows on physical servers as CPU-limited, so "1-10 Gbps for most environments" overstates it, but the missing non-MPTCP measurement stands.

### 3. Renaming the blackhole target breaks scripts and mixed-version backups

fixable · level 2, rubric item 12 · backpatchability, wire protocol · `standards-compat-1`

An operator's benchmark script runs `pg_basebackup --target=blackhole` and fails after upgrade. A new pg_basebackup running `-t server-blackhole` against a Postgres 15-19 server sends TARGET 'server-blackhole', which the older server rejects. An older client sending 'blackhole' to a new server fails the same way.

Patch 0001 changes the TARGET value in protocol.sgml and in the pg_basebackup docs, and its commit message mentions no alias. pg_basebackup is expected to work against older servers, and BASE_BACKUP TARGET values are part of the replication protocol. The rename is cosmetic, because adding `client-blackhole` does not require removing `blackhole`.

Thread: [#1 (Jakub Wartak, 2026-08-13)](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com), [#15 (Jakub Wartak, 2026-11-06)](https://www.postgresql.org/message-id/CAKZiRmyTs0ciaPztkF_VRUp0NXhRLnd%2BAAuJBtW6102QO-rezQ%40mail.gmail.com)

**Question:** Will the server keep accepting `blackhole` as an alias, and how does a new pg_basebackup choose the spelling when connected to a pre-20 server?

Counter-check: held. Patch 0001 changes the TARGET value in protocol.sgml and basebackup_target.c, and its commit message names no alias. pg_basebackup passes the target name to the server, so both mixed-version directions fail as described.

### 4. Direct I/O backup writes lack a defined fallback

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-3`

An operator upgrades the client and points pg_basebackup -D at a filesystem that rejects O_DIRECT or needs a different alignment, such as some NFS or tmpfs setups. The backup fails partway, or writes take an untested path. The operator finds out during a scheduled backup run.

Patch 0010 makes io_uring Direct I/O the default for larger files. The only control is the PG_BASEBACKUP_NODIO environment variable, which is undocumented. The material gives a buffered fallback only for the tar path when posix_fallocate fails (message #1). It does not say what happens when open(O_DIRECT) or an aligned write fails, or how non-Linux builds without liburing behave. These are the writes that produce the user's backup.

Thread: [#1 (Jakub Wartak, 2026-08-13)](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com)

**Question:** When O_DIRECT or io_uring is unavailable or rejected at run time, does pg_basebackup fall back to buffered writes on every platform, and how is that tested?

Counter-check: held. Message #1 describes a buffered fallback only for the tar path when preallocation is not possible. The material says nothing about a rejected O_DIRECT open, alignment failures, or builds without liburing.

### 5. Direct I/O control is an environment variable, not an option

fixable · level 2, rubric item 12 · design, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-3`

After upgrade, an operator writes backups to a filesystem that rejects O_DIRECT or needs different alignment. The only documented way to turn off Direct I/O is PG_BASEBACKUP_NODIO, which is not visible in `--help` or the existing scripts. The operator finds this only when the backup fails or runs slowly.

Patch 0010 turns on io_uring Direct I/O for large files by default and adds PG_BASEBACKUP_NODIO, described in message #1 as "for experimenting". pg_basebackup controls client write behaviour with options such as --no-sync, so an environment variable departs from that practice. The proposal does not say how a build without liburing, or a non-Linux build, behaves.

Thread: [#1 (Jakub Wartak, 2026-08-13)](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com)

**Question:** What happens on filesystems that reject O_DIRECT and on builds without liburing, and why is Direct I/O not controlled by a pg_basebackup option?

Counter-check: held. Message #1 introduces PG_BASEBACKUP_NODIO only "for experimenting", while Direct I/O is on by default for larger files. The material shows no pg_basebackup option and no documentation for turning it off, so the consistency point stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-3: Direct I/O backup writes lack a defined fallback |
| WAL / replication / standby | **yes** | The series rewrites the server-side base backup read path in basebackup.c, including short-read and truncation handling. |
| wire protocol | **yes** | Patch 0001 renames the BASE_BACKUP TARGET value `blackhole` to `server-blackhole` in protocol.sgml. |
| catalog changes | no |  |
| ABI / extension API | **yes** | The libpq patch adds PQgetCopyDataInternalBuf() as exports.txt entry 212. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The target rename makes a new pg_basebackup and an older server disagree on the name, in both directions. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The series adds a Linux-only io_uring Direct I/O write path, a liburing client dependency and a permanent libpq export. |
| performance on the hot path | **yes** | The 256 kB sink buffer and the fadvise calls change the read path of every base backup. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: io_uring Direct I/O benefit shown only above single-flow TCP limits |

## 5. Questions the author must answer before anyone reviews code

1. Why must PQgetCopyDataInternalBuf() be an exported libpq function instead of a frontend-private interface, and which buffer lifetime rules will libpq.sgml promise?
2. Will the server keep accepting `blackhole` as an alias, and what does a new pg_basebackup send to a pre-20 server for `-t server-blackhole`?
3. When the target filesystem rejects O_DIRECT, or the build has no liburing, does pg_basebackup fall back to buffered writes on every platform?
4. What does the io_uring Direct I/O patch gain on a single TCP flow without MPTCP, given that run i2 was slower than run h3?
5. Why is Direct I/O on by default and controlled by the PG_BASEBACKUP_NODIO environment variable instead of a documented pg_basebackup option?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas reviewed the server-side buffer and short-read patches. The client-side patches need someone who knows the libpq API and ABI rules, and someone who knows the pg_basebackup astreamer code.
- **Liability:** The owner signs for a permanent libpq export, for a Linux-only Direct I/O write path that produces users' backups, and for the reworked short-read handling in basebackup.c.
- **Author follow-up:** The author merged the reviewer's refactor into the series and posted new versions up to message #15, after a rebase in October.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The server-side buffer and fadvise patches have measured gains, an independent report of slow incremental backups in message #2, and active committer review, so they can go ahead on their own. The libpq export and the io_uring patches span libpq, fe_utils and pg_basebackup and need their own design answers first.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-2` (fixable, level 2) Second Linux-only write path in pg_basebackup with no reviewer. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-2` New exported libpq function lacks documentation and a stable contract. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same missing contract and ABI commitment more exactly.
- `replication-ondisk-2` Renaming the BASE_BACKUP target breaks mixed-version use. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-1` New public libpq function shaped for pg_basebackup only. Refuted (not specific): Duplicate of replication-ondisk-1.
- `maintenance-fit-3` Renaming the blackhole target breaks cross-version clients and scripts. Refuted (not specific): Duplicate of standards-compat-1.
- `already-exists-1` client-blackhole overlaps with existing tar-to-stdout discard. Refuted (already handled): The objection itself says the stdout route cannot cover tablespaces, WAL streaming or the plain-format extractor, which is the path the benchmarks in message #1 use. A test-only target also follows the existing `blackhole` target.
- `demand-2` Permanent public libpq function with one stated consumer. Refuted (not specific): Duplicate of replication-ondisk-1.
- `demand-3` Renaming the blackhole target has a cost and no requester. Refuted (not specific): Duplicate of standards-compat-1.

### Review angles

- prior-attempts: 0 objection(s). The only precedent, [P17], has no recorded reason, so no earlier rejection reason can be tested against this proposal. Message #1 links the old parallel-backup thread and keeps the single-connection design to avoid its open questions, but the thread does not say what those questions were.
- standards-compat: 3 objection(s)
- replication-ondisk: 3 objection(s). The proposal changes no heap, WAL, catalog or replication-slot state. The server-side read and fadvise changes are under active review by Heikki in messages #6-#14, so this angle does not object to them.
- security: 0 objection(s). The series does not change who may run a base backup or what that role can read; the server work still runs under the existing replication privilege, and the fadvise window is bounded by maintenance_io_concurrency and io_combine_limit. Client-side changes (client-blackhole, fallocate from tar sizes, Direct I/O, the new libpq call) act only for the client user, against a server it already trusts, so I found no security or privilege objection.
- maintenance-fit: 3 objection(s). The server-side patches 0004-0006 have engaged committer review and can be committed on their own, so this angle raises nothing against them.
- already-exists: 1 objection(s). The other parts have no existing equivalent I can name. Postgres has no zero-copy COPY receive call, no fadvise prefetch for incremental reads, and no Direct I/O path in pg_basebackup, so the extension and existing-feature angle finds nothing further.
- demand: 3 objection(s). Demand for faster full and incremental backups is real. Gustavo independently reported slow incremental backups in message #2, and Heikki is engaged on 0004-0005. These objections concern the client-side pieces (0001, 0009-0011), not the server-side read path.

### Precedents retrieved

- P1: [Prevent pg_basebackup running as root](https://commitfest.postgresql.org/patch/2431/) (cf, Rejected, 2020-02-27) - judged not relevant; reason summarised from its thread
- P2: [base backup client as auxiliary backend process](https://commitfest.postgresql.org/patch/2192/) (cf, Returned with feedback, 2020-02-28) - judged not relevant; reason summarised from its thread
- P3: [Revise documentation on base backups](https://commitfest.postgresql.org/patch/379/) (cf, Rejected, 2015-10-30) - judged not relevant; reason summarised from its thread
- P4: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P5: [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://commitfest.postgresql.org/patch/5403/) (cf, Withdrawn, 2024-11-25) - judged not relevant; reason summarised from its thread
- P6: [Application name for pg_basebackup, pg_receivewal and pg_recvlogical](https://commitfest.postgresql.org/patch/2335/) (cf, Rejected, 2019-11-07) - judged not relevant; reason summarised from its thread
- P7: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P8: [pg_basebackup to adjust existing data directory permissions](https://commitfest.postgresql.org/patch/2018/) (cf, Returned with feedback, 2019-11-27) - judged not relevant; reason summarised from its thread
- P9: [pg_basebackup: add test about zstd compress option](https://commitfest.postgresql.org/patch/3835/) (cf, Rejected, 2022-12-06) - judged not relevant; reason summarised from its thread
- P10: [Reorder tablespaces for streaming basebackups](https://commitfest.postgresql.org/patch/1035/) (cf, Rejected, 2017-03-31) - judged not relevant; reason summarised from its thread
- P11: [Revert the patch to check if we've reached end-of-backup also when doing crash recovery, and throw an error if not.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=54685b1c2b356b4b9c0938f6b8dcc52e173c0400) (revert, Reverted, 2011-04-13) - judged not relevant
- P12: [Revert "Prevent running pg_basebackup as root"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dcddc3f813ff39d0e33c2b7c77092a6c0b0e9d0b) (revert, Reverted, 2020-02-07) - judged not relevant
- P13: [pg_basebackup has stricter tablespace rules than the server](https://commitfest.postgresql.org/patch/1125/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P14: [Stores short varlena in an array](https://commitfest.postgresql.org/patch/5461/) (cf, Rejected, 2025-01-25) - judged not relevant
- P15: [Revert changes to pg_basebackup and pg_waldump usage() code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05b5feb60e7cf36db08eb0d34dbd21854cb9e8ce) (revert, Reverted, 2017-05-17) - judged not relevant
- P16: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3689/) (cf, Withdrawn, 2022-06-17) - judged not relevant
- P17: [parallel backup](https://commitfest.postgresql.org/patch/2472/) (cf, Returned with feedback, 2020-07-06) - used in the review
- P18: [Create shorthand for including all extra tests](https://commitfest.postgresql.org/patch/4547/) (cf, Rejected, 2024-01-20) - judged not relevant
- P19: [Avoid Copying WAL Segments Before Divergence to Speed Up pg_rewind](https://commitfest.postgresql.org/patch/3800/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P20: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3690/) (cf, Withdrawn, 2022-06-17) - judged not relevant
- P21: [low wal_retrieve_retry_interval causes missed signals on Windows](https://commitfest.postgresql.org/patch/4118/) (cf, Withdrawn, 2023-02-25) - judged not relevant
- P22: [Improve handling of OOM errors in libpq making process hangling for COPY and bind (take 2)](https://commitfest.postgresql.org/patch/298/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P23: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [File level incremental backup](https://commitfest.postgresql.org/patch/94/) (cf, Rejected, 2015-03-18) - judged not relevant

### Warnings

- attachment limit (8) reached; v06112026-0007-pg_basebackup-eliminate-usage-of-libc-to-c.patch not read
- attachment limit (8) reached; v06112026-0010-pg_basebackup-add-support-for-Direct-I-O-a.patch not read
- attachment limit (8) reached; v06112026-0011-pg_basebackup-preallocate-DIO-writes-also-.patch not read
- attachment limit (8) reached; v06082026-0003-pg_basebackup-report-average-data-transfer.patch not read
- attachment limit (8) reached; v06082026-0002-pg_basebackup-add-new-client-blackhole-ben.patch not read
- attachment limit (8) reached; v06082026-0005-basebackup-issue-posix_fadvise-for-more-ef.patch not read
- attachment limit (8) reached; v06082026-0001-pg_basebackup-rename-the-blackhole-backup-.patch not read
- attachment limit (8) reached; v06082026-0004-basebackup-bump-SINK_BUFFER_LENGTH-to-256k.patch not read
- attachment limit (8) reached; v06082026-0006-pg_basebackup-elimiate-usage-of-libc-to-co.patch not read
- attachment limit (8) reached; v06082026-0007-pg_basebackup-preallocate-extracted-files-.patch not read
- attachment limit (8) reached; v06082026-0009-pg_basebackup-add-support-for-Direct-I-O-a.patch not read
- attachment limit (8) reached; v06082026-0008-libpq-pg_basebackup-add-PQgetCopyDataInter.patch not read
- attachment limit (8) reached; v06082026-0010-pg_basebackup-preallocate-DIO-writes-also-.patch not read
- attachment limit (8) reached; v2-0001-Add-MPTCP-protocol-support-to-server-and-libpq-on.patch not read
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 173 s; cost reported by the CLI: $3.56
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 115 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:24:10.089Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
