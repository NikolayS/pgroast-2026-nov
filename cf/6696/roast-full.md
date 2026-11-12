# Design pre-review: Proposal: Conflict log history table for Logical Replication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6696](https://commitfest.postgresql.org/patch/6696/) (Needs review); authors: Dilip Kumar (dilip.kumar)
- Thread: [Proposal: Conflict log history table for Logical Replication](https://www.postgresql.org/message-id/flat/CAFiTN-u5D5o_AGNbHRZHaOqAMWkxLf%2BhSk_r9X3gv6HbLOB5%2Bg%40mail.gmail.com), started 2025-08-05
- Messages read: 747 of 747
- Patch prose read: v87-0003-Document-and-comment-that-replica-identity-rende.patch, v87-0001-Implement-the-conflict-insertion-infrastructure-.patch, v87-0002-Avoid-stalling-apply-on-the-conflict-log-table-w.patch, v1-0002-Create-conflict-history-table-if-it-does-not-exis.patch, v1-0001-Add-configurable-conflict-log-history-table-for-L.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection to the goal or to demand survived, and a committer who knows logical replication takes part in the thread. The five surviving objections are all fixable level 2 design points. The two_phase lock handling (replication-ondisk-2, security-1) is still open in messages #744 to #746. The other three are recorded key text that depends on session settings (standards-compat-1), `alter subscription` dropping data (standards-compat-3), and the regular dump of rows and grants (standards-compat-2).

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and only 3 of 747 messages were visible directly. The lock handling under two_phase, the cascade drop path and the pg_dump patch need a check against the latest version.

## The proposal, restated

**Problem.** Logical replication conflicts go only to the server log as plain text. Querying them, getting structured fields such as table, LSN or old and new data, and feeding them to monitoring or resolution scripts are all hard. The audience is subscriber administrators and external tools.

**Proposed design.** A new subscription option, conflict_log_destination (log, table or all), makes the system create a table pg_conflict.pg_conflict_log_<subid>, owned by the subscription and dropped with it. The apply worker inserts one row per LOG-level conflict, with the replica identity key as JSON and local tuple metadata as json[]. ERROR-level conflicts still go only to the server log, and the table is never published.

**User-visible surface.**
- Subscription option conflict_log_destination = 'log' | 'table' | 'all' for create subscription and alter subscription.
- New schema pg_conflict with table pg_conflict_log_<subid>; columns include conflict_type, remote_xid, remote_commit_lsn, remote_commit_ts, apply_origin, replica_identity, replica_identity_full, local_conflicts, has_omitted_values.
- Key values over 64kB stored as the marker {"omitted":true,"length":...}.
- drop subscription and alter subscription to 'log' refused while prepared transactions exist, when two_phase is on.
- pg_dump/pg_upgrade and psql \dRs changes, in separate later patches.

**What the author claims.**
- Structured rows are easier to query and analyze than server log text.
- A user table, as opposed to a catalog, allows user purging and later partitioning.
- Omitting oversized values instead of truncating them keeps equality and join queries on the table correct.
- Similar extension-managed tables exist in BDR and pgactive.

**Thread state.** The thread is long, and a core group reviewed it at v93: Amit Kapila, shveta malik, Nisha Moond, vignesh C, Zhijie Hou, Peter Smith and Masahiko Sawada. Sawada's objection to erroring out the apply worker only for the conflict log table under two_phase is still open, and the visible discussion does not settle documentation versus detection.

## 2. Prior attempts

### Detection and resolution of logical replication conflict (related)

- **Link:** [https://commitfest.postgresql.org/patch/5021/](https://commitfest.postgresql.org/patch/5021/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAJpy0uD0-DpYVMtsxK5R=zszXauZBayQMAYET9sWr_w0CNWXxQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAJpy0uD0-DpYVMtsxK5R=zszXauZBayQMAYET9sWr_w0CNWXxQ@mail.gmail.com)
- **Date:** decided 2024-12-11; started 2024-05-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. The patch set was still under review at v17 in October 2024, and the last message shown, from 2024-10-28, has no reply. Two items were still open at that point: clock-skew handling, and a race in which DirtySnapshot index scans miss rows and report update_missing or delete_missing instead of the origin_differs conflicts.
- **Reason (quote):** quote unavailable
- **What is different this time:** P1 added resolvers that choose an outcome per conflict type. This proposal only records conflicts the apply worker already detects and reports, and resolves nothing.


## 3. Objections a reviewer is likely to raise

### 1. Apply worker errors on conflict log table lock contention under two_phase

fixable · level 2, rubric item 7 · WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-2`

A two_phase subscriber with track_commit_timestamp on records update_origin_differs rows after initial sync, while other prepared transactions are pending. Autovacuum truncation of the conflict log table makes the lock unavailable, and the apply worker errors. With disable_on_error, the subscription is disabled again and again.

The prepared-transaction check does not test whether a prepared transaction actually holds the table lock, so it gives false positives (message #731). Message #744 says these conflicts are routine after table sync and that the special error path is no easier to understand than a stall. Messages #745 and #746 still weigh documentation against detection, and the AI-written lock-holder check in message #733 was not adopted.

Thread: [#731 (Masahiko Sawada, 2026-11-03)](https://www.postgresql.org/message-id/CAD21AoBNTnF6Xzo-UqGxALNqyy85EFsPWDRuXGk0aibD2ugkgQ%40mail.gmail.com), [#732 (shveta malik, 2026-11-03)](https://www.postgresql.org/message-id/CAJpy0uCYaXApQ8u6WwApzx-4XnWSZbViVmEBV3uy7YCpdky6wQ%40mail.gmail.com), [#733 (Amit Kapila, 2026-11-03)](https://www.postgresql.org/message-id/CAA4eK1JEEjdmZ99pnDFN0OUEyUiJqq8D_%2BLre7iA0WXw1x6ovA%40mail.gmail.com), [#744 (Masahiko Sawada, 2026-11-09)](https://www.postgresql.org/message-id/CAD21AoBpPuhqmZ3cvdNd3Gg%3DE%2BU0u3oYVWmQ6OpDyEGE%3DPGH%2BQ%40mail.gmail.com), [#745 (Amit Kapila, 2026-11-09)](https://www.postgresql.org/message-id/CAA4eK1J6m%3DVukMa5DjRb0rarCiLR%2BZP_9z7NHH4OUW0O27K_LQ%40mail.gmail.com), [#746 (Amit Kapila, 2026-11-10)](https://www.postgresql.org/message-id/CAA4eK1Jhg2mgiMh5mTFjBgNekDFATAUsRjDaY%3D0D7OmRaf5qGw%40mail.gmail.com)

**Question:** Which conditions lead the apply worker to raise this error when no prepared transaction holds a lock on the conflict log table, and how often do they occur in a two_phase deployment?

Counter-check: held. Message #731 describes the false positive, and #744 says the triggering conditions are common after table sync. Messages #745 and #746 still weigh documentation against detection, so the thread has not answered it.

### 2. Any role can block drop subscription with a fake prepared transaction

fixable · level 2, rubric item 7 · security / privileges, maintenance burden ("who maintains this in 10 years") · `security-1`

An unprivileged user on a subscriber with max_prepared_transactions > 0 runs PREPARE TRANSACTION 'pg_gid_<subid>_1' on an empty transaction. For a two_phase subscription with a conflict log table, drop subscription and alter subscription to 'log' are then refused. The apply worker may also error out on its next lock wait. The administrator sees only an error about prepared transactions.

The new refusals and the apply-worker error depend on finding prepared transactions for the subscription, and message #731 says the GID check tests only the name format. Message #733 compares this to replication slot names. Message #744 answers that creating a slot needs the REPLICATION privilege, while any role can prepare a transaction. The thread does not settle the point, and no fix is proposed, so the patch adds a denial-of-service lever for unprivileged roles that did not exist before.

Thread: [#731 (Masahiko Sawada, 2026-11-03)](https://www.postgresql.org/message-id/CAD21AoBNTnF6Xzo-UqGxALNqyy85EFsPWDRuXGk0aibD2ugkgQ%40mail.gmail.com), [#733 (Amit Kapila, 2026-11-03)](https://www.postgresql.org/message-id/CAA4eK1JEEjdmZ99pnDFN0OUEyUiJqq8D_%2BLre7iA0WXw1x6ovA%40mail.gmail.com), [#744 (Masahiko Sawada, 2026-11-09)](https://www.postgresql.org/message-id/CAD21AoBpPuhqmZ3cvdNd3Gg%3DE%2BU0u3oYVWmQ6OpDyEGE%3DPGH%2BQ%40mail.gmail.com)

**Question:** How do the new prepared-transaction checks tell a GID created by the apply worker from one any role can create with the same name?

Counter-check: held. Message #731 states the check tests only the GID format, and #744 answers the slot-name comparison in #733. A superuser can find and roll back the fake transaction through pg_prepared_xacts, so the harm is bounded, but the proposal has no fix.

### 3. Recorded key text depends on the apply worker's output settings

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An administrator sets DateStyle = 'SQL, DMY' on the subscriber database, and the apply worker records the key 2 January as "02/01/2025". A monitoring job running with MDY casts replica_identity->>'k' back to timestamptz and gets 1 February. Nothing in the row says which settings were used, so the job matches the wrong row.

Message #651 and #661 show the ambiguity, and #674 confirms that IntervalStyle and extra_float_digits behave the same way, and #738 and #741 add timezone and lc_monetary. Message #699 drops the fix and chooses documentation, but the documentation tells readers to cast with "the same settings that the apply worker used". Message #709 notes that a reader cannot find those settings. libpqwalreceiver already forces these settings so that values are unambiguous, so a committer would likely ask why the conflict log table, whose docs recommend casting back, does not do the same.

Thread: [#651 (shveta malik, 2026-10-09)](https://www.postgresql.org/message-id/CAJpy0uB7ExJjtgsSipnGOZT6p%3Dkpx3OWaXSph%2BKRNaQKrhvCLw%40mail.gmail.com), [#661 (shveta malik, 2026-10-12)](https://www.postgresql.org/message-id/CAJpy0uD885SauZ0VQk0fw-VN1eRbuqGsdt%3DYwMEMhyNVFwfrxA%40mail.gmail.com), [#674 (Zhijie Hou, 2026-10-14)](https://www.postgresql.org/message-id/CAFvd2n8Vra0ctRxWVG21FZQeGnZ3Pb%3Djgk0BCrGobivWOqQbOw%40mail.gmail.com), [#699 (Amit Kapila, 2026-10-24)](https://www.postgresql.org/message-id/CAA4eK1LHgNHtZJhbcscOpkQisQYRgdCH7R%2BRPsgpsjN6%3DNEjEQ%40mail.gmail.com), [#701 (shveta malik, 2026-10-26)](https://www.postgresql.org/message-id/CAJpy0uDdKEPgaynN59nBKP4qfnUY6iZDB%2B2J7bR%3D%2BVq83eN9dw%40mail.gmail.com), [#709 (shveta malik, 2026-10-27)](https://www.postgresql.org/message-id/CAJpy0uDwjiH_Zt-gNwxssp06waqaijJwSvAr8-jVJ7fLbcwZJg%40mail.gmail.com), [#738 (Peter Smith, 2026-11-04)](https://www.postgresql.org/message-id/CAHut%2BPveJVZ-fhR6TpZjmMV2FnpwtSK3_iBjj-%3DGvF_DtmrGQA%40mail.gmail.com), [#741 (Peter Smith, 2026-11-04)](https://www.postgresql.org/message-id/CAHut%2BPucAHDiZ6KjaubSB1NKn7Y1wqYLa%2BcjzLdE%3D7HEh%3DzxhA%40mail.gmail.com)

**Question:** How does a reader cast a recorded key back to its original value when the apply worker's DateStyle at write time is not recorded anywhere?

Counter-check: held. The thread weighed this cost and chose documentation in #699 and #702. But #709 shows a row does not record the write-time settings, so a reader cannot follow the documented advice to cast with them.

### 4. Changing a logging option silently drops the table and user objects

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, pg_dump / restore round-trip · `standards-compat-3`

An operator runs alter subscription s set (conflict_log_destination = 'log') to reduce load. The command permanently deletes all recorded conflicts. It also drops views and functions that other roles built on the table, and the subscription owner needs no privilege on those objects. The operator sees only a NOTICE.

Message #654 shows that the drop uses DROP_CASCADE and PERFORM_DELETION_INTERNAL, so event triggers do not see the cascaded objects. It also points out that drop_subscription.sgml still says CASCADE and RESTRICT have no effect. The fix in message #690 makes DROP SUBSCRIPTION honour RESTRICT. Message #654 suggests an error for ALTER, but neither message shows that this was settled. No other ALTER ... SET of a storage parameter or option in Postgres destroys user data, so a committer would likely ask for an explicit verb or a RESTRICT-style error.

Thread: [#654 (Zsolt Parragi, 2026-10-10)](https://www.postgresql.org/message-id/CAN4CZFOqiZtHOHAgd1mA325XQrsShj0C%3DvK2SBboUTFEmdoPZg%40mail.gmail.com), [#658 (Amit Kapila, 2026-10-12)](https://www.postgresql.org/message-id/CAA4eK1LnXEZ-E2%2B1pGeKvPimKdp%3D8RonN7Ueq0pmZheM-1aBRQ%40mail.gmail.com), [#659 (shveta malik, 2026-10-12)](https://www.postgresql.org/message-id/CAJpy0uC3hvzX5qfNe__UMHKYDOY-dd6hw3vkwUW9-uiKMvkfWg%40mail.gmail.com), [#690 (Zsolt Parragi, 2026-10-16)](https://www.postgresql.org/message-id/CAN4CZFNaWLs6rAOX3KNzgQ0ci-6PB%2BmWJNj_6_F-O_vomb8Euw%40mail.gmail.com)

**Question:** Why should alter subscription ... set (conflict_log_destination = 'log') drop the table and its dependents without an error, when drop table refuses by default?

Counter-check: held. The docs warn that the data is deleted, and #658 and #659 agree to honor RESTRICT. The patch in #690 covers only DROP SUBSCRIPTION, and the visible v93 material does not show the ALTER case settled.

### 5. OID-based table name breaks dump, restore and user queries

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, catalog changes · `standards-compat-2`

A user builds views and scripts on pg_conflict.pg_conflict_log_16390 and grants SELECT to a monitoring role. After a pg_dump and restore, or after the subscription is recreated, the table is pg_conflict_log_<new oid>, empty, and without the grants. The user finds out when the monitoring stops returning rows.

The user-facing name of the table is built from the subscription OID, which is not stable across dump and restore. Message #742 states that a regular dump emits neither the table definition nor its privileges, and the brief puts pg_dump and pg_upgrade support in separate later patches. Recorded conflict rows are not carried over by a logical dump at all. pg_toast_<oid> follows the same naming pattern, but users do not query TOAST tables or grant privileges on them.

Thread: [#742 (shveta malik, 2026-11-05)](https://www.postgresql.org/message-id/CAJpy0uB0Cd_Pikxoh3twZnfTfkXNSyCmk8C2x5kKVo3wgQADdQ%40mail.gmail.com), [#733 (Amit Kapila, 2026-11-03)](https://www.postgresql.org/message-id/CAA4eK1JEEjdmZ99pnDFN0OUEyUiJqq8D_%2BLre7iA0WXw1x6ovA%40mail.gmail.com)

**Question:** How does a user refer to a subscription's conflict log table by a stable name, and what does a logical dump and restore do with its rows and grants?

Counter-check: held. Message #742 confirms a regular dump does not carry grants on the conflict log table, and the table name changes with the subscription OID. v93-0002 covers the destination and binary upgrade, but not rows or grants in a regular dump.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | A later patch in the series must carry the per-subscription conflict log table and its destination setting through pg_upgrade. |
| WAL / replication / standby | **yes** | The apply worker inserts conflict rows and gets new lock and error rules under two_phase. |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds the pg_conflict schema, system-created tables named by subscription OID, and a new pg_subscription field. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds the option conflict_log_destination to create subscription and alter subscription. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump must restore the destination setting, and a regular dump carries neither rows nor grants of the table (message #742). |
| psql UX conventions | **yes** | A later patch changes the \dRs output. |
| security / privileges | **yes** | The prepared-transaction checks match GIDs by name format, and the table drop cascades to objects owned by other roles. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds lock rules for one table, the 64kB and 1GB value limits, and pg_dump special cases. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Under two_phase, when does the apply worker raise the lock error although no prepared transaction holds a lock on the conflict log table, and how often does that occur?
2. How do the prepared-transaction checks tell a GID that the apply worker created from one that any role created with the same pg_gid_<subid>_ name?
3. How does a reader cast a recorded key back to its original value when the row does not record the apply worker's DateStyle or timezone?
4. Why does `alter subscription ... set (conflict_log_destination = 'log')` drop the table and its dependent objects without an error, when DROP SUBSCRIPTION now honours RESTRICT?
5. What does a regular pg_dump and restore do with the rows and grants of the conflict log table, and how do users purge old rows?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Amit Kapila, who works on logical replication, reviews the series and plans to commit the insertion patch (message #726). Masahiko Sawada also reviews, and his two_phase objection is open.
- **Liability:** The owner signs for a new schema with system-created user tables, the pg_dump and pg_upgrade special cases, and the two_phase lock rules in the apply worker. The table format and the option values become user-visible and are hard to change later.
- **Author follow-up:** The author has posted more than 90 versions since August 2025, and the entry was rebased through October 2026, so the author stays with the work.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Subscriber administrators who meet conflicts gain a queryable record, and BDR and pgactive have similar tables. The main patch is about 100 KiB in the apply worker and subscription commands, but a qualified committer is engaged and the open points are fixable.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Any user can block DROP SUBSCRIPTION with a fake pg_gid prepared transaction. Refuted (not specific): Duplicate of security-1. Its claim that the administrator cannot find the cause is also weak, because pg_prepared_xacts shows the GID and its owner.
- `replication-ondisk-3` Stored replica_identity text depends on unrecorded apply worker settings. Refuted (not specific): Duplicate of standards-compat-1.
- `replication-ondisk-4` Main patch lands before dump, upgrade and data carry-over are settled. Refuted (not specific): Duplicate of standards-compat-2. Committing the insertion patch before v93-0002 in the same series is the usual infrastructure-first order, and the material does not show a release shipping without it.
- `security-2` One conflict log table exposes key values from every replicated table. Refuted (no failure scenario): The scenario needs an administrator to grant SELECT on the conflict log table, and a grant on any table exposes its contents. The material shows no default access for ordinary roles, so no exposure happens without that deliberate act.
- `maintenance-fit-1` Special lock handling for the conflict log table under two_phase. Refuted (not specific): Duplicate of replication-ondisk-2. Its claim that no test covers the special path does not match #665 and #683, which add tests for the new error and the lock scenario.
- `maintenance-fit-2` Recorded key text depends on the apply worker's output settings. Refuted (not specific): Duplicate of standards-compat-1. The v92 docs were also reworded to be generic, so the list of settings no longer needs an entry per setting.
- `maintenance-fit-3` Dump and upgrade support is in later, uncommitted patches. Refuted (not specific): Duplicate of standards-compat-2. The scenario assumes a release without the dump patch, but v93-0002 is under review in the same series and nothing shows it slipping to a later release.
- `already-exists-1` Structured jsonlog output already covers part of the need. Refuted (already handled): The objection concedes that jsonlog today leaves table, key and origin inside the DETAIL text, so the existing alternative cannot do the job. The thread shows demand from many reviewers, and BDR and pgactive have such tables.
- `already-exists-2` A conflict-report hook could let extensions own the table. Refuted (no failure scenario): No such hook exists in the core apply worker today, and BDR and pgactive use their own apply code. The scenario is not a failure: the feature is opt-in, and the owner can purge rows from the user table.
- `demand-1` Table omits the ERROR-level conflicts that resolution scripts most need. Refuted (already handled): The patch prose says ERROR-level conflicts abort the apply transaction, so they go to the server log and already stop replication visibly. LOG-level conflicts change or skip rows without stopping anything, and the docs state that this is the table's scope.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposed a conflict log table, and P1's open items (clock skew, the DirtySnapshot race) concern detection and resolution, which the current server log shares. Message #8 points to related threads on a COPY error table, and the material does not report a rejection reason for them.
- standards-compat: 3 objection(s). Objections were found from the standards and compatibility angle.
- replication-ondisk: 4 objection(s). The table is never published and its inserts use HEAP_INSERT_NO_LOGICAL. It replicates physically as an ordinary heap, so I see no standby or WAL record format issue.
- security: 2 objection(s). The cross-owner cascade drop raised in message #654 is left out because option b was agreed in messages #658 and #659, and a patch for it was posted in message #690.
- maintenance-fit: 3 objection(s). Hot-path cost falls only on subscriptions that set conflict_log_destination to table or all, and Amit Kapila, the committer in this area, is engaged and plans to commit. For these reasons no objection is raised on performance or ownership.
- already-exists: 2 objection(s). Two objections from the existing-alternative angle. Both are fixable, not fatal.
- demand: 1 objection(s). The thread shows real demand for structured conflict records: BDR and pgactive ship similar tables, several companies review the patch, and outside testers report results in messages #556, #569 and #620. The objection above concerns which conflicts the table covers, not whether anyone wants it.

### Precedents retrieved

- P1: [Detection and resolution of logical replication conflict](https://commitfest.postgresql.org/patch/5021/) (cf, Returned with feedback, 2024-12-11) - used in the review; reason summarised from its thread
- P2: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant; reason summarised from its thread
- P3: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P4: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P5: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P6: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P7: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1710/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P8: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant; reason summarised from its thread
- P9: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant; reason summarised from its thread
- P10: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1862/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P11: [logical replication access control](https://commitfest.postgresql.org/patch/1033/) (cf, Returned with feedback, 2017-04-06) - judged not relevant
- P12: [Subscription code improvements](https://commitfest.postgresql.org/patch/1248/) (cf, Returned with feedback, 2018-04-07) - judged not relevant
- P13: [Revert "Allow ON CONFLICT .. DO NOTHING on a partitioned table."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f05230752d53c4aa74cffa9b699983bbb6bcb118) (revert, Reverted, 2017-03-31) - judged not relevant
- P14: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P15: [Generic type subscription](https://commitfest.postgresql.org/patch/793/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P16: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/2270/) (cf, Returned with feedback, 2020-03-16) - judged not relevant
- P17: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant
- P18: [Revert: Allow table AM to store complex data structures in rd_amcache](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=922c4c461d213a422ee7eb6c38e399607539210a) (revert, Reverted, 2024-04-11) - judged not relevant
- P19: [log_destination=file](https://commitfest.postgresql.org/patch/1274/) (cf, Returned with feedback, 2018-03-02) - judged not relevant
- P20: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P21: [Handling of the input data errors in COPY FROM](https://commitfest.postgresql.org/patch/1179/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P22: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P23: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P24: [Skip replicating the tables specified in except table option](https://commitfest.postgresql.org/patch/3646/) (cf, Withdrawn, 2024-01-09) - judged not relevant

### Warnings

- 620 long messages in the middle of the thread were left out of the prompts

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 218 s; cost reported by the CLI: $12.51
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 124 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:00:29.074Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
