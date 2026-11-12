# Design pre-review: Compress big WAL records

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5621](https://commitfest.postgresql.org/patch/5621/); [thread](https://www.postgresql.org/message-id/flat/4DC38068-976E-4A84-8EE6-4EFACBBD927A%40yandex-team.ru), 37 of 37 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and less WAL is a lasting need. The remaining points are fixable. standards-compat-2 and demand-1 ask for OLTP cost and benefit numbers behind the default-on 512-byte threshold. replication-ondisk-1 asks for a confirmed XLOG_PAGE_MAGIC bump. maintenance-fit-1 and standards-compat-1 concern the two record lengths and the changed meaning of wal_fpi_bytes. A committer who knows the WAL insert and reader paths has to sign the record format.

Not checked: The code was not read. Buffer allocation in critical sections, bounds checks when decompressing, the XLOG_PAGE_MAGIC bump and the GUC contexts could not be checked.

## Prior attempts

None found before 2025-01-12.

## Objections a reviewer is likely to raise

### 1. Existing wal_compression users get whole-record compression by default

A site runs with wal_compression = on, which now means zstd first, and upgrades. Every record over 512 bytes is compressed whole, and each backend holds about 268 KiB more. If OLTP latency rises, the site cannot tell why, because nothing in its configuration changed.

wal_compression has been documented as compression of full-page writes. With a 512-byte default threshold, the patch widens what it does for every current user of lz4 or zstd. Message #7 says 512 was not tested. The brief notes that no CPU or latency numbers for 0001 on OLTP are shown, only WAL size. Message #13 shows a low threshold can make WAL larger, so the default is a real choice that needs data behind it.

**Question:** What OLTP CPU and latency measurements support enabling whole-record compression at 512 bytes by default, rather than leaving it off until a user sets a threshold?

<sub>fixable, level 2, new GUC; messages [#7](https://www.postgresql.org/message-id/87E092DC-0E61-426A-8F73-B61A19BCC00D%40yandex-team.ru), [#13](https://www.postgresql.org/message-id/4D8E3280-632B-4C6C-A728-835B7FDE6325%40yandex-team.ru), [#16](https://www.postgresql.org/message-id/33EDA443-E905-4E15-A7B0-F734C369A19B%40yandex-team.ru), [#25](https://www.postgresql.org/message-id/asRsOl73nf8GeBFd%40paquier.xyz); `standards-compat-2`</sub>

### 2. Gain from whole-record compression is shown on one synthetic workload

A committer takes on a new WAL record format, a new GUC and a changed reader path, and the 15-19% saving appears only on random-data B-tree builds. On typical OLTP WAL, whose records are mostly under 512 bytes, 0001 does nothing. The cost stays even if few users see the gain.

The only measurement for 0001 is CREATE INDEX over 10M random doubles (messages #4 and #17). Message #16 says the records over 512 bytes are 0.8% of pgbench records, and the author points to 0003, which is WIP, for the rest. No field report, -general thread or user in the thread describes WAL volume from large records as their problem. Message #6 liked the idea, but did not report a need.

Precedent: [XLog size reductions: Reduced XLog record header size](https://www.postgresql.org/message-id/flat/CAEze2WgcW5SkPO_TGTqirRcCS_FN0xEZqUsXDE6pPz2TYzXQBA@mail.gmail.com) (Returned with feedback, 2024-04-09)

**Question:** Which real workloads besides random-data index builds produce records above the threshold, and how much WAL does 0001 save on them?

<sub>fixable, level 1b, demand; messages [#4](https://www.postgresql.org/message-id/73E8D8A0-F5C3-45E8-8DFC-60AA86F8B890%40yandex-team.ru), [#6](https://www.postgresql.org/message-id/87d08e95-0e55-4bcf-9f4b-5c9e7b8d91f8%40oss.nttdata.com), [#16](https://www.postgresql.org/message-id/33EDA443-E905-4E15-A7B0-F734C369A19B%40yandex-team.ru), [#17](https://www.postgresql.org/message-id/A2BB6471-C150-423C-B6F5-BFEC46BE544B%40yandex-team.ru); `demand-1`</sub>

### 3. Compressed records hide block references from WAL parsers outside xlogreader

An incremental backup tool that parses raw WAL to collect changed blocks meets an XLR_COMPRESSED record. It does not see the block references inside the compressed body and leaves those pages out of the backup. The restored cluster is missing changes, and the user finds this only after a restore.

The claim that "WAL readers need no new logic" holds only for readers built on the in-core xlogreader. Message #36 shows that in-core code already misread the flag, since replorigin_identity reported records as UNKNOWN. The brief says the material does not show whether XLOG_PAGE_MAGIC is bumped. Message #3 mentions bumping WAL_FILE_MAGIC, and no later message confirms it, so a parser that reads WAL would get no loud signal.

**Question:** Is XLOG_PAGE_MAGIC bumped so that old raw-WAL parsers reject the new format, and how would an external block-tracking tool find the block references in a compressed record?

<sub>fixable, level 2, WAL / replication / standby; messages [#3](https://www.postgresql.org/message-id/ECCAD8F2-032D-42FE-B60D-0D6EB0423FF2%40yandex-team.ru), [#36](https://www.postgresql.org/message-id/CAN4CZFOhMg9oyKUSeK4VhdB1B-cOjS6qUgTk9HCDnbMg9pLQHQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 4. Split between physical and decoded record length burdens every WAL reader

A later patch adds a length check or an xl_info test in xlogreader, an rmgr desc routine or an extension WAL reader, and picks the wrong length or misses the XLR_COMPRESSED bit. Tests pass because wal_compression is off by default, and the bug shows up only on clusters running lz4 or zstd with large records.

Message #20 shows a rebase over an unrelated length check (18992dc9d98) that still compiled, but against a variable that no longer existed. Message #36 found replication origin records reported as UNKNOWN because one routine did not mask the new bit. Both bugs were silent and surfaced only in review. v13 clears the flag after decompression, but every record now has two lengths, and nothing makes the compiler or the default tests catch the wrong one.

**Question:** What makes a future WAL reader or length check that uses the wrong length fail in the default regression run, not only with compression turned on?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#20](https://www.postgresql.org/message-id/04119E77-9C0F-415A-A9E0-65E6E4A57940%40yandex-team.ru), [#36](https://www.postgresql.org/message-id/CAN4CZFOhMg9oyKUSeK4VhdB1B-cOjS6qUgTk9HCDnbMg9pLQHQ%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/17A14E6D-9ECA-4E15-AAF7-1EAED2568788%40yandex-team.ru); `maintenance-fit-1`</sub>

### 5. pg_stat_wal.wal_fpi_bytes can now exceed wal_bytes

A monitoring dashboard computes the FPI share of WAL as wal_fpi_bytes / wal_bytes. After upgrade, with zstd and the default threshold, it reports shares like 845%, as in message #33. The operator finds out from alerts or wrong capacity planning, with no change on their side.

Message #35 keeps wal_fpi_bytes as image bytes before whole-record compression and only documents that it can exceed physical wal_bytes. Today both columns count bytes written, so existing tools read them as parts of one total. The proposal changes what the pair means without a new column or a rename. pg_waldump and pg_walinspect got separate physical and decoded sizes, but pg_stat_wal did not.

**Question:** Why not leave wal_fpi_bytes comparable to wal_bytes and report pre-compression image bytes in a separate column with its own name?

<sub>fixable, level 2, design; messages [#33](https://www.postgresql.org/message-id/CAN4CZFOgbHQFvmBD%3DTo7Hjv1fJ2QMJn0pM-2HcPvH4dqkehxcQ%40mail.gmail.com), [#35](https://www.postgresql.org/message-id/D815DE2E-6EEB-4EF4-8B5F-3374F552D55C%40yandex-team.ru); `standards-compat-1`</sub>

## Questions for the author

1. What CPU and latency cost does 0001 add on pgbench or a similar OLTP workload at the default 512-byte threshold, and how was 512 chosen?
2. Which workloads besides the random-data index build produce records above the threshold, and how much WAL does 0001 save on them compared with HEAD?
3. Is XLOG_PAGE_MAGIC bumped so that WAL parsers that predate the new format reject it instead of misreading compressed records?
4. Why not keep wal_fpi_bytes comparable to wal_bytes and report image bytes before compression in a separate column?
5. For WAL segments smaller than 4MB, does 0003 also restart streams at each segment boundary, or which retention rule keeps the earlier segments?

## Touch points

- **on-disk format / pg_upgrade:** The WAL record format gains the XLR_COMPRESSED bit and a compression header, and 0003 adds 4MB stream restart points.
- **WAL / replication / standby:** Recovery, logical decoding, pg_rewind and pg_waldump must read compressed records; 0003 changes where a reader may start.
- **ABI / extension API:** xlogreader.h and xlogrecord.h change, and 0003 adds XLogBeginReadStreamed() for readers that start mid-WAL.
- **new GUC:** wal_compression_threshold in 0001 and wal_compression_streams in WIP 0003.
- **security / privileges:** Cross-record stream history in 0003 could make WAL size depend on other sessions' data.
- **backpatchability:** see objection replication-ondisk-1: Compressed records hide block references from WAL parsers outside xlogreader
- **maintenance burden ("who maintains this in 10 years"):** Every record has a physical and a decoded length, and 0003 adds an unenforced rule for records read by LSN.
- **performance on the hot path:** Records over 512 bytes are compressed whole in the WAL insert path for every lz4 and zstd user.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Gain from whole-record compression is shown on one synthetic workload

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier committed the zstd context-reuse parts but gave no opinion on whole-record or stream compression. Fujii Masao reviewed the design early. The owner must know xloginsert.c, xlogreader.c and recovery.
- Priority: fix design first; audience a sizeable minority; review cost high. The gain reaches users of lz4 or zstd WAL compression with large records, and is shown on one index-build test. The patch changes the WAL record format and the insert and reader paths, so the default threshold, the format signal and the stats meaning should be settled before detailed review.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
