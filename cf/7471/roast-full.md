# Design pre-review: ZSTD TOAST compression, and an extensible compression method encoding

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7471](https://commitfest.postgresql.org/patch/7471/) (Needs review); authors: Nikhil Kumar Veldanda (nikhilkumarv)
- Thread: [ZSTD TOAST compression, and an extensible compression method encoding](https://www.postgresql.org/message-id/flat/CAFAfj_GGY4W81XjtYLvYOhqPtvfZbBNj%3DYRS6Mq_0SA9AqEwoQ%40mail.gmail.com), started 2026-09-22
- Messages read: 48 of 48
- Patch prose read: v7-0003-Add-zstd-as-a-TOAST-compression-method.patch, v7-0001-Decode-the-compression-method-in-toast_external_i.patch, v7-0002-Allow-more-than-four-TOAST-compression-methods.patch, v2-0002-Allow-more-than-four-TOAST-compression-methods.patch, v2-0003-Add-zstd-as-a-TOAST-compression-method.patch, v2-0001-Refactor-TOAST-compression-header-handling.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection concerns the long-form method ID or plain zstd (0001-0003), and 0002 answers the reason recorded in P3. The dictionary and evolution patches carry open design points. A missed containment check leaves stored values unreadable (maintenance-fit-1), heap data embeds pg_toast_dictionary OIDs while a generic store is still under discussion (replication-ondisk-2), and retirement needs a rewrite (prior-attempts-1). All are fixable, but the list should settle them, and the split of the series, before code review of 0004-0005.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that pglz and lz4 data stays bit for bit the same, and the completeness of the containment checks, rest on the thread's reports. Only 3 of 48 messages were visible here; later messages are known through the objections.

## The proposal, restated

**Problem.** A compressed varlena and an external TOAST pointer store the compression method in two bits, and pglz, lz4 and the invalid ID use three of the four values. Adding zstd would take the last value and block any later method, including dictionary-based ones. The target users want better compression of large values than pglz or lz4 give.

**Proposed design.** Method bits value 3 now means "long form": the method ID sits in an extra byte after the header (inline) or after the pointer (two new vartags), for methods other than pglz and lz4, so up to 255 IDs fit. Zstd becomes method ID 2 on that form. Later patches, added by a reviewer, add column-local zstd and lz4 dictionaries in a new catalog and optional retraining during ANALYZE.

**User-visible surface.**
- COMPRESSION zstd in create table and alter table, and default_toast_compression = 'zstd'; attcompression value 'z'.
- On-disk format: new long-form compressed header (9 bytes) and vartags VARTAG_ONDISK_OID_LONG and VARTAG_ONDISK_OID8_LONG; TOAST_INVALID_COMPRESSION_ID moves from 2 to 3.
- Methods zstd_dict and lz4_dict (attcompression 'd', 'e'), catalog pg_toast_dictionary, and function pg_train_toast_dictionary(rel, col, dict_size).
- Per-column attribute option dictionary_evolution and PGC_SUSET GUCs such as dictionary_evolution_min_interval and dictionary_evolution_growth.
- C API: VARATT_EXTINFO_GET_COMPRESS_METHOD() removed in favor of toast_external_data.compress_method; amcheck checks the pointer form.

**What the author claims.**
- Existing pglz and lz4 data keeps its representation bit for bit; a reviewer confirmed this by hashing pages and running pg_upgrade.
- Zstd stores test corpora at about 27% of raw size against 36% for pglz and 41% for lz4, and saves about a quarter on base64 text.
- Zstd reads of 4 kB values cost about 2.6 times lz4, and a prefix costs as much as the whole value below 128 kB.
- Reusing the zstd decompression context makes small slices of large values about 25 to 40% faster.
- Dictionaries raise lz4 from about 1.9x to 2.5x on 3 kB docs, and evolution recovers about 8% after data drift.

**Thread state.** Michael Paquier committed two refactoring patches and wants the format change held for v21, after the oid8 work settles. Greg Burd now carries the series (v16) with dictionaries and evolution; Hannu Krosing calls 0001-0003 ready for committer, and his generic encoding-context proposal overlaps the dictionary design.

## 2. Prior attempts

### Zstandard support for toast compression (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/3639/](https://commitfest.postgresql.org/patch/3639/)
- **Thread:** [https://www.postgresql.org/message-id/flat/YoMiNmkztrslDbNS@paquier.xyz](https://www.postgresql.org/message-id/flat/YoMiNmkztrslDbNS@paquier.xyz)
- **Date:** decided 2022-06-16; started 2022-05-17
- **Outcome:** Withdrawn (cf)
- **Who objected:** Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Robert Haas saw no urgency and noted that a method added to one of the few ID bit patterns can never be removed. He proposed reserving the fourth pattern for an extensible scheme first, and Michael Paquier agreed the extensible header should come before zstd. The thread then turned to designing that header, and the visible messages do not show the withdrawal itself.
- **Reason (quote):** "If we decide not to add this now, we can always change our minds and add it later, but if we decide to add it now, there's no backing it out." ([message](https://www.postgresql.org/message-id/CA%2BTgmobZkCvpUJAh63XkCPRN3aYYbEEcMTY%3DcqmZdGRLZNbyCg%40mail.gmail.com))
- **What is different this time:** This series first spends the last bit pattern on a long form with a method byte, as the earlier thread asked, and only then adds zstd as ID 2 of 255.

### Compression dictionaries (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3626/](https://commitfest.postgresql.org/patch/3626/)
- **Date:** decided 2024-01-26; started 2022-04-22
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** The dictionaries here are column-local and are retired by rewrite. The precedent row records no reason, so the difference cannot be checked.

### Toast compression method options (related)

- **Link:** [https://commitfest.postgresql.org/patch/3102/](https://commitfest.postgresql.org/patch/3102/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAFiTN-vMHU_yakMgNo90SUih_6LtvmqZ5hpQb2iTgQtVyOjyFA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAFiTN-vMHU_yakMgNo90SUih_6LtvmqZ5hpQb2iTgQtVyOjyFA@mail.gmail.com)
- **Date:** decided 2021-10-01; started 2021-05-03
- **Outcome:** Withdrawn (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier saw no practical use case. Nobody had asked to tune pglz, LZ4 is already cheap in CPU, and users who want to avoid compression cost can store values externally. He also objected that the patch adds a pg_attribute column and complexity to the ALTER TABLE code paths in tablecmds.c. The author withdrew the patch, citing that objection and little interest from other hackers.
- **Reason (quote):** "I don't recall seeing much on this list that users would like to have such a level of tuning for pglz, while lz4 only offers the option to reduce the compression rate while being already very cheap in CPU so the impact is limited." ([message](https://www.postgresql.org/message-id/YVa028Lj99Eqtckt%40paquier.xyz))
- **What is different this time:** Compression level stays out of this series, but 0005 adds per-column options and SUSET GUCs to tune dictionary evolution, which is a similar tuning surface.


## 3. Objections a reviewer is likely to raise

### 1. Dictionary containment depends on a predicate every future datum path must add

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch adds a path that copies or holds datums across statements and tests only VARATT_IS_EXTERNAL(). An inline zstd_dict datum escapes its column. After the next VACUUM FULL retires the dictionary, reads fail with "compression dictionary N does not exist", long after the escape happened.

0004 makes inline dictionary datums safe only by adding VARATT_IS_COMPRESSED_DICT() next to VARATT_IS_EXTERNAL() at each spot that copies a datum. Review has already found escapes in a PL/pgSQL expanded record (message #38), indirect pointers and datumTransfer() (message #41), and check_domain_for_new_field() (message #44), and Manu counted 141 predicate uses to audit (message #45). Nothing fails loudly when a new site forgets the check, and the result is unreadable stored data. Extension code that holds datums has the same exposure.

Thread: [#38 (Hannu Krosing, 2026-10-29)](https://www.postgresql.org/message-id/CAMT0RQS-xv0CzmDz8xwvLRbChs7D8tkb4CEvbSD7rjEaQstUsg%40mail.gmail.com), [#41 (Greg Burd, 2026-10-30)](https://www.postgresql.org/message-id/auTdVHKihzViHELQ%40floki), [#44 (Hannu Krosing, 2026-11-02)](https://www.postgresql.org/message-id/CAMT0RQTWgNJ1%3DQLbmr5rqtsFM5-6Xr0iuen5y4xipG%3DOm_pe8A%40mail.gmail.com), [#45 (Manu, 2026-11-03)](https://www.postgresql.org/message-id/179367850497.1374811.8378346906553509584%40gmail.com)

**Question:** What makes a new datum-copying path that omits VARATT_IS_COMPRESSED_DICT() fail in tests, before a dictionary retirement makes stored values unreadable?

Counter-check: held. Messages #38, #42 and #44 each found a new escape, and nothing in the series makes a missed site fail before retirement. A store-time or amcheck check for nested dictionary datums could close this in a new version, so it is fixable.

### 2. Heap bytes embed pg_toast_dictionary OIDs before the catalog design is settled

fixable · level 2, rubric item 6 · catalog changes, on-disk format / pg_upgrade · `replication-ondisk-2`

v20 ships zstd_dict, and user tables fill with datums carrying a pg_toast_dictionary OID after va_cmid 4. A later release adopts a generic store such as pg_encoding_context. pg_upgrade keeps heap files as they are, so the new release must keep reading pg_toast_dictionary indefinitely or require every user to rewrite their tables.

Message #35 calls re-expressing the dictionary catalog on a generic store "a mechanical change". That holds for code but not for data already on disk, which holds the dictionary OID, method ID 4 and attcompression 'd'/'e'. Hannu's overlapping design is still under discussion (messages #29, #39), and Michael's concern about retaining past dictionaries (message #2) concerns this same on-disk dependency. 0001-0003 do not need this commitment, and 0004 makes it permanent.

Thread: [#2 (Michael Paquier, 2026-09-24)](https://www.postgresql.org/message-id/arS_NVVZZ2XsfCYU%40paquier.xyz), [#18 (Greg Burd, 2026-10-15)](https://www.postgresql.org/message-id/9e2DVFB_h2qGbP2K8kDioxqzP6l67HWLLHEM7Kp8j7Ee1KozM7XPFXpoennCxqqCD4pxMHITBgvqV7wefAaga-xy3aTl_G0cselilKkTT5Y%3D%40burd.me), [#29 (Hannu Krosing, 2026-10-25)](https://www.postgresql.org/message-id/CAMT0RQQEXX7tQf1-yNnweE9JdMMRQezc_5QQMbs7yJfUAz8%3DNQ%40mail.gmail.com), [#35 (Greg Burd, 2026-10-27)](https://www.postgresql.org/message-id/XVi2K1FkIfByjYymEzR-_jZUUmkjZHPvCDX1AI9OXDZX4JiD0ymxTbEpCa2ccN8Ma3O0q4ik9NltiQZWeCLLTWicCZ64DHSBWaN0hxcaxGQ%3D%40burd.me), [#39 (Hannu Krosing, 2026-10-29)](https://www.postgresql.org/message-id/CAMT0RQTZXKWfnsn9DvTononbY4oP7pdbdy6TjNigoWg%2Bh4diOQ%40mail.gmail.com)

**Question:** If a generic encoding-context catalog is adopted later, how does the series let it read existing zstd_dict datums without keeping pg_toast_dictionary forever or rewriting tables at upgrade?

Counter-check: held. Message #18 confirms heap files embed dictionary OIDs that pg_upgrade must keep, and message #35 gives no plan for data already on disk. Hannu agreed in message #39 not to block on the generic store, which lowers the risk but does not answer the question.

### 3. Dictionary accumulation concern from the first thread returns under automatic evolution

fixable · level 2, rubric item 14 · prior rejection, catalog changes, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

An operator turns on dictionary_evolution for a large, busy table. Autoanalyze adopts new dictionaries until the per-column cap, then logs and stops retraining while the data keeps drifting. The only way to retire the old dictionaries is VACUUM FULL, CLUSTER or REPACK, which the operator cannot schedule on that table.

Message #18 reports that the first thread stopped on undroppable dictionaries accumulating and on pg_depend tracking, and that these were deferred, not answered. Michael raised retention of past dictionaries in message #2 and has not commented on 0004-0005. 0005 makes adoption automatic while retirement still needs a rewrite (messages #25, #35), and Hannu calls that rewrite prohibitive on live databases (message #39). The cap only declines with a LOG, so the old problem becomes a silent stall.

Thread: [#2 (Michael Paquier, 2026-09-24)](https://www.postgresql.org/message-id/arS_NVVZZ2XsfCYU%40paquier.xyz), [#18 (Greg Burd, 2026-10-15)](https://www.postgresql.org/message-id/9e2DVFB_h2qGbP2K8kDioxqzP6l67HWLLHEM7Kp8j7Ee1KozM7XPFXpoennCxqqCD4pxMHITBgvqV7wefAaga-xy3aTl_G0cselilKkTT5Y%3D%40burd.me), [#25 (Greg Burd, 2026-10-24)](https://www.postgresql.org/message-id/SNzmb8pP-a8zJJ5yVcfuLceJj2jQV0gJw5v1_akpKONEaO9BZJ_r6ctAEcyx6nMNy8u-mYh8w6f-oIeQoZzz-kWDXp3Q1Gv-GzlOuHPGcRw%3D%40burd.me), [#35 (Greg Burd, 2026-10-27)](https://www.postgresql.org/message-id/XVi2K1FkIfByjYymEzR-_jZUUmkjZHPvCDX1AI9OXDZX4JiD0ymxTbEpCa2ccN8Ma3O0q4ik9NltiQZWeCLLTWicCZ64DHSBWaN0hxcaxGQ%3D%40burd.me), [#39 (Hannu Krosing, 2026-10-29)](https://www.postgresql.org/message-id/CAMT0RQTZXKWfnsn9DvTononbY4oP7pdbdy6TjNigoWg%2Bh4diOQ%40mail.gmail.com)

**Question:** How does a column with evolution enabled get rid of superseded dictionaries without a rewrite under AccessExclusiveLock, and what happens when it hits the cap?

Counter-check: held. Messages #25 and #35 confirm that retirement needs a rewrite and that the cap only declines with a LOG. Message #28 shows a stale dictionary compresses worse than none, so a column at the cap degrades with no path that avoids AccessExclusiveLock.

### 4. pg_toast_dictionary exposes trained column content without privilege checks

fixable · level 2, rubric item 7 · security / privileges, catalog changes · `security-1`

A role with no SELECT on a table, or with RLS limiting it to its own rows, reads pg_toast_dictionary.tddict. Zstd and lz4 dictionaries hold literal substrings taken from the sampled values, so the role sees fragments of other users' data. Nobody notices, because nothing errors.

Message #18 places the dictionary rows next to pg_statistic, but pg_statistic is readable only by superusers, and pg_stats filters it by column privilege. Message #41 lists a pg_stats-like view as a follow-up, which suggests the raw catalog has no such filter yet. Training reads every row as the owner and ignores RLS and column grants, so tddict summarizes data some readers may not see. Retired dictionaries keep this content until the next rewrite, even after the source rows are deleted.

Thread: [#18 (Greg Burd, 2026-10-15)](https://www.postgresql.org/message-id/9e2DVFB_h2qGbP2K8kDioxqzP6l67HWLLHEM7Kp8j7Ee1KozM7XPFXpoennCxqqCD4pxMHITBgvqV7wefAaga-xy3aTl_G0cselilKkTT5Y%3D%40burd.me), [#41 (Greg Burd, 2026-10-30)](https://www.postgresql.org/message-id/auTdVHKihzViHELQ%40floki)

**Question:** Is pg_toast_dictionary revoked from PUBLIC like pg_statistic, and if not, which roles can read tddict for a column they cannot select?

Counter-check: held. The thread compares pg_toast_dictionary rows to pg_statistic but does not say who can read tddict. Trained dictionaries hold literal fragments of sampled values, and message #41 leaves a filtered view as a follow-up.

### 5. Logical dump and restore drops trained dictionaries without notice

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, catalog changes · `standards-compat-2`

A DBA moves a database with pg_dump and pg_restore, which is also the usual path for logical migrations. The zstd_dict column restores with no dictionary, so new rows get plain zstd or no dictionary benefit. The DBA sees the restored table is much larger and has to retrain and run VACUUM FULL.

Message #18 and #43 cover pg_toast_dictionary only in binary-upgrade dumps, where heap files keep dictionary OIDs. The material does not say what a normal dump emits for a dictionary column, or whether it restores the trained bytes. Message #24 shows that the dictionary carries most of the gain: logs go from 2.75 to 4.32. Planner statistics now have a dump path, so dictionary state can follow the same model.

Thread: [#18 (Greg Burd, 2026-10-15)](https://www.postgresql.org/message-id/9e2DVFB_h2qGbP2K8kDioxqzP6l67HWLLHEM7Kp8j7Ee1KozM7XPFXpoennCxqqCD4pxMHITBgvqV7wefAaga-xy3aTl_G0cselilKkTT5Y%3D%40burd.me), [#24 (Greg Burd, 2026-10-24)](https://www.postgresql.org/message-id/ROIZ-hAIfM-v11M1kJOiAdSbeZaC_3paEONbdF67Pi-G0tMvU88x-kcPs_CpGFGPDVab9cKKP8LOkdWNMhcbJqWsXkvGPOjgbJyl5ZPIa-I%3D%40burd.me), [#43 (Greg Burd, 2026-11-02)](https://www.postgresql.org/message-id/aujm92faOjl592iB%40floki)

**Question:** What does a non-binary pg_dump emit for a zstd_dict column with a current dictionary, and does pg_restore get the same compression ratio without a retrain and a rewrite?

Counter-check: held. Messages #18 and #43 describe dictionary handling only for binary-upgrade dumps. The thread does not say what a plain dump emits, and rows restored before a dictionary exists would need a retrain and a rewrite to regain the ratio.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | New 9-byte long-form header, two new vartags, and dictionary OIDs embedded in heap datums that pg_upgrade keeps. |
| WAL / replication / standby | **yes** | Long-form datums and pg_toast_dictionary rows must replay on standbys; the reviewers report this was tested in the thread. |
| wire protocol | no |  |
| catalog changes | **yes** | New catalog pg_toast_dictionary and new attcompression values 'z', 'd' and 'e'. |
| ABI / extension API | **yes** | VARATT_EXTINFO_GET_COMPRESS_METHOD() is removed, and extensions that copy datums must learn the new vartags and VARATT_IS_COMPRESSED_DICT(). |
| SQL-standard conflict or non-standard syntax | **yes** | New values zstd, zstd_dict and lz4_dict for the non-standard COMPRESSION column option. |
| new GUC | **yes** | New PGC_SUSET GUCs for dictionary evolution, and a new value for default_toast_compression. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump emits the zstd column setting; dictionary contents are covered only for binary-upgrade dumps. |
| psql UX conventions | **yes** | psql's \d+ and tab completion learn the new method. |
| security / privileges | **yes** | pg_toast_dictionary.tddict holds fragments of sampled column values, and a shared dictionary makes stored size depend on other rows. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Permanent on-disk format, a new catalog, ANALYZE-time retraining, and a containment check that every datum-copying path must carry. |
| performance on the hot path | **yes** | Detoast and TOAST pointer decoding gain the long-form branch; zstd reads of 4 kB values cost about 2.6 times lz4. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Evolution in 0005 overlaps with a scheduled pg_train_toast_dictionary() call, which 0004 already allows. |

## 5. Questions the author must answer before anyone reviews code

1. What makes a new datum-copying path that omits VARATT_IS_COMPRESSED_DICT() fail in tests or amcheck, before a dictionary retirement makes stored values unreadable?
2. If a generic encoding-context catalog is adopted later, how are existing datums that embed pg_toast_dictionary OIDs read without keeping that catalog forever or rewriting tables?
3. How does a column with evolution enabled retire old dictionaries without a rewrite under AccessExclusiveLock, and what happens to compression once it reaches the cap?
4. Is pg_toast_dictionary revoked from PUBLIC like pg_statistic, and if not, which roles can read tddict for a column they cannot select?
5. Should 0004-0005 move to a separate CommitFest entry so that 0001-0003 can be judged alone for v20 or v21?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier committed two refactoring patches and reviews the format change. No committer has reviewed the dictionary catalog, containment or ANALYZE-time retraining in 0004-0005.
- **Liability:** For 0001-0003: a permanent on-disk header and two vartags. For 0004-0005: a catalog whose OIDs live in heap data, containment in every datum-copying path, and retraining inside autovacuum, with no end date.
- **Author follow-up:** The original author has not posted since early October. Greg Burd now carries the series through v16 and answers review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Plain zstd on the long form has no surviving objection and an engaged committer, so it can go ahead separately. The dictionary and evolution patches add a catalog, on-disk OIDs and GUCs, and their containment, retirement and generic-store questions need answers first.

## Appendix

### Held up in the counter-check, not listed above

- `security-2` (fixable, level 2) Shared column dictionary gives a compression-size oracle across rows. Not listed: not among the strongest objections.
- `maintenance-fit-2` (fixable, level 2) No committer has engaged with the dictionary and evolution patches. Not listed: not among the strongest objections.
- `already-exists-1` (fixable, level 1b) Automatic dictionary evolution repeats what a scheduled training call can do. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-2` Dictionary catalog and on-disk OIDs fix a design before the generic one settles. Refuted (not specific): Duplicate of replication-ondisk-2, which makes the same on-disk point without precedents. [P8] concerned user-defined compression method objects, which this series does not add, so it is not support here.
- `standards-compat-1` Dictionary SQL surface fixed before the generic encoding-context design settles. Refuted (not specific): Duplicate of replication-ondisk-2; the public names follow from the same early commitment to pg_toast_dictionary and method IDs 4 and 5. The question of a separate method against an option on zstd and lz4 is a design preference with no failure shown.
- `standards-compat-3` Evolution policy split across SUSET GUCs and per-column options. Refuted (wrong facts): The repro in message #47 sets dictionary_evolution_min_rows and dictionary_evolution_growth per column with alter table ... set (...), next to a GUC for the interval. So the table owner can override thresholds per column, and the scenario of an owner unable to override does not hold as stated.
- `replication-ondisk-1` Dictionary containment depends on every datum-copying path, and escapes persist. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same containment risk with the audit count from message #45 and the same request for a check that fails early.
- `maintenance-fit-3` Automatic evolution adds dictionaries that only a full rewrite can remove. Refuted (not specific): Duplicate of prior-attempts-1, which covers the same cap and rewrite-only retirement. The added point about permanent evolution knobs is covered by already-exists-1.
- `maintenance-fit-4` TOAST-specific dictionary catalog overlaps with the generic encoding-context proposal. Refuted (not specific): Duplicate of replication-ondisk-2. Both rest on heap values that embed pg_toast_dictionary OIDs while the generic catalog is still under discussion.
- `demand-1` Automatic dictionary evolution has no shown user demand. Refuted (not specific): Duplicate of already-exists-1, which makes the same case against 0005 with the concrete alternative. The scenario of retraining for users who never asked does not hold, because message #28 makes evolution opt-in per column and off by default.
- `demand-2` Dictionary benefit lacks field evidence while containment cost falls on everyone. Refuted (not specific): The containment half is a duplicate of maintenance-fit-1. On evidence, message #24 reports relation-size ratios for stored tables (logs 2.75 to 4.32), and message #22 cites a tweet data set at 210 MB with zstd against 114 MB with a dictionary.

### Review angles

- prior-attempts: 2 objection(s). 0002 answers the [P3] reason for zstd and the long form. Both objections are about the dictionary and evolution patches, 0004-0005.
- standards-compat: 3 objection(s). 0001-0003 (the long form and plain zstd) follow the lz4 precedent for syntax, the GUC, pg_dump, psql and pg_column_compression. I have no compatibility objection to them.
- replication-ondisk: 2 objection(s). The long-form header and tags in 0002-0003 leave pglz and lz4 data unchanged; hash checks and pg_upgrade tests (message #10) back this. WAL, standby replay and pg_upgrade of dictionaries were tested in the thread, so no objection is raised there.
- security: 2 objection(s). The zstd method and the long-form format (0001-0003) add no new privilege or exposure surface; both objections concern the dictionary patches only. The manual trainer already checks ownership, recovery and temp tables, and the evolution GUCs are PGC_SUSET.
- maintenance-fit: 4 objection(s). No objection is raised against 0001-0003. The long-form ID and plain zstd reuse the libzstd already linked in, cost nothing for pglz and lz4, and Michael accepted the one-byte cost.
- already-exists: 1 objection(s). For 0001-0004 there is no existing alternative. Core has only pglz and lz4, and extensions cannot add a TOAST compression method or the containment rules dictionaries need, so plain zstd and dictionaries draw no objection from this angle.
- demand: 2 objection(s). Demand for plain zstd (0001-0003) is reasonably shown: several independent threads, ratio gains on base64 and JSON, and the extensible header that [P3] asked for first. Objections target only the dictionary and evolution patches.

### Precedents retrieved

- P1: [Toast compression method options](https://commitfest.postgresql.org/patch/3102/) (cf, Withdrawn, 2021-10-01) - used in the review; reason summarised from its thread
- P2: [Refactor code around GUC default_toast_compression](https://commitfest.postgresql.org/patch/6727/) (cf, Withdrawn, 2026-09-16) - used in the review; reason summarised from its thread
- P3: [Zstandard support for toast compression](https://commitfest.postgresql.org/patch/3639/) (cf, Withdrawn, 2022-06-16) - used in the review; reason summarised from its thread
- P4: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P5: [Revert the default toast compression strategy to the former behavior where there's no limit on the size of datum we'll try to compress.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=229bffedbc52f12aadb95ca2ee24d5bef36b7c7d) (revert, Reverted, 2009-01-06) - judged not relevant
- P6: [pg_basebackup: add test about zstd compress option](https://commitfest.postgresql.org/patch/3835/) (cf, Rejected, 2022-12-06) - judged not relevant; reason summarised from its thread
- P7: [Compression dictionaries](https://commitfest.postgresql.org/patch/3626/) (cf, Returned with feedback, 2024-01-26) - used in the review
- P8: [Custom compression methods](https://commitfest.postgresql.org/patch/1294/) (cf, Returned with feedback, 2019-09-30) - used in the review; reason summarised from its thread
- P9: [Fix minor issues in astreamer_zstd.c](https://commitfest.postgresql.org/patch/6386/) (cf, Withdrawn, 2026-01-10) - judged not relevant; reason summarised from its thread
- P10: [compress method for spgist](https://commitfest.postgresql.org/patch/12/) (cf, Rejected, 2015-01-16) - judged not relevant; reason summarised from its thread
- P11: [compress method for spgist](https://commitfest.postgresql.org/patch/146/) (cf, Returned with feedback, 2015-08-25) - judged not relevant; reason summarised from its thread
- P12: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant
- P13: [Compress method for SP-GiST](https://commitfest.postgresql.org/patch/1298/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P14: [Shared Ispell dictionaries](https://commitfest.postgresql.org/patch/1461/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P15: [Add minimal C example and SQL registration example for custom table access methods](https://commitfest.postgresql.org/patch/4973/) (cf, Withdrawn, 2024-07-25) - judged not relevant
- P16: [Allow dictionaries to change the token that is passed on to later dictionaries](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Text_Search) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P17: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P18: [Support custom authentication methods using hooks](https://commitfest.postgresql.org/patch/3546/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P19: [Avoid possible null pointer dereference](https://commitfest.postgresql.org/patch/5506/) (cf, Withdrawn, 2025-02-06) - judged not relevant
- P20: [WAL compression setting after PostgreSQL LZ4 default change](https://commitfest.postgresql.org/patch/6979/) (cf, Withdrawn, 2026-07-03) - judged not relevant
- P21: [Pluggable toaster](https://commitfest.postgresql.org/patch/3490/) (cf, Rejected, 2023-04-03) - used in the review
- P22: [Cleanup gcc trick with varattrib_1b_e in VARATT_EXTERNAL_GET_POINTER()](https://commitfest.postgresql.org/patch/5807/) (cf, Withdrawn, 2025-07-14) - judged not relevant
- P23: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P24: [Detect escape of ErrorContextCallback stack pointers (and from PG_TRY() )](https://commitfest.postgresql.org/patch/2722/) (cf, Withdrawn, 2020-09-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 240 s; cost reported by the CLI: $8.91
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 6 of 24 kept
- Tone check: 136 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:59:06.442Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
