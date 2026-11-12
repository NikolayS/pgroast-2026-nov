# Design pre-review: ZSTD TOAST compression, and an extensible compression method encoding

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7471](https://commitfest.postgresql.org/patch/7471/); [thread](https://www.postgresql.org/message-id/flat/CAFAfj_GGY4W81XjtYLvYOhqPtvfZbBNj%3DYRS6Mq_0SA9AqEwoQ%40mail.gmail.com), 48 of 48 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection concerns the long-form method ID or plain zstd (0001-0003), and 0002 answers the reason recorded in P3. The dictionary and evolution patches carry open design points. A missed containment check leaves stored values unreadable (maintenance-fit-1), heap data embeds pg_toast_dictionary OIDs while a generic store is still under discussion (replication-ondisk-2), and retirement needs a rewrite (prior-attempts-1). All are fixable, but the list should settle them, and the split of the series, before code review of 0004-0005.

Not checked: The code was not read, so the claim that pglz and lz4 data stays bit for bit the same, and the completeness of the containment checks, rest on the thread's reports. Only 3 of 48 messages were visible here; later messages are known through the objections.

## Prior attempts

- [Zstandard support for toast compression](https://www.postgresql.org/message-id/flat/YoMiNmkztrslDbNS@paquier.xyz) (same idea; Withdrawn, 2022-06-16). "If we decide not to add this now, we can always change our minds and add it later, but if we decide to add it now, there's no backing it out." ([message](https://www.postgresql.org/message-id/CA%2BTgmobZkCvpUJAh63XkCPRN3aYYbEEcMTY%3DcqmZdGRLZNbyCg%40mail.gmail.com)) Different now: This series first spends the last bit pattern on a long form with a method byte, as the earlier thread asked, and only then adds zstd as ID 2 of 255.
- [Compression dictionaries](https://commitfest.postgresql.org/patch/3626/) (similar idea; Returned with feedback, 2024-01-26). reason not recorded Different now: The dictionaries here are column-local and are retired by rewrite. The precedent row records no reason, so the difference cannot be checked.
- [Toast compression method options](https://www.postgresql.org/message-id/flat/CAFiTN-vMHU_yakMgNo90SUih_6LtvmqZ5hpQb2iTgQtVyOjyFA@mail.gmail.com) (related; Withdrawn, 2021-10-01). "I don't recall seeing much on this list that users would like to have such a level of tuning for pglz, while lz4 only offers the option to reduce the compression rate while being already very cheap in CPU so the impact is limited." ([message](https://www.postgresql.org/message-id/YVa028Lj99Eqtckt%40paquier.xyz)) Different now: Compression level stays out of this series, but 0005 adds per-column options and SUSET GUCs to tune dictionary evolution, which is a similar tuning surface.

## Objections a reviewer is likely to raise

### 1. Dictionary containment depends on a predicate every future datum path must add

A later patch adds a path that copies or holds datums across statements and tests only VARATT_IS_EXTERNAL(). An inline zstd_dict datum escapes its column. After the next VACUUM FULL retires the dictionary, reads fail with "compression dictionary N does not exist", long after the escape happened.

0004 makes inline dictionary datums safe only by adding VARATT_IS_COMPRESSED_DICT() next to VARATT_IS_EXTERNAL() at each spot that copies a datum. Review has already found escapes in a PL/pgSQL expanded record (message #38), indirect pointers and datumTransfer() (message #41), and check_domain_for_new_field() (message #44), and Manu counted 141 predicate uses to audit (message #45). Nothing fails loudly when a new site forgets the check, and the result is unreadable stored data. Extension code that holds datums has the same exposure.

**Question:** What makes a new datum-copying path that omits VARATT_IS_COMPRESSED_DICT() fail in tests, before a dictionary retirement makes stored values unreadable?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#38](https://www.postgresql.org/message-id/CAMT0RQS-xv0CzmDz8xwvLRbChs7D8tkb4CEvbSD7rjEaQstUsg%40mail.gmail.com), [#41](https://www.postgresql.org/message-id/auTdVHKihzViHELQ%40floki), [#44](https://www.postgresql.org/message-id/CAMT0RQTWgNJ1%3DQLbmr5rqtsFM5-6Xr0iuen5y4xipG%3DOm_pe8A%40mail.gmail.com), [#45](https://www.postgresql.org/message-id/179367850497.1374811.8378346906553509584%40gmail.com); `maintenance-fit-1`</sub>

### 2. Heap bytes embed pg_toast_dictionary OIDs before the catalog design is settled

v20 ships zstd_dict, and user tables fill with datums carrying a pg_toast_dictionary OID after va_cmid 4. A later release adopts a generic store such as pg_encoding_context. pg_upgrade keeps heap files as they are, so the new release must keep reading pg_toast_dictionary indefinitely or require every user to rewrite their tables.

Message #35 calls re-expressing the dictionary catalog on a generic store "a mechanical change". That holds for code but not for data already on disk, which holds the dictionary OID, method ID 4 and attcompression 'd'/'e'. Hannu's overlapping design is still under discussion (messages #29, #39), and Michael's concern about retaining past dictionaries (message #2) concerns this same on-disk dependency. 0001-0003 do not need this commitment, and 0004 makes it permanent.

**Question:** If a generic encoding-context catalog is adopted later, how does the series let it read existing zstd_dict datums without keeping pg_toast_dictionary forever or rewriting tables at upgrade?

<sub>fixable, level 2, catalog changes; messages [#2](https://www.postgresql.org/message-id/arS_NVVZZ2XsfCYU%40paquier.xyz), [#18](https://www.postgresql.org/message-id/9e2DVFB_h2qGbP2K8kDioxqzP6l67HWLLHEM7Kp8j7Ee1KozM7XPFXpoennCxqqCD4pxMHITBgvqV7wefAaga-xy3aTl_G0cselilKkTT5Y%3D%40burd.me), [#29](https://www.postgresql.org/message-id/CAMT0RQQEXX7tQf1-yNnweE9JdMMRQezc_5QQMbs7yJfUAz8%3DNQ%40mail.gmail.com), [#35](https://www.postgresql.org/message-id/XVi2K1FkIfByjYymEzR-_jZUUmkjZHPvCDX1AI9OXDZX4JiD0ymxTbEpCa2ccN8Ma3O0q4ik9NltiQZWeCLLTWicCZ64DHSBWaN0hxcaxGQ%3D%40burd.me), [#39](https://www.postgresql.org/message-id/CAMT0RQTZXKWfnsn9DvTononbY4oP7pdbdy6TjNigoWg%2Bh4diOQ%40mail.gmail.com); `replication-ondisk-2`</sub>

### 3. Dictionary accumulation concern from the first thread returns under automatic evolution

An operator turns on dictionary_evolution for a large, busy table. Autoanalyze adopts new dictionaries until the per-column cap, then logs and stops retraining while the data keeps drifting. The only way to retire the old dictionaries is VACUUM FULL, CLUSTER or REPACK, which the operator cannot schedule on that table.

Message #18 reports that the first thread stopped on undroppable dictionaries accumulating and on pg_depend tracking, and that these were deferred, not answered. Michael raised retention of past dictionaries in message #2 and has not commented on 0004-0005. 0005 makes adoption automatic while retirement still needs a rewrite (messages #25, #35), and Hannu calls that rewrite prohibitive on live databases (message #39). The cap only declines with a LOG, so the old problem becomes a silent stall.

**Question:** How does a column with evolution enabled get rid of superseded dictionaries without a rewrite under AccessExclusiveLock, and what happens when it hits the cap?

<sub>fixable, level 2, prior rejection; messages [#2](https://www.postgresql.org/message-id/arS_NVVZZ2XsfCYU%40paquier.xyz), [#18](https://www.postgresql.org/message-id/9e2DVFB_h2qGbP2K8kDioxqzP6l67HWLLHEM7Kp8j7Ee1KozM7XPFXpoennCxqqCD4pxMHITBgvqV7wefAaga-xy3aTl_G0cselilKkTT5Y%3D%40burd.me), [#25](https://www.postgresql.org/message-id/SNzmb8pP-a8zJJ5yVcfuLceJj2jQV0gJw5v1_akpKONEaO9BZJ_r6ctAEcyx6nMNy8u-mYh8w6f-oIeQoZzz-kWDXp3Q1Gv-GzlOuHPGcRw%3D%40burd.me), [#35](https://www.postgresql.org/message-id/XVi2K1FkIfByjYymEzR-_jZUUmkjZHPvCDX1AI9OXDZX4JiD0ymxTbEpCa2ccN8Ma3O0q4ik9NltiQZWeCLLTWicCZ64DHSBWaN0hxcaxGQ%3D%40burd.me), [#39](https://www.postgresql.org/message-id/CAMT0RQTZXKWfnsn9DvTononbY4oP7pdbdy6TjNigoWg%2Bh4diOQ%40mail.gmail.com); `prior-attempts-1`</sub>

### 4. pg_toast_dictionary exposes trained column content without privilege checks

A role with no SELECT on a table, or with RLS limiting it to its own rows, reads pg_toast_dictionary.tddict. Zstd and lz4 dictionaries hold literal substrings taken from the sampled values, so the role sees fragments of other users' data. Nobody notices, because nothing errors.

Message #18 places the dictionary rows next to pg_statistic, but pg_statistic is readable only by superusers, and pg_stats filters it by column privilege. Message #41 lists a pg_stats-like view as a follow-up, which suggests the raw catalog has no such filter yet. Training reads every row as the owner and ignores RLS and column grants, so tddict summarizes data some readers may not see. Retired dictionaries keep this content until the next rewrite, even after the source rows are deleted.

**Question:** Is pg_toast_dictionary revoked from PUBLIC like pg_statistic, and if not, which roles can read tddict for a column they cannot select?

<sub>fixable, level 2, security / privileges; messages [#18](https://www.postgresql.org/message-id/9e2DVFB_h2qGbP2K8kDioxqzP6l67HWLLHEM7Kp8j7Ee1KozM7XPFXpoennCxqqCD4pxMHITBgvqV7wefAaga-xy3aTl_G0cselilKkTT5Y%3D%40burd.me), [#41](https://www.postgresql.org/message-id/auTdVHKihzViHELQ%40floki); `security-1`</sub>

### 5. Logical dump and restore drops trained dictionaries without notice

A DBA moves a database with pg_dump and pg_restore, which is also the usual path for logical migrations. The zstd_dict column restores with no dictionary, so new rows get plain zstd or no dictionary benefit. The DBA sees the restored table is much larger and has to retrain and run VACUUM FULL.

Message #18 and #43 cover pg_toast_dictionary only in binary-upgrade dumps, where heap files keep dictionary OIDs. The material does not say what a normal dump emits for a dictionary column, or whether it restores the trained bytes. Message #24 shows that the dictionary carries most of the gain: logs go from 2.75 to 4.32. Planner statistics now have a dump path, so dictionary state can follow the same model.

**Question:** What does a non-binary pg_dump emit for a zstd_dict column with a current dictionary, and does pg_restore get the same compression ratio without a retrain and a rewrite?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#18](https://www.postgresql.org/message-id/9e2DVFB_h2qGbP2K8kDioxqzP6l67HWLLHEM7Kp8j7Ee1KozM7XPFXpoennCxqqCD4pxMHITBgvqV7wefAaga-xy3aTl_G0cselilKkTT5Y%3D%40burd.me), [#24](https://www.postgresql.org/message-id/ROIZ-hAIfM-v11M1kJOiAdSbeZaC_3paEONbdF67Pi-G0tMvU88x-kcPs_CpGFGPDVab9cKKP8LOkdWNMhcbJqWsXkvGPOjgbJyl5ZPIa-I%3D%40burd.me), [#43](https://www.postgresql.org/message-id/aujm92faOjl592iB%40floki); `standards-compat-2`</sub>

## Questions for the author

1. What makes a new datum-copying path that omits VARATT_IS_COMPRESSED_DICT() fail in tests or amcheck, before a dictionary retirement makes stored values unreadable?
2. If a generic encoding-context catalog is adopted later, how are existing datums that embed pg_toast_dictionary OIDs read without keeping that catalog forever or rewriting tables?
3. How does a column with evolution enabled retire old dictionaries without a rewrite under AccessExclusiveLock, and what happens to compression once it reaches the cap?
4. Is pg_toast_dictionary revoked from PUBLIC like pg_statistic, and if not, which roles can read tddict for a column they cannot select?
5. Should 0004-0005 move to a separate CommitFest entry so that 0001-0003 can be judged alone for v20 or v21?

## Touch points

- **on-disk format / pg_upgrade:** New 9-byte long-form header, two new vartags, and dictionary OIDs embedded in heap datums that pg_upgrade keeps.
- **WAL / replication / standby:** Long-form datums and pg_toast_dictionary rows must replay on standbys; the reviewers report this was tested in the thread.
- **catalog changes:** New catalog pg_toast_dictionary and new attcompression values 'z', 'd' and 'e'.
- **ABI / extension API:** VARATT_EXTINFO_GET_COMPRESS_METHOD() is removed, and extensions that copy datums must learn the new vartags and VARATT_IS_COMPRESSED_DICT().
- **SQL-standard conflict or non-standard syntax:** New values zstd, zstd_dict and lz4_dict for the non-standard COMPRESSION column option.
- **new GUC:** New PGC_SUSET GUCs for dictionary evolution, and a new value for default_toast_compression.
- **pg_dump / restore round-trip:** pg_dump emits the zstd column setting; dictionary contents are covered only for binary-upgrade dumps.
- **psql UX conventions:** psql's \d+ and tab completion learn the new method.
- **security / privileges:** pg_toast_dictionary.tddict holds fragments of sampled column values, and a shared dictionary makes stored size depend on other rows.
- **maintenance burden ("who maintains this in 10 years"):** Permanent on-disk format, a new catalog, ANALYZE-time retraining, and a containment check that every datum-copying path must carry.
- **performance on the hot path:** Detoast and TOAST pointer decoding gain the long-form branch; zstd reads of 4 kB values cost about 2.6 times lz4.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Evolution in 0005 overlaps with a scheduled pg_train_toast_dictionary() call, which 0004 already allows.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier committed two refactoring patches and reviews the format change. No committer has reviewed the dictionary catalog, containment or ANALYZE-time retraining in 0004-0005.
- Priority: fix design first; audience a sizeable minority; review cost high. Plain zstd on the long form has no surviving objection and an engaged committer, so it can go ahead separately. The dictionary and evolution patches add a catalog, on-disk OIDs and GUCs, and their containment, retirement and generic-store questions need answers first.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 8 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
