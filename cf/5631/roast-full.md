# Design pre-review: Allow table AMs to define their own reloptions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5631](https://commitfest.postgresql.org/patch/5631/) (Needs review); authors: Julien Tachoires (julmon), Andrew Dunstan (adunstan)
- Thread: [Allow table AMs to define their own reloptions](https://www.postgresql.org/message-id/flat/20250302085641.hmjom5ru3w554t2n%40poseidon.home.virt), started 2025-03-02
- Messages read: 39 of 39
- Patch prose read: v4-0001-Allow-table-AMs-to-define-their-own-reloptions.patch, v4-0002-Add-the-dummy_table_am-test-module.patch, v1-0001-Allow-table-AMs-to-define-their-own-reloptions.patch, v1-0002-Add-the-dummy_table_am-test-module.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal: demand has lasted since 2020 and the amoptions callback copies index AM practice. The surviving objections are fixable design points at level 2. The strongest is prior-attempts-1: the same feature was reverted twice in 2024 for design issues, and the thread never names them. replication-ondisk-2 and replication-ondisk-1 leave restore and pg_upgrade behaviour open, and maintenance-fit-1 shows a crash class that nothing catches automatically.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only the prose of v1 and v4 was read, not v14. Without the code, this review could not check the binary-upgrade ordering of the separate ALTER entry, whether all rd_options readers are guarded, or how AMs without amoptions behave.

## The proposal, restated

**Problem.** Table AMs other than heap cannot define or validate their own storage parameters, and today they can only use heap's reloptions. The thread names extension AM authors as the users, and Aleksander Alekseev reports a recent commit that used a GUC for lack of this.

**Proposed design.** The current version, Andrew Dunstan's v14, adds an amoptions callback to TableAmRoutine, in line with index AMs. AMs whose options struct starts with StdRdOptions set has_std_options_prefix, and the relcache caches this in rd_stdoptions. pg_dump moves AM-specific options to a separate ALTER TABLE ... SET entry, and ALTER TABLE takes AccessExclusiveLock for options not yet registered in the backend.

**User-visible surface.**
- C API: TableAmRoutine.amoptions callback and has_std_options_prefix flag.
- C API: add_std_reloptions_to_kind() and NUM_STD_RELOPTIONS, so an AM can accept all standard heap options.
- SQL function pg_reloption_is_standard(name), planned for pg_dump in message #29.
- pg_dump emits AM-specific options as a separate ALTER TABLE ... SET (...) TOC entry, which --no-table-access-method skips.
- ALTER TABLE SET/RESET of an unrecognized option name now takes AccessExclusiveLock.

**What the author claims.**
- The design meets most earlier objections and matches what index AMs do.
- Making autovacuum AM-aware would cost a catalog lookup per relation, for a capability nothing needs.
- Caching rd_stdoptions in v11 removed a small performance hit on the heap insert path.
- The lock fix in v14-0001 fixes a bug that exists on master too, shown with bloom, and is probably not worth backpatching.
- An extension that adds reloptions independent of the table AM is a separate project.

**Thread state.** Andrew Dunstan posted v14 on 2026-10-27 with Rui Zhao's lock fix, and nobody has replied since. Several reviewers found bugs that later versions fixed, and no reviewer has said the patch is ready.

## 2. Prior attempts

### Revert "Custom reloptions for table AM" (same idea)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=867cc7b6ddb9d998103688a56048fe9a1ddd972a](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=867cc7b6ddb9d998103688a56048fe9a1ddd972a)
- **Thread:** [https://www.postgresql.org/message-id/flat/11550b536211d5748bb2865ed6cb3502ff073bf7.camel@j-davis.com](https://www.postgresql.org/message-id/flat/11550b536211d5748bb2865ed6cb3502ff073bf7.camel@j-davis.com), [https://www.postgresql.org/message-id/flat/CAPpHfdurb9ycV8udYqM=o0sPS66PJ4RCBM1g-bBpvzUfogY0EA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAPpHfdurb9ycV8udYqM=o0sPS66PJ4RCBM1g-bBpvzUfogY0EA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/AMUA1wBBBxfc3tKRLLdU64rb.1.1683276279979.Hmail.wuhao@hashdata.cn](https://www.postgresql.org/message-id/flat/AMUA1wBBBxfc3tKRLLdU64rb.1.1683276279979.Hmail.wuhao@hashdata.cn)
- **Date:** decided 2024-04-02; started 2024-03-30
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit c95c25f9af4bc77f2f66a587735c50da08c12b37 due to multiple design issues spotted after commit.
- **Reason (quote):** "due to multiple design issues spotted after commit."
- **What is different this time:** v14 uses an amoptions callback that mirrors index AMs, with an StdRdOptions prefix flag. The material does not record the issues behind the revert, so nothing shows whether v14 fixes them.

### Revert: Custom reloptions for table AM (same idea)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bc1e2092ebb857802a9713d0d3588079e2f0216a](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bc1e2092ebb857802a9713d0d3588079e2f0216a)
- **Thread:** [https://www.postgresql.org/message-id/flat/20240410165236.rwyrny7ihi4ddxw4@awork3.anarazel.de](https://www.postgresql.org/message-id/flat/20240410165236.rwyrny7ihi4ddxw4@awork3.anarazel.de), [https://www.postgresql.org/message-id/flat/CAPpHfdurb9ycV8udYqM=o0sPS66PJ4RCBM1g-bBpvzUfogY0EA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAPpHfdurb9ycV8udYqM=o0sPS66PJ4RCBM1g-bBpvzUfogY0EA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/AMUA1wBBBxfc3tKRLLdU64rb.1.1683276279979.Hmail.wuhao@hashdata.cn](https://www.postgresql.org/message-id/flat/AMUA1wBBBxfc3tKRLLdU64rb.1.1683276279979.Hmail.wuhao@hashdata.cn), [https://www.postgresql.org/message-id/flat/CA+hUKGLZzLR50RBvuqOO3MZ=F54ETz-rTp1PDX9uDGP_GqyYqA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKGLZzLR50RBvuqOO3MZ=F54ETz-rTp1PDX9uDGP_GqyYqA@mail.gmail.com)
- **Date:** decided 2024-04-11; started 2024-04-08
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This commit reverts 9bd99f4c26 and 422041542f per review by Andres Freund.
- **Reason (quote):** "This commit reverts 9bd99f4c26 and 422041542f per review by Andres Freund."
- **What is different this time:** This is the follow-up revert of the same 2024 feature, including its extract_autovac_opts change. v14 keeps autovacuum on the heap parser and does not say whether that was the objection.

### Per-table storage parameters for TableAM/IndexAM extensions (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/3495/](https://commitfest.postgresql.org/patch/3495/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAFF0-CG4KZHdtYHMsonWiXNzj16gWZpduXAn8yF7pDDub+GQMg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAFF0-CG4KZHdtYHMsonWiXNzj16gWZpduXAn8yF7pDDub+GQMg@mail.gmail.com)
- **Date:** decided 2022-08-02; started 2021-12-29
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The entry sat in waiting-on-author past the commitfest threshold, so Jacob Champion marked it Returned with Feedback. Before that, the test module failed to build on Windows, and the later patch still had compiler errors in gram.y on cfbot. The author called it work in progress, and no version without these errors appears in the visible messages.
- **Reason (quote):** "This entry has been waiting on author input for a while (our current threshold is roughly two weeks), so I've marked it Returned with Feedback." ([message](https://www.postgresql.org/message-id/5aa39215-e262-4993-cc67-70f73187fb6a%40timescale.com))
- **What is different this time:** v1 of this thread derives from it, including the SET ACCESS METHOD ... OPTIONS syntax. It was returned for inactivity and build errors, which do not apply to the actively maintained v14.

### Reloptions for table access methods (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2717/](https://commitfest.postgresql.org/patch/2717/)
- **Thread:** [https://www.postgresql.org/message-id/flat/429fb58fa3218221bb17c7bf9e70e1aa6cfc6b5d.camel@j-davis.com](https://www.postgresql.org/message-id/flat/429fb58fa3218221bb17c7bf9e70e1aa6cfc6b5d.camel@j-davis.com)
- **Date:** decided 2021-04-06; started 2020-09-01
- **Outcome:** Withdrawn (cf)
- **Who objected:** Simon Riggs, Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Simon Riggs objected that not all heap options should be common to every table AM, that table and index options should work the same way, and that a behavior change needs tests, docs and code comments. Michael Paquier agreed that bypassing a sanity check is not a good long-term solution and sketched a callback-based design. Jeff Davis said he would withdraw the patch if only a fully supported feature was acceptable, and the entry was withdrawn after Simon's promised longer reply did not appear.
- **Reason (quote):** "The simple patch is admirable, but not something we should put into core." ([message](https://www.postgresql.org/message-id/CANP8%2BjLk9hs9UYEdnudi2QcrcOZ4X0ubFJk3_0-TKGDEv7wq6A%40mail.gmail.com))
- **What is different this time:** That patch skipped validation so AMs could add heap-kind options. v14 uses a callback and per-AM option sets, which is the direction the objectors asked for.


## 3. Objections a reviewer is likely to raise

### 1. Thread does not explain how v14 avoids the 2024 revert reasons

fixable · level 2, rubric item 6 · prior rejection, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A committer pushes v14, and extension AM authors build against TableAmRoutine.amoptions and has_std_options_prefix during beta. Post-commit review then finds design issues like those behind the 2024 reverts, and the feature is reverted again. Extensions built against the beta API break before release.

[P9] and [P12] show that custom reloptions for table AMs were committed and reverted twice in 2024 "due to multiple design issues spotted after commit", the second time per review by Andres Freund. The reverted work also touched extract_autovac_opts, and the patch still has autovacuum read StdRdOptions through the heap parser (message #17). Message #8 says the new design "meets most of the previous objections" but does not name them. None of the 39 messages, including the list of earlier attempts in message #1, mentions the reverted commits.

**Precedents.**
- [Revert "Custom reloptions for table AM"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=867cc7b6ddb9d998103688a56048fe9a1ddd972a) (Reverted, 2024-04-02); thread: [#1](https://www.postgresql.org/message-id/flat/11550b536211d5748bb2865ed6cb3502ff073bf7.camel@j-davis.com), [#2](https://www.postgresql.org/message-id/flat/CAPpHfdurb9ycV8udYqM=o0sPS66PJ4RCBM1g-bBpvzUfogY0EA@mail.gmail.com), [#3](https://www.postgresql.org/message-id/flat/AMUA1wBBBxfc3tKRLLdU64rb.1.1683276279979.Hmail.wuhao@hashdata.cn)
- [Revert: Custom reloptions for table AM](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bc1e2092ebb857802a9713d0d3588079e2f0216a) (Reverted, 2024-04-11); thread: [#1](https://www.postgresql.org/message-id/flat/20240410165236.rwyrny7ihi4ddxw4@awork3.anarazel.de), [#2](https://www.postgresql.org/message-id/flat/CAPpHfdurb9ycV8udYqM=o0sPS66PJ4RCBM1g-bBpvzUfogY0EA@mail.gmail.com), [#3](https://www.postgresql.org/message-id/flat/AMUA1wBBBxfc3tKRLLdU64rb.1.1683276279979.Hmail.wuhao@hashdata.cn), [#4](https://www.postgresql.org/message-id/flat/CA+hUKGLZzLR50RBvuqOO3MZ=F54ETz-rTp1PDX9uDGP_GqyYqA@mail.gmail.com)

Thread: [#1 (Julien Tachoires, 2025-03-02)](https://www.postgresql.org/message-id/20250302085641.hmjom5ru3w554t2n%40poseidon.home.virt), [#8 (Andrew Dunstan, 2026-06-03)](https://www.postgresql.org/message-id/ea1c4d33-0780-473c-96dc-1468cf733a04%40dunslane.net), [#17 (Andrew Dunstan, 2026-08-26)](https://www.postgresql.org/message-id/df0ed837-4658-44bc-9d81-c00432a82625%40dunslane.net)

**Question:** Which design issues led to the 2024 reverts, and how does v14 avoid each one?

Counter-check: held. [P9] and [P12] record two reverts of the same feature for design issues, and no message names those issues. Message #8 says only that the design meets "most of the previous objections".

### 2. Restore requires AM options that fix on-disk layout to be settable later

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, pg_dump / restore round-trip · `replication-ondisk-2`

An AM defines an option that sets its on-disk format, such as a page or stripe layout, at CREATE time and rejects later changes. Every pg_dump restore of such a table fails at the separate ALTER TABLE ... SET entry, or the table is restored without the option and data is written in the default layout. The user finds out during a restore.

Message #29 moves all AM-specific options to a later ALTER TABLE ... SET TOC entry. That works only if the AM can apply its options to an empty table, and the author calls this "not ideal". Core keeps creation-time standard options such as toast_value_type in CREATE for this reason, but AM-specific options get no such path. In binary-upgrade mode the ALTER also runs before the relation files are transferred, and the proposal does not say whether an AM may write metadata into a relfilenode that is later replaced.

Thread: [#29 (Andrew Dunstan, 2026-10-01)](https://www.postgresql.org/message-id/cd3fe808-90f1-424f-be92-14535ab91a76%40dunslane.net), [#36 (Andrew Dunstan, 2026-10-19)](https://www.postgresql.org/message-id/5a619b3f-d045-40f5-aaa3-094195c0ee35%40dunslane.net)

**Question:** How does an AM restore an option that only takes effect at creation time, both in a plain restore and in pg_upgrade's binary-upgrade mode?

Counter-check: held. Message #29 states the constraint and calls it "not ideal", so the thread confirms an AM option fixed at creation time cannot be restored. The binary-upgrade part cannot be checked without the code.

### 3. pg_upgrade can fail when an existing table AM starts to supply amoptions

fixable · level 2, rubric item 6 · on-disk format / pg_upgrade, catalog changes, pg_dump / restore round-trip, ABI / extension API · `replication-ondisk-1`

A columnar AM has tables with fillfactor or autovacuum_* set in pg_class.reloptions, which it accepts today through heap's parser. Its release for the new major version supplies amoptions without accepting the standard options. pg_upgrade's schema restore then runs CREATE TABLE ... WITH (fillfactor=...), gets "unrecognized parameter", and the upgrade aborts.

Under v14 the standard options stay in the CREATE statement (message #29), and the new AM's amoptions validates them. Message #36 only says an AM "should accept them all unless it has a good reason not to", which documentation alone cannot enforce. The proposal does not say what happens to options already stored in the catalog, and the brief lists this as unclear. Nothing makes the problem visible before the upgrade is run.

Thread: [#29 (Andrew Dunstan, 2026-10-01)](https://www.postgresql.org/message-id/cd3fe808-90f1-424f-be92-14535ab91a76%40dunslane.net), [#36 (Andrew Dunstan, 2026-10-19)](https://www.postgresql.org/message-id/5a619b3f-d045-40f5-aaa3-094195c0ee35%40dunslane.net)

**Question:** What happens on pg_upgrade to stored heap reloptions of an out-of-core AM that newly supplies amoptions, and can pg_upgrade --check report the conflict before the upgrade starts?

Counter-check: held. Message #19 suggests AMs without amoptions keep the heap parser, so the failure needs an AM to opt in and drop standard options. Only the documentation in message #36 prevents that, and the thread does not cover stored options or pg_upgrade.

### 4. Every core reader of rd_options must remember the rd_stdoptions guard

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A future core patch adds a StdRdOptions field or reads rd_options as StdRdOptions in a new place, without the rd_stdoptions guard. Tests pass because heap and dummy_table_am both embed StdRdOptions. A user of an out-of-core AM that does not set has_std_options_prefix finds out from a backend crash, as in the VACUUM case in message #22.

The thread found unguarded readers several times: four macros (message #16), the TOAST memcpy in vacuum_rel (message #22) and RelationGetToastValueType (message #26). Reviewers found each by hand, and nothing in core fails when a guard is missing. The second test AM in message #27 only catches readers that existing tests exercise, so a new reader added later is not covered.

Thread: [#16 (Andrew Dunstan, 2026-08-21)](https://www.postgresql.org/message-id/5a9683e8-b825-48e9-945b-0ab668cef4f2%40dunslane.net), [#22 (Ajit Awekar, 2026-09-08)](https://www.postgresql.org/message-id/CAER375N_RfkWev54wSj9wmX9dDjX5Ms7wEmpn3f8JJ9RA6QWug%40mail.gmail.com), [#26 (Ajit Awekar, 2026-09-30)](https://www.postgresql.org/message-id/CAER375NyucpO_f735Q69PaHuJvNZiqrsJJQxrNy30Tmw-vE6Nw%40mail.gmail.com), [#27 (Andrew Dunstan, 2026-09-30)](https://www.postgresql.org/message-id/a15db9d5-3ebb-4e6a-80e9-c900aad638a0%40dunslane.net)

**Question:** What makes an unguarded read of rd_options as StdRdOptions fail at build or test time, so that a future patch cannot add one unnoticed?

Counter-check: held. Caching rd_stdoptions in v11 makes the checks agree, but each reader must still apply the guard. Messages #16, #22 and #26 show three rounds of unguarded readers found by hand, one of them a crash.

### 5. New core reloption names can silently weaken an AM option's lock

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API, security / privileges · `maintenance-fit-2`

An out-of-core AM registers an option at AccessExclusiveLock. A later release adds a core reloption with the same name at ShareUpdateExclusiveLock. After upgrade, ALTER TABLE in a backend that has not loaded the AM takes the weaker lock while the AM's readers assume no concurrent change, and nobody is warned.

Message #39 says options are matched by name across all kinds, and covers the conflict only in tableam.sgml. That puts a contract on core as well as on AM authors: every new core reloption name must be checked against an AM namespace the project cannot see. The bloom case in message #38 shows the same lock lookup already goes wrong for index AMs.

Thread: [#38 (Rui Zhao, 2026-10-21)](https://www.postgresql.org/message-id/CAHWVJhFfwJxkWrSJ%3DaruXt0kRP_%2BPz3HcTZTssjOnTF1UensjQ%40mail.gmail.com), [#39 (Andrew Dunstan, 2026-10-27)](https://www.postgresql.org/message-id/2c46e51c-15da-4ea7-94ff-538127939793%40dunslane.net)

**Question:** How will core detect, at ALTER TABLE time or at AM registration, that an AM option shares a name with a core option but needs a stronger lock?

Counter-check: held. Message #39 documents the name collision, but an AM author cannot know which names core will add later. The case is narrow, and message #38 shows index AMs use the same lookup on master, so this is a question to answer, not a blocker.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Reloptions already stored in pg_class for out-of-core AMs must still pass validation during pg_upgrade's schema restore. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds the SQL function pg_reloption_is_standard(name) and changes which pg_class.reloptions values an AM accepts. |
| ABI / extension API | **yes** | TableAmRoutine gains amoptions and has_std_options_prefix, and reloptions.h gains add_std_reloptions_to_kind(). |
| SQL-standard conflict or non-standard syntax | **yes** | v1 to v4 add SET ACCESS METHOD ... OPTIONS (ADD \| SET \| DROP); the material does not show whether v14 keeps it. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump moves AM-specific options to a separate ALTER TABLE ... SET TOC entry that --no-table-access-method skips. |
| psql UX conventions | no |  |
| security / privileges | **yes** | see objection maintenance-fit-2: New core reloption names can silently weaken an AM option's lock |
| backpatchability | **yes** | The lock fix in v14-0001 addresses a bug that exists on master, and the author proposes not to backpatch it. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every core reader of rd_options as StdRdOptions must apply the rd_stdoptions guard, and nothing fails when one is missing. |
| performance on the hot path | **yes** | The heap insert path reads reloptions; v11 added the rd_stdoptions cache to remove a measured hit there. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which design issues led to the 2024 reverts recorded in P9 and P12, and how does v14 avoid each one?
2. How does an AM restore an option that only takes effect at creation time, in a plain restore and in pg_upgrade's binary-upgrade mode?
3. What happens on pg_upgrade to stored standard reloptions of an out-of-core AM that newly supplies amoptions, and can pg_upgrade --check report the conflict?
4. What makes an unguarded read of rd_options as StdRdOptions fail at build or test time?
5. Does v14 keep the SET ACCESS METHOD ... OPTIONS (ADD | SET | DROP) syntax, and do AMs without amoptions keep heap's reloptions?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andrew Dunstan is listed as co-author and posted v14. The owner also needs to know the reloptions, relcache and pg_dump code, and should hear from the reviewer whose review led to the P12 revert.
- **Liability:** A permanent TableAmRoutine callback that out-of-core AMs build against, a new pg_dump TOC entry kind, and the rd_stdoptions guard that every future reader of rd_options must apply. A third revert would break extensions built during beta.
- **Author follow-up:** Good sign: the thread reached v14 over 20 months, later versions fixed bugs that reviewers found, and v14 folds in a reviewer's lock fix.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The users are authors of out-of-core table AMs, but demand has lasted across attempts since 2020. The patch touches reloptions, relcache, tablecmds, autovacuum and pg_dump, and the open restore, pg_upgrade and revert-history points need answers before line-by-line review pays off.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Restore applies AM-specific options after CREATE, changing their meaning. Refuted (not specific): Duplicate of replication-ondisk-2. Message #29 also partly answers its question: pg_restore --no-table-access-method decides at restore time, so the archive needs the separate entry in every dump.
- `standards-compat-2` Stored standard options can break restore when an AM adopts amoptions. Refuted (not specific): Duplicate of replication-ondisk-1, which states the pg_upgrade case more exactly. [P1] is about skipping heap validation and does not support this point.
- `maintenance-fit-3` A table-AM-only API may be replaced by a general options mechanism. Refuted (no failure scenario): The amoptions callback copies the index AM practice, so a later general engine would have to convert both in the same way. Table AM extensions already adapt to TableAmRoutine changes at major versions, and message #21 describes a different feature.

### Review angles

- prior-attempts: 1 objection(s). The options-engine entries [P5] and [P7] died from inactivity, and their reason does not apply here. Message #3 raises the overlap, and other angles cover it.
- standards-compat: 2 objection(s). Two objections above. The lock behaviour for unregistered options and the dropped v1 OPTIONS syntax were not raised, because the thread either fixed them or does not show them in v14.
- replication-ondisk: 2 objection(s). The physical heap format, WAL and the wire protocol do not change. Reloptions stay text in pg_class and are parsed on relcache load, so standbys behave as they do today. The TableAmRoutine change is a normal major-version ABI break.
- security: 0 objection(s). The proposal changes no privilege rule. Only a superuser can install a table AM with its C amoptions callback, and only the table owner can set reloptions, as today. The new AccessExclusiveLock for unregistered option names applies to the owner, who can take that lock already. pg_reloption_is_standard() reports only option names compiled into the server or registered by a loaded AM, which is not sensitive. Without the code, this review could not check that the callback runs with the owner's privileges during pg_restore.
- maintenance-fit: 3 objection(s). Hot-path cost looks settled by the rd_stdoptions cache in message #32. A committer wrote the current version, which lowers the ownership risk.
- already-exists: 0 objection(s). Postgres has no existing way for a table AM to define and check its own per-table options: heap_reloptions rejects unknown names, and a custom GUC or a side table cannot be set or checked through CREATE TABLE ... WITH. The proposal reuses the index AM amoptions pattern rather than adding a second mechanism, and no extension does this job, so this angle finds nothing to raise.
- demand: 0 objection(s). The material shows demand that has lasted: independent attempts in 2020 [P1], 2021 [P6] and 2024 [P9, P12], plus three earlier threads cited in message #1. Rui Zhao (message #38) and Aleksander Alekseev (message #21) also want it. Alekseev's case, options set by an extension independent of the table AM, is a different feature, and the costs of this patch fall mostly on AM users. Not checked: which out-of-core AMs would adopt the callback.

### Precedents retrieved

- P1: [Reloptions for table access methods](https://commitfest.postgresql.org/patch/2717/) (cf, Withdrawn, 2021-04-06) - used in the review; reason summarised from its thread
- P2: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant; reason summarised from its thread
- P3: [Add minimal C example and SQL registration example for custom table access methods](https://commitfest.postgresql.org/patch/4973/) (cf, Withdrawn, 2024-07-25) - judged not relevant; reason summarised from its thread
- P4: [create table like: ACCESS METHOD](https://commitfest.postgresql.org/patch/2865/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P5: [New \[relation\] options engine](https://commitfest.postgresql.org/patch/3536/) (cf, Returned with feedback, 2023-01-31) - used in the review; reason summarised from its thread
- P6: [Per-table storage parameters for TableAM/IndexAM extensions](https://commitfest.postgresql.org/patch/3495/) (cf, Returned with feedback, 2022-08-02) - used in the review; reason summarised from its thread
- P7: [New \[relation\] options engine](https://commitfest.postgresql.org/patch/4160/) (cf, Returned with feedback, 2023-11-11) - used in the review; reason summarised from its thread
- P8: [Small typo fix on tableam documentation](https://commitfest.postgresql.org/patch/2083/) (cf, Rejected, 2019-04-05) - judged not relevant; reason summarised from its thread
- P9: [Revert "Custom reloptions for table AM"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=867cc7b6ddb9d998103688a56048fe9a1ddd972a) (revert, Reverted, 2024-04-02) - used in the review
- P10: [Move all am-related reloption code into src/backend/access/\[am-name\] and get rid of relopt_kind for custom AM](https://commitfest.postgresql.org/patch/992/) (cf, Returned with feedback, 2017-11-28) - used in the review
- P11: [Add table access method as an option to pgbench](https://commitfest.postgresql.org/patch/2884/) (cf, Withdrawn, 2021-01-20) - judged not relevant
- P12: [Revert: Custom reloptions for table AM](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bc1e2092ebb857802a9713d0d3588079e2f0216a) (revert, Reverted, 2024-04-11) - used in the review
- P13: [Add test module for Table Access Method](https://commitfest.postgresql.org/patch/4588/) (cf, Returned with feedback, 2024-01-14) - judged not relevant
- P14: [New Table Access Methods for Multi and Single Inserts](https://commitfest.postgresql.org/patch/2871/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P15: [lax reloptions parsing for views](https://commitfest.postgresql.org/patch/3340/) (cf, Returned with feedback, 2022-03-31) - judged not relevant
- P16: [(Duplicate of CF entry /48/4866/) Table Access Method Enhancements](https://commitfest.postgresql.org/patch/4958/) (cf, Rejected, 2024-07-23) - judged not relevant
- P17: [Add callback table access method to reset filenode when dropping relation](https://commitfest.postgresql.org/patch/3073/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [Custom compression methods](https://commitfest.postgresql.org/patch/1294/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P19: [Support custom authentication methods using hooks](https://commitfest.postgresql.org/patch/3546/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P20: [Remove page-read callback from XLogReaderState](https://commitfest.postgresql.org/patch/2113/) (cf, Returned with feedback, 2021-10-04) - judged not relevant
- P21: [Custom storage managers (SMGR), redux](https://commitfest.postgresql.org/patch/4428/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P22: [Fallback table AM for relkinds without storage](https://commitfest.postgresql.org/patch/2986/) (cf, Returned with feedback, 2021-03-14) - judged not relevant
- P23: [revert: Generalize relation analyze in table AM interface](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6377e12a5a5278446bb0d215439b4825ef8996d1) (revert, Reverted, 2024-04-16) - judged not relevant
- P24: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 20; wall time: 191 s; cost reported by the CLI: $3.98
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 7 of 24 kept
- Tone check: 94 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:34:06.847Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
