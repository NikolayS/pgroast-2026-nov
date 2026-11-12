# Design pre-review: Separate catalog_xmin from xmin in walsender hot standby feedback

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6861](https://commitfest.postgresql.org/patch/6861/) (Needs review); authors: Rui Zhao (zhaorui)
- Thread: [Separate catalog_xmin from xmin in walsender hot standby feedback](https://www.postgresql.org/message-id/flat/3544CABD-2D09-4D7F-AFB7-2C75A09C4E22%40gmail.com), started 2026-04-30
- Messages read: 4 of 4
- Patch prose read: v1-0001-Separate-catalog_xmin-from-xmin-via-ephemeral-phy.patch, v1-0001-Separate-catalog_xmin-from-xmin-in-walsender-hs-feedback.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem is real: the code comment admits it, and Amit Kapila engaged with it in message #3. The thread does not say why wal_receiver_create_temp_slot on the standby is not enough (already-exists-1). The ephemeral-slot version also changes WAL retention and max_replication_slots use for every slotless feedback standby, while only standbys with an older catalog_xmin gain (demand-2). No report from a production system is shown (demand-1).

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and the text of message #4 is not in the material. Whether restart_lsn must be set, how a name collision behaves, and why repack_worker.c, launcher.c and slotsync.c change could not be checked.

## The proposal, restated

**Problem.** Without a physical replication slot, the walsender stores min(xmin, catalog_xmin) from hot standby feedback in its PGPROC xmin. An old catalog_xmin from a logical slot on the standby then holds back vacuum of all user tables on the primary. The author targets HA setups that avoid physical slots and use wal_keep_size or archiving instead.

**Proposed design.** v1/v2 add a catalog_xmin field to PGPROC that ComputeXidHorizons() applies only to the catalog and shared horizons, and GetReplicationHorizons() passes upstream for cascading. After review, the author switched to having the walsender create an ephemeral physical slot (pg_walsender_<pid>) on first feedback and track both horizons through it. If max_replication_slots is full, it falls back to today's min() behavior.

**User-visible surface.**
- A physical slot named pg_walsender_<pid> appears in pg_replication_slots for each slotless feedback standby while it is connected, with temporary = 'f'.
- The new slot pins WAL through restart_lsn, bounded by max_slot_wal_keep_size.
- Slotless feedback walsenders now use up entries of max_replication_slots.
- C API: ReplicationSlotCreate gets a new error_if_full argument.

**What the author claims.**
- The data horizon is no longer held back by a standby's catalog_xmin, while catalog protection stays in place.
- The PGPROC version showed no measurable pgbench cost on a 104-core machine; the patched build was 1.3% to 1.9% faster, which the author calls noise.
- The ephemeral slot avoids the extra 4 bytes per PGPROC and updates both horizons together under the slot mutex.
- Pinning WAL for a connected standby is desirable, and the slot drops when the walsender exits.
- No replication connection breaks, because a full slot pool falls back to the current behavior.

**Thread state.** Amit Kapila objected to the PGPROC version (4 bytes per process, a possible torn read of xmin and catalog_xmin) and suggested ephemeral slots. The author posted an ephemeral-slot prototype; nobody has replied to it, the last message is truncated, and CFbot reports the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-04-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. wal_receiver_create_temp_slot already gives slotless standbys a separate catalog_xmin

fixable (first rated fatal; changed in the counter-check) · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An HA operator who avoids permanent slots can already set wal_receiver_create_temp_slot = on on the standby. The walreceiver then creates a temporary physical slot, and the primary tracks xmin and catalog_xmin separately through that slot, which drops on disconnect. The patch adds a second, server-side mechanism for the same job.

Message #1 lists the slot downsides as lost slots after failover, re-creating slots on the new primary, and dangling slots. A temporary slot created by the walreceiver avoids all three: the standby creates it on each new upstream, and the upstream drops it at disconnect. The ephemeral-slot version in message #4 is close to this existing feature, but primary-side and with temporary = 'f'. The thread does not mention the existing setting or say why it is not enough.

Thread: [#1 (Rui Zhao, 2026-04-30)](https://www.postgresql.org/message-id/3544CABD-2D09-4D7F-AFB7-2C75A09C4E22%40gmail.com), [#4 (Rui Zhao, 2026-06-15)](https://www.postgresql.org/message-id/57E2263F-455D-49A5-A16D-E66562DBC027%40gmail.com)

**Question:** What does a primary-created pg_walsender_<pid> slot do that a standby with wal_receiver_create_temp_slot = on cannot do today?

Counter-check: held. wal_receiver_create_temp_slot exists and the thread does not discuss it, so the objection stands. It is fixable, because the code comment and Amit Kapila both point to this direction, and the PGPROC version splits the horizons without any slot.

### 2. All slotless feedback standbys pay for a fix few of them need

fixable · level 1b, rubric item 4 · desirability, WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `demand-2`

A DBA runs standbys without slots on purpose and relies on wal_keep_size. After an upgrade, every walsender with feedback creates pg_walsender_<pid>. A lagging standby now pins WAL, and with the default max_slot_wal_keep_size = -1 the primary disk can fill. Slot-count monitoring also starts alerting on slots with temporary = 'f'.

Only standbys that host a logical slot gain from the change. The ephemeral-slot version still changes WAL retention and max_replication_slots use for every slotless standby that sends feedback, and the material shows no way to turn it off. Message #4 calls the WAL pinning 'a feature', but these users chose to avoid slots for this reason, as message #1 itself says. The claimed bound depends on max_slot_wal_keep_size, which is unlimited by default.

Thread: [#1 (Rui Zhao, 2026-04-30)](https://www.postgresql.org/message-id/3544CABD-2D09-4D7F-AFB7-2C75A09C4E22%40gmail.com), [#4 (Rui Zhao, 2026-06-15)](https://www.postgresql.org/message-id/57E2263F-455D-49A5-A16D-E66562DBC027%40gmail.com)

**Question:** Can the slot be created only when feedback carries a catalog_xmin different from xmin, and how do sites that use wal_keep_size keep today's WAL retention?

Counter-check: held. The scenario overlaps replication-ondisk-1, but the point is separate. The slot is created on first feedback for every slotless standby, and only standbys that report an older catalog_xmin gain from it.

### 3. No field evidence that the affected group is more than a narrow case

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer accepts a new walsender slot lifecycle and a changed ReplicationSlotCreate API. The bug only bites setups that combine a slotless physical standby, hot_standby_feedback, and a logical slot on that standby with an old catalog_xmin. If few sites run that combination, the project carries the code and its edge cases for little gain.

Message #1 asserts that 'many production HA deployments' avoid physical slots and points to 'Patroni with certain configurations', but it gives no bug report, -general thread or bloat numbers from a real system. Nobody besides the author asks for the fix in the thread. The only validation in message #4 shows the horizons differing by one xid on a test pair. A workaround also exists: give that one standby a physical slot, bounded by max_slot_wal_keep_size.

Thread: [#1 (Rui Zhao, 2026-04-30)](https://www.postgresql.org/message-id/3544CABD-2D09-4D7F-AFB7-2C75A09C4E22%40gmail.com), [#4 (Rui Zhao, 2026-06-15)](https://www.postgresql.org/message-id/57E2263F-455D-49A5-A16D-E66562DBC027%40gmail.com)

**Question:** Which production systems hit this combination, how much bloat did it cause, and why was a physical slot for that standby not an acceptable fix for them?

Counter-check: held. The thread has no report from a second user and no bloat numbers from a real system. The code comment admits the limitation, which supports that the problem exists but does not show how many sites hit it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The walsender creates a physical slot whose restart_lsn pins WAL, and hot standby feedback handling changes. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | ReplicationSlotCreate gets a new error_if_full argument; the earlier version added a field to PGPROC. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #1 calls it a fix for a long-standing issue, but it changes a C API and WAL retention. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new long-lived use of ephemeral slots in the walsender, with a fallback path when the slot pool is full. |
| performance on the hot path | **yes** | The PGPROC version adds a read per PGPROC in ComputeXidHorizons(); the slot version has no numbers. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | wal_receiver_create_temp_slot already lets a slotless standby get a temporary physical slot that tracks both horizons. |

## 5. Questions the author must answer before anyone reviews code

1. What does a primary-created pg_walsender_<pid> slot do that a standby with wal_receiver_create_temp_slot = on cannot do today?
2. Can the slot be created only when feedback carries a catalog_xmin older than xmin, so that other slotless standbys see no change?
3. Can the slot track xmin and catalog_xmin with restart_lsn left invalid, so that wal_keep_size setups keep today's WAL retention?
4. Which production systems hit this combination, how much bloat did it cause, and why was a physical slot for that standby not acceptable?
5. Should server-created walsender slots count against max_replication_slots, and what happens when a slot named pg_walsender_<pid> already exists?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Amit Kapila reviewed the PGPROC version in message #3 and suggested ephemeral slots; nobody has replied to the ephemeral-slot prototype. The owner needs to know replication slots and the xid horizon code.
- **Liability:** The owner signs for the walsender slot lifecycle, the changed WAL retention for slotless standbys, and the correctness of the catalog horizon, with no end date.
- **Author follow-up:** The author rebased, added measurements, wrote a new prototype after review in message #3, and moved the entry across three CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Only slotless feedback standbys that host a logical slot gain, but the slot version changes behavior for every slotless feedback standby. The patch is small but touches slot, walsender and horizon code, and the choice between PGPROC, an implicit slot and the existing standby setting is open.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Implicit slots take max_replication_slots entries from user-created slots. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Ephemeral slot adds WAL retention that slotless standbys chose to avoid. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-3` (fixable, level 2) Long-lived RS_EPHEMERAL slot with a pid-based name stretches slot semantics. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-3` (fixable, level 2) Best-effort fallback makes the vacuum horizon depend on slot pool state. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-1` Slotless standbys start pinning WAL with no way to opt out. Refuted (not specific): Duplicate of replication-ondisk-1. That one states the same WAL retention scenario and asks the more useful question about leaving restart_lsn invalid.
- `standards-compat-3` New slots look persistent and share the user slot namespace. Refuted (not specific): Duplicate of replication-ondisk-3, which names the error a name collision raises. The pid reuse part is not a failure, because the slot drops when the walsender exits.
- `replication-ondisk-2` Transient walsender slots can block real slot creation. Refuted (not specific): Duplicate of standards-compat-2, which describes the same full slot pool and also asks whether the server-created slots should be counted separately.
- `security-1` Slotless feedback standbys now pin WAL with no cap by default. Refuted (not specific): Duplicate of replication-ondisk-1. Its scenario is also weaker: a physical slot's restart_lsn follows the standby's flush position, so a replay stall alone does not pin WAL.
- `maintenance-fit-1` Implicit slot adds WAL retention to setups that avoid slots on purpose. Refuted (not specific): Duplicate of replication-ondisk-1. It gives the same scenario, the same default of max_slot_wal_keep_size = -1 and the same missing opt-out.
- `maintenance-fit-2` Walsenders take max_replication_slots entries that other features need. Refuted (not specific): Duplicate of standards-compat-2. Its point that raising max_replication_slots needs a restart is correct and belongs with that objection.
- `maintenance-fit-4` Long-lived ephemeral slots shown as non-temporary, with a reserved name. Refuted (not specific): Duplicate of replication-ondisk-3. The pid reuse part is not a failure, because the slot drops when the walsender exits.
- `already-exists-2` Implicit slots override deployments that avoid slots on purpose. Refuted (not specific): Duplicate of replication-ondisk-1 for WAL retention and of standards-compat-2 for the slot pool. The comparison with the opt-in setting is already in already-exists-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found for this idea. [P9] is a revert because a held xmin was ignored and HOT pruning then broke CREATE INDEX CONCURRENTLY; here the catalog horizon stays held, and slots already split the two horizons this way, so that reason does not carry over. [P3] failed because it broke the catalog_xmin guarantee, which this proposal keeps. The only earlier trace in the thread is the XXX comment in ProcessStandbyHSFeedbackMessage() about ephemeral slots (messages #1, #2, #4). The thread reports no earlier discussion or rejection of that comment's idea.
- standards-compat: 3 objection(s). Objections listed above. Neither precedent applies to the user-facing contract this angle covers.
- replication-ondisk: 3 objection(s). The PGPROC version (messages #1 and #2) is superseded, so the objections address the ephemeral-slot design in message #4. Message #4 is truncated, so later parts of its recommendation could not be checked.
- security: 1 objection(s). The patch adds no new privilege. A REPLICATION role could already create slots, so using up max_replication_slots or pinning WAL is not an escalation, and the objection above is about the safe default. Possible slot name collisions with pg_walsender_<pid> could not be checked without the code.
- maintenance-fit: 4 objection(s). The four objections above cover the ephemeral-slot design. No committer has reviewed that design yet, and only Amit Kapila commented on the earlier one.
- already-exists: 2 objection(s). Two objections are raised. The listed precedents ([P9], [P3]) concern horizon safety and slot sync, not an existing alternative, so none is cited.
- demand: 2 objection(s). Two objections are raised. Neither listed precedent fits the demand angle.

### Precedents retrieved

- P1: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P2: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P3: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - used in the review; reason summarised from its thread
- P4: [Revert "Add test case for obsoleting slot with active walsender"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96795176810b979a2bf1f4563bdcb161679d5b95) (revert, Reverted, 2021-06-20) - judged not relevant
- P5: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P6: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant; reason summarised from its thread
- P7: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P8: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - used in the review
- P10: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant; reason summarised from its thread
- P11: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P12: [Synchronizing slots from primary to standby](https://commitfest.postgresql.org/patch/1961/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P13: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P14: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P15: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant
- P16: [expose confirmed_flush for replication slots](https://commitfest.postgresql.org/patch/295/) (cf, Rejected, 2015-07-08) - judged not relevant
- P17: [Switch to FullTransactionId for PGPROC->xid and XLogRecord->xl_xid](https://commitfest.postgresql.org/patch/4741/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P18: [enable pg_recvlogical to use temporary slots](https://commitfest.postgresql.org/patch/5363/) (cf, Returned with feedback, 2025-01-27) - judged not relevant
- P19: [Useless field ispartitioned in CreateStmtContext](https://commitfest.postgresql.org/patch/5340/) (cf, Rejected, 2024-11-28) - judged not relevant
- P20: [walsender.c fileheader comment](https://commitfest.postgresql.org/patch/5055/) (cf, Withdrawn, 2024-07-19) - judged not relevant
- P21: [GUC-ify WALSender's MAX_SEND_SIZE constant](https://commitfest.postgresql.org/patch/4781/) (cf, Rejected, 2024-05-14) - judged not relevant
- P22: [Optimize walsender handling invalid messages of 'drop publication'](https://commitfest.postgresql.org/patch/4393/) (cf, Withdrawn, 2023-10-20) - judged not relevant
- P23: [Catalog version access](https://commitfest.postgresql.org/patch/2999/) (cf, Returned with feedback, 2022-01-31) - judged not relevant
- P24: [Revert changes for SSL compression in libpq](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0ba71107efeeccde9158f47118f95043afdca0bb) (revert, Reverted, 2021-03-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 185 s; cost reported by the CLI: $1.93
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 126 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:05:46.306Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
