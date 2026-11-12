# Design pre-review: Proposal: Conflict log history table for Logical Replication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6696](https://commitfest.postgresql.org/patch/6696/); [thread](https://www.postgresql.org/message-id/flat/CAFiTN-u5D5o_AGNbHRZHaOqAMWkxLf%2BhSk_r9X3gv6HbLOB5%2Bg%40mail.gmail.com), 747 of 747 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection to the goal or to demand survived, and a committer who knows logical replication takes part in the thread. The five surviving objections are all fixable level 2 design points. The two_phase lock handling (replication-ondisk-2, security-1) is still open in messages #744 to #746. The other three are recorded key text that depends on session settings (standards-compat-1), `alter subscription` dropping data (standards-compat-3), and the regular dump of rows and grants (standards-compat-2).

Not checked: The code was not read, and only 3 of 747 messages were visible directly. The lock handling under two_phase, the cascade drop path and the pg_dump patch need a check against the latest version.

## Prior attempts

- [Detection and resolution of logical replication conflict](https://www.postgresql.org/message-id/flat/CAJpy0uD0-DpYVMtsxK5R=zszXauZBayQMAYET9sWr_w0CNWXxQ@mail.gmail.com) (related; Returned with feedback, 2024-12-11). The visible messages do not show why. The patch set was still under review at v17 in October 2024, and the last message shown, from 2024-10-28, has no reply. Two items were still open at that point: clock-skew handling, and a race in which DirtySnapshot index scans miss rows and report update_missing or delete_missing instead of the origin_differs conflicts. (AI summary of the thread) Different now: P1 added resolvers that choose an outcome per conflict type. This proposal only records conflicts the apply worker already detects and reports, and resolves nothing.

## Objections a reviewer is likely to raise

### 1. Apply worker errors on conflict log table lock contention under two_phase

A two_phase subscriber with track_commit_timestamp on records update_origin_differs rows after initial sync, while other prepared transactions are pending. Autovacuum truncation of the conflict log table makes the lock unavailable, and the apply worker errors. With disable_on_error, the subscription is disabled again and again.

The prepared-transaction check does not test whether a prepared transaction actually holds the table lock, so it gives false positives (message #731). Message #744 says these conflicts are routine after table sync and that the special error path is no easier to understand than a stall. Messages #745 and #746 still weigh documentation against detection, and the AI-written lock-holder check in message #733 was not adopted.

**Question:** Which conditions lead the apply worker to raise this error when no prepared transaction holds a lock on the conflict log table, and how often do they occur in a two_phase deployment?

<sub>fixable, level 2, WAL / replication / standby; messages [#731](https://www.postgresql.org/message-id/CAD21AoBNTnF6Xzo-UqGxALNqyy85EFsPWDRuXGk0aibD2ugkgQ%40mail.gmail.com), [#732](https://www.postgresql.org/message-id/CAJpy0uCYaXApQ8u6WwApzx-4XnWSZbViVmEBV3uy7YCpdky6wQ%40mail.gmail.com), [#733](https://www.postgresql.org/message-id/CAA4eK1JEEjdmZ99pnDFN0OUEyUiJqq8D_%2BLre7iA0WXw1x6ovA%40mail.gmail.com), [#744](https://www.postgresql.org/message-id/CAD21AoBpPuhqmZ3cvdNd3Gg%3DE%2BU0u3oYVWmQ6OpDyEGE%3DPGH%2BQ%40mail.gmail.com), [#745](https://www.postgresql.org/message-id/CAA4eK1J6m%3DVukMa5DjRb0rarCiLR%2BZP_9z7NHH4OUW0O27K_LQ%40mail.gmail.com), [#746](https://www.postgresql.org/message-id/CAA4eK1Jhg2mgiMh5mTFjBgNekDFATAUsRjDaY%3D0D7OmRaf5qGw%40mail.gmail.com); `replication-ondisk-2`</sub>

### 2. Any role can block drop subscription with a fake prepared transaction

An unprivileged user on a subscriber with max_prepared_transactions > 0 runs PREPARE TRANSACTION 'pg_gid_<subid>_1' on an empty transaction. For a two_phase subscription with a conflict log table, drop subscription and alter subscription to 'log' are then refused. The apply worker may also error out on its next lock wait. The administrator sees only an error about prepared transactions.

The new refusals and the apply-worker error depend on finding prepared transactions for the subscription, and message #731 says the GID check tests only the name format. Message #733 compares this to replication slot names. Message #744 answers that creating a slot needs the REPLICATION privilege, while any role can prepare a transaction. The thread does not settle the point, and no fix is proposed, so the patch adds a denial-of-service lever for unprivileged roles that did not exist before.

**Question:** How do the new prepared-transaction checks tell a GID created by the apply worker from one any role can create with the same name?

<sub>fixable, level 2, security / privileges; messages [#731](https://www.postgresql.org/message-id/CAD21AoBNTnF6Xzo-UqGxALNqyy85EFsPWDRuXGk0aibD2ugkgQ%40mail.gmail.com), [#733](https://www.postgresql.org/message-id/CAA4eK1JEEjdmZ99pnDFN0OUEyUiJqq8D_%2BLre7iA0WXw1x6ovA%40mail.gmail.com), [#744](https://www.postgresql.org/message-id/CAD21AoBpPuhqmZ3cvdNd3Gg%3DE%2BU0u3oYVWmQ6OpDyEGE%3DPGH%2BQ%40mail.gmail.com); `security-1`</sub>

### 3. Recorded key text depends on the apply worker's output settings

An administrator sets DateStyle = 'SQL, DMY' on the subscriber database, and the apply worker records the key 2 January as "02/01/2025". A monitoring job running with MDY casts replica_identity->>'k' back to timestamptz and gets 1 February. Nothing in the row says which settings were used, so the job matches the wrong row.

Message #651 and #661 show the ambiguity, and #674 confirms that IntervalStyle and extra_float_digits behave the same way, and #738 and #741 add timezone and lc_monetary. Message #699 drops the fix and chooses documentation, but the documentation tells readers to cast with "the same settings that the apply worker used". Message #709 notes that a reader cannot find those settings. libpqwalreceiver already forces these settings so that values are unambiguous, so a committer would likely ask why the conflict log table, whose docs recommend casting back, does not do the same.

**Question:** How does a reader cast a recorded key back to its original value when the apply worker's DateStyle at write time is not recorded anywhere?

<sub>fixable, level 2, design; messages [#651](https://www.postgresql.org/message-id/CAJpy0uB7ExJjtgsSipnGOZT6p%3Dkpx3OWaXSph%2BKRNaQKrhvCLw%40mail.gmail.com), [#661](https://www.postgresql.org/message-id/CAJpy0uD885SauZ0VQk0fw-VN1eRbuqGsdt%3DYwMEMhyNVFwfrxA%40mail.gmail.com), [#674](https://www.postgresql.org/message-id/CAFvd2n8Vra0ctRxWVG21FZQeGnZ3Pb%3Djgk0BCrGobivWOqQbOw%40mail.gmail.com), [#699](https://www.postgresql.org/message-id/CAA4eK1LHgNHtZJhbcscOpkQisQYRgdCH7R%2BRPsgpsjN6%3DNEjEQ%40mail.gmail.com), [#701](https://www.postgresql.org/message-id/CAJpy0uDdKEPgaynN59nBKP4qfnUY6iZDB%2B2J7bR%3D%2BVq83eN9dw%40mail.gmail.com), [#709](https://www.postgresql.org/message-id/CAJpy0uDwjiH_Zt-gNwxssp06waqaijJwSvAr8-jVJ7fLbcwZJg%40mail.gmail.com), [#738](https://www.postgresql.org/message-id/CAHut%2BPveJVZ-fhR6TpZjmMV2FnpwtSK3_iBjj-%3DGvF_DtmrGQA%40mail.gmail.com), [#741](https://www.postgresql.org/message-id/CAHut%2BPucAHDiZ6KjaubSB1NKn7Y1wqYLa%2BcjzLdE%3D7HEh%3DzxhA%40mail.gmail.com); `standards-compat-1`</sub>

### 4. Changing a logging option silently drops the table and user objects

An operator runs alter subscription s set (conflict_log_destination = 'log') to reduce load. The command permanently deletes all recorded conflicts. It also drops views and functions that other roles built on the table, and the subscription owner needs no privilege on those objects. The operator sees only a NOTICE.

Message #654 shows that the drop uses DROP_CASCADE and PERFORM_DELETION_INTERNAL, so event triggers do not see the cascaded objects. It also points out that drop_subscription.sgml still says CASCADE and RESTRICT have no effect. The fix in message #690 makes DROP SUBSCRIPTION honour RESTRICT. Message #654 suggests an error for ALTER, but neither message shows that this was settled. No other ALTER ... SET of a storage parameter or option in Postgres destroys user data, so a committer would likely ask for an explicit verb or a RESTRICT-style error.

**Question:** Why should alter subscription ... set (conflict_log_destination = 'log') drop the table and its dependents without an error, when drop table refuses by default?

<sub>fixable, level 2, design; messages [#654](https://www.postgresql.org/message-id/CAN4CZFOqiZtHOHAgd1mA325XQrsShj0C%3DvK2SBboUTFEmdoPZg%40mail.gmail.com), [#658](https://www.postgresql.org/message-id/CAA4eK1LnXEZ-E2%2B1pGeKvPimKdp%3D8RonN7Ueq0pmZheM-1aBRQ%40mail.gmail.com), [#659](https://www.postgresql.org/message-id/CAJpy0uC3hvzX5qfNe__UMHKYDOY-dd6hw3vkwUW9-uiKMvkfWg%40mail.gmail.com), [#690](https://www.postgresql.org/message-id/CAN4CZFNaWLs6rAOX3KNzgQ0ci-6PB%2BmWJNj_6_F-O_vomb8Euw%40mail.gmail.com); `standards-compat-3`</sub>

### 5. OID-based table name breaks dump, restore and user queries

A user builds views and scripts on pg_conflict.pg_conflict_log_16390 and grants SELECT to a monitoring role. After a pg_dump and restore, or after the subscription is recreated, the table is pg_conflict_log_<new oid>, empty, and without the grants. The user finds out when the monitoring stops returning rows.

The user-facing name of the table is built from the subscription OID, which is not stable across dump and restore. Message #742 states that a regular dump emits neither the table definition nor its privileges, and the brief puts pg_dump and pg_upgrade support in separate later patches. Recorded conflict rows are not carried over by a logical dump at all. pg_toast_<oid> follows the same naming pattern, but users do not query TOAST tables or grant privileges on them.

**Question:** How does a user refer to a subscription's conflict log table by a stable name, and what does a logical dump and restore do with its rows and grants?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#742](https://www.postgresql.org/message-id/CAJpy0uB0Cd_Pikxoh3twZnfTfkXNSyCmk8C2x5kKVo3wgQADdQ%40mail.gmail.com), [#733](https://www.postgresql.org/message-id/CAA4eK1JEEjdmZ99pnDFN0OUEyUiJqq8D_%2BLre7iA0WXw1x6ovA%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Under two_phase, when does the apply worker raise the lock error although no prepared transaction holds a lock on the conflict log table, and how often does that occur?
2. How do the prepared-transaction checks tell a GID that the apply worker created from one that any role created with the same pg_gid_<subid>_ name?
3. How does a reader cast a recorded key back to its original value when the row does not record the apply worker's DateStyle or timezone?
4. Why does `alter subscription ... set (conflict_log_destination = 'log')` drop the table and its dependent objects without an error, when DROP SUBSCRIPTION now honours RESTRICT?
5. What does a regular pg_dump and restore do with the rows and grants of the conflict log table, and how do users purge old rows?

## Touch points

- **on-disk format / pg_upgrade:** A later patch in the series must carry the per-subscription conflict log table and its destination setting through pg_upgrade.
- **WAL / replication / standby:** The apply worker inserts conflict rows and gets new lock and error rules under two_phase.
- **catalog changes:** The patch adds the pg_conflict schema, system-created tables named by subscription OID, and a new pg_subscription field.
- **SQL-standard conflict or non-standard syntax:** The patch adds the option conflict_log_destination to create subscription and alter subscription.
- **pg_dump / restore round-trip:** pg_dump must restore the destination setting, and a regular dump carries neither rows nor grants of the table (message #742).
- **psql UX conventions:** A later patch changes the \dRs output.
- **security / privileges:** The prepared-transaction checks match GIDs by name format, and the table drop cascades to objects owned by other roles.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds lock rules for one table, the 64kB and 1GB value limits, and pg_dump special cases.

## Who would commit it, and when

- Owner: qualified committer engaged. Amit Kapila, who works on logical replication, reviews the series and plans to commit the insertion patch (message #726). Masahiko Sawada also reviews, and his two_phase objection is open.
- Priority: proceed; audience a sizeable minority; review cost high. Subscriber administrators who meet conflicts gain a queryable record, and BDR and pgactive have similar tables. The main patch is about 100 KiB in the apply worker and subscription commands, but a qualified committer is engaged and the open points are fixable.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 10 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
