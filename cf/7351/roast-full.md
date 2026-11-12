# Design pre-review: Crashes on a partition whose concurrent detach never finished

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7351](https://commitfest.postgresql.org/patch/7351/) (Needs review); authors: shveta malik (shvetam), Mihail Nikalayeu (mihailnikalayeu), Zsolt Parragi (zsolt.parragi)
- Thread: [Crashes on a partition whose concurrent detach never finished](https://www.postgresql.org/message-id/flat/CADzfLwWoFPT%2Ba73%3DA%3DbsNWRMZQ98NpBEMgE%3Dt1FS4O4_%3DQVLfA%40mail.gmail.com), started 2026-08-15
- Messages read: 37 of 37
- Patch prose read: v2-0002-Fix-crash-inserting-into-a-partition-pending-deta.patch, v2-0001-Fix-crash-decoding-a-change-to-a-partition-pendin.patch, v2-0004-Invalidate-a-partition-s-relcache-when-its-concur.patch, v2-0003-Include-partitions-pending-detach-in-publication-.patch, v1-0001-Fix-crash-decoding-a-change-to-a-partition-pendin.patch, v1-0002-Fix-crash-inserting-into-a-partition-pending-deta.patch, v1-0003-Fix-crash-on-UPDATE-or-DELETE-of-a-partition-pend.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The crash fixes are wanted bug fixes, and no objection at level 1a or 1b survived. The surviving objections (standards-compat-1, standards-compat-2, maintenance-fit-2) are all fixable and concern back-patching 0003 and 0004, which change replication output and locking in stable branches. Message #32 says the invalidation question needs more discussion, so the scope decision belongs to the committers engaged in the thread.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check whether 0004's AccessShareLock on descendants and the extra invalidation are safe against concurrent drops and large partition trees. Only 3 of 37 messages were visible, so later agreement on scope may exist.

## The proposal, restated

**Problem.** When the wait in DETACH PARTITION ... CONCURRENTLY is interrupted, the partition keeps relispartition = true while get_partition_ancestors() returns nothing. Several callers assume a non-empty ancestor list and hit an assertion failure or NULL dereference. Users of logical replication and identity columns on partitioned tables are affected, and the walsender can enter a crash loop.

**Proposed design.** Each caller checks the ancestor list instead of relispartition, and a partition pending detach is published as a standalone table, as after FINALIZE. getIdentitySequence() still climbs to the root, because the sequence belongs to it. The first detach transaction also invalidates the partition's relcache and its descendants', taking AccessShareLock on the descendants.

**User-visible surface.**
- pg_publication_tables and pg_get_publication_tables() list a partition pending detach under publish_via_partition_root = true publications.
- New DETAIL and HINT when a detach-pending partition is named in a publication EXCEPT clause (committed part).
- Changes to a detach-pending partition are no longer replicated via the former root once its relcache is invalidated.

**What the author claims.**
- A plain UPDATE on a detach-pending partition crashed the server in v19 without any publication; that fix is committed.
- The pgoutput crash exists since v14, and fd366065e06a widened it to any FOR ALL TABLES publication, a v19 regression per Sawada.
- Publishing via the former root would make publisher and subscriber data diverge, so standalone publishing is the chosen rule.
- Without invalidation, whether a session sees old or new publication state depends on whether it touched the partition before the detach.
- Each patch carries a test that reproduces the problem; the series is mostly AI-generated, per the author.

**Thread state.** The v19 relcache crash fix (v1-0003) is committed; Zhijie Hou posted v3 of the pgoutput crash fix for PG15 through HEAD to close the open item, with Amit asking Sawada to confirm scope. The invalidation patch (0004) and the publication-list patch (0003) await more discussion, and CFbot reports a rebase is needed.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-15; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-patching 0004 changes which rows replicate in a minor release

fixable · level 2, rubric item 6 · backpatchability, WAL / replication / standby · `standards-compat-1`

A publisher on PG15 has a FOR TABLE root publication with publish_via_partition_root = true, and one partition is stuck in detach pending. The subscriber receives that partition's changes through the root today. After a minor update with 0004, those changes stop arriving, and the subscriber finds out only when its rows no longer match the publisher's.

0004 carries Backpatch-through: 14, but in message #32 Amit calls the invalidation question unrelated to PG-19 and in need of more discussion. In message #11 Shveta was unsure whether the extra invalidation has other effects. Moving from root-routed to standalone may be the right rule, as message #12 argues, but changing it in a stable branch alters replication output for users who did not ask for it. The thread has not agreed whether this is HEAD-only.

Thread: [#11 (shveta malik, 2026-09-08)](https://www.postgresql.org/message-id/CAJpy0uA6womAP7fkmT%3DsSfKG_0CfKUSb7HhP_J9mw071MXc39w%40mail.gmail.com), [#12 (Nisha Moond, 2026-09-10)](https://www.postgresql.org/message-id/CABdArM4fJ8JAWyP0MAF%3D8n66M6%3DCouUmtSGwqizn2pD%3DgfPEZw%40mail.gmail.com), [#32 (Amit Kapila, 2026-11-11)](https://www.postgresql.org/message-id/CAA4eK1L%3D8-8NM0Ui2wttmq3fceODPAWrYLHmVLTt276KQkdFeg%40mail.gmail.com)

**Question:** Why is changing replication routing for detach-pending partitions acceptable in stable branches, instead of HEAD only with a release note?

Counter-check: held. The thread has not settled whether 0004 is back-patched; message #32 says it needs more discussion. The scenario's baseline holds only for sessions with a stale cache entry (message #11), but the routing change in a stable branch is still unanswered.

### 2. 0003 changes pg_publication_tables output in stable branches without subscriber guidance

fixable · level 2, rubric item 6 · backpatchability, WAL / replication / standby · `standards-compat-2`

After a minor update, pg_publication_tables lists a detach-pending partition next to its root for publish_via_partition_root publications. A subscriber that has only the root table runs ALTER SUBSCRIPTION ... REFRESH PUBLICATION. The refresh now needs a table the subscriber does not have, and the user finds out when the refresh fails or that partition's changes are skipped.

The 0003 commit message says it lets REFRESH PUBLICATION pick up the partition before FINALIZE, with Backpatch-through: 14. Today the documented outcome of publish_via_partition_root is that only the root is listed, and monitoring or schema-sync tools may rely on that. The proposal does not say what subscribers must do after the update, or whether a refresh is required. The brief lists both points as open.

Thread: [#23 (Nisha Moond, 2026-09-22)](https://www.postgresql.org/message-id/CABdArM7Bc7SZPSHk5uT-yA6%3DDYVFW-Os4noB6CANH6GQw9rvew%40mail.gmail.com), [#25 (shveta malik, 2026-09-23)](https://www.postgresql.org/message-id/CAJpy0uAzTDwmq1F_N0XgADpp9TFFqEvvscWbH17Ym5LHeSP7sQ%40mail.gmail.com), [#26 (Mihail Nikalayeu, 2026-09-27)](https://www.postgresql.org/message-id/CADzfLwU97r3QC5oZDeVcWvcvXhNy9VnoFNDVHw%2BERg_DEG4%3DFQ%40mail.gmail.com)

**Question:** What must an existing subscriber do after the minor update, and how does the release note tell users that pg_publication_tables can now list a partition next to its root?

Counter-check: held. 0003 carries Backpatch-through: 14 and changes what pg_publication_tables lists. A refresh on a subscriber without the leaf table fails on the missing relation, and the proposal does not say what subscribers must do.

### 3. Back-patching 0004 adds locking and invalidation to stable branches

fixable · level 2, rubric item 6 · backpatchability, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

After a minor upgrade, a user runs DETACH PARTITION ... CONCURRENTLY on a partitioned partition with thousands of leaves. The first transaction now takes AccessShareLock on every leaf. It can wait behind a leaf held in AccessExclusiveLock, or run out of shared lock table space, where it succeeded before the upgrade.

0004 says Backpatch-through: 14, but Shveta in message #11 was not sure about side effects, and Amit in message #32 said the invalidation needs more discussion. The commit message explains the lock choice only against VACUUM. It does not cover lock count, wait behavior, or the cost of invalidation for large trees. A changed lock footprint in a minor release is a change that the project has to defend in every supported branch.

Thread: [#11 (shveta malik, 2026-09-08)](https://www.postgresql.org/message-id/CAJpy0uA6womAP7fkmT%3DsSfKG_0CfKUSb7HhP_J9mw071MXc39w%40mail.gmail.com), [#32 (Amit Kapila, 2026-11-11)](https://www.postgresql.org/message-id/CAA4eK1L%3D8-8NM0Ui2wttmq3fceODPAWrYLHmVLTt276KQkdFeg%40mail.gmail.com)

**Question:** What does the first DETACH CONCURRENTLY transaction now wait for or allocate on a partition tree with many leaves, and why is that acceptable in a minor release?

Counter-check: held. Shveta's side-effect question from message #11 has no answer, and the first transaction gains a new wait on descendants. The lock-count part is weak, because the 0004 commit message notes that DetachPartitionFinalize() already takes AccessExclusiveLock on them.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | pgoutput publishes a detach-pending partition as a standalone table, and 0004 changes when that takes effect. |
| wire protocol | no |  |
| catalog changes | **yes** | 0003 changes the rows that pg_publication_tables and pg_get_publication_tables() return; no catalog format changes. |
| ABI / extension API | **yes** | v2-0001 touches partition.c, and the suggested flag for get_partition_ancestors() would change a function that extensions can call. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | 0003 and 0004 carry Backpatch-through: 14 and change publication lists, replication routing and locking in stable branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each caller of get_partition_ancestors() must handle the detach-pending state itself; the HEAD-only interface change has no patch yet. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Are 0003 and 0004 meant for HEAD only, or for stable branches as their Backpatch-through: 14 lines say?
2. If 0004 is back-patched, why is it acceptable for a minor release to stop replicating a detach-pending partition's changes via the former root?
3. What must an existing subscriber do after the update when pg_publication_tables starts listing a detach-pending partition next to its root, and is REFRESH PUBLICATION required?
4. What does the first DETACH CONCURRENTLY transaction now wait for on a partitioned partition with many leaves, and does the extra invalidation affect anything beyond replication?
5. With PG14 at end of life on 2026-11-12, which branch is the oldest back-patch target for each patch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Alvaro Herrera, who wrote concurrent detach, offered in message #2 to push the fixes; Amit Kapila reviews the logical replication parts.
- **Liability:** The owner signs for changed replication output and a new lock and invalidation step in DETACH CONCURRENTLY, in every supported branch if 0003 and 0004 are back-patched.
- **Author follow-up:** The authors posted v2 and v3 after review, and the v19 relcache fix is committed; this suggests they stay for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** These are crash and correctness fixes for logical replication and identity columns on partitioned tables, with a walsender crash loop and a v19 open item. The patches are small, but 0003 and 0004 touch pgoutput, publication lookup and tablecmds.c, and their back-patch scope is not settled.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Standalone publishing of a detach-pending partition can stop root-only subscribers. Refuted (no failure scenario): A root-only subscriber of a FOR ALL TABLES publication gets the same apply error after FINALIZE, or today after a walsender restart. Message #12 weighed root routing and rejected it for data divergence; the back-branch part is a duplicate of standards-compat-1.
- `replication-ondisk-2` Back-patched 0003 and 0004 change publication membership without subscriber guidance. Refuted (not specific): Duplicate of standards-compat-1 for 0004 and of standards-compat-2 for 0003.
- `maintenance-fit-1` Per-caller checks leave the get_partition_ancestors() trap for future callers. Refuted (already handled): Messages #2, #7 and #8 weighed the interface change and deferred it to a separate HEAD-only patch, because back branches cannot take a changed function signature. The missing follow-up is not a defect of these crash fixes.
- `maintenance-fit-3` 0003 changes publication membership in back branches without upgrade guidance. Refuted (not specific): Duplicate of standards-compat-2, which has the same root-only subscriber and refresh scenario.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P2] was a revert for a release code freeze, which does not apply here, and [P3] routed inheritance children through a parent, which this proposal does not do; the thread rejects that routing in message #12. The earlier discussions the thread mentions (Nisha's report in message #4, the get_partition_root thread in message #5, Alvaro's API idea in message #2) are related open work, not rejections.
- standards-compat: 2 objection(s). The crash fixes themselves raise no standards or compatibility concern. Both objections are about back-patching the behavior changes in 0003 and 0004.
- replication-ondisk: 2 objection(s). No on-disk, WAL record or catalog format change is proposed. The v19 relcache fix and the pgoutput crash fix (v3-0001) look sound from this angle.
- security: 0 objection(s). The series opens no new security exposure. It removes crashes that a table owner can trigger by cancelling a concurrent detach. The publication change follows the existing rule that a detached table in a FOR ALL TABLES publication is published on its own, and the extra AccessShareLock in 0004 is weaker than the lock FINALIZE already takes.
- maintenance-fit: 3 objection(s). The crash fixes are wanted and engaged committers own them; objections cover the API trap and the stable-branch scope of 0003 and 0004.
- already-exists: 0 objection(s). The series fixes crashes and wrong publication state in core code paths (pgoutput, getIdentitySequence(), relcache, pg_publication), so no extension, hook or existing option can stand in for it. The only nearby workaround, DETACH PARTITION ... FINALIZE, ends the bad state but does not prevent the crashes, and walsender decodes already-written changes against historic catalogs.
- demand: 0 objection(s). Demand is not in doubt. These are crash and correctness fixes in shipped behavior: walsender crash loops and assertion failures. Several people hit them on their own: the author in stress testing, Nisha while testing another patch (message #4), Zsolt from an automated review (message #22), and Sawada in his own work (message #27). Committers treat the v19 part as an open item. The demand angle has no grounds to object; the open questions about 0004 side effects and back-patch scope belong to the design angle.

### Precedents retrieved

- P1: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant; reason summarised from its thread
- P2: [Revert "Fix partition pruning setup during DETACH CONCURRENTLY"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b0ea16528cda1f3a993bbfeb2b93eed9762af4e6) (revert, Reverted, 2024-06-24) - used in the review
- P3: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - used in the review; reason summarised from its thread
- P4: [propagating replica identity to partitions](https://commitfest.postgresql.org/patch/2044/) (cf, Returned with feedback, 2019-09-11) - judged not relevant; reason summarised from its thread
- P5: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P6: [Prevent pg_basebackup running as root](https://commitfest.postgresql.org/patch/2431/) (cf, Rejected, 2020-02-27) - judged not relevant; reason summarised from its thread
- P7: [use AV work items infrastructure for GIN pending list's cleanup](https://commitfest.postgresql.org/patch/3060/) (cf, Returned with feedback, 2021-09-08) - judged not relevant; reason summarised from its thread
- P8: [Revert "Fix issues with Windows' stat() for files pending on deletion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c9b46c090e76c62f24563b9be2c34e6b92e9329) (revert, Reverted, 2021-07-12) - judged not relevant
- P9: [Revert patch pending more discussion:](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=782df58a1cafbef5defb329a7583dfa0154dfe72) (revert, Reverted, 2006-04-30) - judged not relevant
- P10: [Revert error message on GLOBAL/LOCAL pending further discussion](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=28ac7972873bd314d4837807396fe44571b5cb50) (revert, Reverted, 2012-06-10) - judged not relevant
- P11: [Separate memory contexts for relcache and catcache](https://commitfest.postgresql.org/patch/4554/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P12: [Revert exports.list change pending closer study.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=36b870623698332144c73e950d6a8f91e883d708) (revert, Reverted, 2006-12-28) - judged not relevant
- P13: [USAGE privilege on PUBLICATION](https://commitfest.postgresql.org/patch/3641/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P14: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P15: [Multi-column list partitioning](https://commitfest.postgresql.org/patch/3383/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P16: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P17: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P18: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant
- P19: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4739/) (cf, Rejected, 2024-04-08) - judged not relevant
- P20: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P21: [Revert "Avoid the creation of the free space map for small heap relations".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7db0cde6b58eef2ba0c70437324cbc7622230320) (revert, Reverted, 2019-05-07) - judged not relevant
- P22: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P23: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4750/) (cf, Withdrawn, 2024-10-12) - judged not relevant
- P24: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 146 s; cost reported by the CLI: $2.82
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 81 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:43:44.339Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
