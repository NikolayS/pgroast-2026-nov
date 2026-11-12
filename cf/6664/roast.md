# Design pre-review: Support EXCEPT for ALL SEQUENCES publications

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6664](https://commitfest.postgresql.org/patch/6664/); [thread](https://www.postgresql.org/message-id/flat/CANhcyEVSXyQkvmrsOWPdQqnm2J3GMyQQrKhyCJiBQzqs6AvSow%40mail.gmail.com), 82 of 82 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the feature extends the committed FOR ALL TABLES ... EXCEPT mechanism with pg_dump, psql and ALTER covered. Three fixable points remain. Demand rests on two reviewer +1s (demand-1). The subscriber's behaviour after an exclusion list changes is not documented (replication-ondisk-1). The choice to store sequences in pg_publication_rel was defended on lookup cost only (maintenance-fit-1). A committer who knows the publication catalog has to settle the last two.

Not checked: The code was not read, so the handling of relkind and prexcept filters in each reader of pg_publication_rel could not be checked. The subscriber's behaviour between ALTER PUBLICATION and REFRESH PUBLICATION is inferred from a test fix in message #62.

## Prior attempts

- [Skip replicating the tables specified in except table option](https://www.postgresql.org/message-id/flat/CALDaNm3=JrucjhiiwsYQw5-PGtBHFONa6F7hhWCXMsGvh=tamA@mail.gmail.com) (similar idea; Withdrawn, 2024-01-09). "As I did not see much interest from others, I'm withdrawing this patch for now." ([message](https://www.postgresql.org/message-id/CALDaNm1AQZYgT0tALRrkvpP1Q%2B8%2Be7vkGCUjQ-jim1C0q3e%3DzA%40mail.gmail.com)) Different now: P3 proposed EXCEPT TABLE for FOR ALL TABLES. Message #1 reports that the table version has since been committed, and this proposal extends the same mechanism to FOR ALL SEQUENCES.

## Objections a reviewer is likely to raise

### 1. Use case for excluding sequences is stated but not shown

The project adds EXCEPT (SEQUENCE ...) syntax and starts storing sequences in pg_publication_rel, and pg_dump, psql and later publication patches must handle them. If few users ever exclude a sequence, that cost stays and buys little. The proposal does not say what goes wrong today when a node-local sequence is synchronized anyway.

The only demand in the thread is two reviewer +1s describing a general case: temporary-purpose but persistent, node-specific sequences (messages #2, #3). There is no field report, no -general or -bugs thread, and no description of the harm, such as a subscriber error or an overwritten value. Symmetry with ALL TABLES EXCEPT (message #1) is the main argument, and it lowers the cost of the feature but does not show users need it. An earlier EXCEPT TABLE proposal was withdrawn for lack of interest [P3], but EXCEPT TABLE has since gone in, so that precedent weighs little here.

Precedent: [Skip replicating the tables specified in except table option](https://www.postgresql.org/message-id/flat/CALDaNm3=JrucjhiiwsYQw5-PGtBHFONa6F7hhWCXMsGvh=tamA@mail.gmail.com) (Withdrawn, 2024-01-09)

**Question:** What concrete failure or cost do users hit today when FOR ALL SEQUENCES synchronizes a node-local sequence, and is there a user report of it?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CANhcyEVSXyQkvmrsOWPdQqnm2J3GMyQQrKhyCJiBQzqs6AvSow%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CALDaNm2DysPZD%2B5uJvygkws9i%2B7Q3EvUdofEmYguvfnod_ZDnQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAJpy0uDU0PrH%3DgvFZjpphOX7t%3D2jH5wTqYry%3DC22vnuJJ9Q5%3Dg%40mail.gmail.com); `demand-1`</sub>

### 2. Subscriber state after changing a sequence exclusion list is undefined

An administrator runs alter publication ... set all sequences except (sequence s1) because s1 is node-local. The subscriber still lists s1 in pg_subscription_rel. A later alter subscription ... refresh sequences may overwrite the local s1 value, and the administrator finds out only when the subscriber's s1 hands out values that clash with local use.

Message #62 shows the TAP test failed until alter subscription ... refresh publication was added, so the subscriber's pg_subscription_rel keeps an excluded sequence until that command runs. The documentation hunks for alter_publication.sgml and logical-replication.sgml do not tell users to run it, or what refresh sequences does in the meantime. Message #54 asked for this flow to be tested, but the material shows no answer on what users must do.

**Question:** After set all sequences except adds a sequence, what does refresh sequences do on the subscriber before refresh publication runs, and where is the required step documented?

<sub>fixable, level 2, WAL / replication / standby; messages [#54](https://www.postgresql.org/message-id/CAE9k0PnBrRaqOAqGVKmcaCKKKoZEx_sEf6p%3DhQRn_ZW2%2B04afw%40mail.gmail.com), [#62](https://www.postgresql.org/message-id/CANhcyEUg0c2rMEWt%3D2DFLF%2Bfrxx72AyqXn2viyye5uvb8Fmn0g%40mail.gmail.com); `replication-ondisk-1`</sub>

### 3. Mixed tables and sequences in pg_publication_rel rely on unchecked invariants

A later patch adds FOR SEQUENCE s1, so a publication can include some sequences and exclude others. Existing readers assume that any sequence row for an ALL SEQUENCES publication is an exclusion. They will then treat included sequences as excluded, or the reverse. Nothing fails at build or test time; users find out when subscriber sequence values are wrong.

Peter Smith (message #9) listed the row combinations that now exist and asked why sequences share the catalog. The answer in messages #14 and #19 covered lookup cost, which is a different concern. Several readers depend on implicit rules: psql omits the prexcept check (messages #31, #39), GetIncludedPublicationRelations returns an empty list for ALL SEQUENCES (messages #66, #67), and get_delete_rels must be called per relkind (message #72). The idea in message #72 to check relkind once in OpenRelationList got no reply.

**Question:** Which mechanism makes a future reader of pg_publication_rel fail visibly if it forgets the relkind or prexcept filter, for example once FOR SEQUENCE exists?

<sub>fixable, level 2, catalog changes; messages [#9](https://www.postgresql.org/message-id/CAHut%2BPsGFPE7vEpp1UrW1bjRHvNChL6aQErAt6Ugc-VtV93G3w%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAJpy0uCAC7TwbWpJofbayHky_qUhOJfih1OSygR2RPPghr05aQ%40mail.gmail.com), [#19](https://www.postgresql.org/message-id/CANhcyEXiwG6HJiNp8eteOmGXnrSE%2B2eXRAv7Bh5i3QP7vt1nkg%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/CANhcyEVCJOZTH9GL2wCOA%2Bea5uM_yqemUu2vAG-chNuA29HuJA%40mail.gmail.com), [#39](https://www.postgresql.org/message-id/CANhcyEW%2BaJ1rAAodQZDvCd-WpQ%2Bb8r5wBs8H%2BmNdQS8URmkODg%40mail.gmail.com), [#66](https://www.postgresql.org/message-id/CANhcyEWeOnseA-OGvbHa6DGKpOT_XZ0dWt-mW69-TCuE1LMwLg%40mail.gmail.com), [#67](https://www.postgresql.org/message-id/CAHut%2BPvG4OjooxEO-OgiGqG9nvAqS6_ZR%2BU2%2B9ThwRUQ_y6K7A%40mail.gmail.com), [#72](https://www.postgresql.org/message-id/CANhcyEXU61q1MwbfGga%2BC6st7qZBiq6H%3DCWZXgvqDEL0y1O3_A%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. After SET ALL SEQUENCES EXCEPT adds a sequence, what does ALTER SUBSCRIPTION ... REFRESH SEQUENCES do on the subscriber before REFRESH PUBLICATION runs?
2. Will the ALTER PUBLICATION documentation say that the subscriber needs REFRESH PUBLICATION after the exclusion list changes?
3. What makes a future reader of pg_publication_rel fail visibly if it omits the relkind or prexcept filter, for example once FOR SEQUENCE exists?
4. What concrete failure or cost do users hit today when FOR ALL SEQUENCES synchronizes a node-local sequence, and is there a user report of it?
5. Will OpenRelationList check relkind once, as message #72 offered, so that ALTER does not need separate table and sequence lists?

## Touch points

- **WAL / replication / standby:** The exclusion list changes which sequences logical replication synchronizes to the subscriber.
- **catalog changes:** pg_publication_rel.prrelid and prexcept now also cover sequences, so readers tell rows apart by relkind.
- **ABI / extension API:** The patch changes pg_publication.h and parsenodes.h, which extensions that read publications compile against.
- **SQL-standard conflict or non-standard syntax:** gram.y gains EXCEPT (SEQUENCE ...) for CREATE PUBLICATION and ALTER PUBLICATION ... SET ALL SEQUENCES.
- **pg_dump / restore round-trip:** pg_dump must emit the EXCEPT (SEQUENCE ...) clause in CREATE PUBLICATION.
- **psql UX conventions:** \dRp+ gains an "Except sequences:" footer, \d+ on a sequence lists excluding publications, and tab completion changes.
- **maintenance burden ("who maintains this in 10 years"):** Every later reader of pg_publication_rel must filter by relkind and prexcept, and nothing fails visibly if one does not.
- **performance on the hot path:** Publication membership checks now need relkind lookups; the author says these are cached on hot paths.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the publication catalog and logical replication of sequences. No committer has replied in the visible thread.
- Priority: needs champion; audience niche; review cost medium. Reviewers reported no further comments at v24 and the patch has only been rebased since, with no committer reply. The feature serves users with node-local persistent sequences, and it adds syntax and mixed-relkind catalog rows across the parser, the catalog code, pg_dump and psql.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
