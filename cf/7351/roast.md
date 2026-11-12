# Design pre-review: Crashes on a partition whose concurrent detach never finished

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7351](https://commitfest.postgresql.org/patch/7351/); [thread](https://www.postgresql.org/message-id/flat/CADzfLwWoFPT%2Ba73%3DA%3DbsNWRMZQ98NpBEMgE%3Dt1FS4O4_%3DQVLfA%40mail.gmail.com), 37 of 37 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The crash fixes are wanted bug fixes, and no objection at level 1a or 1b survived. The surviving objections (standards-compat-1, standards-compat-2, maintenance-fit-2) are all fixable and concern back-patching 0003 and 0004, which change replication output and locking in stable branches. Message #32 says the invalidation question needs more discussion, so the scope decision belongs to the committers engaged in the thread.

Not checked: Without the code, this review could not check whether 0004's AccessShareLock on descendants and the extra invalidation are safe against concurrent drops and large partition trees. Only 3 of 37 messages were visible, so later agreement on scope may exist.

## Prior attempts

None found before 2026-08-15.

## Objections a reviewer is likely to raise

### 1. Back-patching 0004 changes which rows replicate in a minor release

A publisher on PG15 has a FOR TABLE root publication with publish_via_partition_root = true, and one partition is stuck in detach pending. The subscriber receives that partition's changes through the root today. After a minor update with 0004, those changes stop arriving, and the subscriber finds out only when its rows no longer match the publisher's.

0004 carries Backpatch-through: 14, but in message #32 Amit calls the invalidation question unrelated to PG-19 and in need of more discussion. In message #11 Shveta was unsure whether the extra invalidation has other effects. Moving from root-routed to standalone may be the right rule, as message #12 argues, but changing it in a stable branch alters replication output for users who did not ask for it. The thread has not agreed whether this is HEAD-only.

**Question:** Why is changing replication routing for detach-pending partitions acceptable in stable branches, instead of HEAD only with a release note?

<sub>fixable, level 2, backpatchability; messages [#11](https://www.postgresql.org/message-id/CAJpy0uA6womAP7fkmT%3DsSfKG_0CfKUSb7HhP_J9mw071MXc39w%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CABdArM4fJ8JAWyP0MAF%3D8n66M6%3DCouUmtSGwqizn2pD%3DgfPEZw%40mail.gmail.com), [#32](https://www.postgresql.org/message-id/CAA4eK1L%3D8-8NM0Ui2wttmq3fceODPAWrYLHmVLTt276KQkdFeg%40mail.gmail.com); `standards-compat-1`</sub>

### 2. 0003 changes pg_publication_tables output in stable branches without subscriber guidance

After a minor update, pg_publication_tables lists a detach-pending partition next to its root for publish_via_partition_root publications. A subscriber that has only the root table runs ALTER SUBSCRIPTION ... REFRESH PUBLICATION. The refresh now needs a table the subscriber does not have, and the user finds out when the refresh fails or that partition's changes are skipped.

The 0003 commit message says it lets REFRESH PUBLICATION pick up the partition before FINALIZE, with Backpatch-through: 14. Today the documented outcome of publish_via_partition_root is that only the root is listed, and monitoring or schema-sync tools may rely on that. The proposal does not say what subscribers must do after the update, or whether a refresh is required. The brief lists both points as open.

**Question:** What must an existing subscriber do after the minor update, and how does the release note tell users that pg_publication_tables can now list a partition next to its root?

<sub>fixable, level 2, backpatchability; messages [#23](https://www.postgresql.org/message-id/CABdArM7Bc7SZPSHk5uT-yA6%3DDYVFW-Os4noB6CANH6GQw9rvew%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/CAJpy0uAzTDwmq1F_N0XgADpp9TFFqEvvscWbH17Ym5LHeSP7sQ%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CADzfLwU97r3QC5oZDeVcWvcvXhNy9VnoFNDVHw%2BERg_DEG4%3DFQ%40mail.gmail.com); `standards-compat-2`</sub>

### 3. Back-patching 0004 adds locking and invalidation to stable branches

After a minor upgrade, a user runs DETACH PARTITION ... CONCURRENTLY on a partitioned partition with thousands of leaves. The first transaction now takes AccessShareLock on every leaf. It can wait behind a leaf held in AccessExclusiveLock, or run out of shared lock table space, where it succeeded before the upgrade.

0004 says Backpatch-through: 14, but Shveta in message #11 was not sure about side effects, and Amit in message #32 said the invalidation needs more discussion. The commit message explains the lock choice only against VACUUM. It does not cover lock count, wait behavior, or the cost of invalidation for large trees. A changed lock footprint in a minor release is a change that the project has to defend in every supported branch.

**Question:** What does the first DETACH CONCURRENTLY transaction now wait for or allocate on a partition tree with many leaves, and why is that acceptable in a minor release?

<sub>fixable, level 2, backpatchability; messages [#11](https://www.postgresql.org/message-id/CAJpy0uA6womAP7fkmT%3DsSfKG_0CfKUSb7HhP_J9mw071MXc39w%40mail.gmail.com), [#32](https://www.postgresql.org/message-id/CAA4eK1L%3D8-8NM0Ui2wttmq3fceODPAWrYLHmVLTt276KQkdFeg%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Are 0003 and 0004 meant for HEAD only, or for stable branches as their Backpatch-through: 14 lines say?
2. If 0004 is back-patched, why is it acceptable for a minor release to stop replicating a detach-pending partition's changes via the former root?
3. What must an existing subscriber do after the update when pg_publication_tables starts listing a detach-pending partition next to its root, and is REFRESH PUBLICATION required?
4. What does the first DETACH CONCURRENTLY transaction now wait for on a partitioned partition with many leaves, and does the extra invalidation affect anything beyond replication?
5. With PG14 at end of life on 2026-11-12, which branch is the oldest back-patch target for each patch?

## Touch points

- **WAL / replication / standby:** pgoutput publishes a detach-pending partition as a standalone table, and 0004 changes when that takes effect.
- **catalog changes:** 0003 changes the rows that pg_publication_tables and pg_get_publication_tables() return; no catalog format changes.
- **ABI / extension API:** v2-0001 touches partition.c, and the suggested flag for get_partition_ancestors() would change a function that extensions can call.
- **backpatchability:** 0003 and 0004 carry Backpatch-through: 14 and change publication lists, replication routing and locking in stable branches.
- **maintenance burden ("who maintains this in 10 years"):** Each caller of get_partition_ancestors() must handle the detach-pending state itself; the HEAD-only interface change has no patch yet.

## Who would commit it, and when

- Owner: qualified committer engaged. Alvaro Herrera, who wrote concurrent detach, offered in message #2 to push the fixes; Amit Kapila reviews the logical replication parts.
- Priority: proceed; audience a sizeable minority; review cost medium. These are crash and correctness fixes for logical replication and identity columns on partitioned tables, with a walsender crash loop and a v19 open item. The patches are small, but 0003 and 0004 touch pgoutput, publication lookup and tablecmds.c, and their back-patch scope is not settled.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
