# Design pre-review: Issue with synced logical replication slot during switchover

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6132](https://commitfest.postgresql.org/patch/6132/); [thread](https://www.postgresql.org/message-id/flat/CAA5-nLAqGpBFEAr2XNYMj3E%2B39caQra_SJeB5MCtp7PCyLTiOg%40mail.gmail.com), 66 of 66 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The entry does not name one design (maintenance-fit-3). The allow_overwrite series, the message #42 patch and the message #66 rule differ in on-disk and API cost. The demand is real and the goal is accepted in the thread. The reviewers still disagree on whether name plus failover=true is enough to overwrite a slot, so the list must settle the rule before code review makes sense.

Not checked: Without the code, this review cannot tell whether the message #42 patch checks any local slot property before the drop. It also cannot tell whether the recreated slot becomes sync-ready in the same cycle.

## Prior attempts

None found before 2025-08-07.

## Objections a reviewer is likely to raise

### 1. Two incompatible designs with no stated choice

A committer picking up the CF entry cannot tell which design is proposed. The v6 series adds an allow_overwrite field to on-disk slot data and changes the ReplicationSlotCreate() signature that extensions call. The message #42 patch changes only sync behaviour, so the commit and pg_upgrade cost differ completely between the two.

The CF entry still points to the allow_overwrite series as reviewed in messages #28 to #31. The latest commit message does not say whether it replaces that series. Message #64 shows a cascading switchover where allow_overwrite blocks a wanted overwrite. Message #66 proposes a third rule based on synchronized_standby_slots and primary_conninfo, with no patch. A permanent on-disk field and ABI change should not land while the semantics are still moving.

**Question:** Which single design does the entry now propose, and does it change on-disk slot data or the ReplicationSlotCreate() signature?

<sub>fixable, level 1a, problem statement; messages [#31](https://www.postgresql.org/message-id/CAA5-nLDXU3zVAKmeE5TYF0nyWjUDrxzK_2w5hmy0mgwqsEspRw%40mail.gmail.com), [#42](https://www.postgresql.org/message-id/CAA5-nLDqGFgZhV1T2GwjRAAWA-b0oBuWCkd1g1g0Aup45Dpm8A%40mail.gmail.com), [#64](https://www.postgresql.org/message-id/CAD21AoDH%2BNv62qoJ8P%2B2249u%3DHTyrWg3qGuk9vqOb014%3DzzWzg%40mail.gmail.com), [#66](https://www.postgresql.org/message-id/CAA5-nLCx5qL%3DGk7DYFmtyoOtASAEtsvDpo%3DZMyYFnuMv%3DqVFuQ%40mail.gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. Which single design does the entry now propose: the allow_overwrite series, the message #42 patch, or the message #66 rule?
2. Does the chosen design change the on-disk slot data or the ReplicationSlotCreate() signature?
3. Which properties of the local slot (plugin, database, LSNs, catalog_xmin) does the patch compare with the remote slot before dropping it, and what happens when they differ?
4. What slot state, other than failover=true, would tell a former primary's slot apart once standbys may create failover slots?
5. How does the message #42 patch shorten the time without sync-ready slots compared with pg_drop_replication_slot() followed by pg_sync_replication_slots()?

## Touch points

- **on-disk format / pg_upgrade:** The earlier series adds an allow_overwrite field to the on-disk slot data.
- **WAL / replication / standby:** The latest patch changes when the slot sync worker drops and recreates local logical slots on a standby.
- **catalog changes:** The earlier series changes pg_proc.dat and system_functions.sql and adds a column to pg_replication_slots.
- **ABI / extension API:** The earlier series changes the ReplicationSlotCreate() C signature in slot.h.
- **backpatchability:** The CF entry is tagged Bugfix for a failure reported on 17.5, but the fix changes slot sync behavior.
- **maintenance burden ("who maintains this in 10 years"):** The latest patch reads failover=true as slot provenance, which later changes to failover slot creation must recheck.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** HA tools can already drop the slot and call pg_sync_replication_slots(), as message #40 notes.

## Who would commit it, and when

- Owner: qualified committer engaged. Amit Kapila and Masahiko Sawada, who know slot sync, take part in the thread; neither has endorsed a design.
- Priority: needs design rethink; audience a sizeable minority; review cost medium. Message #1 reports a production failure with Patroni, and a Patroni maintainer confirms the gap in message #36, so the goal is valid. The latest patch is small and touches only slotsync.c, but three candidate rules are open and none has consensus.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
