# Design pre-review: Compress big WAL records

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5621](https://commitfest.postgresql.org/patch/5621/); [thread](https://www.postgresql.org/message-id/flat/4DC38068-976E-4A84-8EE6-4EFACBBD927A%40yandex-team.ru), 37 of 37 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and none stops review at level 1 or 2. The goal extends the existing wal_compression setting, and the open points are fixable: the default and opt-out (standards-compat-1), the codec needed to read block references (replication-ondisk-1), and workload evidence (demand-1). The WAL record format change and the WIP streams rule (replication-ondisk-2) need a committer who knows xlogreader and recovery to sign.

Not checked: Without the code, I could not check that every reader masks XLR_COMPRESSED and uses the right length, or whether the page magic changes. The truncated messages #16 and #17 hide the stream lock cost.

## Prior attempts

None found before 2025-01-12.

## Objections a reviewer is likely to raise

### 1. Gain for whole-record compression is shown on one synthetic workload only

A user turns on zstd with the default 512-byte threshold on an OLTP cluster and expects lower WAL volume. In that workload few records exceed the threshold, so the change in WAL volume is small. Meanwhile the record format change, the second compression buffer and the extra reader path are permanent for every release and every WAL tool.

The only measured benefit of 0002 is CREATE INDEX over 10M random doubles: 19% less WAL with zstd and 15% with lz4 (messages #4, #17). The 2PC case from message #1 has no numbers. No field report, -general thread or production WAL sample shows how much real WAL sits in large multi-FPI records. Michael Paquier committed the context-reuse parts but had no opinion on 0002 and 0003 (message #25), and the other interested reviewers (messages #6, #21) state liking the idea without giving a workload.

**Question:** On which production or realistic workloads, other than a B-tree build on random data, does 0002 reduce WAL volume, and by how much at the default threshold?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/4DC38068-976E-4A84-8EE6-4EFACBBD927A%40yandex-team.ru), [#4](https://www.postgresql.org/message-id/73E8D8A0-F5C3-45E8-8DFC-60AA86F8B890%40yandex-team.ru), [#17](https://www.postgresql.org/message-id/A2BB6471-C150-423C-B6F5-BFEC46BE544B%40yandex-team.ru), [#25](https://www.postgresql.org/message-id/asRsOl73nf8GeBFd%40paquier.xyz); `demand-1`</sub>

### 2. Default threshold turns on whole-record compression for every existing wal_compression user

A site runs wal_compression = on and upgrades. With the 512-byte default, large records are now compressed whole and need a second buffer per backend, and WAL size and CPU change. To get the old behaviour back, the DBA has to find a second GUC and set it to about 1GB.

In message #11 Fujii Masao asked to keep FPI-only compression as a real choice, and message #12 agreed to complement it. But the opt-out in message #13 is a magic threshold value, with no explicit setting. The 512 default has no measured basis (message #7), and message #16 shows a low threshold can make WAL larger. wal_compression already selects the method, so a committer would likely ask why the scope is not chosen there.

**Question:** Why is whole-record compression on by default for existing wal_compression users, and why is the opt-out a large threshold value with no explicit off setting?

<sub>fixable, level 2, new GUC; messages [#7](https://www.postgresql.org/message-id/87E092DC-0E61-426A-8F73-B61A19BCC00D%40yandex-team.ru), [#11](https://www.postgresql.org/message-id/CAHGQGwG7krZ%2Bpktg9c2e8xk-6TK-sK0kMSWriSQtz%3DJvJ_E6Pg%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/19C9772F-9377-4ABA-88CF-F5B7C35C5A4C%40yandex-team.ru), [#13](https://www.postgresql.org/message-id/4D8E3280-632B-4C6C-A728-835B7FDE6325%40yandex-team.ru), [#16](https://www.postgresql.org/message-id/33EDA443-E905-4E15-A7B0-F734C369A19B%40yandex-team.ru); `standards-compat-1`</sub>

### 3. Whole-record compression hides block references behind the codec

A cluster runs with wal_compression = on, which now picks zstd first, so every record over 512 bytes is compressed as one unit. A pg_rewind binary, or an out-of-tree WAL reader built without zstd, can no longer parse the block headers of those records. It fails during a failover rewind or an incremental backup scan, not when the setting is changed.

Today a reader without lz4 or zstd can still decode record headers and block references and fails only on restoring an image. With 0002 the block headers sit inside the compressed payload (message #1), so codec support decides whether the record can be read at all. The default threshold turns this on for every existing wal_compression = lz4, zstd or on user, with no opt-in (messages #7, #20). The material also does not say whether XLOG_PAGE_MAGIC is bumped, so older tools may misread records instead of rejecting them (message #3).

**Question:** Why should whole-record compression be on by default for existing wal_compression users, and how does a reader without the codec report the problem instead of misparsing?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/4DC38068-976E-4A84-8EE6-4EFACBBD927A%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/ECCAD8F2-032D-42FE-B60D-0D6EB0423FF2%40yandex-team.ru), [#7](https://www.postgresql.org/message-id/87E092DC-0E61-426A-8F73-B61A19BCC00D%40yandex-team.ru), [#20](https://www.postgresql.org/message-id/04119E77-9C0F-415A-A9E0-65E6E4A57940%40yandex-team.ru); `replication-ondisk-1`</sub>

### 4. wal_fpi_bytes exceeding wal_bytes changes the meaning of a pg_stat_wal column

A monitoring dashboard plots wal_fpi_bytes / wal_bytes as the full-page-image share of WAL. After upgrade the ratio goes above 1, as in message #33 (90 MB against 10 MB), and alerts or capacity plans built on it give wrong answers. Users learn of the change only from a release-notes line or the documentation.

Today wal_fpi_bytes is a subset of wal_bytes, and external tools depend on that. Message #35 keeps the pre-compression count, documents that it can exceed wal_bytes, and says the jointly compressed bytes cannot be split. That reason holds, but the result is one view whose two byte columns count in different units. A committer would likely ask for a separate column, or for wal_fpi_bytes to stay physical.

Precedent: [Tracking the compression effect](https://www.postgresql.org/message-id/flat/0489c74a83dbd54ed33feb6690d7ca4f@oss.nttdata.com) (Withdrawn, 2022-10-31)

**Question:** Can pg_stat_wal keep wal_fpi_bytes no larger than wal_bytes, for example by reporting pre-compression image bytes in a new column instead?

<sub>fixable, level 2, design; messages [#33](https://www.postgresql.org/message-id/CAN4CZFOgbHQFvmBD%3DTo7Hjv1fJ2QMJn0pM-2HcPvH4dqkehxcQ%40mail.gmail.com), [#35](https://www.postgresql.org/message-id/D815DE2E-6EEB-4EF4-8B5F-3374F552D55C%40yandex-team.ru); `standards-compat-2`</sub>

### 5. Two record lengths that every future xlogreader change must choose between

A later patch adds a sanity check, a statistic or a buffer size based on xl_tot_len in xlogreader or pg_waldump. It is correct for uncompressed records, which is all the default test setup produces. With whole-record compression on, it checks or allocates the wrong side, and this shows up as a decode failure or a wrong report in the field.

0002 splits each record into a physical length and a decoded length. Message #20 shows a rebase across 18992dc9d98 that needed a choice of side, and a plain textual rebase compiled against a variable that no longer existed. Message #36 found replorigin_identity reading xl_info without masking XLR_COMPRESSED. v13 clears the flag after decompression, but nothing makes the wrong choice of length fail loudly.

**Question:** Which test configuration runs the recovery and tool suites with whole-record compression on, so that a check on the wrong length fails in CI?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#20](https://www.postgresql.org/message-id/04119E77-9C0F-415A-A9E0-65E6E4A57940%40yandex-team.ru), [#33](https://www.postgresql.org/message-id/CAN4CZFOgbHQFvmBD%3DTo7Hjv1fJ2QMJn0pM-2HcPvH4dqkehxcQ%40mail.gmail.com), [#35](https://www.postgresql.org/message-id/D815DE2E-6EEB-4EF4-8B5F-3374F552D55C%40yandex-team.ru), [#36](https://www.postgresql.org/message-id/CAN4CZFOhMg9oyKUSeK4VhdB1B-cOjS6qUgTk9HCDnbMg9pLQHQ%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. On which realistic workloads, other than a B-tree build on random data, does whole-record compression reduce WAL volume at the 512-byte default, and by how much?
2. Why is whole-record compression on by default for existing lz4 and zstd users, with a large threshold value as the only way back to FPI-only compression?
3. Which in-tree tools and known out-of-tree WAL readers need only block references today, and how do they behave on a compressed record when built without the codec?
4. Can pg_stat_wal keep wal_fpi_bytes no larger than wal_bytes, for example by reporting image bytes before compression in a new column?
5. Which test configuration runs the recovery and tool suites with whole-record compression on, and with streams on, so a wrong length or a direct read by LSN fails in CI?

## Touch points

- **WAL / replication / standby:** The patch adds the XLR_COMPRESSED bit in xl_info and a compression header to the WAL record format.
- **ABI / extension API:** The patch changes xlogrecord.h and xlogreader.h, and the streams patch adds XLogBeginReadStreamed() for WAL readers.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Default threshold turns on whole-record compression for every existing wal_compression user
- **new GUC:** The series adds wal_compression_threshold and, in the WIP patch, wal_compression_streams.
- **maintenance burden ("who maintains this in 10 years"):** Each record gets a physical and a decoded length, and streams add a rule for every reader that reads by LSN.
- **performance on the hot path:** The WAL insert path in xloginsert.c gains a compression attempt and a second buffer per backend.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier committed the zstd context reuse split out of this series, but in message #25 gave no opinion on whole-record compression or streams. The owner must know xloginsert, xlogreader and recovery.
- Priority: fix design first; audience a sizeable minority; review cost high. Users of wal_compression with lz4 or zstd gain, with 15% to 19% less WAL shown on one index build. The patch changes the WAL record format and the insert and reader paths, so the default, the opt-out and the pg_stat_wal meaning should be settled before detailed review.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
