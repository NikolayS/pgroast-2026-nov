# Design pre-review: pg_dump flag for dumping tables data in multiple chunks, for huge tables can often give near-linear speedup when parallelized

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6219](https://commitfest.postgresql.org/patch/6219/) (Needs review); authors: Hannu Krosing (hannukrosing)
- Thread: [Patch: dumping tables data in multiple chunks in pg_dump](https://www.postgresql.org/message-id/flat/CAMT0RQT_0qVxcTT6ycM20QUN-pEQ6iMLbz6gLWgLpeF0NmNOUA%40mail.gmail.com), started 2025-11-11
- Messages read: 46 of 46
- Patch prose read: v17--0001-Add-max-table-segment-pages-option-to-pg_dump-fo.patch, 0001-adds-ability-to-dump-data-for-tables-in-multiple-chu.patch, README.pg_dump.md (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is clear and measured, and no surviving objection is against the idea. All four surviving objections are fixable level 2 points about restore correctness and the archive: replication-ondisk-2 and standards-compat-1 concern lost rows on restore, and replication-ondisk-1 concerns the unchanged archive version. Message #45 showed data loss in v16, and no reviewer has confirmed the v17 fix. A committer would likely want these answered before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check how v17 keeps depCount correct, which desc comparisons remain, or how segments behave under the truncate and WAL-skip paths. It also could not check snapshot behaviour across segments or formats other than directory.

## The proposal, restated

**Problem.** A parallel pg_dump gains little when one or a few huge tables hold most of the data, because each table is dumped by a single worker. The author also cites file size limits on target file systems, such as 4GB on FAT and 5TB on GCS.

**Proposed design.** A new pg_dump option splits the data of heap tables larger than a page threshold into segments of that many main-fork pages. pg_dump dumps each segment as its own TABLE DATA entry with a ctid range query, so parallel workers can share one table. pg_restore loads all segments into the same table, and v17 adds restore dependencies so indexes and constraints wait for all segments.

**User-visible surface.**
- pg_dump option --max-table-segment-pages=npages, first named --huge-table-chunk-pages.
- TOC entries labelled "TABLE DATA (pages N:M)", one per segment, visible in pg_restore --list.
- Segmented tables produce several data files per table in a directory-format dump.
- New frontend helper option_parse_uint32() in fe_utils/option_utils.

**What the author claims.**
- A 408GB table without TOAST dumped about 7 times faster with 8 workers and 1GB segments, 40 minutes down to 6 minutes.
- Speedup stopped at 16 workers because disk read throughput of about 4TB/hour became the limit.
- A 53GB mostly-TOAST table dumped in 30 seconds with 16 workers and 1000-page segments, against 240 seconds serially.
- pg_restore needed no format change, since all segments are tied to the table in the TOC.
- Segmenting works on servers before version 14, but without efficient TID range scans.

**Thread state.** Rowley, Kumar, Parragi, Stehule and Banck reviewed it, and Rowley said he would drop out if quality did not improve by v11. Parragi found an assertion failure and data loss in parallel restore in v16, and v17 claims a fix that no one has yet confirmed.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-11-11; 19 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Restore of several data entries into one table breaks the truncate-before-load invariant

fixable · level 2, rubric item 7 · WAL / replication / standby, pg_dump / restore round-trip · `replication-ondisk-2`

An operator runs a parallel or --clean restore of a segmented dump. One segment's load path truncates the table, or uses the created-in-this-transaction WAL-skip optimization, after another segment has already loaded rows. Rows go missing, and the operator learns this only when the restored data is checked.

pg_restore assumes one TABLE DATA entry per table. It uses that to decide when a table may be truncated, and when a COPY may skip WAL under wal_level=minimal. Message #37 reported truncation problems once segments were treated as table data. Message #45 showed an assertion failure and data loss in v16, and nobody has confirmed the v17 fix from message #46. The proposal does not state which invariant now guarantees that only the first segment truncates or skips WAL.

Thread: [#37 (Zsolt Parragi, 2026-02-03)](https://www.postgresql.org/message-id/CAN4CZFMHr0gFx-DpKBy4SkVzuJC5f3S%3DpnkrMDC54OswBSuJNQ%40mail.gmail.com), [#45 (Zsolt Parragi, 2026-03-30)](https://www.postgresql.org/message-id/CAN4CZFN%3DiKYAhaki0eC-ADutk-aB3B5jOV26FrTewWouQiD6jQ%40mail.gmail.com), [#46 (Hannu Krosing, 2026-08-06)](https://www.postgresql.org/message-id/CAMT0RQTCipoCSHVZ7UamMZ9JnF-prN37hZNqrDwzQr5CpgJrzw%40mail.gmail.com)

**Question:** Which rule in v17 ensures that only one segment per table truncates or uses the WAL-skip path, and which TAP test covers --clean, --single-transaction and --jobs together?

Counter-check: held. Parallel pg_restore truncates a table it created before loading its data entry, so a second segment can remove rows a first one loaded. Message #37 saw this, and message #39 answers only "should solve", with no reviewer confirmation.

### 2. Page range stored in the TOC desc field breaks desc-based matching

fixable · level 2, rubric item 12 · pg_dump / restore round-trip, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A user restores a segmented dump with pg_restore -t big_table, -L from an edited list, or a pg_restore that does not know the new desc. Code that compares desc with "TABLE DATA" skips or misorders the segment entries. Some or all of the table's rows are missing after restore, and nothing reports an error.

The patch writes segment entries with desc "TABLE DATA (pages N:M)". Message #37 found that pg_backup_archiver.c tests desc for the exact string "TABLE DATA" in several places. v17 fixes dependency scheduling, but the proposal does not say whether every other desc check was audited, such as table selection, section and data-only filtering, and truncation. Encoding a parameter in desc departs from how the TOC records entry types today, so a later strcmp(desc, "TABLE DATA") can miss segments without any test failing.

Thread: [#37 (Zsolt Parragi, 2026-02-03)](https://www.postgresql.org/message-id/CAN4CZFMHr0gFx-DpKBy4SkVzuJC5f3S%3DpnkrMDC54OswBSuJNQ%40mail.gmail.com), [#45 (Zsolt Parragi, 2026-03-30)](https://www.postgresql.org/message-id/CAN4CZFN%3DiKYAhaki0eC-ADutk-aB3B5jOV26FrTewWouQiD6jQ%40mail.gmail.com), [#46 (Hannu Krosing, 2026-08-06)](https://www.postgresql.org/message-id/CAMT0RQTCipoCSHVZ7UamMZ9JnF-prN37hZNqrDwzQr5CpgJrzw%40mail.gmail.com)

**Question:** Which pg_restore paths match desc against "TABLE DATA", and does each one handle segment entries, including archives read by an older pg_restore of the same archive version?

Counter-check: held. Message #37 confirms that pg_restore compares desc with the exact string "TABLE DATA" in several places. The v17 commit message covers only depCount and --inserts, so selection and filtering paths are unanswered.

### 3. Segment identity lives only in the TOC desc string, not the archive format

fixable · level 2, rubric item 12 · on-disk format / pg_upgrade, pg_dump / restore round-trip, backpatchability · `replication-ondisk-1`

A user restores a segmented dump with a pg_restore that accepts the same archive version but predates this patch. That pg_restore matches "TABLE DATA" exactly, so it does not treat "TABLE DATA (pages N:M)" entries as table data. Indexes and constraints may then build before all segments load, and --data-only or --section filtering can misclassify the segments.

Message #1 says no pg_restore format change is needed, and the README in message #1 shows the dump version unchanged at 1.16-0. Message #37 found that pg_restore keys dependency tracking on the literal "TABLE DATA" desc. In message #39, v17 rebuilds segment dependencies at restore time, so only a new pg_restore knows that segments belong together. Third-party tools that parse pg_restore --list or the TOC face the same mismatch.

Thread: [#1 (Hannu Krosing, 2025-11-11)](https://www.postgresql.org/message-id/CAMT0RQT_0qVxcTT6ycM20QUN-pEQ6iMLbz6gLWgLpeF0NmNOUA%40mail.gmail.com), [#37 (Zsolt Parragi, 2026-02-03)](https://www.postgresql.org/message-id/CAN4CZFMHr0gFx-DpKBy4SkVzuJC5f3S%3DpnkrMDC54OswBSuJNQ%40mail.gmail.com), [#39 (Hannu Krosing, 2026-03-28)](https://www.postgresql.org/message-id/CAMT0RQSwbGpf4WSCb41dcyvHUEYAnVTYxdJ6SqLYf7HiUag%2BTA%40mail.gmail.com), [#46 (Hannu Krosing, 2026-08-06)](https://www.postgresql.org/message-id/CAMT0RQTCipoCSHVZ7UamMZ9JnF-prN37hZNqrDwzQr5CpgJrzw%40mail.gmail.com)

**Question:** Will the archive version be bumped, or segment membership stored in the TOC, so that an older pg_restore rejects or handles these archives?

Counter-check: held. The README shows the archive version unchanged at 1.16, so a released pg_restore would accept a segmented dump without knowing the new desc. The thread does not discuss a version bump; the third-party tools point adds little.

### 4. Option unit is heap pages, an internal detail in a user tool

fixable · level 2, rubric item 10 · design, pg_dump / restore round-trip · `standards-compat-2`

An administrator sets --max-table-segment-pages=131072 to get 1GB segments, as in message #3, against a server built with 32kB blocks. Each segment is about 4GB, and files exceed the FAT limit that motivated the option. The administrator finds out when a copy to the target file system fails.

Message #40 asked for a size such as '10GB', and message #41 kept pages because heap pages do not predict output size. But the page count also depends on the server's block_size, which the user may not know. No other pg_dump option exposes block counts. A size converted to pages via block_size would keep the author's honest caveat about TOAST and match how users think about file limits.

Thread: [#1 (Hannu Krosing, 2025-11-11)](https://www.postgresql.org/message-id/CAMT0RQT_0qVxcTT6ycM20QUN-pEQ6iMLbz6gLWgLpeF0NmNOUA%40mail.gmail.com), [#3 (Hannu Krosing, 2025-11-13)](https://www.postgresql.org/message-id/CAMT0RQTHoL8S7OonFWC_aDSC-2oX7BGBBLAQ%2BOOBhRPcxV2eiw%40mail.gmail.com), [#40 (Michael Banck, 2026-03-28)](https://www.postgresql.org/message-id/20260328112607.GA3202%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#41 (Hannu Krosing, 2026-03-28)](https://www.postgresql.org/message-id/CAMT0RQRtLwi_CrOcD7KxYL0Gm1nGXb-HWmerVg%3DajEs6JP7m%2Bw%40mail.gmail.com)

**Question:** Why should users reason in server block counts, when a size argument converted with the server's block_size would give the same segmentation with a familiar unit?

Counter-check: held. Message #41 explains why output size cannot be promised, but a size converted with the server's block_size would give the same segmentation. The unit is still open after message #40, though the 32kB FAT scenario is a rare case.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Segmented dumps add a new kind of TOC entry to the archive while the archive version stays at 1.16. |
| WAL / replication / standby | **yes** | Several data entries per table reach the pg_restore truncate and WAL-skip load path used under wal_level=minimal. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_restore must load several "TABLE DATA (pages N:M)" entries into one table before indexes and constraints. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable, and an older pg_restore accepts the archive without knowing the segment entries. |
| maintenance burden ("who maintains this in 10 years") | **yes** | pg_backup_archiver.c code that assumes one data entry per table must now handle segments. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which rule in v17 ensures that only one segment per table truncates the table or uses the WAL-skip path during restore?
2. Which pg_restore paths compare desc with "TABLE DATA", and does each one handle segment entries for -t, -L, --section and --data-only?
3. Will the archive version be bumped, or segment membership stored in the TOC, so that an older pg_restore rejects or handles segmented dumps?
4. Which TAP test reproduces the v16 depCount assertion failure and data loss from message #45 and passes with v17?
5. Why should the option take heap pages when a size converted with the server's block_size would give the same segments?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. David Rowley reviewed the dump side but said he would drop out if quality did not improve. The restore side needs someone who knows parallel pg_restore scheduling in pg_backup_archiver.c.
- **Liability:** The owner signs for restore correctness of every segmented dump, including silent row loss under -j, --clean and filtering. The option name, its unit and the new TOC entries stay while such archives exist.
- **Author follow-up:** The author has posted 17 versions over a year and moved the entry through five CommitFests, so follow-up looks likely.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Users with one or a few huge tables gain a measured speedup of about 7 times, and users who do not set the option pay nothing. The 32 KiB patch changes the parallel restore code in pg_backup_archiver.c, where v16 lost data and the v17 fix is unconfirmed.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Many TABLE DATA entries per table break silent pg_restore assumptions. Refuted (not specific): Duplicate of standards-compat-1 for the desc checks and of replication-ondisk-2 for the truncation path. Those two state the same failures with more concrete scenarios.
- `maintenance-fit-2` Heap page count becomes a permanent pg_dump user contract. Refuted (not specific): Duplicate of standards-compat-2, which states the unit concern with a concrete alternative. The remaining scenario of a future AM or block size change is vague, and skipping non-heap tables follows Rowley's advice in message #16.
- `demand-1` The file size limit use case is not served by a page-based option. Refuted (already handled): The v17 commit message claims only parallel dumping, and the documentation note warns that one heap page can dump as a 200GB file. The file size claim appears only in message #1, and the measured use case does not depend on it.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] and [P4] add a user-supplied --where filter to pg_dump; their recorded reasons were a missing rebase and arbitrary SQL pasted into the query, and neither applies to internally generated ctid ranges. The thread does not point to any earlier attempt at intra-table parallel dump.
- standards-compat: 2 objection(s). Two objections listed, both fixable.
- replication-ondisk: 2 objection(s). The proposal changes no server-side on-disk format, WAL, catalog or wire protocol. Both objections concern the pg_dump archive as persistent state read by pg_restore.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. pg_dump still runs as the connecting role, and each segment query adds only ctid bounds that pg_dump computes from integers, with no user-supplied SQL, so the injection concern recorded in [P4] does not apply. The page count comes from pg_relation_size(), which any role can already call on any relation. Row-level security handling and the locks pg_dump takes are unchanged, and the number of connections is still set by -j. Not checked without the code: whether the --enable-row-security path keeps the user's policies when ctid quals are added.
- maintenance-fit: 2 objection(s). Users who do not set the option pay no hot-path cost, and the feature adds no new dependency. The two objections concern long-run restore correctness and the permanence of the user interface. Without the code, I could not check how v17 keeps depCount correct.
- already-exists: 0 objection(s). No existing Postgres feature splits one table's data across pg_dump workers inside a single archive with restore ordering. A manual route exists: pg_export_snapshot, pg_dump --snapshot and parallel ctid-range COPY sessions. It loses the single archive, needs hand-built restore ordering, and is far more than modest effort, so the new option adds something real. No extension does this. Precedents [P3] and [P4] cover row filtering, not this need.
- demand: 1 objection(s). Demand for parallel dump of one huge table is plausible, and the material backs it with measured speedups of about 7 times on a 408GB table and 8 times on a TOAST-heavy table, plus support from Kumar and Rowley. The cost falls only on users who set the option. The material has no field reports from independent users and no restore timings.

### Precedents retrieved

- P1: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P2: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant; reason summarised from its thread
- P3: [Add --include-table-data-where option to pg_dump, to export only a subset of table data](https://commitfest.postgresql.org/patch/1644/) (cf, Returned with feedback, 2019-01-31) - used in the review; reason summarised from its thread
- P4: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - used in the review; reason summarised from its thread
- P5: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P6: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant; reason summarised from its thread
- P7: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P8: [Revert "pg_dump: Lock all relations, not just plain tables".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d3adaabaf7d555ec8bb1d83c43f72e79f1bf0b7d) (revert, Reverted, 2020-11-06) - judged not relevant
- P9: [Add force flag while doing pg_restore to force dropping database in case of existing connections](https://commitfest.postgresql.org/patch/4485/) (cf, Returned with feedback, 2024-01-14) - judged not relevant; reason summarised from its thread
- P10: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P11: [support tid range scan in parallel](https://commitfest.postgresql.org/patch/4982/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P12: [Tid range scan](https://commitfest.postgresql.org/patch/1755/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P13: [Improve pg_restore toc file parsing and format for better performances](https://commitfest.postgresql.org/patch/4509/) (cf, Returned with feedback, 2024-01-14) - judged not relevant
- P14: [pg_restore cleanups](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=40b9c27014d90d77c61e8f5c77ddeb84fa9a6b69) (revert, Partially reverted, 2025-04-16) - judged not relevant
- P15: [Detect supported SET parameters when pg_restore is run](https://commitfest.postgresql.org/patch/803/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P16: [Parallel Inserts in CREATE TABLE AS](https://commitfest.postgresql.org/patch/2841/) (cf, Withdrawn, 2021-03-20) - judged not relevant
- P17: [Allow setting parallel_workers on partitioned tables](https://commitfest.postgresql.org/patch/2987/) (cf, Returned with feedback, 2021-09-04) - judged not relevant
- P18: [Improving test coverage of extensions with pg_dump](https://commitfest.postgresql.org/patch/187/) (cf, Returned with feedback, 2015-12-02) - judged not relevant
- P19: [Revert "Have dtrace depend on object files directly, not objfiles.txt"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=08fbad0afd62690cc82990c0504529ef238ac24d) (revert, Reverted, 2015-10-15) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 158 s; cost reported by the CLI: $3.32
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 19 kept
- Tone check: 82 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:46:35.183Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
