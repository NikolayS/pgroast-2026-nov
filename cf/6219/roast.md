# Design pre-review: pg_dump flag for dumping tables data in multiple chunks, for huge tables can often give near-linear speedup when parallelized

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6219](https://commitfest.postgresql.org/patch/6219/); [thread](https://www.postgresql.org/message-id/flat/CAMT0RQT_0qVxcTT6ycM20QUN-pEQ6iMLbz6gLWgLpeF0NmNOUA%40mail.gmail.com), 46 of 46 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is clear and measured, and no surviving objection is against the idea. All four surviving objections are fixable level 2 points about restore correctness and the archive: replication-ondisk-2 and standards-compat-1 concern lost rows on restore, and replication-ondisk-1 concerns the unchanged archive version. Message #45 showed data loss in v16, and no reviewer has confirmed the v17 fix. A committer would likely want these answered before code review.

Not checked: Without the code, this review could not check how v17 keeps depCount correct, which desc comparisons remain, or how segments behave under the truncate and WAL-skip paths. It also could not check snapshot behaviour across segments or formats other than directory.

## Prior attempts

None found before 2025-11-11.

## Objections a reviewer is likely to raise

### 1. Restore of several data entries into one table breaks the truncate-before-load invariant

An operator runs a parallel or --clean restore of a segmented dump. One segment's load path truncates the table, or uses the created-in-this-transaction WAL-skip optimization, after another segment has already loaded rows. Rows go missing, and the operator learns this only when the restored data is checked.

pg_restore assumes one TABLE DATA entry per table. It uses that to decide when a table may be truncated, and when a COPY may skip WAL under wal_level=minimal. Message #37 reported truncation problems once segments were treated as table data. Message #45 showed an assertion failure and data loss in v16, and nobody has confirmed the v17 fix from message #46. The proposal does not state which invariant now guarantees that only the first segment truncates or skips WAL.

**Question:** Which rule in v17 ensures that only one segment per table truncates or uses the WAL-skip path, and which TAP test covers --clean, --single-transaction and --jobs together?

<sub>fixable, level 2, WAL / replication / standby; messages [#37](https://www.postgresql.org/message-id/CAN4CZFMHr0gFx-DpKBy4SkVzuJC5f3S%3DpnkrMDC54OswBSuJNQ%40mail.gmail.com), [#45](https://www.postgresql.org/message-id/CAN4CZFN%3DiKYAhaki0eC-ADutk-aB3B5jOV26FrTewWouQiD6jQ%40mail.gmail.com), [#46](https://www.postgresql.org/message-id/CAMT0RQTCipoCSHVZ7UamMZ9JnF-prN37hZNqrDwzQr5CpgJrzw%40mail.gmail.com); `replication-ondisk-2`</sub>

### 2. Page range stored in the TOC desc field breaks desc-based matching

A user restores a segmented dump with pg_restore -t big_table, -L from an edited list, or a pg_restore that does not know the new desc. Code that compares desc with "TABLE DATA" skips or misorders the segment entries. Some or all of the table's rows are missing after restore, and nothing reports an error.

The patch writes segment entries with desc "TABLE DATA (pages N:M)". Message #37 found that pg_backup_archiver.c tests desc for the exact string "TABLE DATA" in several places. v17 fixes dependency scheduling, but the proposal does not say whether every other desc check was audited, such as table selection, section and data-only filtering, and truncation. Encoding a parameter in desc departs from how the TOC records entry types today, so a later strcmp(desc, "TABLE DATA") can miss segments without any test failing.

**Question:** Which pg_restore paths match desc against "TABLE DATA", and does each one handle segment entries, including archives read by an older pg_restore of the same archive version?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#37](https://www.postgresql.org/message-id/CAN4CZFMHr0gFx-DpKBy4SkVzuJC5f3S%3DpnkrMDC54OswBSuJNQ%40mail.gmail.com), [#45](https://www.postgresql.org/message-id/CAN4CZFN%3DiKYAhaki0eC-ADutk-aB3B5jOV26FrTewWouQiD6jQ%40mail.gmail.com), [#46](https://www.postgresql.org/message-id/CAMT0RQTCipoCSHVZ7UamMZ9JnF-prN37hZNqrDwzQr5CpgJrzw%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Segment identity lives only in the TOC desc string, not the archive format

A user restores a segmented dump with a pg_restore that accepts the same archive version but predates this patch. That pg_restore matches "TABLE DATA" exactly, so it does not treat "TABLE DATA (pages N:M)" entries as table data. Indexes and constraints may then build before all segments load, and --data-only or --section filtering can misclassify the segments.

Message #1 says no pg_restore format change is needed, and the README in message #1 shows the dump version unchanged at 1.16-0. Message #37 found that pg_restore keys dependency tracking on the literal "TABLE DATA" desc. In message #39, v17 rebuilds segment dependencies at restore time, so only a new pg_restore knows that segments belong together. Third-party tools that parse pg_restore --list or the TOC face the same mismatch.

**Question:** Will the archive version be bumped, or segment membership stored in the TOC, so that an older pg_restore rejects or handles these archives?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#1](https://www.postgresql.org/message-id/CAMT0RQT_0qVxcTT6ycM20QUN-pEQ6iMLbz6gLWgLpeF0NmNOUA%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/CAN4CZFMHr0gFx-DpKBy4SkVzuJC5f3S%3DpnkrMDC54OswBSuJNQ%40mail.gmail.com), [#39](https://www.postgresql.org/message-id/CAMT0RQSwbGpf4WSCb41dcyvHUEYAnVTYxdJ6SqLYf7HiUag%2BTA%40mail.gmail.com), [#46](https://www.postgresql.org/message-id/CAMT0RQTCipoCSHVZ7UamMZ9JnF-prN37hZNqrDwzQr5CpgJrzw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 4. Option unit is heap pages, an internal detail in a user tool

An administrator sets --max-table-segment-pages=131072 to get 1GB segments, as in message #3, against a server built with 32kB blocks. Each segment is about 4GB, and files exceed the FAT limit that motivated the option. The administrator finds out when a copy to the target file system fails.

Message #40 asked for a size such as '10GB', and message #41 kept pages because heap pages do not predict output size. But the page count also depends on the server's block_size, which the user may not know. No other pg_dump option exposes block counts. A size converted to pages via block_size would keep the author's honest caveat about TOAST and match how users think about file limits.

**Question:** Why should users reason in server block counts, when a size argument converted with the server's block_size would give the same segmentation with a familiar unit?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAMT0RQT_0qVxcTT6ycM20QUN-pEQ6iMLbz6gLWgLpeF0NmNOUA%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAMT0RQTHoL8S7OonFWC_aDSC-2oX7BGBBLAQ%2BOOBhRPcxV2eiw%40mail.gmail.com), [#40](https://www.postgresql.org/message-id/20260328112607.GA3202%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#41](https://www.postgresql.org/message-id/CAMT0RQRtLwi_CrOcD7KxYL0Gm1nGXb-HWmerVg%3DajEs6JP7m%2Bw%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Which rule in v17 ensures that only one segment per table truncates the table or uses the WAL-skip path during restore?
2. Which pg_restore paths compare desc with "TABLE DATA", and does each one handle segment entries for -t, -L, --section and --data-only?
3. Will the archive version be bumped, or segment membership stored in the TOC, so that an older pg_restore rejects or handles segmented dumps?
4. Which TAP test reproduces the v16 depCount assertion failure and data loss from message #45 and passes with v17?
5. Why should the option take heap pages when a size converted with the server's block_size would give the same segments?

## Touch points

- **on-disk format / pg_upgrade:** Segmented dumps add a new kind of TOC entry to the archive while the archive version stays at 1.16.
- **WAL / replication / standby:** Several data entries per table reach the pg_restore truncate and WAL-skip load path used under wal_level=minimal.
- **pg_dump / restore round-trip:** pg_restore must load several "TABLE DATA (pages N:M)" entries into one table before indexes and constraints.
- **backpatchability:** The CommitFest entry targets stable, and an older pg_restore accepts the archive without knowing the segment entries.
- **maintenance burden ("who maintains this in 10 years"):** pg_backup_archiver.c code that assumes one data entry per table must now handle segments.

## Who would commit it, and when

- Owner: qualified committer engaged. David Rowley reviewed the dump side but said he would drop out if quality did not improve. The restore side needs someone who knows parallel pg_restore scheduling in pg_backup_archiver.c.
- Priority: fix design first; audience a sizeable minority; review cost medium. Users with one or a few huge tables gain a measured speedup of about 7 times, and users who do not set the option pay nothing. The 32 KiB patch changes the parallel restore code in pg_backup_archiver.c, where v16 lost data and the v17 fix is unconfirmed.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
