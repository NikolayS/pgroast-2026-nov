# Design pre-review: Collect ALTER PUBLICATION commands for event triggers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7055](https://commitfest.postgresql.org/patch/7055/) (Needs review); authors: Fujii Masao (masaofujii)
- Thread: [Collect ALTER PUBLICATION commands for event triggers](https://www.postgresql.org/message-id/flat/CAHGQGwH-wWO%3D0406dbSiPvxNMhc-6TQK7ORZ_%2B9SyaR5fDsvyg%40mail.gmail.com), started 2026-07-24
- Messages read: 11 of 11
- Patch prose read: v2-0001-Collect-ALTER-PUBLICATION-mapping-drops-for-event.patch, v2-0002-Collect-ALTER-PUBLICATION-SET-ALL-commands-for-ev.patch, v2-0003-Invoke-post-alter-hook-for-ALTER-PUBLICATION-SET-.patch, v1-0001-Collect-ALTER-PUBLICATION-mapping-drops-for-event.patch, v1-0002-Collect-ALTER-PUBLICATION-SET-ALL-commands-for-ev.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal: message #2 notes the documentation expects a ddl_command_end entry for the main command after sql_drop. Three fixable design points remain. The patches do not state whether a row means the command ran or the catalog changed (standards-compat-2). The v2 commit messages still plan a back-patch that changes output in minor releases (standards-compat-1). Per-path collection can miss future subcommands (maintenance-fit-1).

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check how 0001 detects removals, how AlterPublicationAllFlags() decides the state changed, or whether mixed SET can return duplicate publication rows. Only 3 of 11 messages were read directly.

## The proposal, restated

**Problem.** Some ALTER PUBLICATION forms fire ddl_command_end, but pg_event_trigger_ddl_commands() returns no entry for them. This affects DROP TABLE, DROP TABLES IN SCHEMA, SET forms that only remove members, and SET ALL TABLES or SET ALL SEQUENCES. Event trigger users who audit or deparse commands miss these commands.

**Proposed design.** 0001 collects the publication itself as the ALTER PUBLICATION command when the command removes table or schema members, and keeps the existing per-member entries for additions. 0002 makes AlterPublicationAllFlags() report whether the publication state changed, and collects the publication when it did. 0003 calls the post-alter object access hook for SET ALL.

**User-visible surface.**
- pg_event_trigger_ddl_commands() returns a publication-level ALTER PUBLICATION row for DROP TABLE, DROP TABLES IN SCHEMA and remove-only SET.
- pg_event_trigger_ddl_commands() returns a row for SET ALL TABLES and SET ALL SEQUENCES when the publication state changes.
- SET with both additions and removals, and SET ALL TABLES ... EXCEPT, now return a publication row plus the per-member rows.
- Object access hooks now get a post-alter call for ALTER PUBLICATION SET ALL (0003).

**What the author claims.**
- The missing entry is a bug, since ddl_command_end fires but reports no command.
- The extra publication row next to per-member rows is intentional, as each covers a different part of the command.
- 0001 should go to all supported branches, or only to master if the behaviour change is too risky.
- 0002 and 0003 should go back to v19, where SET ALL TABLES and SET ALL SEQUENCES appeared.
- Command-level auditing needs the entry even when sql_drop gives enough data to deparse.

**Thread state.** Ian Barwick and Chao Li find v2 reasonable, and Zhijie Hou supports 0002 but asks what 0001 adds beyond sql_drop. shihao zhong measured that no-op commands still return zero rows and asks for a fixed 0002 commit message and a rebase; the author has not replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-24; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Row is reported only when the catalog changed, unlike other ALTER forms

fixable · level 2, rubric item 12 · design, backpatchability · `standards-compat-2`

An auditor relies on the new rows to log every ALTER PUBLICATION that ran. ALTER PUBLICATION p SET (publish='insert') returns a row every time, but a repeated SET TABLE with the same set or SET ALL TABLES on an all-tables publication returns none. The audit log misses these commands, and the gap shows up only during a review of the log.

Message #10 measured both rules on the same command. The author's stated goal in message #7 is command-level auditing, and that goal needs a row whenever ddl_command_end fires. The 0002 commit message says it fixes the case of no entry, but the no-op SET ALL case still returns no entry. The proposal does not state which rule ddl_command_end follows, and it attaches no documentation hunk that says so.

Thread: [#7 (Fujii Masao, 2026-07-24)](https://www.postgresql.org/message-id/CAHGQGwFsLEMs1FmkK8Ff8WhD2zNiavPfpjsRHWYHgHv5638hCA%40mail.gmail.com), [#10 (shihao zhong, 2026-09-05)](https://www.postgresql.org/message-id/CAGRkXqQ1WDOE17dESEyUpyrVeDenjGqgnXRjJfste_45LjYxVA%40mail.gmail.com)

**Question:** Should pg_event_trigger_ddl_commands() return a row for every ALTER PUBLICATION that fires ddl_command_end, and if not, where is the catalog-changed rule documented?

Counter-check: held. Message #10 measured that SET (publish=...) returns a row every time while no-op SET TABLE and SET ALL TABLES return none. The author has not replied, and no patch states which rule applies.

### 2. Back-patching 0001 changes event trigger output in minor releases

fixable · level 2, rubric item 6 · backpatchability, pg_dump / restore round-trip · `standards-compat-1`

A site runs an event trigger that rebuilds commands from the rows of pg_event_trigger_ddl_commands(), for example to replay DDL elsewhere. After a minor upgrade, ALTER PUBLICATION p SET TABLE with both additions and removals returns an extra publication row. The tool replays or logs the command twice, and the site finds out when the replica or audit log diverges.

The v2 commit messages still say 0001 goes to all supported versions and 0002 to v19. In message #2, Barwick said this kind of behaviour change usually does not get back-patched when nobody has complained. The author agreed in message #5 but did not change the plan. Zhijie Hou notes in message #4 that sql_drop already lets tools deparse the drop forms, so existing tools may already handle these cases.

Thread: [#2 (Ian Lawrence Barwick, 2026-07-24)](https://www.postgresql.org/message-id/CAB8KJ%3DjwEsFKhx%2Bv0xg9K5RQKrAjExWh9nrp7LgNhUkndcNcaQ%40mail.gmail.com), [#4 ("Zhijie Hou (Fujitsu)", 2026-07-24)](https://www.postgresql.org/message-id/TY4PR01MB1771848D7483FC22C8617E68A94CF2%40TY4PR01MB17718.jpnprd01.prod.outlook.com), [#5 (Fujii Masao, 2026-07-24)](https://www.postgresql.org/message-id/CAHGQGwEscdd%3DUaxojkWthY8MT%2BJJ9-KY61QLVSfAZ02RwJiEqA%40mail.gmail.com)

**Question:** Which existing event trigger users need the drop-form row in back branches, and why is a master-only change for 0001 not enough?

Counter-check: held. The v2 commit message for 0001 still says "Backpatch to all supported versions", although the author accepted Barwick's point in message #5. The extra publication row for mixed SET is a real output change in a minor release.

### 3. Per-path change detection leaves future ALTER PUBLICATION forms easy to miss

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later release adds a new ALTER PUBLICATION subcommand and its author does not add a collection call for that path. No regression test fails. Audit triggers again see ddl_command_end with no row, and users find out only when an audit gap is reported.

0001 and 0002 add collection inside each code path that changes state. AlterPublicationAllFlags() also has to report whether anything changed. The v19 SET ALL feature already missed both the collection and the post-alter hook (0002, 0003; message #3), so this per-path pattern has been missed once before. Collecting the publication once per command would also remove the split shihao zhong measured, where no-op SET ALL TABLES returns zero rows and a no-op SET (publish=...) returns a row (message #10).

Thread: [#3 (Chao Li, 2026-07-24)](https://www.postgresql.org/message-id/B2B24887-00C6-4A4D-97E0-89529D476D47%40gmail.com), [#10 (shihao zhong, 2026-09-05)](https://www.postgresql.org/message-id/CAGRkXqQ1WDOE17dESEyUpyrVeDenjGqgnXRjJfste_45LjYxVA%40mail.gmail.com)

**Question:** Why not collect the publication once for every ALTER PUBLICATION, so a new subcommand cannot silently skip event trigger reporting?

Counter-check: held. Per-path collection is the existing practice for ALTER PUBLICATION, so the patch did not create the pattern. But v19 SET ALL already missed both the collection and the hook, and the proposal does not say why one collection per command was not chosen.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0003 adds a post-alter object access hook call for SET ALL, and extensions read the rows of pg_event_trigger_ddl_commands(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | see objection standards-compat-1: Back-patching 0001 changes event trigger output in minor releases |
| psql UX conventions | no |  |
| security / privileges | **yes** | 0003 makes ALTER PUBLICATION SET ALL visible to object access hook users such as sepgsql. |
| backpatchability | **yes** | 0001 targets all supported branches, 0002 and 0003 target v19, and mixed SET returns an extra row. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each ALTER PUBLICATION code path must collect the command itself, and v19 SET ALL already missed it. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should pg_event_trigger_ddl_commands() return a row for every ALTER PUBLICATION that fires ddl_command_end, including a no-op SET TABLE or SET ALL TABLES, or only when the catalog changed?
2. Why not collect the publication once per ALTER PUBLICATION command, so a new subcommand cannot skip event trigger reporting?
3. Which existing event trigger users need the drop-form row in back branches, and why is a master-only change for 0001 not enough?
4. How should a trigger that reads both the publication row and the per-member rows of a mixed SET avoid counting the command twice?
5. Does the event trigger documentation need a change to state the rule, and can 0003 be committed apart from 0001 and 0002?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The maintenance-fit review notes that a committer in the area wrote the patch; the owner needs to know event trigger command collection and publicationcmds.c.
- **Liability:** The owner signs for the rows ALTER PUBLICATION returns to event triggers, in every branch that gets the patch. If back-patched, that includes any breakage of tools that parse these rows after a minor update.
- **Author follow-up:** The author posted v2 in message #5 and answered the sql_drop question in message #7, but has not yet replied to the measurements in message #10.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patches touch one backend file plus the event_trigger regression test and fix a mismatch with documented behaviour. The users are event trigger and object access hook consumers of publication DDL, and the open points are a reporting rule and the back-patch range.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Back-patching 0001 changes event trigger output in a minor release. Refuted (not specific): Duplicate of standards-compat-1. Its added point on 0002 is weaker, because message #4 says SET ALL has no other means of deparsing, which supports a fix in v19.
- `already-exists-1` 0001 repeats what sql_drop already reports for publication member removals. Refuted (already handled): Message #2 shows the documentation expects a ddl_command_end entry for the main command after sql_drop, as ALTER TABLE ... DROP COLUMN gives, so 0001 restores existing practice. The back-branch risk is a duplicate of standards-compat-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P8] is about replicating DDL through a hand-written deparser and was withdrawn over that deparser's long-term maintenance cost. This proposal adds no deparser and does not touch parse node structures, so that reason does not apply. No thread message points to an earlier discussion of collecting ALTER PUBLICATION for event triggers; message #3 only names the commit that added SET ALL.
- standards-compat: 2 objection(s). Two objections raised. 0003, the post-alter hook call, follows the existing practice of AlterPublicationOptions() and raises no compatibility concern.
- replication-ondisk: 1 objection(s). The patches add no WAL records, catalog columns, on-disk format changes or protocol messages, so there is nothing for pg_upgrade or a standby to handle. The only cross-node effect is through user tools that forward event trigger output, covered above.
- security: 0 objection(s). The patches add no new privilege or exposure. Event triggers still need superuser or a role allowed to create them, the new rows describe a publication that the session's role already altered, and 0003 closes a gap for object access hooks such as sepgsql instead of opening one.
- maintenance-fit: 1 objection(s). Nothing else from this angle. There is no hot-path cost, no new dependency and no fit problem, and a committer in the area wrote the patch. [P8] is about the DDL deparser and does not apply here.
- already-exists: 1 objection(s). No existing mechanism covers 0002 or 0003. Message #4 confirms that nothing reports SET ALL today, and the post-alter hook has no substitute.
- demand: 0 objection(s). No demand objection. The patches fix a mismatch with documented behaviour: message #2 notes the docs say sql_drop is followed by a ddl_command_end entry for the main command. The cost falls on event trigger users, and message #4 says SET ALL has no other way to be deparsed. The lack of field reports in message #2 bears on backpatching 0001, which is not a demand question.

### Precedents retrieved

- P1: [USAGE privilege on PUBLICATION](https://commitfest.postgresql.org/patch/3641/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P2: [Fix cleanup of WSA events in WIN32 wait event code](https://commitfest.postgresql.org/patch/904/) (cf, Rejected, 2016-12-27) - judged not relevant
- P3: [Add new wait event to XactLockTableWait](https://commitfest.postgresql.org/patch/5804/) (cf, Withdrawn, 2026-03-31) - judged not relevant; reason summarised from its thread
- P4: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P5: [Revert "pg_createsubscriber: Add log message when no publications exist to drop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=45a7faf130176ae7c152413967c7c90068476a94) (revert, Reverted, 2025-10-15) - judged not relevant
- P6: [Optimize walsender handling invalid messages of 'drop publication'](https://commitfest.postgresql.org/patch/4393/) (cf, Withdrawn, 2023-10-20) - judged not relevant; reason summarised from its thread
- P7: [Add accumulated statistics for wait event](https://commitfest.postgresql.org/patch/2332/) (cf, Returned with feedback, 2020-08-02) - judged not relevant; reason summarised from its thread
- P8: [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (cf, Withdrawn, 2024-03-28) - used in the review; reason summarised from its thread
- P9: [Xact event callback at command start](https://commitfest.postgresql.org/patch/2860/) (cf, Rejected, 2021-09-01) - judged not relevant; reason summarised from its thread
- P10: [Fix NULL dereference in pg_get_database_ddl()](https://commitfest.postgresql.org/patch/6669/) (cf, Withdrawn, 2026-04-13) - judged not relevant; reason summarised from its thread
- P11: [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://commitfest.postgresql.org/patch/2664/) (cf, Returned with feedback, 2020-11-24) - judged not relevant; reason summarised from its thread
- P12: [Revert refactoring of restore command code to shell_restore.c](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f6e15ac93c58c1140e4a4affe61e78f7346497a) (revert, Reverted, 2023-02-05) - judged not relevant
- P13: [Revert changes to CREATE TRIGGER and ALTER TABLE ADD FOREIGN KEY locking, per request from Tom.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50ce8ab9fcd49dfcbff9e19c5aa53efe0fc6dc6e) (revert, Reverted, 2005-03-24) - judged not relevant
- P14: [Parallel tests publication and subscription might fail due to concurrent tuple update](https://commitfest.postgresql.org/patch/5459/) (cf, Rejected, 2025-03-27) - judged not relevant
- P15: [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://commitfest.postgresql.org/patch/5992/) (cf, Returned with feedback, 2026-04-02) - judged not relevant
- P16: [Storage layer hooks to enable data transformation in extensions](https://commitfest.postgresql.org/patch/6351/) (cf, Withdrawn, 2026-02-08) - judged not relevant
- P17: [Allows database-specific role memberships](https://commitfest.postgresql.org/patch/3374/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [Make deparsing of column defaults faster](https://commitfest.postgresql.org/patch/1657/) (cf, Returned with feedback, 2018-09-24) - judged not relevant
- P19: [Hooks and API to allow loadable wire protocol extensions](https://commitfest.postgresql.org/patch/3018/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [Add support for --end-segment-command in pg_receivewal](https://commitfest.postgresql.org/patch/1040/) (cf, Returned with feedback, 2017-03-21) - judged not relevant
- P21: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant
- P22: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P23: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P24: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 109 s; cost reported by the CLI: $1.42
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 72 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:16:58.255Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
