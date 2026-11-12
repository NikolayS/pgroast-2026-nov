# Design pre-review: Per-table resync for logical replication subscriptions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7126](https://commitfest.postgresql.org/patch/7126/) (Needs review); authors: Cagri Biroglu (cbiroglu)
- Thread: [Per-table resync for logical replication subscriptions](https://www.postgresql.org/message-id/flat/CAA36mso9zgvMVr4UNHMMAQyU-dQy33EqxjZ%3DyEDG8SAB6yYDJw%40mail.gmail.com), started 2026-07-07
- Messages read: 36 of 36
- Patch prose read: v15-0001-refresh-table.patch, v15-0002-refresh-table-partitions.patch, v1-0001-refresh-table.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection to the goal survived: message #1 shows by test that the existing workarounds are non-local or race the apply worker. Three fixable level 2 points remain. replication-ondisk-1 is the strongest: a refresh on a forwarding node can stop a downstream apply worker. maintenance-fit-1 and standards-compat-1 concern lock-order invariants and trigger rules that only a committer who knows logical replication can accept.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The lock order, the slot-then-origin drop sequence and the placement of the origin on the truncate WAL record could not be checked without the code. Only 3 of 36 messages were read directly, so later discussion is known only through the objections.

## The proposal, restated

**Problem.** Postgres has no supported way to re-copy one table that a subscription already tracks. REFRESH PUBLICATION only picks up newly added tables, so operators change publication membership, which affects sibling and downstream subscribers, or edit pg_subscription_rel by hand, which races the apply worker.

**Proposed design.** A new subscriber-local command truncates the named tables in one statement and resets their pg_subscription_rel state to init, so tablesync workers re-copy them on enable. The subscription must be disabled with no workers left, and tables another subscription also feeds are refused. The truncate WAL record carries the subscription's own origin, and mid-sync tablesync slots and then origins are dropped last.

**User-visible surface.**
- New syntax: ALTER SUBSCRIPTION name REFRESH TABLE table_name [, ...], with no WITH options in the current version.
- New errors: refused when the subscription is enabled or two_phase is on, or when another subscription populates the table.
- The command fires TRUNCATE triggers as a session TRUNCATE would and needs TRUNCATE privilege on each named table.
- The replicated truncate carries the subscription's origin, so origin = none downstream subscribers skip both the truncate and the re-copy.
- C API: ExecuteTruncateGuts() gains an origin parameter, and truncate_check_rel/perms/activity become extern; 0002 accepts partitioned tables and partitions.

**What the author claims.**
- Replaces dropping and recreating the whole subscription, which re-copies every table and loses the others' progress.
- Leaves publication membership and other subscribers untouched, which the publication-membership workaround does not.
- All validation runs before anything irreversible, so ordinary failures leave no changes and re-running fixes a failure in the slot drop phase.
- Tagging the truncate with the subscription origin prevents data loss in two-way and origin = none cascade setups, for any cycle length.
- Truncating local-only partitions matches what a replicated TRUNCATE of the root already does.

**Thread state.** Sawada, Kuroda, Ajin Cherian, Shlok Kyal, Yilin Zhang and Song Yanli reviewed through v15; each round found lock-order deadlocks, data-loss cases or origin-state bugs, and the author fixed them. Nobody has replied to v14 or v15, and the cascade case where an intermediate node withholds TRUNCATE still leaves stale rows and apply conflicts.

## 2. Prior attempts

### Add `truncate` option to subscription commands (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2758/](https://commitfest.postgresql.org/patch/2758/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CF3B6672-2A43-4204-A60A-68F359218A9B@endpoint.com](https://www.postgresql.org/message-id/flat/CF3B6672-2A43-4204-A60A-68F359218A9B@endpoint.com)
- **Date:** decided 2020-11-25; started 2020-10-09
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P10 added a truncate option to existing subscription commands. This proposal adds a separate command that truncates and re-copies tables the subscription already tracks, with origin tagging, cross-subscription checks and slot cleanup.


## 3. Objections a reviewer is likely to raise

### 1. Refresh on a forwarding node can stop a downstream subscriber's apply worker

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-1`

In A -> B -> C, B's publication omits truncate and C uses origin = any. An operator runs REFRESH TABLE on B, and the re-copied rows flow to C, which still holds the old rows. C logs insert_exists, its apply worker stops, and every table C subscribes to falls behind.

Message #28 shows the case itself: the 'withholds TRUNCATE, origin = any' row ends with C stale and its apply worker stopped. The patch documents this but neither detects nor refuses it. The command runs on B, and B can see its own publications on the table and their publish options. A local repair on one node therefore halts replication on another node, and the operator finds out only from C's server log.

Thread: [#27 ("Hayato Kuroda (Fujitsu)", 2026-10-23)](https://www.postgresql.org/message-id/OS7PR01MB1831757BF5C212B9E04F7E615F5642%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#28 (Cagri Biroglu, 2026-10-25)](https://www.postgresql.org/message-id/CAA36mspCc4wqZJMuPTxG_sy15_yBkY3sw%3DTB9Y1orrp-uS6r9Q%40mail.gmail.com)

**Question:** Why does the command not warn or refuse when the node republishes the table through a publication that does not publish truncate?

Counter-check: held. Message #28 confirms that C ends with insert_exists and a stopped apply worker, and v15 does not change that. B can see its own publications' publish options, so a warning or refusal is possible and nobody has decided against it.

### 2. Command duplicates tablesync and catalog lock-order invariants with no loud failure

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), catalog changes, WAL / replication / standby · `maintenance-fit-1`

A later patch changes the lock level REFRESH PUBLICATION, CREATE SUBSCRIPTION or the tablesync worker takes on pg_subscription_rel, or adds a sync state. REFRESH TABLE keeps its own ordering and slot-then-origin drop sequence, so a deadlock or a lost-row case comes back. Users find out in production, through a worker killed by the deadlock detector or rows that are never copied back.

Each review round found a new interaction outside the command: a deadlock with a tablesync worker (#15, #16), with user transactions (#16), with partition lock order (#23), with REFRESH PUBLICATION (#29), and an origin Assert reached from triggers (#34). The fixes rest on ordering rules held in comments and TAP tests, and other subscription commands do not depend on those rules. Sawada, a committer, reviewed v3 (#9), but no committer has commented on the design since.

Thread: [#9 (Masahiko Sawada, 2026-08-10)](https://www.postgresql.org/message-id/CAD21AoAe3XmT03NLqLNEJbiYb3dyW3iTBY-RV51qYDy%2Bs0zpLA%40mail.gmail.com), [#15 ("Hayato Kuroda (Fujitsu)", 2026-08-17)](https://www.postgresql.org/message-id/OS9PR01MB12149489D6B3C370CE8FEC97CF5A72%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#16 (Cagri Biroglu, 2026-08-19)](https://www.postgresql.org/message-id/CAA36mspx4Szp02%3DG5Lop9gyeqv8r2yKs-R5%2B8xB-YTtv5d9e0Q%40mail.gmail.com), [#23 ("Hayato Kuroda (Fujitsu)", 2026-09-24)](https://www.postgresql.org/message-id/TY5PR01MB183149575EEA744B1CFA8BA6DF5812%40TY5PR01MB18314.jpnprd01.prod.outlook.com), [#29 (Shlok Kyal, 2026-10-29)](https://www.postgresql.org/message-id/CANhcyEVjG62LDOuUTDAAa_iS%3DLTxUzqSm1RhRHBh_B8vewMY8g%40mail.gmail.com), [#34 (Shlok Kyal, 2026-11-03)](https://www.postgresql.org/message-id/CANhcyEW66sNdiie1GdSusUCh1njEbVcC7PkyfFSoh4Yx%3DGpTtA%40mail.gmail.com)

**Question:** Which lock-order and state-machine invariants does REFRESH TABLE depend on, and what fails at build or test time if another subscription command changes them?

Counter-check: held. Five review rounds each found a new lock-order or state interaction with another command, which supports the concern. v13 adopts the lock order of AlterSubscription_refresh() and adds a TAP assertion, but no reviewer has confirmed that this covers later changes.

### 3. Truncate and re-copy fire different trigger sets

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A subscriber keeps an ordinary AFTER TRUNCATE trigger and a row trigger that maintain a summary table. REFRESH TABLE fires the truncate trigger in the user's session, but the tablesync re-copy runs as replica and skips the ordinary row trigger. The summary table stays empty, and the user learns this only when reports go wrong.

No other ALTER SUBSCRIPTION command fires user triggers (message #33). Under v15 one operation has two halves with different trigger rules. Message #35 keeps TRUNCATE semantics and points users to ENABLE ALWAYS, but that changes how the trigger behaves for all replicated changes. Nobody has replied to that choice.

Thread: [#33 (Shlok Kyal, 2026-11-03)](https://www.postgresql.org/message-id/CANhcyEWG6DsUudBSnj7Swsf%2BsJXioDgAphtHXFvgimqXJ%2Bq2SQ%40mail.gmail.com), [#35 (Cagri Biroglu, 2026-11-06)](https://www.postgresql.org/message-id/CAA36msotccBGaiej3n3qsUgxAqjtwpB-F3vYt8UoW0krHUFK9w%40mail.gmail.com)

**Question:** Why should one command fire the session's TRUNCATE triggers for the discard but only replica or always triggers for the refill, and not one rule for both halves?

Counter-check: held. Message #33 asked which trigger rule applies, and message #35 kept session TRUNCATE semantics with no reviewer reply. The scenario is weak, because ordinary row triggers never fire for replicated rows on a subscriber, but the consistency question stays open.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The truncate WAL record carries the subscription's origin, and the command drops tablesync slots and origins. |
| wire protocol | no |  |
| catalog changes | **yes** | The command resets pg_subscription_rel rows to init from a user backend; it adds no catalog column. |
| ABI / extension API | **yes** | ExecuteTruncateGuts() gains an origin parameter, and three truncate_check functions become extern. |
| SQL-standard conflict or non-standard syntax | **yes** | New ALTER SUBSCRIPTION name REFRESH TABLE table_name [, ...] syntax in gram.y. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | The patch adds tab completion in src/bin/psql/tab-complete.in.c. |
| security / privileges | **yes** | The command needs TRUNCATE privilege on each table and fires TRUNCATE triggers in the caller's session. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The command keeps its own lock order and slot-then-origin drop sequence next to tablesync and REFRESH PUBLICATION. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should REFRESH TABLE warn or refuse when the node republishes the table through a publication that publishes inserts but not truncate?
2. Which lock-order and sync-state invariants does REFRESH TABLE depend on, and which test fails if another subscription command changes them?
3. Why does the discard fire the session's TRUNCATE triggers while the re-copy fires only replica or always triggers, instead of one rule for both?
4. Would a later enabled-subscription version keep truncate-and-recopy with the table empty during sync, or use a different mechanism?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Masahiko Sawada is the CommitFest reviewer and reviewed early versions, but nobody has replied to v14 or v15. The owner must know tablesync, replication origins and truncate.
- **Liability:** The owner signs for lock ordering against tablesync workers, REFRESH PUBLICATION and CREATE SUBSCRIPTION, for origin-tagged truncate WAL in cascade setups, and for the trigger behaviour, for as long as the command exists.
- **Author follow-up:** The author fixed every reported problem through fifteen versions and rebased twice after CFbot reports, which suggests continued follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The need is shown by test in message #1 and the author responds to review. The two patches total about 135 KiB across subscriptioncmds.c, tablecmds.c and worker.c, and each review round found a new lock-order or origin problem.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Origin-tagged truncate breaks downstream nodes that do not publish TRUNCATE. Refuted (not specific): Duplicate of replication-ondisk-1. The title also blames the origin tag, but the matrix in message #28 shows the same stale result in v11, before the tag existed.
- `standards-compat-3` Refreshing a partitioned root discards local-only partitions. Refuted (already handled): The 0002 documentation states that subscriber-only partitions are truncated and not refilled. The re-copy routes rows by the subscriber's partition bounds, so the command cannot tell which partitions the publisher never fills, and the user names the root explicitly.
- `replication-ondisk-2` Partial failure during slot drops leaves a mid-sync table unable to resume. Refuted (already handled): The command returns an error to the client when a slot drop fails, and message #28 documents re-running as the remedy. REFRESH PUBLICATION and DROP SUBSCRIPTION also drop tablesync slots last with missing_ok, so a rollback there restores rows for dropped slots too.
- `maintenance-fit-2` User backend writing WAL under a subscription origin widens what origins mean. Refuted (no failure scenario): Any session can already write WAL under an origin through pg_replication_origin_session_setup(), so the assumed guarantee does not exist today. A changed C signature in a major release is normal, and the parameter itself limits the origin to the WAL record.
- `maintenance-fit-3` Visible semantics are fixed before the enabled or concurrent design exists. Refuted (no failure scenario): The scenario names no behaviour that a later enabled or concurrent version could not relax, since lifting a restriction breaks no runbook. Its concrete parts repeat standards-compat-1 and replication-ondisk-1.
- `demand-1` Demand rests on one operator's topology, with no field reports. Refuted (already handled): Message #1 shows by test that the publication workaround wipes the downstream node and leaves the sibling behind, and that the catalog edit races the apply worker. Core has REFRESH SEQUENCES for the same need, and pglogical ships a per-table resynchronize function.

### Review angles

- prior-attempts: 0 objection(s). Only [P10] is close, and it was withdrawn with no recorded reason, so no earlier objection can be tested against this design. [P23] concerns TRUNCATE under session_replication_role = replica and also has no recorded reason. Within the thread, message #1 and message #3 point only to the REFRESH SEQUENCES syntax debate, which supports the chosen spelling.
- standards-compat: 3 objection(s). The REFRESH TABLE spelling follows REFRESH SEQUENCES and was agreed in messages #3 and #4. It adds no keyword, GUC or catalog object, so pg_dump and psql conventions are not affected.
- replication-ondisk: 2 objection(s). The proposal adds no new WAL record type, catalog column or on-disk format, so physical standbys and pg_upgrade are not affected. The ExecuteTruncateGuts() signature change targets master only and is a normal major-version API change.
- security: 0 objection(s). No security objection survives. Running the command takes subscription ownership plus TRUNCATE on each table, rechecked under AccessExclusiveLock. Triggers fire as in a TRUNCATE in that session. The origin-tagged truncate gives a subscription owner nothing the subscription's own apply could not already write under that origin.
- maintenance-fit: 3 objection(s). Objections were found. Hot-path cost is not one of them, because the work runs only when a user runs the command.
- already-exists: 0 objection(s). Postgres has no supported way to re-copy one already-subscribed table at a slot-consistent point. REFRESH PUBLICATION skips tracked tables, REFRESH SEQUENCES covers only sequences, and the publication-membership and pg_subscription_rel workarounds are non-local or race the apply worker, as message #1 shows. A manual COPY cannot line up with the apply position, and no extension or hook does this job, so the angle finds the need unmet. [P10] has no recorded reason and was not used.
- demand: 1 objection(s). Demand is plausible: the workarounds in message #1 are shown to be non-local, and a committer is engaged. So this is a request to show demand, not a rejection.

### Precedents retrieved

- P1: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant; reason summarised from its thread
- P2: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant; reason summarised from its thread
- P3: [Revert fix missing call to table_finish_bulk_insert during COPY](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5db56fc4d6e95c582b61c99328ea0702b869fa0) (revert, Reverted, 2019-07-01) - judged not relevant
- P4: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P5: [Fix alter subscription concurrency errors](https://commitfest.postgresql.org/patch/3842/) (cf, Withdrawn, 2023-03-28) - judged not relevant; reason summarised from its thread
- P6: [Allow single table VACUUM in transaction block](https://commitfest.postgresql.org/patch/3969/) (cf, Returned with feedback, 2022-11-24) - judged not relevant; reason summarised from its thread
- P7: [Bug in ALTER SUBSCRIPTION ... SERVER / ... CONNECTION with broken old server](https://commitfest.postgresql.org/patch/6707/) (cf, Withdrawn, 2026-06-22) - judged not relevant; reason summarised from its thread
- P8: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant; reason summarised from its thread
- P9: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant; reason summarised from its thread
- P10: [Add `truncate` option to subscription commands](https://commitfest.postgresql.org/patch/2758/) (cf, Withdrawn, 2020-11-25) - used in the review
- P11: [Add force flag while doing pg_restore to force dropping database in case of existing connections](https://commitfest.postgresql.org/patch/4485/) (cf, Returned with feedback, 2024-01-14) - judged not relevant
- P12: [Add a reset_computed_values function in pg_stat_statements](https://commitfest.postgresql.org/patch/2275/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P13: [pg_basebackup to adjust existing data directory permissions](https://commitfest.postgresql.org/patch/2018/) (cf, Returned with feedback, 2019-11-27) - judged not relevant
- P14: [New Table Access Methods for Multi and Single Inserts](https://commitfest.postgresql.org/patch/2871/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P15: [New Table Access Methods for Multi and Single Inserts](https://commitfest.postgresql.org/patch/6643/) (cf, Withdrawn, 2026-04-28) - judged not relevant
- P16: [Reloption tests iprovement. Test resetting illegal option that was actually set for some reason](https://commitfest.postgresql.org/patch/3534/) (cf, Returned with feedback, 2022-06-29) - judged not relevant
- P17: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P18: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - judged not relevant
- P19: [Respect client-initiated CopyDone during logical streaming replication](https://commitfest.postgresql.org/patch/621/) (cf, Withdrawn, 2019-07-08) - judged not relevant
- P20: [Revert "Allow logical replication snapshots to be database-specific"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=01a80f062146af1b17b411c35cb8d992c487fa7c) (revert, Reverted, 2026-05-24) - judged not relevant
- P21: [Reloption tests iprovement. Test resetting illegal option that was actually set for some reason](https://commitfest.postgresql.org/patch/3731/) (cf, Withdrawn, 2022-07-01) - judged not relevant
- P22: [Allow using replication origins in SQL level parallel sessions](https://commitfest.postgresql.org/patch/6035/) (cf, Withdrawn, 2025-09-16) - judged not relevant
- P23: [TRUNCATE behavior when session_replication_role = replica](https://commitfest.postgresql.org/patch/1447/) (cf, Rejected, 2018-04-07) - used in the review
- P24: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 162 s; cost reported by the CLI: $4.41
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 93 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:22:12.817Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
