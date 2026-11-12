# Design pre-review: Allow table AMs to define their own reloptions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5631](https://commitfest.postgresql.org/patch/5631/); [thread](https://www.postgresql.org/message-id/flat/20250302085641.hmjom5ru3w554t2n%40poseidon.home.virt), 39 of 39 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal: demand has lasted since 2020 and the amoptions callback copies index AM practice. The surviving objections are fixable design points at level 2. The strongest is prior-attempts-1: the same feature was reverted twice in 2024 for design issues, and the thread never names them. replication-ondisk-2 and replication-ondisk-1 leave restore and pg_upgrade behaviour open, and maintenance-fit-1 shows a crash class that nothing catches automatically.

Not checked: Only the prose of v1 and v4 was read, not v14. Without the code, this review could not check the binary-upgrade ordering of the separate ALTER entry, whether all rd_options readers are guarded, or how AMs without amoptions behave.

## Prior attempts

- [Revert "Custom reloptions for table AM"](https://www.postgresql.org/message-id/flat/11550b536211d5748bb2865ed6cb3502ff073bf7.camel@j-davis.com) (same idea; Reverted, 2024-04-02). "due to multiple design issues spotted after commit." Different now: v14 uses an amoptions callback that mirrors index AMs, with an StdRdOptions prefix flag. The material does not record the issues behind the revert, so nothing shows whether v14 fixes them.
- [Revert: Custom reloptions for table AM](https://www.postgresql.org/message-id/flat/20240410165236.rwyrny7ihi4ddxw4@awork3.anarazel.de) (same idea; Reverted, 2024-04-11). "This commit reverts 9bd99f4c26 and 422041542f per review by Andres Freund." Different now: This is the follow-up revert of the same 2024 feature, including its extract_autovac_opts change. v14 keeps autovacuum on the heap parser and does not say whether that was the objection.
- [Per-table storage parameters for TableAM/IndexAM extensions](https://www.postgresql.org/message-id/flat/CAFF0-CG4KZHdtYHMsonWiXNzj16gWZpduXAn8yF7pDDub+GQMg@mail.gmail.com) (same idea; Returned with feedback, 2022-08-02). "This entry has been waiting on author input for a while (our current threshold is roughly two weeks), so I've marked it Returned with Feedback." ([message](https://www.postgresql.org/message-id/5aa39215-e262-4993-cc67-70f73187fb6a%40timescale.com)) Different now: v1 of this thread derives from it, including the SET ACCESS METHOD ... OPTIONS syntax. It was returned for inactivity and build errors, which do not apply to the actively maintained v14.
- [Reloptions for table access methods](https://www.postgresql.org/message-id/flat/429fb58fa3218221bb17c7bf9e70e1aa6cfc6b5d.camel@j-davis.com) (similar idea; Withdrawn, 2021-04-06). "The simple patch is admirable, but not something we should put into core." ([message](https://www.postgresql.org/message-id/CANP8%2BjLk9hs9UYEdnudi2QcrcOZ4X0ubFJk3_0-TKGDEv7wq6A%40mail.gmail.com)) Different now: That patch skipped validation so AMs could add heap-kind options. v14 uses a callback and per-AM option sets, which is the direction the objectors asked for.

## Objections a reviewer is likely to raise

### 1. Thread does not explain how v14 avoids the 2024 revert reasons

A committer pushes v14, and extension AM authors build against TableAmRoutine.amoptions and has_std_options_prefix during beta. Post-commit review then finds design issues like those behind the 2024 reverts, and the feature is reverted again. Extensions built against the beta API break before release.

[P9] and [P12] show that custom reloptions for table AMs were committed and reverted twice in 2024 "due to multiple design issues spotted after commit", the second time per review by Andres Freund. The reverted work also touched extract_autovac_opts, and the patch still has autovacuum read StdRdOptions through the heap parser (message #17). Message #8 says the new design "meets most of the previous objections" but does not name them. None of the 39 messages, including the list of earlier attempts in message #1, mentions the reverted commits.

Precedent: [Revert "Custom reloptions for table AM"](https://www.postgresql.org/message-id/flat/11550b536211d5748bb2865ed6cb3502ff073bf7.camel@j-davis.com) (Reverted, 2024-04-02)

Precedent: [Revert: Custom reloptions for table AM](https://www.postgresql.org/message-id/flat/20240410165236.rwyrny7ihi4ddxw4@awork3.anarazel.de) (Reverted, 2024-04-11)

**Question:** Which design issues led to the 2024 reverts, and how does v14 avoid each one?

<sub>fixable, level 2, prior rejection; messages [#1](https://www.postgresql.org/message-id/20250302085641.hmjom5ru3w554t2n%40poseidon.home.virt), [#8](https://www.postgresql.org/message-id/ea1c4d33-0780-473c-96dc-1468cf733a04%40dunslane.net), [#17](https://www.postgresql.org/message-id/df0ed837-4658-44bc-9d81-c00432a82625%40dunslane.net); `prior-attempts-1`</sub>

### 2. Restore requires AM options that fix on-disk layout to be settable later

An AM defines an option that sets its on-disk format, such as a page or stripe layout, at CREATE time and rejects later changes. Every pg_dump restore of such a table fails at the separate ALTER TABLE ... SET entry, or the table is restored without the option and data is written in the default layout. The user finds out during a restore.

Message #29 moves all AM-specific options to a later ALTER TABLE ... SET TOC entry. That works only if the AM can apply its options to an empty table, and the author calls this "not ideal". Core keeps creation-time standard options such as toast_value_type in CREATE for this reason, but AM-specific options get no such path. In binary-upgrade mode the ALTER also runs before the relation files are transferred, and the proposal does not say whether an AM may write metadata into a relfilenode that is later replaced.

**Question:** How does an AM restore an option that only takes effect at creation time, both in a plain restore and in pg_upgrade's binary-upgrade mode?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#29](https://www.postgresql.org/message-id/cd3fe808-90f1-424f-be92-14535ab91a76%40dunslane.net), [#36](https://www.postgresql.org/message-id/5a619b3f-d045-40f5-aaa3-094195c0ee35%40dunslane.net); `replication-ondisk-2`</sub>

### 3. pg_upgrade can fail when an existing table AM starts to supply amoptions

A columnar AM has tables with fillfactor or autovacuum_* set in pg_class.reloptions, which it accepts today through heap's parser. Its release for the new major version supplies amoptions without accepting the standard options. pg_upgrade's schema restore then runs CREATE TABLE ... WITH (fillfactor=...), gets "unrecognized parameter", and the upgrade aborts.

Under v14 the standard options stay in the CREATE statement (message #29), and the new AM's amoptions validates them. Message #36 only says an AM "should accept them all unless it has a good reason not to", which documentation alone cannot enforce. The proposal does not say what happens to options already stored in the catalog, and the brief lists this as unclear. Nothing makes the problem visible before the upgrade is run.

**Question:** What happens on pg_upgrade to stored heap reloptions of an out-of-core AM that newly supplies amoptions, and can pg_upgrade --check report the conflict before the upgrade starts?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#29](https://www.postgresql.org/message-id/cd3fe808-90f1-424f-be92-14535ab91a76%40dunslane.net), [#36](https://www.postgresql.org/message-id/5a619b3f-d045-40f5-aaa3-094195c0ee35%40dunslane.net); `replication-ondisk-1`</sub>

### 4. Every core reader of rd_options must remember the rd_stdoptions guard

A future core patch adds a StdRdOptions field or reads rd_options as StdRdOptions in a new place, without the rd_stdoptions guard. Tests pass because heap and dummy_table_am both embed StdRdOptions. A user of an out-of-core AM that does not set has_std_options_prefix finds out from a backend crash, as in the VACUUM case in message #22.

The thread found unguarded readers several times: four macros (message #16), the TOAST memcpy in vacuum_rel (message #22) and RelationGetToastValueType (message #26). Reviewers found each by hand, and nothing in core fails when a guard is missing. The second test AM in message #27 only catches readers that existing tests exercise, so a new reader added later is not covered.

**Question:** What makes an unguarded read of rd_options as StdRdOptions fail at build or test time, so that a future patch cannot add one unnoticed?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#16](https://www.postgresql.org/message-id/5a9683e8-b825-48e9-945b-0ab668cef4f2%40dunslane.net), [#22](https://www.postgresql.org/message-id/CAER375N_RfkWev54wSj9wmX9dDjX5Ms7wEmpn3f8JJ9RA6QWug%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CAER375NyucpO_f735Q69PaHuJvNZiqrsJJQxrNy30Tmw-vE6Nw%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/a15db9d5-3ebb-4e6a-80e9-c900aad638a0%40dunslane.net); `maintenance-fit-1`</sub>

### 5. New core reloption names can silently weaken an AM option's lock

An out-of-core AM registers an option at AccessExclusiveLock. A later release adds a core reloption with the same name at ShareUpdateExclusiveLock. After upgrade, ALTER TABLE in a backend that has not loaded the AM takes the weaker lock while the AM's readers assume no concurrent change, and nobody is warned.

Message #39 says options are matched by name across all kinds, and covers the conflict only in tableam.sgml. That puts a contract on core as well as on AM authors: every new core reloption name must be checked against an AM namespace the project cannot see. The bloom case in message #38 shows the same lock lookup already goes wrong for index AMs.

**Question:** How will core detect, at ALTER TABLE time or at AM registration, that an AM option shares a name with a core option but needs a stronger lock?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#38](https://www.postgresql.org/message-id/CAHWVJhFfwJxkWrSJ%3DaruXt0kRP_%2BPz3HcTZTssjOnTF1UensjQ%40mail.gmail.com), [#39](https://www.postgresql.org/message-id/2c46e51c-15da-4ea7-94ff-538127939793%40dunslane.net); `maintenance-fit-2`</sub>

## Questions for the author

1. Which design issues led to the 2024 reverts recorded in P9 and P12, and how does v14 avoid each one?
2. How does an AM restore an option that only takes effect at creation time, in a plain restore and in pg_upgrade's binary-upgrade mode?
3. What happens on pg_upgrade to stored standard reloptions of an out-of-core AM that newly supplies amoptions, and can pg_upgrade --check report the conflict?
4. What makes an unguarded read of rd_options as StdRdOptions fail at build or test time?
5. Does v14 keep the SET ACCESS METHOD ... OPTIONS (ADD | SET | DROP) syntax, and do AMs without amoptions keep heap's reloptions?

## Touch points

- **on-disk format / pg_upgrade:** Reloptions already stored in pg_class for out-of-core AMs must still pass validation during pg_upgrade's schema restore.
- **catalog changes:** The patch adds the SQL function pg_reloption_is_standard(name) and changes which pg_class.reloptions values an AM accepts.
- **ABI / extension API:** TableAmRoutine gains amoptions and has_std_options_prefix, and reloptions.h gains add_std_reloptions_to_kind().
- **SQL-standard conflict or non-standard syntax:** v1 to v4 add SET ACCESS METHOD ... OPTIONS (ADD | SET | DROP); the material does not show whether v14 keeps it.
- **pg_dump / restore round-trip:** pg_dump moves AM-specific options to a separate ALTER TABLE ... SET TOC entry that --no-table-access-method skips.
- **security / privileges:** see objection maintenance-fit-2: New core reloption names can silently weaken an AM option's lock
- **backpatchability:** The lock fix in v14-0001 addresses a bug that exists on master, and the author proposes not to backpatch it.
- **maintenance burden ("who maintains this in 10 years"):** Every core reader of rd_options as StdRdOptions must apply the rd_stdoptions guard, and nothing fails when one is missing.
- **performance on the hot path:** The heap insert path reads reloptions; v11 added the rd_stdoptions cache to remove a measured hit there.

## Who would commit it, and when

- Owner: qualified committer engaged. Andrew Dunstan is listed as co-author and posted v14. The owner also needs to know the reloptions, relcache and pg_dump code, and should hear from the reviewer whose review led to the P12 revert.
- Priority: fix design first; audience niche; review cost high. The users are authors of out-of-core table AMs, but demand has lasted across attempts since 2020. The patch touches reloptions, relcache, tablecmds, autovacuum and pg_dump, and the open restore, pg_upgrade and revert-history points need answers before line-by-line review pays off.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
