# Design pre-review: Compress big WAL records

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5621](https://commitfest.postgresql.org/patch/5621/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Compression of bigger WAL records](https://www.postgresql.org/message-id/flat/4DC38068-976E-4A84-8EE6-4EFACBBD927A%40yandex-team.ru), started 2025-01-12
- Messages read: 37 of 37
- Patch prose read: v2-0001-Compress-big-WAL-records.patch, v0-0001-Compress-big-WAL-records.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and less WAL is a lasting need. The remaining points are fixable. standards-compat-2 and demand-1 ask for OLTP cost and benefit numbers behind the default-on 512-byte threshold. replication-ondisk-1 asks for a confirmed XLOG_PAGE_MAGIC bump. maintenance-fit-1 and standards-compat-1 concern the two record lengths and the changed meaning of wal_fpi_bytes. A committer who knows the WAL insert and reader paths has to sign the record format.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Buffer allocation in critical sections, bounds checks when decompressing, the XLOG_PAGE_MAGIC bump and the GUC contexts could not be checked.

## The proposal, restated

**Problem.** Each full-page image is compressed on its own today, so redundancy between images in one record, or between records, goes unused. B-tree index builds and other large-record workloads write more WAL than they need to. Small records, which carry about half of pgbench WAL, do not compress on their own.

**Proposed design.** Context reuse for zstd is already committed. The remaining 0001 compresses a record larger than wal_compression_threshold as one unit with lz4 or zstd, flags it with XLR_COMPRESSED and a codec and length header, and falls back to per-FPI compression when that does not save space. WIP 0003 keeps zstd history across records in leased streams that restart at fixed 4MB boundaries so readers can start mid-WAL.

**User-visible surface.**
- New GUC wal_compression_threshold, default 512 bytes; only lz4 and zstd use it.
- New GUC wal_compression_streams in WIP 0003, a pool of cross-record compression streams.
- WAL record format: XLR_COMPRESSED bit in xl_info plus a compression header with codec and decompressed length.
- pg_waldump and pg_walinspect report physical record sizes separately from decoded sizes.
- pg_stat_wal.wal_fpi_bytes counts image bytes before whole-record compression and can exceed wal_bytes.

**What the author claims.**
- CREATE INDEX over 10M random doubles writes about 19% less WAL with zstd and 15% less with lz4.
- Records below the threshold are untouched, and a threshold above the largest record restores current behaviour.
- In 0001 every record still decodes on its own, so WAL readers need no new logic.
- Streams in 0003 cut pgbench WAL per transaction by 17% to 37%, at 0% to 12% throughput cost.
- A backend with whole-record compression holds about twice the old compression buffer memory, about 268 KiB extra.

**Thread state.** Michael Paquier committed the zstd context-reuse parts for replay and insertion in October 2026 but gave no opinion on whole-record or stream compression. Zsolt Parragi is reviewing v13; the author asked for advice on buffer allocation in critical sections and got no answer.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-01-12; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Existing wal_compression users get whole-record compression by default

fixable · level 2, rubric item 10 · new GUC, performance on the hot path · `standards-compat-2`

A site runs with wal_compression = on, which now means zstd first, and upgrades. Every record over 512 bytes is compressed whole, and each backend holds about 268 KiB more. If OLTP latency rises, the site cannot tell why, because nothing in its configuration changed.

wal_compression has been documented as compression of full-page writes. With a 512-byte default threshold, the patch widens what it does for every current user of lz4 or zstd. Message #7 says 512 was not tested. The brief notes that no CPU or latency numbers for 0001 on OLTP are shown, only WAL size. Message #13 shows a low threshold can make WAL larger, so the default is a real choice that needs data behind it.

Thread: [#7 (Andrey Borodin, 2025-01-30)](https://www.postgresql.org/message-id/87E092DC-0E61-426A-8F73-B61A19BCC00D%40yandex-team.ru), [#13 (Andrey Borodin, 2026-03-09)](https://www.postgresql.org/message-id/4D8E3280-632B-4C6C-A728-835B7FDE6325%40yandex-team.ru), [#16 (Andrey Borodin, 2026-07-26)](https://www.postgresql.org/message-id/33EDA443-E905-4E15-A7B0-F734C369A19B%40yandex-team.ru), [#25 (Michael Paquier, 2026-10-06)](https://www.postgresql.org/message-id/asRsOl73nf8GeBFd%40paquier.xyz)

**Question:** What OLTP CPU and latency measurements support enabling whole-record compression at 512 bytes by default, rather than leaving it off until a user sets a threshold?

Counter-check: held. The thread shows only WAL size for 0001, and message #7 confirms 512 was not tested. The low-threshold figure is in message #16, not #13, but the missing OLTP CPU and latency numbers for a default-on path remain.

### 2. Gain from whole-record compression is shown on one synthetic workload

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer takes on a new WAL record format, a new GUC and a changed reader path, and the 15-19% saving appears only on random-data B-tree builds. On typical OLTP WAL, whose records are mostly under 512 bytes, 0001 does nothing. The cost stays even if few users see the gain.

The only measurement for 0001 is CREATE INDEX over 10M random doubles (messages #4 and #17). Message #16 says the records over 512 bytes are 0.8% of pgbench records, and the author points to 0003, which is WIP, for the rest. No field report, -general thread or user in the thread describes WAL volume from large records as their problem. Message #6 liked the idea, but did not report a need.

**Precedents.**
- [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (Returned with feedback, 2024-04-09); thread: [link](https://www.postgresql.org/message-id/flat/CAEze2WgcW5SkPO_TGTqirRcCS_FN0xEZqUsXDE6pPz2TYzXQBA@mail.gmail.com)

Thread: [#4 ("Andrey M(dot) Borodin", 2025-01-21)](https://www.postgresql.org/message-id/73E8D8A0-F5C3-45E8-8DFC-60AA86F8B890%40yandex-team.ru), [#6 (Fujii Masao, 2025-01-28)](https://www.postgresql.org/message-id/87d08e95-0e55-4bcf-9f4b-5c9e7b8d91f8%40oss.nttdata.com), [#16 (Andrey Borodin, 2026-07-26)](https://www.postgresql.org/message-id/33EDA443-E905-4E15-A7B0-F734C369A19B%40yandex-team.ru), [#17 (Andrey Borodin, 2026-08-07)](https://www.postgresql.org/message-id/A2BB6471-C150-423C-B6F5-BFEC46BE544B%40yandex-team.ru)

**Question:** Which real workloads besides random-data index builds produce records above the threshold, and how much WAL does 0001 save on them?

Counter-check: held. The only 0001 measurement is the random-data index build. "Does nothing" on OLTP overstates, since message #16 puts 47.5% of pgbench bytes in records over 512 bytes, but the thread shows no HEAD comparison for that workload.

### 3. Compressed records hide block references from WAL parsers outside xlogreader

fixable · level 2, rubric item 7 · WAL / replication / standby, ABI / extension API, backpatchability · `replication-ondisk-1`

An incremental backup tool that parses raw WAL to collect changed blocks meets an XLR_COMPRESSED record. It does not see the block references inside the compressed body and leaves those pages out of the backup. The restored cluster is missing changes, and the user finds this only after a restore.

The claim that "WAL readers need no new logic" holds only for readers built on the in-core xlogreader. Message #36 shows that in-core code already misread the flag, since replorigin_identity reported records as UNKNOWN. The brief says the material does not show whether XLOG_PAGE_MAGIC is bumped. Message #3 mentions bumping WAL_FILE_MAGIC, and no later message confirms it, so a parser that reads WAL would get no loud signal.

Thread: [#3 (Andrey Borodin, 2025-01-13)](https://www.postgresql.org/message-id/ECCAD8F2-032D-42FE-B60D-0D6EB0423FF2%40yandex-team.ru), [#36 (Zsolt Parragi, 2026-10-26)](https://www.postgresql.org/message-id/CAN4CZFOhMg9oyKUSeK4VhdB1B-cOjS6qUgTk9HCDnbMg9pLQHQ%40mail.gmail.com)

**Question:** Is XLOG_PAGE_MAGIC bumped so that old raw-WAL parsers reject the new format, and how would an external block-tracking tool find the block references in a compressed record?

Counter-check: held. Any WAL format change breaks parsers that do not use xlogreader, so that part is not specific to this patch. The surviving point is that no message after #3 confirms an XLOG_PAGE_MAGIC bump, which is what makes old parsers fail loudly.

### 4. Split between physical and decoded record length burdens every WAL reader

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby, ABI / extension API · `maintenance-fit-1`

A later patch adds a length check or an xl_info test in xlogreader, an rmgr desc routine or an extension WAL reader, and picks the wrong length or misses the XLR_COMPRESSED bit. Tests pass because wal_compression is off by default, and the bug shows up only on clusters running lz4 or zstd with large records.

Message #20 shows a rebase over an unrelated length check (18992dc9d98) that still compiled, but against a variable that no longer existed. Message #36 found replication origin records reported as UNKNOWN because one routine did not mask the new bit. Both bugs were silent and surfaced only in review. v13 clears the flag after decompression, but every record now has two lengths, and nothing makes the compiler or the default tests catch the wrong one.

Thread: [#20 (Andrey Borodin, 2026-08-14)](https://www.postgresql.org/message-id/04119E77-9C0F-415A-A9E0-65E6E4A57940%40yandex-team.ru), [#36 (Zsolt Parragi, 2026-10-26)](https://www.postgresql.org/message-id/CAN4CZFOhMg9oyKUSeK4VhdB1B-cOjS6qUgTk9HCDnbMg9pLQHQ%40mail.gmail.com), [#37 (Andrey Borodin, 2026-11-08)](https://www.postgresql.org/message-id/17A14E6D-9ECA-4E15-AAF7-1EAED2568788%40yandex-team.ru)

**Question:** What makes a future WAL reader or length check that uses the wrong length fail in the default regression run, not only with compression turned on?

Counter-check: held. v13 clears XLR_COMPRESSED after decompression, so rmgr desc routines and readers built on xlogreader see ordinary records; that part is answered. The two lengths inside xlogreader.c remain, and message #20 shows a rebase can pick the wrong one silently.

### 5. pg_stat_wal.wal_fpi_bytes can now exceed wal_bytes

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A monitoring dashboard computes the FPI share of WAL as wal_fpi_bytes / wal_bytes. After upgrade, with zstd and the default threshold, it reports shares like 845%, as in message #33. The operator finds out from alerts or wrong capacity planning, with no change on their side.

Message #35 keeps wal_fpi_bytes as image bytes before whole-record compression and only documents that it can exceed physical wal_bytes. Today both columns count bytes written, so existing tools read them as parts of one total. The proposal changes what the pair means without a new column or a rename. pg_waldump and pg_walinspect got separate physical and decoded sizes, but pg_stat_wal did not.

Thread: [#33 (Zsolt Parragi, 2026-10-12)](https://www.postgresql.org/message-id/CAN4CZFOgbHQFvmBD%3DTo7Hjv1fJ2QMJn0pM-2HcPvH4dqkehxcQ%40mail.gmail.com), [#35 (Andrey Borodin, 2026-10-24)](https://www.postgresql.org/message-id/D815DE2E-6EEB-4EF4-8B5F-3374F552D55C%40yandex-team.ru)

**Question:** Why not leave wal_fpi_bytes comparable to wal_bytes and report pre-compression image bytes in a separate column with its own name?

Counter-check: held. Message #35 keeps wal_fpi_bytes as pre-compression image bytes and only documents that it can exceed wal_bytes. Only the author weighed this choice, so the changed meaning of an existing column stands as a UI question.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The WAL record format gains the XLR_COMPRESSED bit and a compression header, and 0003 adds 4MB stream restart points. |
| WAL / replication / standby | **yes** | Recovery, logical decoding, pg_rewind and pg_waldump must read compressed records; 0003 changes where a reader may start. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | xlogreader.h and xlogrecord.h change, and 0003 adds XLogBeginReadStreamed() for readers that start mid-WAL. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | wal_compression_threshold in 0001 and wal_compression_streams in WIP 0003. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Cross-record stream history in 0003 could make WAL size depend on other sessions' data. |
| backpatchability | **yes** | see objection replication-ondisk-1: Compressed records hide block references from WAL parsers outside xlogreader |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every record has a physical and a decoded length, and 0003 adds an unenforced rule for records read by LSN. |
| performance on the hot path | **yes** | Records over 512 bytes are compressed whole in the WAL insert path for every lz4 and zstd user. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Gain from whole-record compression is shown on one synthetic workload |

## 5. Questions the author must answer before anyone reviews code

1. What CPU and latency cost does 0001 add on pgbench or a similar OLTP workload at the default 512-byte threshold, and how was 512 chosen?
2. Which workloads besides the random-data index build produce records above the threshold, and how much WAL does 0001 save on them compared with HEAD?
3. Is XLOG_PAGE_MAGIC bumped so that WAL parsers that predate the new format reject it instead of misreading compressed records?
4. Why not keep wal_fpi_bytes comparable to wal_bytes and report image bytes before compression in a separate column?
5. For WAL segments smaller than 4MB, does 0003 also restart streams at each segment boundary, or which retention rule keeps the earlier segments?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier committed the zstd context-reuse parts but gave no opinion on whole-record or stream compression. Fujii Masao reviewed the design early. The owner must know xloginsert.c, xlogreader.c and recovery.
- **Liability:** The owner signs for a new WAL record format, the decompression path in recovery, logical decoding and pg_rewind, and the default-on behaviour for lz4 and zstd users. The format stays for as long as the feature exists.
- **Author follow-up:** The author has carried the patch through 7 CommitFests and 13 versions, rebasing and answering review, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The gain reaches users of lz4 or zstd WAL compression with large records, and is shown on one index-build test. The patch changes the WAL record format and the insert and reader paths, so the default threshold, the format signal and the stats meaning should be settled before detailed review.

## Appendix

### Held up in the counter-check, not listed above

- `replication-ondisk-2` (fixable, level 2) Fixed 4MB stream restart does not fit 1MB or 2MB WAL segments. Not listed: not among the strongest objections.
- `security-1` (fixable, level 2) Cross-record zstd streams could act as a compression oracle. Not listed: not among the strongest objections.
- `maintenance-fit-3` (fixable, level 2) Stream compression makes every future WAL reader use a new start API. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `replication-ondisk-3` Stream exclusion rule for LSN-addressed records has no enforcement. Refuted (not specific): Duplicate of maintenance-fit-3, which covers the same unenforced exclusion rule together with the new start call.
- `maintenance-fit-2` No committer has taken on whole-record or stream compression. Refuted (wrong facts): Fujii Masao, a committer and listed CF reviewer, gave design-level review in messages #6 and #11, and the author adopted his suggestion to keep per-FPI compression. Michael Paquier is active in the thread, and weak committability alone is not a design flaw.
- `maintenance-fit-4` Default threshold puts unmeasured CPU cost on every compression user. Refuted (not specific): Duplicate of standards-compat-2. The 12% tps figure comes from the 0003 stream lease, a different mechanism, and P8 concerned the size of a gain, so neither supports this point.
- `demand-2` Default threshold puts unmeasured costs on every lz4 and zstd user. Refuted (not specific): Duplicate of standards-compat-2 for the unmeasured default cost and of standards-compat-1 for the wal_fpi_bytes change.

### Review angles

- prior-attempts: 0 objection(s). No precedent found for compressing whole WAL records or keeping compression history across records. [P1], [P8], [P23], [P3] and [P2] are in nearby WAL areas, but none was rejected for a reason that still applies to this proposal. The thread points to no earlier discussion of the same idea.
- standards-compat: 2 objection(s). Two objections only. The record-format change uses a reserved xl_info bit and only needs the usual XLOG_PAGE_MAGIC bump, which the material does not confirm. The WIP streams GUC is not mature enough to judge from this angle.
- replication-ondisk: 3 objection(s). Objections 2 and 3 apply to WIP 0003 only. For 0001 the main open point is the format signal to external WAL parsers. Message #21's page LSN concern was not tied to a specific design, so it is left out.
- security: 1 objection(s). 0001 adds no new exposure: per-FPI compression already makes WAL size depend on page contents. The threshold and stream GUCs do not give unprivileged users new abilities, assuming they keep superuser or postmaster context. The material does not state the GUC contexts.
- maintenance-fit: 4 objection(s). Objections listed above.
- already-exists: 0 objection(s). Postgres has no existing way to compress whole records or to use cross-record redundancy in pg_wal or the replication stream. Per-FPI wal_compression is extended here, and compressing archived segments in archive_command shrinks only the archive, so neither covers live WAL I/O or standby traffic.
- demand: 2 objection(s). Less WAL volume is a broad and lasting need, so neither objection is fatal. Both ask for evidence that the 0001 gain reaches beyond index builds and outweighs its default-on cost.

### Precedents retrieved

- P1: [Generic WAL compression](https://commitfest.postgresql.org/patch/1363/) (cf, Returned with feedback, 2018-03-21) - used in the review; reason summarised from its thread
- P2: [Tracking the compression effect](https://commitfest.postgresql.org/patch/3851/) (cf, Withdrawn, 2022-10-31) - used in the review; reason summarised from its thread
- P3: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - used in the review; reason summarised from its thread
- P4: [Compression dictionaries](https://commitfest.postgresql.org/patch/3626/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P5: [Adding CommandID to heap xlog records](https://commitfest.postgresql.org/patch/3882/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P6: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - judged not relevant; reason summarised from its thread
- P7: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P8: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - used in the review; reason summarised from its thread
- P9: [Page compression for OLTP](https://commitfest.postgresql.org/patch/3783/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P10: [Revert "Add some temporary code to record stack usage at server process exit."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96112ee7c60557bb192a9aa07b514db2400fd45e) (revert, Reverted, 2016-07-10) - judged not relevant
- P11: [pg_basebackup: add test about zstd compress option](https://commitfest.postgresql.org/patch/3835/) (cf, Rejected, 2022-12-06) - judged not relevant
- P12: [Move walreceiver state assignment (to WALRCV_STREAMING) in WalReceiverMain()](https://commitfest.postgresql.org/patch/4698/) (cf, Withdrawn, 2024-01-22) - judged not relevant
- P13: [Making WAL receiver startup rely on GUC context for primary_conninfo and primary_slot_name](https://commitfest.postgresql.org/patch/1916/) (cf, Rejected, 2019-02-16) - judged not relevant
- P14: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P15: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P16: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P17: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P18: [Making autovacuum logs indicate if insert-based threshold was the triggering condition](https://commitfest.postgresql.org/patch/3802/) (cf, Withdrawn, 2022-11-27) - judged not relevant
- P19: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P20: [Compensate for full_page_writes when spreading checkpoint I/O](https://commitfest.postgresql.org/patch/283/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P21: [Reduce wakeup on idle for bgwriter & walwriter for >5s](https://commitfest.postgresql.org/patch/4029/) (cf, Withdrawn, 2023-03-30) - judged not relevant
- P22: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3668/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P23: [XLog size reductions: smaller XLogRecordBlockHeader](https://commitfest.postgresql.org/patch/4324/) (cf, Returned with feedback, 2024-10-07) - used in the review
- P24: [Improving replay of XLOG_BTREE_VACUUM records](https://commitfest.postgresql.org/patch/233/) (cf, Returned with feedback, 2015-08-25) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 236 s; cost reported by the CLI: $3.61
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 111 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:47:12.850Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
