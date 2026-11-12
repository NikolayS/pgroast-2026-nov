# Design pre-review: Extension security improvement: Add support for extensions with an owned schema

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5018](https://commitfest.postgresql.org/patch/5018/) (Needs review); authors: Jelte Fennema-Nio (jeltef)
- Thread: [Extension security improvement: Add support for extensions with an owned schema](https://www.postgresql.org/message-id/flat/CAGECzQQzDqDzakBkR71ZkQ1N1ffTjAaruRSqppQAKu3WF%2B6rNQ%40mail.gmail.com), started 2024-06-02
- Messages read: 43 of 43
- Patch prose read: v4-0001-Add-support-for-extensions-with-an-owned-schema.patch, v1-0001-Add-support-for-extensions-with-an-owned-schema.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has shown demand and committer support, and no surviving objection is fatal. The decisive open point is replication-ondisk-1: on v12 a prerequisite installed into an owned schema gives a dump that does not restore and a pg_upgrade that loses a pg_depend row. security-1, maintenance-fit-2 and standards-compat-2 are fixable design points that need a decision from someone who knows extension dependencies and pg_dump. Only 3 of 43 messages were read directly, so confidence is medium.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: No code was read, and only 3 of 43 messages were shown directly. Whether v12's pg_dump ordering and binary upgrade handle all dependency shapes of an owned schema cannot be judged from the prose.

## The proposal, restated

**Problem.** Extension scripts run by CREATE EXTENSION and ALTER EXTENSION UPDATE use a search_path that contains the extension schema. If a less privileged user can create functions or operators in that schema, unqualified references in the script can resolve to them and run as superuser. Managed services that run these commands as superuser for their users are the main case.

**Proposed design.** A new control file option owned_schema = true makes CREATE EXTENSION create the schema itself, and the command fails if the schema already exists. The schema becomes a member of the extension, so DROP EXTENSION ... CASCADE drops it. ALTER EXTENSION SET SCHEMA renames the owned schema to a name that must not exist yet.

**User-visible surface.**
- Control file option owned_schema (boolean, default false).
- CREATE EXTENSION ... SCHEMA: the named schema must not exist when owned_schema is true.
- ALTER EXTENSION ... SET SCHEMA renames the owned schema instead of moving objects.
- New catalog column pg_extension.extownedschema.
- pg_dump output and a changed binary_upgrade_create_empty_extension signature in pg_upgrade support.

**What the author claims.**
- Extension authors get a safe search_path without schema-qualifying every function and operator in their scripts.
- SECURITY DEFINER functions can safely set search_path to the extension's owned schema.
- The option removes the need for the pg_cron workaround of schema = pg_catalog plus create schema, and users still choose the schema name.
- It helps avoid name conflicts between extensions.
- The feature is useful even if only new extensions can use it.

**Thread state.** The latest patch is v12, which also adds a check that fails test_pg_dump on any pg_dump warning. Rui Zhao reported a three-object dependency cycle through CREATE EXTENSION ... SCHEMA ... CASCADE that breaks restore and loses a dependency in pg_upgrade, plus doc gaps; no reply yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-06-02; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Owned schema shared via CASCADE breaks restore and pg_upgrade drops a dependency

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, on-disk format / pg_upgrade, catalog changes · `replication-ondisk-1`

A user runs CREATE EXTENSION os_owned SCHEMA shared_schema CASCADE, which installs the required os_base into the owned schema as well. The plain or directory dump then emits os_owned before os_base, so restore fails with "required extension is not installed". pg_upgrade reports success, but pg_depend no longer records that os_owned depends on os_base.

Message #43 shows this on v12, and nobody has replied. The v12 fix for the two-object loop in message #41 drops one edge from the dump sort, so any longer cycle through the owned schema is still unhandled. A silent loss of a pg_depend row in binary upgrade is the worst outcome, because no error appears and later DROP or ALTER EXTENSION acts on the wrong dependency graph. The proposal does not yet say whether to forbid this layout or to support it.

Thread: [#41 ("Jelte Fennema-Nio", 2026-09-26)](https://www.postgresql.org/message-id/DLPDSPWPW6SS.3KZI2YC6VZVI0%40jeltef.nl), [#43 (Rui Zhao, 2026-09-29)](https://www.postgresql.org/message-id/CAHWVJhERrZ0df6L8GaOL9RMi8%2BoTm%3DH0STa-XL-iD2pjjqPcfw%40mail.gmail.com)

**Question:** Will CREATE EXTENSION reject a prerequisite extension landing in an owned schema, or will pg_dump and binary upgrade keep every pg_depend edge for this layout?

Counter-check: held. Message #43 gives a reproducer on v12 for a dump that does not restore and a pg_upgrade that loses a pg_depend row. No reply exists, and the proposal does not choose between rejecting the layout and supporting it.

### 2. Safe search_path claim fails when a required extension sits in a writable schema

fixable · level 2, rubric item 7 · security / privileges · `security-1`

Extension X sets owned_schema = true and requires hstore, which is installed in public or in another schema where an application role has CREATE. That role creates an overloaded operator there. The control plane runs CREATE EXTENSION X as superuser, an unqualified operator in X's script resolves to the planted one, and the role becomes superuser.

The extension script's search_path also includes the schemas of required extensions, and owned_schema does not make those schemas safe. The v4 docs say that with an owned schema "the search_path now only contains trusted schemas" and that general-purpose SQL is "safe by default". In message #21 the author answers that the dependency can harden its own scripts, but that does not protect unqualified names in the dependent extension's script. Message #18 raised the transitive case, and the docs still state the guarantee without this limit.

Thread: [#18 (Sadeq Dousti, 2025-07-27)](https://www.postgresql.org/message-id/CADE6LvjnX8MbZPSTkrdoRGwf%2B_q6deNN1%2B-N_bmjdUgvFNy%2BfQ%40mail.gmail.com), [#21 (Jelte Fennema-Nio, 2025-07-28)](https://www.postgresql.org/message-id/CAGECzQRFa-RLf2FMsL51TqXbtrbJjtC%3DSCMo50ZO7Pf_e0O7Sg%40mail.gmail.com)

**Question:** What keeps an owned_schema extension's script safe when a required extension lives in a schema that less privileged roles can write to, and do the docs state this limit?

Counter-check: held. CREATE EXTENSION adds the schemas of required extensions to the script's search_path, so an overload planted there still wins. Message #21 answers only for the dependency's own scripts, and the shown docs say the path "only contains trusted schemas".

### 3. ALTER EXTENSION SET SCHEMA flips its precondition per control file

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax · `standards-compat-2`

A DBA script runs create schema app_ext followed by alter extension e set schema app_ext, which is the documented pattern today. For an owned_schema extension the second command fails because app_ext exists. The DBA cannot see from the command which behaviour applies without reading the control file or pg_extension.extownedschema.

Message #4 confirms that for owned schemas the command renames the schema and requires the target to be absent, which is the opposite of the existing rule. The same verb now has two contracts, and a flag the installing user did not choose selects between them. A committer would likely ask whether a rename is better exposed as its own form, or whether SET SCHEMA should keep its contract and report when the extension owns its schema. Message #43 shows the docs still describe only the old behaviour.

Thread: [#4 (Jelte Fennema-Nio, 2024-06-05)](https://www.postgresql.org/message-id/CAGECzQTe3y6xwP9aMbFBCxKnQEEn3G3%3DcEDqoD5xaOwPwypd_A%40mail.gmail.com), [#43 (Rui Zhao, 2026-09-29)](https://www.postgresql.org/message-id/CAHWVJhERrZ0df6L8GaOL9RMi8%2BoTm%3DH0STa-XL-iD2pjjqPcfw%40mail.gmail.com)

**Question:** Why should one ALTER EXTENSION SET SCHEMA command require an existing schema for some extensions and a new one for others, and not use a separate rename form?

Counter-check: held. Message #4 confirms that SET SCHEMA requires an absent target for owned schemas and an existing one otherwise. Message #43 shows the ALTER EXTENSION docs still describe only the old rule, and the thread gives no reason for reusing the verb.

### 4. Schema comment survives pg_upgrade but is lost on dump/restore

fixable · level 2, rubric item 13 · pg_dump / restore round-trip · `standards-compat-4`

A user adds a comment on the extension's owned schema and moves the database with pg_dump and pg_restore. The comment is gone in the new database. The same cluster upgraded with pg_upgrade keeps it, so the two migration paths give different results.

Message #42 shows the ACL is dumped but the COMMENT is not, because the schema is now an extension member. A schema comment that survived a dump before this patch no longer does. The thread does not say which fix is planned.

Thread: [#42 (Manu, 2026-09-26)](https://www.postgresql.org/message-id/179044614455.109724.18419741920404710654%40gmail.com)

**Question:** Will pg_dump emit the owned schema's comment as it does its ACL, so dump/restore and pg_upgrade give the same result?

Counter-check: held. Message #42 measured it: a schema comment that survived dump and restore before the patch is lost, while pg_upgrade keeps it. The loss follows the existing rule for extension members, but the thread has no answer on which fix is planned.

### 5. Two permanent models of extension schema, with no path between them

fixable · level 1b, rubric item 6 · maintenance burden ("who maintains this in 10 years"), catalog changes, pg_dump / restore round-trip · `maintenance-fit-2`

An author of an existing extension such as pg_cron sets owned_schema = true in a new release. Clusters that already have the extension installed keep the old layout, because the control file option is read only at CREATE EXTENSION. The author either keeps both layouts or writes direct catalog updates in the update script, which pg_dump and pg_upgrade must then handle correctly.

Message #16 says the feature serves only new extensions, and the only migration route it offers is direct catalog changes in update scripts. pg_extension.extownedschema, the changed signature of binary_upgrade_create_empty_extension, and two meanings of ALTER EXTENSION SET SCHEMA (move objects, or rename the schema) all become permanent. Messages #4 and #21 already float making owned_schema the default, which would be a second compatibility break later. A committer would likely ask for the migration design before the format is fixed.

Thread: [#4 (Jelte Fennema-Nio, 2024-06-05)](https://www.postgresql.org/message-id/CAGECzQTe3y6xwP9aMbFBCxKnQEEn3G3%3DcEDqoD5xaOwPwypd_A%40mail.gmail.com), [#16 (Jelte Fennema-Nio, 2024-10-04)](https://www.postgresql.org/message-id/CAGECzQS02M6YPDXemo36tShO-ZYObjqnyTJyVttua1PGyN4xRw%40mail.gmail.com), [#21 (Jelte Fennema-Nio, 2025-07-28)](https://www.postgresql.org/message-id/CAGECzQRFa-RLf2FMsL51TqXbtrbJjtC%3DSCMo50ZO7Pf_e0O7Sg%40mail.gmail.com)

**Question:** How does an installed extension switch to an owned schema without direct catalog changes, and would that route need different catalog or dump support than v12?

Counter-check: held. Message #16 confirms installed extensions have no supported route and points to catalog changes in update scripts. Reading control file options only at CREATE EXTENSION is existing practice, which lowers the weight, but nobody has shown that a later migration fits the v12 catalog design.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The patch changes the signature of binary_upgrade_create_empty_extension, and message #43 reports a lost pg_depend row after pg_upgrade. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds the column pg_extension.extownedschema and makes the schema a member of the extension. |
| ABI / extension API | **yes** | The patch adds the control file option owned_schema for extension authors and changes src/include/commands/extension.h. |
| SQL-standard conflict or non-standard syntax | **yes** | CREATE EXTENSION ... SCHEMA and ALTER EXTENSION ... SET SCHEMA change their precondition when owned_schema is true. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump must order the owned schema and its extension; message #43 shows a dump that does not restore. |
| psql UX conventions | no |  |
| security / privileges | **yes** | The feature exists to keep less privileged roles out of the search_path of extension scripts run as superuser. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every later change to extension dumping or binary upgrade must handle a schema that is an extension member. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will CREATE EXTENSION reject a prerequisite extension that lands in an owned schema, or will pg_dump and binary upgrade keep every pg_depend row for that layout?
2. What keeps an owned_schema extension's script safe when a required extension lives in a schema that less privileged roles can write to, and will the docs state this limit?
3. How does an installed extension switch to an owned schema without direct catalog changes, and does that route fit the v12 catalog and dump design?
4. Why should ALTER EXTENSION SET SCHEMA require an absent schema for owned_schema extensions and an existing one otherwise, and not use a separate rename form?
5. Will pg_dump emit the comment on an owned schema as it emits its ACL, so that dump and restore match pg_upgrade?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Committers took part in the design discussion (messages #7, #12, #23, #30), but none has said they will commit it. The owner needs to know extension dependencies, pg_dump ordering and binary upgrade.
- **Liability:** The owner signs for a permanent pg_extension column, a second meaning of ALTER EXTENSION SET SCHEMA, and correct pg_dump and pg_upgrade handling of a schema that is an extension member.
- **Author follow-up:** The author reached v12 and moved the entry between CommitFests himself until February 2026. Message #43 has no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Extension authors and managed services need this, as message #2 describes, and costs fall only on extension DDL, pg_dump and pg_upgrade. The open cycle from message #43 and the search_path of required extensions need a design decision before more code review.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Dump of a shared-schema CASCADE layout does not restore. Refuted (not specific): Duplicate of replication-ondisk-1, which describes the same message #43 cycle and also states the pg_depend loss in binary upgrade.
- `standards-compat-3` Direct ALTER SCHEMA on the owned schema has no defined behaviour. Refuted (already handled): Messages #35 and #36 settle on no special restrictions, so normal schema rules apply, and Postgres already refuses a direct drop of an extension member. The scenario needs an admin to hand the schema to an untrusted role, which the thread weighed.
- `replication-ondisk-2` Catalog migration to owned_schema is left to hand edits in update scripts. Refuted (no failure scenario): The scenario depends on an extension author writing direct catalog updates, which a superuser update script can do today with the same risks. The patch does not add or document that route; the missing migration path is covered by maintenance-fit-2.
- `replication-ondisk-3` Making the schema an extension member drops its comment from dumps. Refuted (not specific): Duplicate of standards-compat-4. The added points on security labels and default privileges have no evidence in the thread.
- `security-2` Documented SECURITY DEFINER safety depends on schema ACLs nothing enforces. Refuted (already handled): Messages #24 to #36 weighed this case, and a committer concluded in #33 and #35 that it is not a security issue and needs no special restriction. A grant of CREATE on any schema that holds SECURITY DEFINER functions carries the same risk today.
- `maintenance-fit-1` An extension that owns its schema adds dependency cycles that pg_dump and pg_upgrade do not handle. Refuted (not specific): Duplicate of replication-ondisk-1 for the open cycle and of standards-compat-4 for the comment. The two-object loop is fixed in v12, and v12-0001 makes test_pg_dump fail on any pg_dump warning.
- `already-exists-1` Most of the protection is already available through the pg_cron control file idiom. Refuted (already handled): The idiom cannot give a user-chosen schema name or relocation, and message #2 says it works by disabling extension schemas. An extension developer and three committers (messages #2, #7, #12, #15) support the need, so the gap is more than one author's convenience.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposes extension-owned schemas; [P2], [P3], [P4] and [P6] cover search_path for maintenance commands and functions, and their recorded reasons do not apply here. Messages #8 and #10 link earlier search_path threads but report no rejection of this idea.
- standards-compat: 4 objection(s). Not applicable; the objections above are fixable and concern dump round-trip and command contracts, not the goal itself.
- replication-ondisk: 3 objection(s). Objections are limited to catalog, pg_dump and pg_upgrade state. The feature adds no WAL records or wire protocol changes and does not change the heap format.
- security: 2 objection(s). Two fixable objections. The aim, keeping less privileged roles out of the schema used by superuser-run extension scripts, closes a real privilege escalation path that message #2 describes on managed services.
- maintenance-fit: 2 objection(s). Hot-path cost is not a concern: the feature adds work only in extension DDL, pg_dump and pg_upgrade. Robert Haas reviewed the design (messages #23 and #30), but no committer has said they will commit it over 13 CommitFests.
- already-exists: 1 objection(s). One objection from this angle. Several reviewers support the direction and agree the user-chosen schema name is worth having, so this objection weighs less.
- demand: 0 objection(s). Demand is shown. In message #2 Marco Slot describes managed services where extension scripts were exploited to gain superuser, and treats the workaround as a hard requirement. Message #1 cites the pg_cron workaround, message #15 reports consensus, and Robert Haas agrees the problem is real in messages #7 and #12. The gain applies only to new extensions or new installs, which the author accepts in message #16, but that limits reach, not whether anyone needs it.

### Precedents retrieved

- P1: [Prevent extension creation in temporary schemas](https://commitfest.postgresql.org/patch/1969/) (cf, Returned with feedback, 2019-03-07) - judged not relevant; reason summarised from its thread
- P2: [Allow excluding pg_temp or pg_catalog from search_path.](https://commitfest.postgresql.org/patch/4508/) (cf, Withdrawn, 2023-11-17) - used in the review; reason summarised from its thread
- P3: [Fix search_path for maintenance commands](https://commitfest.postgresql.org/patch/4367/) (cf, Withdrawn, 2024-01-18) - used in the review; reason summarised from its thread
- P4: [Add checks on search_path for IMMUTABLE and SECURITY DEFINER functions](https://commitfest.postgresql.org/patch/3684/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P5: [Tab completion for CREATE SCHEMA AUTHORIZATION](https://commitfest.postgresql.org/patch/3252/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P6: [Revert "Fix search_path to a safe value during maintenance operations."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2fcc7ee7affe2fb708d7b2727e93981216b5d2f8) (revert, Reverted, 2023-06-10) - used in the review
- P7: [psql: add schema size information to \dn+](https://commitfest.postgresql.org/patch/2010/) (cf, Withdrawn, 2019-02-25) - judged not relevant; reason summarised from its thread
- P8: [Support % wildcard in extension upgrade script filenames](https://commitfest.postgresql.org/patch/4489/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [Refactor add_path for clarity and extensibility](https://commitfest.postgresql.org/patch/3599/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P10: [Revert "Forbid DROP SCHEMA on temporary namespaces"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65192e02441cedd106b6abebe0036fb8cc124fb3) (revert, Reverted, 2020-01-08) - judged not relevant
- P11: [Adaptive query optimization: create extented statistics using auto_explain extension](https://commitfest.postgresql.org/patch/2958/) (cf, Returned with feedback, 2021-11-04) - judged not relevant; reason summarised from its thread
- P12: [Support % wildcard in extension upgrade scripts](https://commitfest.postgresql.org/patch/3654/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P13: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P14: [Support % wildcard in extension upgrade scripts](https://commitfest.postgresql.org/patch/3892/) (cf, Withdrawn, 2022-09-14) - judged not relevant
- P15: [revert "warn of SECURITY DEFINER schemas for non-sql_body funcs"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f77becbe5a99c9853b1ae7f22aa15babd01368b7) (revert, Reverted, 2022-09-28) - judged not relevant
- P16: [Vertical Clustered Index (columnar storage extension)](https://commitfest.postgresql.org/patch/945/) (cf, Returned with feedback, 2017-03-06) - judged not relevant
- P17: [Configurable location for extension .control files](https://commitfest.postgresql.org/patch/170/) (cf, Returned with feedback, 2015-07-30) - judged not relevant
- P18: [Allows database-specific role memberships](https://commitfest.postgresql.org/patch/3374/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P19: [CREATE ROLE IF NOT EXISTS](https://commitfest.postgresql.org/patch/3370/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P20: [Catalog version access](https://commitfest.postgresql.org/patch/2999/) (cf, Returned with feedback, 2022-01-31) - judged not relevant
- P21: [Fix PGXS so built control files work with VPATH builds](https://commitfest.postgresql.org/patch/2461/) (cf, Returned with feedback, 2020-07-31) - judged not relevant
- P22: [Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file](https://commitfest.postgresql.org/patch/3442/) (cf, Withdrawn, 2022-02-27) - judged not relevant
- P23: [Fixes for non-atomic read of read of control file on ext4 + ntfs](https://commitfest.postgresql.org/patch/4025/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P24: [extension patch of CREATE OR REPLACE TRIGGER](https://commitfest.postgresql.org/patch/2034/) (cf, Returned with feedback, 2019-09-30) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (maintenance-fit-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 197 s; cost reported by the CLI: $3.95
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 108 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:28:47.406Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
