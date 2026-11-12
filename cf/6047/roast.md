# Design pre-review: Fix replica identity mismatch for partitioned tables with publish_via_partition_root

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6047](https://commitfest.postgresql.org/patch/6047/); [thread](https://www.postgresql.org/message-id/flat/CAKkoVatDVuH%3DAcnecrZSOQ0_Md6RfW4ExvCKJzJJ4JTrX1wbxQ%40mail.gmail.com), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal of the tag fix is sound: message #3 reports a real CDC failure and the docs define 'O' as a full old row. The design is not settled. Per standards-compat-2, a 'K' tuple still contradicts the root's Relation message, which reports FULL identity. Per standards-compat-1 and demand-1, v4-0002 rejects setups that work today without a restore or upgrade plan. The stable target (standards-compat-3) also needs agreement before code review.

Not checked: Without the code, it is not possible to check which DDL paths v4-0002 covers, whether v4-0001 keeps the _extended functions, or what the leaf lookup costs per decoded change.

## Prior attempts

- [propagating replica identity to partitions](https://www.postgresql.org/message-id/flat/201902041630.gpadougzab7v@alvherre.pgsql) (related; Returned with feedback, 2019-09-11). "I've not had time to work on this issue, and there are a few remaining problems in the patch; mostly that I will need to change the way pg_dump represents the replica identity so that it is only defined when the index is dumped in all partitions rather than immediately after creating the index." ([message](https://www.postgresql.org/message-id/20190911174423.GA28675%40alvherre.pgsql)) Different now: P1 made replica identity recurse to partitions. v4-0002 keeps identities independent and rejects leaves that do not cover the root; v4-0001 only changes the old-tuple tag.
- [Allow logical replication via inheritance root table](https://www.postgresql.org/message-id/flat/CAAWbhmiRRJDzbA4fv0vAViu_wfoeYqtT+VYOOu7845WO-yT35w@mail.gmail.com) (related; Returned with feedback, 2023-08-31). "Given that, I think I need to pivot and focus on the subscriber side first. That might(?) be a smaller effort anyway, and if we can't make headway there then publisher-side support probably doesn't make sense at all." ([message](https://www.postgresql.org/message-id/CAAWbhmjcnoV7Xu6LHr_hxqWmVtehv404bvDye%2BQZcUDSg8NSKw%40mail.gmail.com)) Different now: P3 routed inheritance children through a parent and paused on subscriber-side row lookup. This patch fixes only the publisher tag; the subscriber miss in message #7 stays unless v4-0002 forbids the setup.

## Objections a reviewer is likely to raise

### 1. A 'K' tuple conflicts with the root Relation message's FULL identity

A CDC consumer reads the Relation message for root p, which reports replica identity FULL. It then gets a 'K' DELETE in root layout with NULL in non-key columns, and matches on every column marked as key, NULLs included. The rows are still processed wrongly, now under a tag the protocol defines differently.

The doc hunk keeps the relation OID and tuple layout of the root, but takes the tag from the leaf. Message #9 confirms the payload does not change: b still arrives as NULL. So the stream now contradicts itself in a different place, because the Relation message describes the key columns by the root's identity. The protocol docs do not define a 'K' tuple whose key columns are given by a different relation than the one announced.

**Question:** Given the root's Relation message, how does a consumer tell which columns of a 'K' tuple are real key values?

<sub>fixable, level 2, wire protocol; messages [#7](https://www.postgresql.org/message-id/CAD21AoD42cJ61nKjriDuX0K-iZWK6xXA2w35NBK_jNN9qKmW8w%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/178167335561.992.15549275027332613587.pgcf%40coridan.postgresql.org); `standards-compat-2`</sub>

### 2. v4-0002 makes existing setups fail at restore, upgrade and minor update

A site runs a root with REPLICA IDENTITY FULL, a leaf with default identity, and a publication with publish_via_partition_root. After the change, pg_dump output fails at CREATE PUBLICATION or ATTACH PARTITION on restore, and pg_upgrade fails partway. With the stable target, an unrelated ALTER TABLE starts to error after a minor update.

Message #11 adds new errors at four DDL points, but the configurations it rejects are valid and in use today. Message #1 and message #7 build exactly this setup. The CommitFest entry targets stable, and no message says whether 0002 is meant for back branches. No message covers dump ordering, pg_upgrade checks, or what happens to catalogs that already violate the rule.

Precedent: [propagating replica identity to partitions](https://www.postgresql.org/message-id/flat/201902041630.gpadougzab7v@alvherre.pgsql) (Returned with feedback, 2019-09-11)

**Question:** How do pg_dump restore and pg_upgrade handle existing publications that break the coverage rule, and is 0002 intended for back branches?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#1](https://www.postgresql.org/message-id/CAKkoVatDVuH%3DAcnecrZSOQ0_Md6RfW4ExvCKJzJJ4JTrX1wbxQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAD21AoD42cJ61nKjriDuX0K-iZWK6xXA2w35NBK_jNN9qKmW8w%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CAPpHfds-NTdAMqy1%2B8i07WBoMDM9agr9773Xy2hsQquZEwedEQ%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Coverage check in four DDL points leaves other paths unguarded

A user drops the index behind a leaf's REPLICA IDENTITY USING INDEX, or creates a leaf with CREATE TABLE ... PARTITION OF under a FOR ALL TABLES publication. Neither path is among the four checked points, so the mismatch v4-0002 forbids exists anyway. Nobody finds out until a subscriber fails to find a row.

Message #11 enforces the rule at CREATE PUBLICATION, ALTER PUBLICATION SET, ATTACH PARTITION and ALTER TABLE REPLICA IDENTITY only. The invariant depends on root identity, leaf identity, the identity index, partition membership and the publication option, so many more DDL paths can break it. Any later feature that touches these paths must remember the check, and nothing fails loudly if it does not. A committer would likely prefer one place that decides the rule, or a rule that cannot drift.

**Question:** Which DDL paths can produce a leaf whose replica identity does not cover the root's, and how does v4-0002 handle each one beyond the four listed?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#11](https://www.postgresql.org/message-id/CAPpHfds-NTdAMqy1%2B8i07WBoMDM9agr9773Xy2hsQquZEwedEQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. Changing pgoutput wire output in a minor release

An existing consumer handles today's 'O' messages for these partitions, for example by ignoring NULLs. After a minor update, it receives 'K' for the same relation with no protocol version change. If it treats 'K' as unexpected for a FULL-identity relation, it errors or drops changes.

The entry targets stable, so the tag change would reach running consumers without them asking for it. Message #3 cites one client that gains from the change. No message checks for consumers that rely on the current output, and the brief lists this as unknown. Message #9 shows native apply does not change, so the back-patch helps external consumers only, and those are also the ones it can break.

**Question:** Why should this tag change go to back branches without a protocol version bump, rather than to master only?

<sub>fixable, level 1b, backpatchability; messages [#3](https://www.postgresql.org/message-id/CAKkoVau0NBdKApEOqnuN%2Bvba6RP41Xs73Y%3DL7SALuOyhPQhHVg%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/178167335561.992.15549275027332613587.pgcf%40coridan.postgresql.org); `standards-compat-3`</sub>

### 5. New DDL errors in v4-0002 charge all mismatched setups for a narrow gain

A site has a FULL root with key-identity leaves and subscribers that have a primary key, and replication works. After the change, ATTACH PARTITION or ALTER PUBLICATION SET fails, and so may a restore by pg_dump. The site finds out during a maintenance window or a restore.

Message #9 shows that subscribers with a key index apply these changes correctly, and only subscribers with no index diverge. v4-0002 makes every user of the mixed setup pay, but only that second group gains. No one in the thread asked for enforcement, and no one reported the divergence from the field. Message #11 proposes the check, and the thread has not weighed what it costs users who run this setup today.

**Question:** How many deployments run a root and leaves with different replica identity, and why should those whose subscribers apply correctly accept new DDL errors?

<sub>fixable, level 1b, desirability; messages [#7](https://www.postgresql.org/message-id/CAD21AoD42cJ61nKjriDuX0K-iZWK6xXA2w35NBK_jNN9qKmW8w%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/178167335561.992.15549275027332613587.pgcf%40coridan.postgresql.org), [#11](https://www.postgresql.org/message-id/CAPpHfds-NTdAMqy1%2B8i07WBoMDM9agr9773Xy2hsQquZEwedEQ%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which of v4-0001 and v4-0002 is meant for back branches, and why is a tag change without a protocol version bump acceptable in a minor release?
2. With the root's Relation message marking all columns as key, how does a consumer tell which columns of a 'K' tuple hold real key values?
3. What do pg_dump restore and pg_upgrade do with an existing publication whose leaves do not cover the root's replica identity?
4. Why should sites whose subscribers have a key index, and apply correctly today, get an error and not a warning from v4-0002?
5. Which DDL paths beyond the four listed can produce a leaf that does not cover the root's identity, and does v4-0002 check each one?

## Touch points

- **on-disk format / pg_upgrade:** v4-0002 adds DDL errors that pg_upgrade's schema restore can hit for existing mixed-identity setups.
- **WAL / replication / standby:** The patch changes what pgoutput sends for UPDATE and DELETE under publish_via_partition_root.
- **wire protocol:** The old-tuple tag changes from 'O' to 'K' for some leaves, with no protocol version change.
- **catalog changes:** see objection maintenance-fit-1: Coverage check in four DDL points leaves other paths unguarded
- **ABI / extension API:** v3 adds logicalrep_write_update_extended and logicalrep_write_delete_extended to logicalproto.h.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: A 'K' tuple conflicts with the root Relation message's FULL identity
- **pg_dump / restore round-trip:** v4-0002 errors at CREATE PUBLICATION, ATTACH PARTITION and ALTER TABLE REPLICA IDENTITY, which pg_dump output replays.
- **backpatchability:** The CommitFest entry targets stable, and both the tag change and the new errors are user-visible.
- **maintenance burden ("who maintains this in 10 years"):** v4-0002 puts a cross-object rule at four DDL points that later DDL work must keep in step.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: New DDL errors in v4-0002 charge all mismatched setups for a narrow gain

## Who would commit it, and when

- Owner: qualified committer engaged. Korotkov posted v4 in message #11. The owner needs to know pgoutput, the logical replication protocol, and publication and partition DDL.
- Priority: fix design first; audience niche; review cost medium. The affected setup is a publish_via_partition_root publication with mixed replica identities read by an external consumer. The tag patch is small, but v4-0002 adds DDL rules with restore and upgrade effects, and the back-branch scope is not agreed.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
