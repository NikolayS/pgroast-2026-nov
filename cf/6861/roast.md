# Design pre-review: Separate catalog_xmin from xmin in walsender hot standby feedback

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6861](https://commitfest.postgresql.org/patch/6861/); [thread](https://www.postgresql.org/message-id/flat/3544CABD-2D09-4D7F-AFB7-2C75A09C4E22%40gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem is real: the code comment admits it, and Amit Kapila engaged with it in message #3. The thread does not say why wal_receiver_create_temp_slot on the standby is not enough (already-exists-1). The ephemeral-slot version also changes WAL retention and max_replication_slots use for every slotless feedback standby, while only standbys with an older catalog_xmin gain (demand-2). No report from a production system is shown (demand-1).

Not checked: The code was not read, and the text of message #4 is not in the material. Whether restart_lsn must be set, how a name collision behaves, and why repack_worker.c, launcher.c and slotsync.c change could not be checked.

## Prior attempts

None found before 2026-04-30.

## Objections a reviewer is likely to raise

### 1. wal_receiver_create_temp_slot already gives slotless standbys a separate catalog_xmin

An HA operator who avoids permanent slots can already set wal_receiver_create_temp_slot = on on the standby. The walreceiver then creates a temporary physical slot, and the primary tracks xmin and catalog_xmin separately through that slot, which drops on disconnect. The patch adds a second, server-side mechanism for the same job.

Message #1 lists the slot downsides as lost slots after failover, re-creating slots on the new primary, and dangling slots. A temporary slot created by the walreceiver avoids all three: the standby creates it on each new upstream, and the upstream drops it at disconnect. The ephemeral-slot version in message #4 is close to this existing feature, but primary-side and with temporary = 'f'. The thread does not mention the existing setting or say why it is not enough.

**Question:** What does a primary-created pg_walsender_<pid> slot do that a standby with wal_receiver_create_temp_slot = on cannot do today?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/3544CABD-2D09-4D7F-AFB7-2C75A09C4E22%40gmail.com), [#4](https://www.postgresql.org/message-id/57E2263F-455D-49A5-A16D-E66562DBC027%40gmail.com); `already-exists-1`</sub>

### 2. All slotless feedback standbys pay for a fix few of them need

A DBA runs standbys without slots on purpose and relies on wal_keep_size. After an upgrade, every walsender with feedback creates pg_walsender_<pid>. A lagging standby now pins WAL, and with the default max_slot_wal_keep_size = -1 the primary disk can fill. Slot-count monitoring also starts alerting on slots with temporary = 'f'.

Only standbys that host a logical slot gain from the change. The ephemeral-slot version still changes WAL retention and max_replication_slots use for every slotless standby that sends feedback, and the material shows no way to turn it off. Message #4 calls the WAL pinning 'a feature', but these users chose to avoid slots for this reason, as message #1 itself says. The claimed bound depends on max_slot_wal_keep_size, which is unlimited by default.

**Question:** Can the slot be created only when feedback carries a catalog_xmin different from xmin, and how do sites that use wal_keep_size keep today's WAL retention?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/3544CABD-2D09-4D7F-AFB7-2C75A09C4E22%40gmail.com), [#4](https://www.postgresql.org/message-id/57E2263F-455D-49A5-A16D-E66562DBC027%40gmail.com); `demand-2`</sub>

### 3. No field evidence that the affected group is more than a narrow case

A committer accepts a new walsender slot lifecycle and a changed ReplicationSlotCreate API. The bug only bites setups that combine a slotless physical standby, hot_standby_feedback, and a logical slot on that standby with an old catalog_xmin. If few sites run that combination, the project carries the code and its edge cases for little gain.

Message #1 asserts that 'many production HA deployments' avoid physical slots and points to 'Patroni with certain configurations', but it gives no bug report, -general thread or bloat numbers from a real system. Nobody besides the author asks for the fix in the thread. The only validation in message #4 shows the horizons differing by one xid on a test pair. A workaround also exists: give that one standby a physical slot, bounded by max_slot_wal_keep_size.

**Question:** Which production systems hit this combination, how much bloat did it cause, and why was a physical slot for that standby not an acceptable fix for them?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/3544CABD-2D09-4D7F-AFB7-2C75A09C4E22%40gmail.com), [#4](https://www.postgresql.org/message-id/57E2263F-455D-49A5-A16D-E66562DBC027%40gmail.com); `demand-1`</sub>

## Questions for the author

1. What does a primary-created pg_walsender_<pid> slot do that a standby with wal_receiver_create_temp_slot = on cannot do today?
2. Can the slot be created only when feedback carries a catalog_xmin older than xmin, so that other slotless standbys see no change?
3. Can the slot track xmin and catalog_xmin with restart_lsn left invalid, so that wal_keep_size setups keep today's WAL retention?
4. Which production systems hit this combination, how much bloat did it cause, and why was a physical slot for that standby not acceptable?
5. Should server-created walsender slots count against max_replication_slots, and what happens when a slot named pg_walsender_<pid> already exists?

## Touch points

- **WAL / replication / standby:** The walsender creates a physical slot whose restart_lsn pins WAL, and hot standby feedback handling changes.
- **ABI / extension API:** ReplicationSlotCreate gets a new error_if_full argument; the earlier version added a field to PGPROC.
- **backpatchability:** Message #1 calls it a fix for a long-standing issue, but it changes a C API and WAL retention.
- **maintenance burden ("who maintains this in 10 years"):** A new long-lived use of ephemeral slots in the walsender, with a fallback path when the slot pool is full.
- **performance on the hot path:** The PGPROC version adds a read per PGPROC in ComputeXidHorizons(); the slot version has no numbers.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** wal_receiver_create_temp_slot already lets a slotless standby get a temporary physical slot that tracks both horizons.

## Who would commit it, and when

- Owner: qualified committer engaged. Amit Kapila reviewed the PGPROC version in message #3 and suggested ephemeral slots; nobody has replied to the ephemeral-slot prototype. The owner needs to know replication slots and the xid horizon code.
- Priority: needs design rethink; audience niche; review cost medium. Only slotless feedback standbys that host a logical slot gain, but the slot version changes behavior for every slotless feedback standby. The patch is small but touches slot, walsender and horizon code, and the choice between PGPROC, an implicit slot and the existing standby setting is open.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 8 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
