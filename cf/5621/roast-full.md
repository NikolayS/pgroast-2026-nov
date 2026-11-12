# Design pre-review: Compress big WAL records

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5621](https://commitfest.postgresql.org/patch/5621/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Compression of bigger WAL records](https://www.postgresql.org/message-id/flat/4DC38068-976E-4A84-8EE6-4EFACBBD927A%40yandex-team.ru), started 2025-01-12
- Messages read: 37 of 37
- Patch prose read: v2-0001-Compress-big-WAL-records.patch, v0-0001-Compress-big-WAL-records.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and none stops review at level 1 or 2. The goal extends the existing wal_compression setting, and the open points are fixable: the default and opt-out (standards-compat-1), the codec needed to read block references (replication-ondisk-1), and workload evidence (demand-1). The WAL record format change and the WIP streams rule (replication-ondisk-2) need a committer who knows xlogreader and recovery to sign.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check that every reader masks XLR_COMPRESSED and uses the right length, or whether the page magic changes. The truncated messages #16 and #17 hide the stream lock cost.

## The proposal, restated

**Problem.** Today wal_compression compresses each full-page image on its own, so it cannot use redundancy shared by several images in one record, or between records. Large non-FPI records, such as 2PC records, are not compressed at all. The target is any cluster where WAL volume matters, with B-tree index builds as the main example.

**Proposed design.** A record larger than wal_compression_threshold is compressed as one unit with lz4 or zstd, alongside per-FPI compression, and is marked by a spare xl_info bit plus a small header. If whole-record compression does not save space, the record falls back to per-FPI compression. A WIP patch keeps zstd history across records in a pool of streams that reset at fixed 4MB boundaries, so readers can start mid-WAL.

**User-visible surface.**
- New GUC wal_compression_threshold, default 512 bytes, which sets the record size above which the whole record is compressed.
- New GUC wal_compression_streams (WIP patch), a pool of compression streams that keep history across records.
- WAL record format change: the XLR_COMPRESSED bit in xl_info and a compression header with method and decompressed length.
- pg_waldump and pg_walinspect report physical record sizes separately from decoded sizes.
- pg_stat_wal wal_fpi_bytes counts image bytes before whole-record compression, so it can exceed wal_bytes.

**What the author claims.**
- CREATE INDEX over 10M random doubles writes 19% less WAL with zstd and 15% less with lz4.
- Records below the threshold are unchanged, and each record still decodes on its own without streams.
- Setting the threshold above the largest possible record restores today's behaviour.
- Streams cut pgbench WAL per transaction by 17% to 37%, with a throughput cost between none and 12%.
- A backend with whole-record compression enabled holds about twice the old compression buffer memory.

**Thread state.** The zstd context reuse split out of this series was committed in two parts by Michael Paquier in October 2026. v13 of whole-record compression, the pg_walinspect change and WIP streams answers Zsolt Parragi's review and has no reply, and the buffer allocation question asked of Michael Paquier is still open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-01-12; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Gain for whole-record compression is shown on one synthetic workload only

fixable · level 1b, rubric item 3 · demand, WAL / replication / standby, new GUC, maintenance burden ("who maintains this in 10 years") · `demand-1`

A user turns on zstd with the default 512-byte threshold on an OLTP cluster and expects lower WAL volume. In that workload few records exceed the threshold, so the change in WAL volume is small. Meanwhile the record format change, the second compression buffer and the extra reader path are permanent for every release and every WAL tool.

The only measured benefit of 0002 is CREATE INDEX over 10M random doubles: 19% less WAL with zstd and 15% with lz4 (messages #4, #17). The 2PC case from message #1 has no numbers. No field report, -general thread or production WAL sample shows how much real WAL sits in large multi-FPI records. Michael Paquier committed the context-reuse parts but had no opinion on 0002 and 0003 (message #25), and the other interested reviewers (messages #6, #21) state liking the idea without giving a workload.

Thread: [#1 ("Andrey M(dot) Borodin", 2025-01-12)](https://www.postgresql.org/message-id/4DC38068-976E-4A84-8EE6-4EFACBBD927A%40yandex-team.ru), [#4 ("Andrey M(dot) Borodin", 2025-01-21)](https://www.postgresql.org/message-id/73E8D8A0-F5C3-45E8-8DFC-60AA86F8B890%40yandex-team.ru), [#17 (Andrey Borodin, 2026-08-07)](https://www.postgresql.org/message-id/A2BB6471-C150-423C-B6F5-BFEC46BE544B%40yandex-team.ru), [#25 (Michael Paquier, 2026-10-06)](https://www.postgresql.org/message-id/asRsOl73nf8GeBFd%40paquier.xyz)

**Question:** On which production or realistic workloads, other than a B-tree build on random data, does 0002 reduce WAL volume, and by how much at the default threshold?

Counter-check: held. Message #16 shows records over 512 bytes hold 47.5% of pgbench WAL bytes, which weakens the "few records" scenario. But the only measured gain over per-FPI compression is the CREATE INDEX case, and no field report is shown.

### 2. Default threshold turns on whole-record compression for every existing wal_compression user

fixable · level 2, rubric item 12 · new GUC, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A site runs wal_compression = on and upgrades. With the 512-byte default, large records are now compressed whole and need a second buffer per backend, and WAL size and CPU change. To get the old behaviour back, the DBA has to find a second GUC and set it to about 1GB.

In message #11 Fujii Masao asked to keep FPI-only compression as a real choice, and message #12 agreed to complement it. But the opt-out in message #13 is a magic threshold value, with no explicit setting. The 512 default has no measured basis (message #7), and message #16 shows a low threshold can make WAL larger. wal_compression already selects the method, so a committer would likely ask why the scope is not chosen there.

Thread: [#7 (Andrey Borodin, 2025-01-30)](https://www.postgresql.org/message-id/87E092DC-0E61-426A-8F73-B61A19BCC00D%40yandex-team.ru), [#11 (Fujii Masao, 2026-01-16)](https://www.postgresql.org/message-id/CAHGQGwG7krZ%2Bpktg9c2e8xk-6TK-sK0kMSWriSQtz%3DJvJ_E6Pg%40mail.gmail.com), [#12 (Andrey Borodin, 2026-01-16)](https://www.postgresql.org/message-id/19C9772F-9377-4ABA-88CF-F5B7C35C5A4C%40yandex-team.ru), [#13 (Andrey Borodin, 2026-03-09)](https://www.postgresql.org/message-id/4D8E3280-632B-4C6C-A728-835B7FDE6325%40yandex-team.ru), [#16 (Andrey Borodin, 2026-07-26)](https://www.postgresql.org/message-id/33EDA443-E905-4E15-A7B0-F734C369A19B%40yandex-team.ru)

**Question:** Why is whole-record compression on by default for existing wal_compression users, and why is the opt-out a large threshold value with no explicit off setting?

Counter-check: held. Message #13 gives a 1GB threshold as the only way back to FPI-only compression, and message #7 gives no measured basis for 512. Message #35 spares FPI-only settings the new buffers, but lz4 and zstd users still get the new path by default.

### 3. Whole-record compression hides block references behind the codec

fixable · level 2, rubric item 7 · WAL / replication / standby, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A cluster runs with wal_compression = on, which now picks zstd first, so every record over 512 bytes is compressed as one unit. A pg_rewind binary, or an out-of-tree WAL reader built without zstd, can no longer parse the block headers of those records. It fails during a failover rewind or an incremental backup scan, not when the setting is changed.

Today a reader without lz4 or zstd can still decode record headers and block references and fails only on restoring an image. With 0002 the block headers sit inside the compressed payload (message #1), so codec support decides whether the record can be read at all. The default threshold turns this on for every existing wal_compression = lz4, zstd or on user, with no opt-in (messages #7, #20). The material also does not say whether XLOG_PAGE_MAGIC is bumped, so older tools may misread records instead of rejecting them (message #3).

Thread: [#1 ("Andrey M(dot) Borodin", 2025-01-12)](https://www.postgresql.org/message-id/4DC38068-976E-4A84-8EE6-4EFACBBD927A%40yandex-team.ru), [#3 (Andrey Borodin, 2025-01-13)](https://www.postgresql.org/message-id/ECCAD8F2-032D-42FE-B60D-0D6EB0423FF2%40yandex-team.ru), [#7 (Andrey Borodin, 2025-01-30)](https://www.postgresql.org/message-id/87E092DC-0E61-426A-8F73-B61A19BCC00D%40yandex-team.ru), [#20 (Andrey Borodin, 2026-08-14)](https://www.postgresql.org/message-id/04119E77-9C0F-415A-A9E0-65E6E4A57940%40yandex-team.ru)

**Question:** Why should whole-record compression be on by default for existing wal_compression users, and how does a reader without the codec report the problem instead of misparsing?

Counter-check: held. Message #18 quotes a "not supported by build" error, so a reader without the codec reports the problem and does not misparse. The narrower point stands: tools that need only block references now need the codec, by default.

### 4. wal_fpi_bytes exceeding wal_bytes changes the meaning of a pg_stat_wal column

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A monitoring dashboard plots wal_fpi_bytes / wal_bytes as the full-page-image share of WAL. After upgrade the ratio goes above 1, as in message #33 (90 MB against 10 MB), and alerts or capacity plans built on it give wrong answers. Users learn of the change only from a release-notes line or the documentation.

Today wal_fpi_bytes is a subset of wal_bytes, and external tools depend on that. Message #35 keeps the pre-compression count, documents that it can exceed wal_bytes, and says the jointly compressed bytes cannot be split. That reason holds, but the result is one view whose two byte columns count in different units. A committer would likely ask for a separate column, or for wal_fpi_bytes to stay physical.

**Precedents.**
- [Tracking the compression effect](https://commitfest.postgresql.org/patch/3851/) (Withdrawn, 2022-10-31); thread: [link](https://www.postgresql.org/message-id/flat/0489c74a83dbd54ed33feb6690d7ca4f@oss.nttdata.com)

Thread: [#33 (Zsolt Parragi, 2026-10-12)](https://www.postgresql.org/message-id/CAN4CZFOgbHQFvmBD%3DTo7Hjv1fJ2QMJn0pM-2HcPvH4dqkehxcQ%40mail.gmail.com), [#35 (Andrey Borodin, 2026-10-24)](https://www.postgresql.org/message-id/D815DE2E-6EEB-4EF4-8B5F-3374F552D55C%40yandex-team.ru)

**Question:** Can pg_stat_wal keep wal_fpi_bytes no larger than wal_bytes, for example by reporting pre-compression image bytes in a new column instead?

Counter-check: held. Message #35 documents the change but does not keep the ratio that monitoring tools compute today. P2 must not be used as support, because its recorded reason argues against adding counters to pg_stat_wal.

### 5. Two record lengths that every future xlogreader change must choose between

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby, ABI / extension API · `maintenance-fit-2`

A later patch adds a sanity check, a statistic or a buffer size based on xl_tot_len in xlogreader or pg_waldump. It is correct for uncompressed records, which is all the default test setup produces. With whole-record compression on, it checks or allocates the wrong side, and this shows up as a decode failure or a wrong report in the field.

0002 splits each record into a physical length and a decoded length. Message #20 shows a rebase across 18992dc9d98 that needed a choice of side, and a plain textual rebase compiled against a variable that no longer existed. Message #36 found replorigin_identity reading xl_info without masking XLR_COMPRESSED. v13 clears the flag after decompression, but nothing makes the wrong choice of length fail loudly.

Thread: [#20 (Andrey Borodin, 2026-08-14)](https://www.postgresql.org/message-id/04119E77-9C0F-415A-A9E0-65E6E4A57940%40yandex-team.ru), [#33 (Zsolt Parragi, 2026-10-12)](https://www.postgresql.org/message-id/CAN4CZFOgbHQFvmBD%3DTo7Hjv1fJ2QMJn0pM-2HcPvH4dqkehxcQ%40mail.gmail.com), [#35 (Andrey Borodin, 2026-10-24)](https://www.postgresql.org/message-id/D815DE2E-6EEB-4EF4-8B5F-3374F552D55C%40yandex-team.ru), [#36 (Zsolt Parragi, 2026-10-26)](https://www.postgresql.org/message-id/CAN4CZFOhMg9oyKUSeK4VhdB1B-cOjS6qUgTk9HCDnbMg9pLQHQ%40mail.gmail.com)

**Question:** Which test configuration runs the recovery and tool suites with whole-record compression on, so that a check on the wrong length fails in CI?

Counter-check: held. The series has a dedicated test, and message #14 shows v6 set lz4 for the recovery tests. The material does not say v13 keeps that setting, so the question about suite-wide coverage stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch adds the XLR_COMPRESSED bit in xl_info and a compression header to the WAL record format. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes xlogrecord.h and xlogreader.h, and the streams patch adds XLogBeginReadStreamed() for WAL readers. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Default threshold turns on whole-record compression for every existing wal_compression user |
| new GUC | **yes** | The series adds wal_compression_threshold and, in the WIP patch, wal_compression_streams. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each record gets a physical and a decoded length, and streams add a rule for every reader that reads by LSN. |
| performance on the hot path | **yes** | The WAL insert path in xloginsert.c gains a compression attempt and a second buffer per backend. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. On which realistic workloads, other than a B-tree build on random data, does whole-record compression reduce WAL volume at the 512-byte default, and by how much?
2. Why is whole-record compression on by default for existing lz4 and zstd users, with a large threshold value as the only way back to FPI-only compression?
3. Which in-tree tools and known out-of-tree WAL readers need only block references today, and how do they behave on a compressed record when built without the codec?
4. Can pg_stat_wal keep wal_fpi_bytes no larger than wal_bytes, for example by reporting image bytes before compression in a new column?
5. Which test configuration runs the recovery and tool suites with whole-record compression on, and with streams on, so a wrong length or a direct read by LSN fails in CI?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier committed the zstd context reuse split out of this series, but in message #25 gave no opinion on whole-record compression or streams. The owner must know xloginsert, xlogreader and recovery.
- **Liability:** The owner signs for a WAL record format change read by recovery, standbys, pg_rewind, pg_waldump, logical decoding and out-of-tree readers, for every later release. With streams, the owner also signs for the rule on records read by LSN.
- **Author follow-up:** The author has posted 13 versions since January 2025, rebased across seven CommitFests and answered the latest review in v13.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Users of wal_compression with lz4 or zstd gain, with 15% to 19% less WAL shown on one index build. The patch changes the WAL record format and the insert and reader paths, so the default, the opt-out and the pg_stat_wal meaning should be settled before detailed review.

## Appendix

### Held up in the counter-check, not listed above

- `replication-ondisk-2` (fixable, level 2) Stream mode depends on an unenforced rule about LSN-addressed reads. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `security-1` Shared compression streams may leak other sessions' data through WAL size. Refuted (wrong facts): Message #16 says a backend that takes over a lease restarts the stream, because compressor state is backend-local. Message #35 keeps one compressor per writer, so the material does not show a record compressed against another backend's records.
- `maintenance-fit-1` Streams make every future read-by-LSN path depend on an exclusion rule. Refuted (not specific): Duplicate of replication-ondisk-2, which states the same rule and scenario with the fitting severity. A missing test-time check in a WIP patch is fixable.
- `maintenance-fit-3` Existing wal_compression users get new memory and CPU costs by default. Refuted (not specific): Duplicate of standards-compat-1, which already covers the default threshold, the second buffer and the CPU cost for existing wal_compression users. The request for many-connection measurements belongs with that objection.
- `already-exists-1` Archive-side compression already captures cross-record redundancy for stored WAL. Refuted (no failure scenario): Compressing archived segments cannot reduce WAL written to pg_wal or sent to standbys, which is what the existing wal_compression setting targets. An operator who gains little from a setting they can raise or turn off is not a failure.
- `demand-2` Archive-side compression already captures cross-record redundancy for stored WAL. Refuted (no failure scenario): wal_compression_streams is off by default in a WIP patch, so the throughput cost and the reader rewind fall only on clusters that enable it. Item 4 fails when everyone pays, and the scenario describes an operator who opted in.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] is a delta format for generic WAL and was returned for lack of review time, and [P2] proposed pg_stat_wal counters, which this proposal does not add. [P4] and [P10] record no reason. The thread cites only a related wal_compression default discussion (messages #17 and #28), not an earlier attempt at whole-record compression.
- standards-compat: 2 objection(s). The WAL format change itself is not objected to: WAL is tied to a major version, and standbys and pg_upgrade do not carry WAL across versions. The WIP streams GUC was not judged.
- replication-ondisk: 2 objection(s). 0002 keeps each record decodable on its own and leaves pg_upgrade and the catalog unchanged. The page LSN concern in message #21 seems to concern another patch, so it was not used.
- security: 1 objection(s). 0002 alone adds little new exposure: records stay independent, and message #20 caps the decompressed length at XLogRecordMaxSize. The GUC contexts for the new settings are not shown, so I could not check who can set them.
- maintenance-fit: 3 objection(s). The context-reuse part is committed. These objections apply to 0002 and 0003 only.
- already-exists: 1 objection(s). Only one objection from this angle. Per-FPI wal_compression cannot use redundancy across images, and core has no compression of the replication stream, so no existing feature fully replaces the proposal.
- demand: 2 objection(s). Interest in lower WAL volume is broad, and wal_compression already exists. The objections ask for workload evidence, not a rejection.

### Precedents retrieved

- P1: [Generic WAL compression](https://commitfest.postgresql.org/patch/1363/) (cf, Returned with feedback, 2018-03-21) - used in the review; reason summarised from its thread
- P2: [Tracking the compression effect](https://commitfest.postgresql.org/patch/3851/) (cf, Withdrawn, 2022-10-31) - used in the review; reason summarised from its thread
- P3: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant; reason summarised from its thread
- P4: [Compression dictionaries](https://commitfest.postgresql.org/patch/3626/) (cf, Returned with feedback, 2024-01-26) - used in the review
- P5: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3668/) (cf, Withdrawn, 2022-07-20) - judged not relevant; reason summarised from its thread
- P6: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - judged not relevant; reason summarised from its thread
- P7: [Move walreceiver state assignment (to WALRCV_STREAMING) in WalReceiverMain()](https://commitfest.postgresql.org/patch/4698/) (cf, Withdrawn, 2024-01-22) - judged not relevant; reason summarised from its thread
- P8: [pg_basebackup: add test about zstd compress option](https://commitfest.postgresql.org/patch/3835/) (cf, Rejected, 2022-12-06) - judged not relevant; reason summarised from its thread
- P9: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P10: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - used in the review
- P11: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant
- P12: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P13: [Adding CommandID to heap xlog records](https://commitfest.postgresql.org/patch/3882/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P14: [Add WAL recovery info such as current WAL file and WAL source to server logs](https://commitfest.postgresql.org/patch/3443/) (cf, Withdrawn, 2022-10-26) - judged not relevant
- P15: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P16: [Page compression for OLTP](https://commitfest.postgresql.org/patch/3783/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P17: [Making WAL receiver startup rely on GUC context for primary_conninfo and primary_slot_name](https://commitfest.postgresql.org/patch/1916/) (cf, Rejected, 2019-02-16) - judged not relevant
- P18: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P19: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P20: [Reduce wakeup on idle for bgwriter & walwriter for >5s](https://commitfest.postgresql.org/patch/4029/) (cf, Withdrawn, 2023-03-30) - judged not relevant
- P21: [Compensate for full_page_writes when spreading checkpoint I/O](https://commitfest.postgresql.org/patch/283/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P22: [Changing WAL Header to reduce contention during ReserveXLogInsertLocation()](https://commitfest.postgresql.org/patch/1455/) (cf, Returned with feedback, 2018-07-09) - judged not relevant
- P23: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - judged not relevant
- P24: [Making autovacuum logs indicate if insert-based threshold was the triggering condition](https://commitfest.postgresql.org/patch/3802/) (cf, Withdrawn, 2022-11-27) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 199 s; cost reported by the CLI: $3.48
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 104 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:33:58.678Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
