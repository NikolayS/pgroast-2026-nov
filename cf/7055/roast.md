# Design pre-review: Collect ALTER PUBLICATION commands for event triggers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7055](https://commitfest.postgresql.org/patch/7055/); [thread](https://www.postgresql.org/message-id/flat/CAHGQGwH-wWO%3D0406dbSiPvxNMhc-6TQK7ORZ_%2B9SyaR5fDsvyg%40mail.gmail.com), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal: message #2 notes the documentation expects a ddl_command_end entry for the main command after sql_drop. Three fixable design points remain. The patches do not state whether a row means the command ran or the catalog changed (standards-compat-2). The v2 commit messages still plan a back-patch that changes output in minor releases (standards-compat-1). Per-path collection can miss future subcommands (maintenance-fit-1).

Not checked: Without the code, this review could not check how 0001 detects removals, how AlterPublicationAllFlags() decides the state changed, or whether mixed SET can return duplicate publication rows. Only 3 of 11 messages were read directly.

## Prior attempts

None found before 2026-07-24.

## Objections a reviewer is likely to raise

### 1. Row is reported only when the catalog changed, unlike other ALTER forms

An auditor relies on the new rows to log every ALTER PUBLICATION that ran. ALTER PUBLICATION p SET (publish='insert') returns a row every time, but a repeated SET TABLE with the same set or SET ALL TABLES on an all-tables publication returns none. The audit log misses these commands, and the gap shows up only during a review of the log.

Message #10 measured both rules on the same command. The author's stated goal in message #7 is command-level auditing, and that goal needs a row whenever ddl_command_end fires. The 0002 commit message says it fixes the case of no entry, but the no-op SET ALL case still returns no entry. The proposal does not state which rule ddl_command_end follows, and it attaches no documentation hunk that says so.

**Question:** Should pg_event_trigger_ddl_commands() return a row for every ALTER PUBLICATION that fires ddl_command_end, and if not, where is the catalog-changed rule documented?

<sub>fixable, level 2, design; messages [#7](https://www.postgresql.org/message-id/CAHGQGwFsLEMs1FmkK8Ff8WhD2zNiavPfpjsRHWYHgHv5638hCA%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAGRkXqQ1WDOE17dESEyUpyrVeDenjGqgnXRjJfste_45LjYxVA%40mail.gmail.com); `standards-compat-2`</sub>

### 2. Back-patching 0001 changes event trigger output in minor releases

A site runs an event trigger that rebuilds commands from the rows of pg_event_trigger_ddl_commands(), for example to replay DDL elsewhere. After a minor upgrade, ALTER PUBLICATION p SET TABLE with both additions and removals returns an extra publication row. The tool replays or logs the command twice, and the site finds out when the replica or audit log diverges.

The v2 commit messages still say 0001 goes to all supported versions and 0002 to v19. In message #2, Barwick said this kind of behaviour change usually does not get back-patched when nobody has complained. The author agreed in message #5 but did not change the plan. Zhijie Hou notes in message #4 that sql_drop already lets tools deparse the drop forms, so existing tools may already handle these cases.

**Question:** Which existing event trigger users need the drop-form row in back branches, and why is a master-only change for 0001 not enough?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/CAB8KJ%3DjwEsFKhx%2Bv0xg9K5RQKrAjExWh9nrp7LgNhUkndcNcaQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/TY4PR01MB1771848D7483FC22C8617E68A94CF2%40TY4PR01MB17718.jpnprd01.prod.outlook.com), [#5](https://www.postgresql.org/message-id/CAHGQGwEscdd%3DUaxojkWthY8MT%2BJJ9-KY61QLVSfAZ02RwJiEqA%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Per-path change detection leaves future ALTER PUBLICATION forms easy to miss

A later release adds a new ALTER PUBLICATION subcommand and its author does not add a collection call for that path. No regression test fails. Audit triggers again see ddl_command_end with no row, and users find out only when an audit gap is reported.

0001 and 0002 add collection inside each code path that changes state. AlterPublicationAllFlags() also has to report whether anything changed. The v19 SET ALL feature already missed both the collection and the post-alter hook (0002, 0003; message #3), so this per-path pattern has been missed once before. Collecting the publication once per command would also remove the split shihao zhong measured, where no-op SET ALL TABLES returns zero rows and a no-op SET (publish=...) returns a row (message #10).

**Question:** Why not collect the publication once for every ALTER PUBLICATION, so a new subcommand cannot silently skip event trigger reporting?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/B2B24887-00C6-4A4D-97E0-89529D476D47%40gmail.com), [#10](https://www.postgresql.org/message-id/CAGRkXqQ1WDOE17dESEyUpyrVeDenjGqgnXRjJfste_45LjYxVA%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Should pg_event_trigger_ddl_commands() return a row for every ALTER PUBLICATION that fires ddl_command_end, including a no-op SET TABLE or SET ALL TABLES, or only when the catalog changed?
2. Why not collect the publication once per ALTER PUBLICATION command, so a new subcommand cannot skip event trigger reporting?
3. Which existing event trigger users need the drop-form row in back branches, and why is a master-only change for 0001 not enough?
4. How should a trigger that reads both the publication row and the per-member rows of a mixed SET avoid counting the command twice?
5. Does the event trigger documentation need a change to state the rule, and can 0003 be committed apart from 0001 and 0002?

## Touch points

- **ABI / extension API:** 0003 adds a post-alter object access hook call for SET ALL, and extensions read the rows of pg_event_trigger_ddl_commands().
- **pg_dump / restore round-trip:** see objection standards-compat-1: Back-patching 0001 changes event trigger output in minor releases
- **security / privileges:** 0003 makes ALTER PUBLICATION SET ALL visible to object access hook users such as sepgsql.
- **backpatchability:** 0001 targets all supported branches, 0002 and 0003 target v19, and mixed SET returns an extra row.
- **maintenance burden ("who maintains this in 10 years"):** Each ALTER PUBLICATION code path must collect the command itself, and v19 SET ALL already missed it.

## Who would commit it, and when

- Owner: qualified committer engaged. The maintenance-fit review notes that a committer in the area wrote the patch; the owner needs to know event trigger command collection and publicationcmds.c.
- Priority: proceed; audience niche; review cost low. The patches touch one backend file plus the event_trigger regression test and fix a mismatch with documented behaviour. The users are event trigger and object access hook consumers of publication DDL, and the open points are a reporting rule and the back-patch range.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
