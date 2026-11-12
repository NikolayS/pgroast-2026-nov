# Design pre-review: Extension security improvement: Add support for extensions with an owned schema

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5018](https://commitfest.postgresql.org/patch/5018/); [thread](https://www.postgresql.org/message-id/flat/CAGECzQQzDqDzakBkR71ZkQ1N1ffTjAaruRSqppQAKu3WF%2B6rNQ%40mail.gmail.com), 43 of 43 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has shown demand and committer support, and no surviving objection is fatal. The decisive open point is replication-ondisk-1: on v12 a prerequisite installed into an owned schema gives a dump that does not restore and a pg_upgrade that loses a pg_depend row. security-1, maintenance-fit-2 and standards-compat-2 are fixable design points that need a decision from someone who knows extension dependencies and pg_dump. Only 3 of 43 messages were read directly, so confidence is medium.

Not checked: No code was read, and only 3 of 43 messages were shown directly. Whether v12's pg_dump ordering and binary upgrade handle all dependency shapes of an owned schema cannot be judged from the prose.

## Prior attempts

None found before 2024-06-02.

## Objections a reviewer is likely to raise

### 1. Owned schema shared via CASCADE breaks restore and pg_upgrade drops a dependency

A user runs CREATE EXTENSION os_owned SCHEMA shared_schema CASCADE, which installs the required os_base into the owned schema as well. The plain or directory dump then emits os_owned before os_base, so restore fails with "required extension is not installed". pg_upgrade reports success, but pg_depend no longer records that os_owned depends on os_base.

Message #43 shows this on v12, and nobody has replied. The v12 fix for the two-object loop in message #41 drops one edge from the dump sort, so any longer cycle through the owned schema is still unhandled. A silent loss of a pg_depend row in binary upgrade is the worst outcome, because no error appears and later DROP or ALTER EXTENSION acts on the wrong dependency graph. The proposal does not yet say whether to forbid this layout or to support it.

**Question:** Will CREATE EXTENSION reject a prerequisite extension landing in an owned schema, or will pg_dump and binary upgrade keep every pg_depend edge for this layout?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#41](https://www.postgresql.org/message-id/DLPDSPWPW6SS.3KZI2YC6VZVI0%40jeltef.nl), [#43](https://www.postgresql.org/message-id/CAHWVJhERrZ0df6L8GaOL9RMi8%2BoTm%3DH0STa-XL-iD2pjjqPcfw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Safe search_path claim fails when a required extension sits in a writable schema

Extension X sets owned_schema = true and requires hstore, which is installed in public or in another schema where an application role has CREATE. That role creates an overloaded operator there. The control plane runs CREATE EXTENSION X as superuser, an unqualified operator in X's script resolves to the planted one, and the role becomes superuser.

The extension script's search_path also includes the schemas of required extensions, and owned_schema does not make those schemas safe. The v4 docs say that with an owned schema "the search_path now only contains trusted schemas" and that general-purpose SQL is "safe by default". In message #21 the author answers that the dependency can harden its own scripts, but that does not protect unqualified names in the dependent extension's script. Message #18 raised the transitive case, and the docs still state the guarantee without this limit.

**Question:** What keeps an owned_schema extension's script safe when a required extension lives in a schema that less privileged roles can write to, and do the docs state this limit?

<sub>fixable, level 2, security / privileges; messages [#18](https://www.postgresql.org/message-id/CADE6LvjnX8MbZPSTkrdoRGwf%2B_q6deNN1%2B-N_bmjdUgvFNy%2BfQ%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CAGECzQRFa-RLf2FMsL51TqXbtrbJjtC%3DSCMo50ZO7Pf_e0O7Sg%40mail.gmail.com); `security-1`</sub>

### 3. ALTER EXTENSION SET SCHEMA flips its precondition per control file

A DBA script runs create schema app_ext followed by alter extension e set schema app_ext, which is the documented pattern today. For an owned_schema extension the second command fails because app_ext exists. The DBA cannot see from the command which behaviour applies without reading the control file or pg_extension.extownedschema.

Message #4 confirms that for owned schemas the command renames the schema and requires the target to be absent, which is the opposite of the existing rule. The same verb now has two contracts, and a flag the installing user did not choose selects between them. A committer would likely ask whether a rename is better exposed as its own form, or whether SET SCHEMA should keep its contract and report when the extension owns its schema. Message #43 shows the docs still describe only the old behaviour.

**Question:** Why should one ALTER EXTENSION SET SCHEMA command require an existing schema for some extensions and a new one for others, and not use a separate rename form?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#4](https://www.postgresql.org/message-id/CAGECzQTe3y6xwP9aMbFBCxKnQEEn3G3%3DcEDqoD5xaOwPwypd_A%40mail.gmail.com), [#43](https://www.postgresql.org/message-id/CAHWVJhERrZ0df6L8GaOL9RMi8%2BoTm%3DH0STa-XL-iD2pjjqPcfw%40mail.gmail.com); `standards-compat-2`</sub>

### 4. Schema comment survives pg_upgrade but is lost on dump/restore

A user adds a comment on the extension's owned schema and moves the database with pg_dump and pg_restore. The comment is gone in the new database. The same cluster upgraded with pg_upgrade keeps it, so the two migration paths give different results.

Message #42 shows the ACL is dumped but the COMMENT is not, because the schema is now an extension member. A schema comment that survived a dump before this patch no longer does. The thread does not say which fix is planned.

**Question:** Will pg_dump emit the owned schema's comment as it does its ACL, so dump/restore and pg_upgrade give the same result?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#42](https://www.postgresql.org/message-id/179044614455.109724.18419741920404710654%40gmail.com); `standards-compat-4`</sub>

### 5. Two permanent models of extension schema, with no path between them

An author of an existing extension such as pg_cron sets owned_schema = true in a new release. Clusters that already have the extension installed keep the old layout, because the control file option is read only at CREATE EXTENSION. The author either keeps both layouts or writes direct catalog updates in the update script, which pg_dump and pg_upgrade must then handle correctly.

Message #16 says the feature serves only new extensions, and the only migration route it offers is direct catalog changes in update scripts. pg_extension.extownedschema, the changed signature of binary_upgrade_create_empty_extension, and two meanings of ALTER EXTENSION SET SCHEMA (move objects, or rename the schema) all become permanent. Messages #4 and #21 already float making owned_schema the default, which would be a second compatibility break later. A committer would likely ask for the migration design before the format is fixed.

**Question:** How does an installed extension switch to an owned schema without direct catalog changes, and would that route need different catalog or dump support than v12?

<sub>fixable, level 1b, maintenance burden ("who maintains this in 10 years"); messages [#4](https://www.postgresql.org/message-id/CAGECzQTe3y6xwP9aMbFBCxKnQEEn3G3%3DcEDqoD5xaOwPwypd_A%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/CAGECzQS02M6YPDXemo36tShO-ZYObjqnyTJyVttua1PGyN4xRw%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CAGECzQRFa-RLf2FMsL51TqXbtrbJjtC%3DSCMo50ZO7Pf_e0O7Sg%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Will CREATE EXTENSION reject a prerequisite extension that lands in an owned schema, or will pg_dump and binary upgrade keep every pg_depend row for that layout?
2. What keeps an owned_schema extension's script safe when a required extension lives in a schema that less privileged roles can write to, and will the docs state this limit?
3. How does an installed extension switch to an owned schema without direct catalog changes, and does that route fit the v12 catalog and dump design?
4. Why should ALTER EXTENSION SET SCHEMA require an absent schema for owned_schema extensions and an existing one otherwise, and not use a separate rename form?
5. Will pg_dump emit the comment on an owned schema as it emits its ACL, so that dump and restore match pg_upgrade?

## Touch points

- **on-disk format / pg_upgrade:** The patch changes the signature of binary_upgrade_create_empty_extension, and message #43 reports a lost pg_depend row after pg_upgrade.
- **catalog changes:** The patch adds the column pg_extension.extownedschema and makes the schema a member of the extension.
- **ABI / extension API:** The patch adds the control file option owned_schema for extension authors and changes src/include/commands/extension.h.
- **SQL-standard conflict or non-standard syntax:** CREATE EXTENSION ... SCHEMA and ALTER EXTENSION ... SET SCHEMA change their precondition when owned_schema is true.
- **pg_dump / restore round-trip:** pg_dump must order the owned schema and its extension; message #43 shows a dump that does not restore.
- **security / privileges:** The feature exists to keep less privileged roles out of the search_path of extension scripts run as superuser.
- **maintenance burden ("who maintains this in 10 years"):** Every later change to extension dumping or binary upgrade must handle a schema that is an extension member.

## Who would commit it, and when

- Owner: qualified committer engaged. Committers took part in the design discussion (messages #7, #12, #23, #30), but none has said they will commit it. The owner needs to know extension dependencies, pg_dump ordering and binary upgrade.
- Priority: fix design first; audience a sizeable minority; review cost medium. Extension authors and managed services need this, as message #2 describes, and costs fall only on extension DDL, pg_dump and pg_upgrade. The open cycle from message #43 and the search_path of required extensions need a design decision before more code review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
