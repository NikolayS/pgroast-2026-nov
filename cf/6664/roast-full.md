# Design pre-review: Support EXCEPT for ALL SEQUENCES publications

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6664](https://commitfest.postgresql.org/patch/6664/) (Needs review); authors: Shlok Kyal (shlokkyal)
- Thread: [Support EXCEPT for ALL SEQUENCES publications](https://www.postgresql.org/message-id/flat/CANhcyEVSXyQkvmrsOWPdQqnm2J3GMyQQrKhyCJiBQzqs6AvSow%40mail.gmail.com), started 2026-04-10
- Messages read: 82 of 82
- Patch prose read: v28-0001-Support-EXCEPT-for-ALL-SEQUENCES-in-CREATE-PUBLI.patch, v28-0002-Support-EXCEPT-for-ALL-SEQUENCES-in-ALTER-PUBLIC.patch, v1-0001-Support-EXCEPT-for-ALL-SEQUENCES-in-CREATE-PUBLIC.patch, v1-0002-Support-EXCEPT-for-ALL-SEQUENCES-in-ALTER-PUBLICA.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the feature extends the committed FOR ALL TABLES ... EXCEPT mechanism with pg_dump, psql and ALTER covered. Three fixable points remain. Demand rests on two reviewer +1s (demand-1). The subscriber's behaviour after an exclusion list changes is not documented (replication-ondisk-1). The choice to store sequences in pg_publication_rel was defended on lookup cost only (maintenance-fit-1). A committer who knows the publication catalog has to settle the last two.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the handling of relkind and prexcept filters in each reader of pg_publication_rel could not be checked. The subscriber's behaviour between ALTER PUBLICATION and REFRESH PUBLICATION is inferred from a test fix in message #62.

## The proposal, restated

**Problem.** FOR ALL SEQUENCES publishes every persistent sequence, so a user cannot leave out one sequence. Reviewers named node-specific, temporary-purpose but persistent sequences that need no replication. The patch also aims to match the existing FOR ALL TABLES ... EXCEPT support.

**Proposed design.** The patch adds an EXCEPT (SEQUENCE ...) clause to CREATE PUBLICATION and ALTER PUBLICATION ... SET ALL SEQUENCES. It stores excluded sequences in pg_publication_rel with prexcept set, next to excluded tables, so readers tell them apart by relkind. SET ALL SEQUENCES with EXCEPT replaces the exclusion list, and without EXCEPT it clears the list.

**User-visible surface.**
- SQL: create publication ... for all sequences except (sequence s1, s2), also combined with all tables except (...).
- SQL: alter publication ... set all sequences [except (sequence ...)] replaces or clears sequence exclusions.
- Catalog: pg_publication_rel.prrelid and prexcept now also cover sequences; prattrs and prqual are always null for sequences.
- psql: \dRp+ shows an "Except sequences:" footer, and \d+ on a sequence lists publications that exclude it.
- pg_dump emits the EXCEPT (SEQUENCE ...) clause in CREATE PUBLICATION, and psql tab completion covers the new syntax.

**What the author claims.**
- The change brings sequence publications in line with FOR ALL TABLES ... EXCEPT.
- Excluded sequences are not sent to the subscriber.
- An exclusion follows the sequence across rename and ALTER SEQUENCE ... SET SCHEMA.
- A sequence excluded by one publication but included by another in the same subscription is replicated.
- Storing sequences in pg_publication_rel adds no meaningful cost, because relkind lookups are cached on hot paths.

**Thread state.** Peter Smith, shveta malik, vignesh C, Ashutosh Sharma and Zsolt Parragi reviewed through v28, and solai v tested v11 without finding problems. shveta reported no further comments at v24; the patch has since been only rebased and fixed after rebase errors, and no committer has replied.

## 2. Prior attempts

### Skip replicating the tables specified in except table option (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3646/](https://commitfest.postgresql.org/patch/3646/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CALDaNm3=JrucjhiiwsYQw5-PGtBHFONa6F7hhWCXMsGvh=tamA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALDaNm3=JrucjhiiwsYQw5-PGtBHFONa6F7hhWCXMsGvh=tamA@mail.gmail.com)
- **Date:** decided 2024-01-09; started 2022-03-22
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the patch after the thread went quiet. From mid-2022 the messages are rebases and commitfest reminders, and the author said he saw little interest from others. Earlier, Peter Eisentraut doubted the feature was worth the trouble and pointed to the cost of tracking negative dependencies. Peter Smith found the meaning of ALTER PUBLICATION combined with EXCEPT ambiguous, but these points did not decide the outcome.
- **Reason (quote):** "As I did not see much interest from others, I'm withdrawing this patch for now." ([message](https://www.postgresql.org/message-id/CALDaNm1AQZYgT0tALRrkvpP1Q%2B8%2Be7vkGCUjQ-jim1C0q3e%3DzA%40mail.gmail.com))
- **What is different this time:** P3 proposed EXCEPT TABLE for FOR ALL TABLES. Message #1 reports that the table version has since been committed, and this proposal extends the same mechanism to FOR ALL SEQUENCES.


## 3. Objections a reviewer is likely to raise

### 1. Use case for excluding sequences is stated but not shown

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), catalog changes · `demand-1`

The project adds EXCEPT (SEQUENCE ...) syntax and starts storing sequences in pg_publication_rel, and pg_dump, psql and later publication patches must handle them. If few users ever exclude a sequence, that cost stays and buys little. The proposal does not say what goes wrong today when a node-local sequence is synchronized anyway.

The only demand in the thread is two reviewer +1s describing a general case: temporary-purpose but persistent, node-specific sequences (messages #2, #3). There is no field report, no -general or -bugs thread, and no description of the harm, such as a subscriber error or an overwritten value. Symmetry with ALL TABLES EXCEPT (message #1) is the main argument, and it lowers the cost of the feature but does not show users need it. An earlier EXCEPT TABLE proposal was withdrawn for lack of interest [P3], but EXCEPT TABLE has since gone in, so that precedent weighs little here.

**Precedents.**
- [Skip replicating the tables specified in except table option](https://commitfest.postgresql.org/patch/3646/) (Withdrawn, 2024-01-09); thread: [link](https://www.postgresql.org/message-id/flat/CALDaNm3=JrucjhiiwsYQw5-PGtBHFONa6F7hhWCXMsGvh=tamA@mail.gmail.com)

Thread: [#1 (Shlok Kyal, 2026-04-10)](https://www.postgresql.org/message-id/CANhcyEVSXyQkvmrsOWPdQqnm2J3GMyQQrKhyCJiBQzqs6AvSow%40mail.gmail.com), [#2 (vignesh C, 2026-04-10)](https://www.postgresql.org/message-id/CALDaNm2DysPZD%2B5uJvygkws9i%2B7Q3EvUdofEmYguvfnod_ZDnQ%40mail.gmail.com), [#3 (shveta malik, 2026-04-10)](https://www.postgresql.org/message-id/CAJpy0uDU0PrH%3DgvFZjpphOX7t%3D2jH5wTqYry%3DC22vnuJJ9Q5%3Dg%40mail.gmail.com)

**Question:** What concrete failure or cost do users hit today when FOR ALL SEQUENCES synchronizes a node-local sequence, and is there a user report of it?

Counter-check: held. The thread shows two reviewer +1s with a general use case and no user report or described harm. P3 must not be used as support, because EXCEPT for ALL TABLES was committed later; the fitting outcome is "show demand".

### 2. Subscriber state after changing a sequence exclusion list is undefined

fixable · level 2, rubric item 14 · WAL / replication / standby, catalog changes · `replication-ondisk-1`

An administrator runs alter publication ... set all sequences except (sequence s1) because s1 is node-local. The subscriber still lists s1 in pg_subscription_rel. A later alter subscription ... refresh sequences may overwrite the local s1 value, and the administrator finds out only when the subscriber's s1 hands out values that clash with local use.

Message #62 shows the TAP test failed until alter subscription ... refresh publication was added, so the subscriber's pg_subscription_rel keeps an excluded sequence until that command runs. The documentation hunks for alter_publication.sgml and logical-replication.sgml do not tell users to run it, or what refresh sequences does in the meantime. Message #54 asked for this flow to be tested, but the material shows no answer on what users must do.

Thread: [#54 (Ashutosh Sharma, 2026-06-30)](https://www.postgresql.org/message-id/CAE9k0PnBrRaqOAqGVKmcaCKKKoZEx_sEf6p%3DhQRn_ZW2%2B04afw%40mail.gmail.com), [#62 (Shlok Kyal, 2026-07-07)](https://www.postgresql.org/message-id/CANhcyEUg0c2rMEWt%3D2DFLF%2Bfrxx72AyqXn2viyye5uvb8Fmn0g%40mail.gmail.com)

**Question:** After set all sequences except adds a sequence, what does refresh sequences do on the subscriber before refresh publication runs, and where is the required step documented?

Counter-check: held. Message #62 shows the subscriber keeps the excluded sequence until REFRESH PUBLICATION runs. The shown documentation hunks do not say this, or what REFRESH SEQUENCES does before then, so I could not refute it.

### 3. Mixed tables and sequences in pg_publication_rel rely on unchecked invariants

fixable · level 2, rubric item 9 · catalog changes, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds FOR SEQUENCE s1, so a publication can include some sequences and exclude others. Existing readers assume that any sequence row for an ALL SEQUENCES publication is an exclusion. They will then treat included sequences as excluded, or the reverse. Nothing fails at build or test time; users find out when subscriber sequence values are wrong.

Peter Smith (message #9) listed the row combinations that now exist and asked why sequences share the catalog. The answer in messages #14 and #19 covered lookup cost, which is a different concern. Several readers depend on implicit rules: psql omits the prexcept check (messages #31, #39), GetIncludedPublicationRelations returns an empty list for ALL SEQUENCES (messages #66, #67), and get_delete_rels must be called per relkind (message #72). The idea in message #72 to check relkind once in OpenRelationList got no reply.

Thread: [#9 (Peter Smith, 2026-04-13)](https://www.postgresql.org/message-id/CAHut%2BPsGFPE7vEpp1UrW1bjRHvNChL6aQErAt6Ugc-VtV93G3w%40mail.gmail.com), [#14 (shveta malik, 2026-04-13)](https://www.postgresql.org/message-id/CAJpy0uCAC7TwbWpJofbayHky_qUhOJfih1OSygR2RPPghr05aQ%40mail.gmail.com), [#19 (Shlok Kyal, 2026-04-14)](https://www.postgresql.org/message-id/CANhcyEXiwG6HJiNp8eteOmGXnrSE%2B2eXRAv7Bh5i3QP7vt1nkg%40mail.gmail.com), [#31 (Shlok Kyal, 2026-05-29)](https://www.postgresql.org/message-id/CANhcyEVCJOZTH9GL2wCOA%2Bea5uM_yqemUu2vAG-chNuA29HuJA%40mail.gmail.com), [#39 (Shlok Kyal, 2026-06-18)](https://www.postgresql.org/message-id/CANhcyEW%2BaJ1rAAodQZDvCd-WpQ%2Bb8r5wBs8H%2BmNdQS8URmkODg%40mail.gmail.com), [#66 (Shlok Kyal, 2026-07-24)](https://www.postgresql.org/message-id/CANhcyEWeOnseA-OGvbHa6DGKpOT_XZ0dWt-mW69-TCuE1LMwLg%40mail.gmail.com), [#67 (Peter Smith, 2026-07-27)](https://www.postgresql.org/message-id/CAHut%2BPvG4OjooxEO-OgiGqG9nvAqS6_ZR%2BU2%2B9ThwRUQ_y6K7A%40mail.gmail.com), [#72 (Shlok Kyal, 2026-07-30)](https://www.postgresql.org/message-id/CANhcyEXU61q1MwbfGga%2BC6st7qZBiq6H%3DCWZXgvqDEL0y1O3_A%40mail.gmail.com)

**Question:** Which mechanism makes a future reader of pg_publication_rel fail visibly if it forgets the relkind or prexcept filter, for example once FOR SEQUENCE exists?

Counter-check: held. Messages #14 and #19 answered Peter Smith's catalog question on lookup cost only, and no committer has weighed in. The FOR SEQUENCE scenario is hypothetical, but message #9 names it and nothing shown makes a reader fail visibly.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The exclusion list changes which sequences logical replication synchronizes to the subscriber. |
| wire protocol | no |  |
| catalog changes | **yes** | pg_publication_rel.prrelid and prexcept now also cover sequences, so readers tell rows apart by relkind. |
| ABI / extension API | **yes** | The patch changes pg_publication.h and parsenodes.h, which extensions that read publications compile against. |
| SQL-standard conflict or non-standard syntax | **yes** | gram.y gains EXCEPT (SEQUENCE ...) for CREATE PUBLICATION and ALTER PUBLICATION ... SET ALL SEQUENCES. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump must emit the EXCEPT (SEQUENCE ...) clause in CREATE PUBLICATION. |
| psql UX conventions | **yes** | \dRp+ gains an "Except sequences:" footer, \d+ on a sequence lists excluding publications, and tab completion changes. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every later reader of pg_publication_rel must filter by relkind and prexcept, and nothing fails visibly if one does not. |
| performance on the hot path | **yes** | Publication membership checks now need relkind lookups; the author says these are cached on hot paths. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. After SET ALL SEQUENCES EXCEPT adds a sequence, what does ALTER SUBSCRIPTION ... REFRESH SEQUENCES do on the subscriber before REFRESH PUBLICATION runs?
2. Will the ALTER PUBLICATION documentation say that the subscriber needs REFRESH PUBLICATION after the exclusion list changes?
3. What makes a future reader of pg_publication_rel fail visibly if it omits the relkind or prexcept filter, for example once FOR SEQUENCE exists?
4. What concrete failure or cost do users hit today when FOR ALL SEQUENCES synchronizes a node-local sequence, and is there a user report of it?
5. Will OpenRelationList check relkind once, as message #72 offered, so that ALTER does not need separate table and sequence lists?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the publication catalog and logical replication of sequences. No committer has replied in the visible thread.
- **Liability:** The owner signs for the meaning of sequence rows in pg_publication_rel, the new syntax, and the pg_dump and psql output. All of these stay for every later release and constrain a future FOR SEQUENCE design.
- **Author follow-up:** The author answered reviews and rebased the series through v28 over seven months, which suggests continued follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Reviewers reported no further comments at v24 and the patch has only been rebased since, with no committer reply. The feature serves users with node-local persistent sequences, and it adds syntax and mixed-relkind catalog rows across the parser, the catalog code, pg_dump and psql.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Excluded sequences change what existing pg_publication_rel queries return. Refuted (not specific): Duplicate of replication-ondisk-2, which states the same catalog-reader concern and is judged there.
- `standards-compat-2` Subscriber does not show when a changed exclusion list takes effect. Refuted (not specific): Duplicate of replication-ondisk-1, which hedges the overwrite claim where this one states it as fact.
- `replication-ondisk-2` Existing readers of prexcept rows will now also see sequences. Refuted (no failure scenario): Postgres does not promise stable catalog contents across major versions, and psql does not support newer servers. The catalogs.sgml hunk documents the new meaning of prrelid and prexcept, as Postgres 19 did when prexcept rows first appeared.

### Review angles

- prior-attempts: 0 objection(s). P3 died for lack of interest, and its doubts about negative dependencies came up in an AI summary only. Both points have lost weight since the table version of EXCEPT was committed, and here several reviewers back the sequence extension (messages #2, #3, #15). The thread cites no other earlier attempt.
- standards-compat: 2 objection(s). The syntax follows the existing FOR ALL TABLES EXCEPT form. pg_dump, psql describe output and tab completion are covered, so no objection is raised on those points.
- replication-ondisk: 2 objection(s). Nothing changes WAL, on-disk relation formats or the wire protocol; the catalog has no new columns, so pg_upgrade only needs pg_dump to emit the new except (sequence ...) clause.
- security: 0 objection(s). The proposal adds no new privilege path. FOR ALL SEQUENCES publications, and changes to them, already need superuser, and an exclusion only reduces what gets sent. The new psql queries read pg_publication_rel, which is already readable by all users. The pg_temp_N schema name in the new error message exposes nothing a session cannot already see.
- maintenance-fit: 1 objection(s). The hot-path cost was weighed in message #14 and seems small. Shared EXCEPT syntax copies committed practice for tables. No committer has engaged, which affects committability, not the merits.
- already-exists: 0 objection(s). Postgres has no existing way to drop one persistent sequence from a FOR ALL SEQUENCES publication. Per-sequence FOR SEQUENCE does not exist (message #9), and making a sequence unlogged to exclude it changes its crash behaviour, so that is no substitute. The patch reuses the FOR ALL TABLES ... EXCEPT mechanism and pg_publication_rel, so it does not add a second mechanism. No extension can change publication membership.
- demand: 1 objection(s). Demand is thin but the feature copies existing ALL TABLES EXCEPT practice and its cost falls mostly on its users, so the objection is 'show demand', not rejection.

### Precedents retrieved

- P1: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P2: [logical decoding and replication of sequences, take 2](https://commitfest.postgresql.org/patch/3823/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P3: [Skip replicating the tables specified in except table option](https://commitfest.postgresql.org/patch/3646/) (cf, Withdrawn, 2024-01-09) - used in the review; reason summarised from its thread
- P4: [USAGE privilege on PUBLICATION](https://commitfest.postgresql.org/patch/3641/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P5: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P6: [Revert "pg_createsubscriber: Add log message when no publications exist to drop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=45a7faf130176ae7c152413967c7c90068476a94) (revert, Reverted, 2025-10-15) - judged not relevant
- P7: [rich constructor for plpy.Error class, possible raise structured exception](https://commitfest.postgresql.org/patch/387/) (cf, Returned with feedback, 2016-02-03) - judged not relevant; reason summarised from its thread
- P8: [Optimize walsender handling invalid messages of 'drop publication'](https://commitfest.postgresql.org/patch/4393/) (cf, Withdrawn, 2023-10-20) - judged not relevant; reason summarised from its thread
- P9: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P10: [Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet](https://commitfest.postgresql.org/patch/4077/) (cf, Withdrawn, 2022-12-27) - judged not relevant; reason summarised from its thread
- P11: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant; reason summarised from its thread
- P12: [Parallel tests publication and subscription might fail due to concurrent tuple update](https://commitfest.postgresql.org/patch/5459/) (cf, Rejected, 2025-03-27) - judged not relevant
- P13: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P14: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1710/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P15: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1862/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P16: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/2270/) (cf, Returned with feedback, 2020-03-16) - judged not relevant
- P17: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant
- P18: [pg_dumpall --exclude-database option](https://commitfest.postgresql.org/patch/1743/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P19: [Reduce the log spam by excluding certain SQLSTATEs](https://commitfest.postgresql.org/patch/4964/) (cf, Withdrawn, 2025-04-03) - judged not relevant
- P20: [Table AM modifications to accept column projection lists](https://commitfest.postgresql.org/patch/2922/) (cf, Withdrawn, 2021-06-30) - judged not relevant
- P21: [Allow excluding pg_temp or pg_catalog from search_path.](https://commitfest.postgresql.org/patch/4508/) (cf, Withdrawn, 2023-11-17) - judged not relevant
- P22: [add option to pg_dumpall to exclude tables from the dump](https://commitfest.postgresql.org/patch/671/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P23: [ALTER ROLE SET/RESET for multiple options](https://commitfest.postgresql.org/patch/486/) (cf, Rejected, 2016-02-18) - judged not relevant
- P24: [Make executor's Range Table an array instead of a List](https://commitfest.postgresql.org/patch/1771/) (cf, Returned with feedback, 2018-08-23) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 160 s; cost reported by the CLI: $7.56
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 80 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:58:59.791Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
