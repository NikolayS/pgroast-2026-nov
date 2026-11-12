# Design pre-review: Fix replica identity mismatch for partitioned tables with publish_via_partition_root

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6047](https://commitfest.postgresql.org/patch/6047/) (Needs review); authors: Mikhail Kharitonov (mkharitonov)
- Thread: [\[PATCH\] Fix replica identity mismatch for partitioned tables with publish_via_partition_root](https://www.postgresql.org/message-id/flat/CAKkoVatDVuH%3DAcnecrZSOQ0_Md6RfW4ExvCKJzJJ4JTrX1wbxQ%40mail.gmail.com), started 2025-05-05
- Messages read: 11 of 11
- Patch prose read: v3-0002-replica-identity-test.patch, v3-0001-Fix-replica-identity-flags-for-partitioned-tables.patch, 0001-Fix-replica-identity-flags-for-partitioned-tables.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal of the tag fix is sound: message #3 reports a real CDC failure and the docs define 'O' as a full old row. The design is not settled. Per standards-compat-2, a 'K' tuple still contradicts the root's Relation message, which reports FULL identity. Per standards-compat-1 and demand-1, v4-0002 rejects setups that work today without a restore or upgrade plan. The stable target (standards-compat-3) also needs agreement before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it is not possible to check which DDL paths v4-0002 covers, whether v4-0001 keeps the _extended functions, or what the leaf lookup costs per decoded change.

## The proposal, restated

**Problem.** With publish_via_partition_root = true and REPLICA IDENTITY FULL on the root only, pgoutput tags UPDATE and DELETE old tuples 'O' even for leaves whose replica identity is a key. External CDC consumers that trust the 'O' tag as meaning a full old row then process data wrongly. Sawada also showed native subscribers without a key index fail to find rows, because non-key columns arrive as NULL.

**Proposed design.** pgoutput picks the 'O' or 'K' tag from the replica identity of the leaf partition that held the old row. Message layout and relation OID stay those of the root. Korotkov's v4-0002 also rejects configurations where the leaf's replica identity does not cover the root's, checked at CREATE PUBLICATION, ALTER PUBLICATION SET, ATTACH PARTITION and ALTER TABLE REPLICA IDENTITY.

**User-visible surface.**
- Protocol: UPDATE and DELETE messages may carry 'K' instead of 'O' for leaves without FULL replica identity under publish_via_partition_root.
- Doc note in logical-replication.sgml saying the old-tuple flag follows the leaf's replica identity.
- C API (v3): new logicalrep_write_update_extended and logicalrep_write_delete_extended, with the old functions kept as wrappers.
- v4-0002: new errors in CREATE PUBLICATION, ALTER PUBLICATION SET, ATTACH PARTITION and ALTER TABLE REPLICA IDENTITY when leaf identity does not cover root identity.

**What the author claims.**
- The current 'O' tag with key-only content violates the documented logical replication message format.
- Native Postgres-to-Postgres apply is unaffected by the tag change; data applies correctly.
- A client's custom CDC pipeline to Oracle processes rows wrongly because it trusts the 'O' tag.
- The fix is minimal and does not change protocol structure or add behaviour.
- Reviewer bahn confirmed v3 changes the tag to 'K' while the payload, with NULL for non-key columns, stays the same.

**Thread state.** Korotkov posted v4 in August 2026, with a revised tag fix and a new DDL check, and asked for feedback on 0002. Henson Choi's question about the _extended wrappers and the native subscriber divergence are still open, and CFbot reports the patch needs a rebase.

## 2. Prior attempts

### propagating replica identity to partitions (related)

- **Link:** [https://commitfest.postgresql.org/patch/2044/](https://commitfest.postgresql.org/patch/2044/)
- **Thread:** [https://www.postgresql.org/message-id/flat/201902041630.gpadougzab7v@alvherre.pgsql](https://www.postgresql.org/message-id/flat/201902041630.gpadougzab7v@alvherre.pgsql)
- **Date:** decided 2019-09-11; started 2019-02-04
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew it for lack of time, with work still left in the patch. pg_dump had to emit the replica identity only after the index exists in all partitions, and a bug reported by Dmitry Dolgov was still open. Reviewers agreed with the recursion; the author planned to resubmit for November.
- **Reason (quote):** "I've not had time to work on this issue, and there are a few remaining problems in the patch; mostly that I will need to change the way pg_dump represents the replica identity so that it is only defined when the index is dumped in all partitions rather than immediately after creating the index." ([message](https://www.postgresql.org/message-id/20190911174423.GA28675%40alvherre.pgsql))
- **What is different this time:** P1 made replica identity recurse to partitions. v4-0002 keeps identities independent and rejects leaves that do not cover the root; v4-0001 only changes the old-tuple tag.

### Allow logical replication via inheritance root table (related)

- **Link:** [https://commitfest.postgresql.org/patch/4225/](https://commitfest.postgresql.org/patch/4225/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAAWbhmiRRJDzbA4fv0vAViu_wfoeYqtT+VYOOu7845WO-yT35w@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAWbhmiRRJDzbA4fv0vAViu_wfoeYqtT+VYOOu7845WO-yT35w@mail.gmail.com)
- **Date:** decided 2023-08-31; started 2022-12-09
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Jacob Champion, Amit Kapila (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author paused the entry after finding that the subscriber side could not handle the routed data: when incoming tuples are moved to other tables, a later UPDATE or DELETE checks the root table and does not find the rows. He decided to work on subscriber-side support first. Amit Kapila also said that one extension's needs are not a reason to accept the feature in core.
- **Reason (quote):** "Given that, I think I need to pivot and focus on the subscriber side first. That might(?) be a smaller effort anyway, and if we can't make headway there then publisher-side support probably doesn't make sense at all." ([message](https://www.postgresql.org/message-id/CAAWbhmjcnoV7Xu6LHr_hxqWmVtehv404bvDye%2BQZcUDSg8NSKw%40mail.gmail.com))
- **What is different this time:** P3 routed inheritance children through a parent and paused on subscriber-side row lookup. This patch fixes only the publisher tag; the subscriber miss in message #7 stays unless v4-0002 forbids the setup.


## 3. Objections a reviewer is likely to raise

### 1. A 'K' tuple conflicts with the root Relation message's FULL identity

fixable · level 2, rubric item 12 · wire protocol, SQL-standard conflict or non-standard syntax · `standards-compat-2`

A CDC consumer reads the Relation message for root p, which reports replica identity FULL. It then gets a 'K' DELETE in root layout with NULL in non-key columns, and matches on every column marked as key, NULLs included. The rows are still processed wrongly, now under a tag the protocol defines differently.

The doc hunk keeps the relation OID and tuple layout of the root, but takes the tag from the leaf. Message #9 confirms the payload does not change: b still arrives as NULL. So the stream now contradicts itself in a different place, because the Relation message describes the key columns by the root's identity. The protocol docs do not define a 'K' tuple whose key columns are given by a different relation than the one announced.

Thread: [#7 (Masahiko Sawada, 2026-01-07)](https://www.postgresql.org/message-id/CAD21AoD42cJ61nKjriDuX0K-iZWK6xXA2w35NBK_jNN9qKmW8w%40mail.gmail.com), [#9 (jihyun bahn, 2026-06-17)](https://www.postgresql.org/message-id/178167335561.992.15549275027332613587.pgcf%40coridan.postgresql.org)

**Question:** Given the root's Relation message, how does a consumer tell which columns of a 'K' tuple are real key values?

Counter-check: held. pgoutput marks every column as key in the Relation message when the published relation is FULL, and message #9 confirms the payload still has NULL there. v4-0002 would forbid this setup, but it is not settled and 0001 stands alone without it.

### 2. v4-0002 makes existing setups fail at restore, upgrade and minor update

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 13 · pg_dump / restore round-trip, on-disk format / pg_upgrade, backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A site runs a root with REPLICA IDENTITY FULL, a leaf with default identity, and a publication with publish_via_partition_root. After the change, pg_dump output fails at CREATE PUBLICATION or ATTACH PARTITION on restore, and pg_upgrade fails partway. With the stable target, an unrelated ALTER TABLE starts to error after a minor update.

Message #11 adds new errors at four DDL points, but the configurations it rejects are valid and in use today. Message #1 and message #7 build exactly this setup. The CommitFest entry targets stable, and no message says whether 0002 is meant for back branches. No message covers dump ordering, pg_upgrade checks, or what happens to catalogs that already violate the rule.

**Precedents.**
- [propagating replica identity to partitions](https://commitfest.postgresql.org/patch/2044/) (Returned with feedback, 2019-09-11); thread: [link](https://www.postgresql.org/message-id/flat/201902041630.gpadougzab7v@alvherre.pgsql)

Thread: [#1 (Mikhail Kharitonov, 2025-05-05)](https://www.postgresql.org/message-id/CAKkoVatDVuH%3DAcnecrZSOQ0_Md6RfW4ExvCKJzJJ4JTrX1wbxQ%40mail.gmail.com), [#7 (Masahiko Sawada, 2026-01-07)](https://www.postgresql.org/message-id/CAD21AoD42cJ61nKjriDuX0K-iZWK6xXA2w35NBK_jNN9qKmW8w%40mail.gmail.com), [#11 (Alexander Korotkov, 2026-08-02)](https://www.postgresql.org/message-id/CAPpHfds-NTdAMqy1%2B8i07WBoMDM9agr9773Xy2hsQquZEwedEQ%40mail.gmail.com)

**Question:** How do pg_dump restore and pg_upgrade handle existing publications that break the coverage rule, and is 0002 intended for back branches?

Counter-check: held. Messages #1 and #7 build the setup that v4-0002 rejects, and no message covers restore, pg_upgrade or back branches. A master-only 0002 with a pg_upgrade check could solve it, so it is fixable; [P1] must not be used as support.

### 3. Coverage check in four DDL points leaves other paths unguarded

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), catalog changes, pg_dump / restore round-trip · `maintenance-fit-1`

A user drops the index behind a leaf's REPLICA IDENTITY USING INDEX, or creates a leaf with CREATE TABLE ... PARTITION OF under a FOR ALL TABLES publication. Neither path is among the four checked points, so the mismatch v4-0002 forbids exists anyway. Nobody finds out until a subscriber fails to find a row.

Message #11 enforces the rule at CREATE PUBLICATION, ALTER PUBLICATION SET, ATTACH PARTITION and ALTER TABLE REPLICA IDENTITY only. The invariant depends on root identity, leaf identity, the identity index, partition membership and the publication option, so many more DDL paths can break it. Any later feature that touches these paths must remember the check, and nothing fails loudly if it does not. A committer would likely prefer one place that decides the rule, or a rule that cannot drift.

Thread: [#11 (Alexander Korotkov, 2026-08-02)](https://www.postgresql.org/message-id/CAPpHfds-NTdAMqy1%2B8i07WBoMDM9agr9773Xy2hsQquZEwedEQ%40mail.gmail.com)

**Question:** Which DDL paths can produce a leaf whose replica identity does not cover the root's, and how does v4-0002 handle each one beyond the four listed?

Counter-check: held. Message #11 lists four check points, and create table ... partition of, ALTER PUBLICATION ADD and dropping a leaf's identity index are not among them. Whether the code catches these cannot be checked from the material, so the question stands.

### 4. Changing pgoutput wire output in a minor release

fixable · level 1b, rubric item 6 · backpatchability, wire protocol · `standards-compat-3`

An existing consumer handles today's 'O' messages for these partitions, for example by ignoring NULLs. After a minor update, it receives 'K' for the same relation with no protocol version change. If it treats 'K' as unexpected for a FULL-identity relation, it errors or drops changes.

The entry targets stable, so the tag change would reach running consumers without them asking for it. Message #3 cites one client that gains from the change. No message checks for consumers that rely on the current output, and the brief lists this as unknown. Message #9 shows native apply does not change, so the back-patch helps external consumers only, and those are also the ones it can break.

Thread: [#3 (Mikhail Kharitonov, 2025-05-29)](https://www.postgresql.org/message-id/CAKkoVau0NBdKApEOqnuN%2Bvba6RP41Xs73Y%3DL7SALuOyhPQhHVg%40mail.gmail.com), [#9 (jihyun bahn, 2026-06-17)](https://www.postgresql.org/message-id/178167335561.992.15549275027332613587.pgcf%40coridan.postgresql.org)

**Question:** Why should this tag change go to back branches without a protocol version bump, rather than to master only?

Counter-check: held. The CommitFest entry targets stable and no message discusses consumers of the current output or a master-only option. The bug-fix label does not answer this, because message #9 shows the payload stays the same and only the tag changes.

### 5. New DDL errors in v4-0002 charge all mismatched setups for a narrow gain

fixable · level 1b, rubric item 4 · desirability, pg_dump / restore round-trip, backpatchability, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A site has a FULL root with key-identity leaves and subscribers that have a primary key, and replication works. After the change, ATTACH PARTITION or ALTER PUBLICATION SET fails, and so may a restore by pg_dump. The site finds out during a maintenance window or a restore.

Message #9 shows that subscribers with a key index apply these changes correctly, and only subscribers with no index diverge. v4-0002 makes every user of the mixed setup pay, but only that second group gains. No one in the thread asked for enforcement, and no one reported the divergence from the field. Message #11 proposes the check, and the thread has not weighed what it costs users who run this setup today.

Thread: [#7 (Masahiko Sawada, 2026-01-07)](https://www.postgresql.org/message-id/CAD21AoD42cJ61nKjriDuX0K-iZWK6xXA2w35NBK_jNN9qKmW8w%40mail.gmail.com), [#9 (jihyun bahn, 2026-06-17)](https://www.postgresql.org/message-id/178167335561.992.15549275027332613587.pgcf%40coridan.postgresql.org), [#11 (Alexander Korotkov, 2026-08-02)](https://www.postgresql.org/message-id/CAPpHfds-NTdAMqy1%2B8i07WBoMDM9agr9773Xy2hsQquZEwedEQ%40mail.gmail.com)

**Question:** How many deployments run a root and leaves with different replica identity, and why should those whose subscribers apply correctly accept new DDL errors?

Counter-check: held. Message #9 shows that subscribers with a key index apply correctly under the mixed setup, and v4-0002 would reject that setup. Message #9 asked whether a warning is preferred and message #11 asked for feedback, so the thread has not weighed this cost.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | v4-0002 adds DDL errors that pg_upgrade's schema restore can hit for existing mixed-identity setups. |
| WAL / replication / standby | **yes** | The patch changes what pgoutput sends for UPDATE and DELETE under publish_via_partition_root. |
| wire protocol | **yes** | The old-tuple tag changes from 'O' to 'K' for some leaves, with no protocol version change. |
| catalog changes | **yes** | see objection maintenance-fit-1: Coverage check in four DDL points leaves other paths unguarded |
| ABI / extension API | **yes** | v3 adds logicalrep_write_update_extended and logicalrep_write_delete_extended to logicalproto.h. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: A 'K' tuple conflicts with the root Relation message's FULL identity |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | v4-0002 errors at CREATE PUBLICATION, ATTACH PARTITION and ALTER TABLE REPLICA IDENTITY, which pg_dump output replays. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable, and both the tag change and the new errors are user-visible. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v4-0002 puts a cross-object rule at four DDL points that later DDL work must keep in step. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: New DDL errors in v4-0002 charge all mismatched setups for a narrow gain |

## 5. Questions the author must answer before anyone reviews code

1. Which of v4-0001 and v4-0002 is meant for back branches, and why is a tag change without a protocol version bump acceptable in a minor release?
2. With the root's Relation message marking all columns as key, how does a consumer tell which columns of a 'K' tuple hold real key values?
3. What do pg_dump restore and pg_upgrade do with an existing publication whose leaves do not cover the root's replica identity?
4. Why should sites whose subscribers have a key index, and apply correctly today, get an error and not a warning from v4-0002?
5. Which DDL paths beyond the four listed can produce a leaf that does not cover the root's identity, and does v4-0002 check each one?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Korotkov posted v4 in message #11. The owner needs to know pgoutput, the logical replication protocol, and publication and partition DDL.
- **Liability:** The owner signs for a wire-visible change that external consumers parse, and, with v4-0002, for a DDL rule that pg_dump, pg_upgrade and all later partition or publication DDL must respect indefinitely.
- **Author follow-up:** The author posted revised versions through v3 and moved the entry to a new CommitFest in September 2026. The patch needs a rebase.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The affected setup is a publish_via_partition_root publication with mixed replica identities read by an external consumer. The tag patch is small, but v4-0002 adds DDL rules with restore and upgrade effects, and the back-branch scope is not agreed.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-4` (fixable, level 2) Extended write functions add API surface with no external caller. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` Coverage check repeats the pg_dump ordering problem of the 2019 attempt. Refuted (not specific): Duplicate of standards-compat-1, which states the restore and upgrade failure more concretely. [P1] stopped on pg_dump ordering for a recursing identity, a related but different problem, so it is weak support.
- `replication-ondisk-1` Coverage check in v4-0002 can break pg_dump restore and pg_upgrade. Refuted (not specific): Duplicate of standards-compat-1.
- `replication-ondisk-2` 'K' tag with a FULL root Relation message stays inconsistent. Refuted (not specific): Duplicate of standards-compat-2; its stable-branch sentence duplicates standards-compat-3.
- `replication-ondisk-3` The four DDL checks leave other ways to break coverage. Refuted (not specific): Duplicate of maintenance-fit-1. It also lists SET (publish_via_partition_root = true) as unchecked, which the ALTER PUBLICATION SET check in message #11 plausibly covers.
- `maintenance-fit-2` New DDL errors may break pg_dump restore and pg_upgrade. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-3` Wire-visible tag change and new errors proposed for stable branches. Refuted (not specific): Duplicate of standards-compat-3 for the tag change and of standards-compat-1 for the new errors.
- `already-exists-1` Consistent leaf replica identity already avoids the mislabeled old tuple. Refuted (wrong facts): 0001 and 0002 do not overlap fully. With an index-identity root and a FULL leaf, which 0002 allows, pgoutput today sends 'K' with a full old row, and only 0001 corrects that tag. The stable-branch point duplicates standards-compat-3.

### Review angles

- prior-attempts: 1 objection(s). v4-0001 does not repeat any recorded rejection reason. The one objection is about v4-0002.
- standards-compat: 3 objection(s). Objections were found; see the list.
- replication-ondisk: 3 objection(s). Objections above cover the wire tag semantics and the v4-0002 catalog check. No on-disk format or WAL record changes are proposed.
- security: 0 objection(s). No privilege or exposure aspect. v4-0001 changes only the 'O'/'K' flag, and the old-tuple payload stays the same, so no one sees more data than before. The v4-0002 errors are visible to table owners, but publication names in pg_publication are already readable by everyone. One thing for the design angle: a FOR ALL TABLES publication with publish_via_partition_root can block another owner's ATTACH PARTITION or REPLICA IDENTITY change.
- maintenance-fit: 4 objection(s). A committer who knows the area is engaged (message #11). This lowers the doubt that anyone would own the fix, but it does not settle the objections above.
- already-exists: 1 objection(s). No extension or contrib module fixes the pgoutput tag. The only existing alternative is to set replica identity on each leaf, and the objection above rests on that.
- demand: 1 objection(s). The tag fix in v4-0001 has enough demand for a bug fix. Postgres documents that 'O' means a full old row, a client's CDC pipeline breaks today (message #3), and message #9 confirms the fix. The objection above covers only v4-0002.

### Precedents retrieved

- P1: [propagating replica identity to partitions](https://commitfest.postgresql.org/patch/2044/) (cf, Returned with feedback, 2019-09-11) - used in the review; reason summarised from its thread
- P2: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P3: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - used in the review; reason summarised from its thread
- P4: [Revert "Fix bug in checking of IDENTIFY_SYSTEM result."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c4762886539b06da682045e19ba73dcee3789638) (revert, Reverted, 2014-08-19) - judged not relevant
- P5: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P6: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P7: [Prevent pg_basebackup running as root](https://commitfest.postgresql.org/patch/2431/) (cf, Rejected, 2020-02-27) - judged not relevant; reason summarised from its thread
- P8: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [logical decoding and replication of sequences, take 2](https://commitfest.postgresql.org/patch/3823/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P10: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P11: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1710/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P12: [possibility to rename root namespace in plpgsql](https://commitfest.postgresql.org/patch/2839/) (cf, Rejected, 2022-03-07) - judged not relevant
- P13: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1862/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P14: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P15: [Tab completion for ATTACH PARTITION](https://commitfest.postgresql.org/patch/4561/) (cf, Returned with feedback, 2024-01-29) - judged not relevant
- P16: [Respect client-initiated CopyDone during logical streaming replication](https://commitfest.postgresql.org/patch/621/) (cf, Withdrawn, 2019-07-08) - judged not relevant
- P17: [Revert "Make pg_dump emit ATTACH PARTITION instead of PARTITION OF"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a1ec7402e9a0392f95bc79542af0efcd5c6e7929) (revert, Reverted, 2019-05-06) - judged not relevant
- P18: [Propagate sanity checks of ProcessUtility() to standard_ProcessUtility()?](https://commitfest.postgresql.org/patch/4877/) (cf, Withdrawn, 2024-05-18) - judged not relevant
- P19: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P20: [Vacuuming relations on full tablespaces (Add EMERGENCY option)](https://commitfest.postgresql.org/patch/649/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P21: [Autovacuum (analyze) on partitioned tables for ATTACH/DETACH/DROP commands](https://commitfest.postgresql.org/patch/3188/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P22: [Revert analyze support for partitioned tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8127b7390119c21479f5ce495b7d2168930e82) (revert, Reverted, 2021-08-16) - judged not relevant
- P23: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - judged not relevant
- P24: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 186 s; cost reported by the CLI: $2.38
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 117 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:40:13.453Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
