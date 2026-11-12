# Design pre-review: Add pg_get_table_ddl() to reconstruct CREATE TABLE statements

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6837](https://commitfest.postgresql.org/patch/6837/); [thread](https://www.postgresql.org/message-id/flat/CANxoLDfjQnhM%3DE6JSyYo9s9OdjqoN8s_3wE5yL%3DkaDu_X8j-dA%40mail.gmail.com), 73 of 73 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has shown interest from several users, so level 1b does not stop the review. The design does: prior-attempts-1 shows the signature follows sibling functions that were reverted pending a redesign, and the thread has no answer on how this patch follows it. standards-compat-1 shows that v26 changes pg_get_constraintdef() output and breaks pg_dump --no-tablespaces. maintenance-fit-1 adds that no in-tree test keeps this function in sync with pg_dump.

Not checked: Only 3 of 73 messages and no code were read, and the content of the redesign thread is not in the material. The permission check, the search_path reset on error and the full set of changed ruleutils.c helpers could not be checked.

## Prior attempts

- [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://www.postgresql.org/message-id/flat/191ef0d76e30a36f7d6326a3ce58bd0e@postgrespro.ru) (similar idea; Returned with feedback, 2020-11-24). reason not recorded Different now: P14 put CREATE output for tables and other objects in psql commands. This proposal is a server-side SQL function. P14's reason is not recorded.
- [Support logical replication of DDL commands](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com) (related; Withdrawn, 2024-03-28). "As presented, this stuff has an extremely high maintenance cost long-term because it requires anybody doing a change in the parsed tree structures of the DDLs replicated to also change the code paths introduced by this patch set." ([message](https://www.postgresql.org/message-id/ZIgf3qBvFoTsMhIc%40paquier.xyz)) Different now: P4 deparsed DDL commands from parse trees for replication. This proposal deparses catalog state for one table, but it is also a hand-written deparser that must follow every DDL change.
- [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com) (related; Returned with feedback, 2022-11-30). "In other words, I think your patch is too small." ([message](https://www.postgresql.org/message-id/20220810085850.ghwvrkcqzimif5ok%40alvherre.pgsql)) Different now: P3 changed schema qualification in pg_get_constraintdef call by call. This patch narrows search_path during deparse and adds thin ruleutils.c wrappers.

## Objections a reviewer is likely to raise

### 1. Extends a function family that was reverted pending a redesign

The patch is committed with the named-boolean signature copied from pg_get_role_ddl() and its siblings. The redesign of the reverted siblings then picks a different option model or output contract. Either pg_get_table_ddl() ships out of line with its family, or its interface changes again before or after release.

Message #39 bases the v14+ signature on a commit to pg_get_role_ddl(), pg_get_tablespace_ddl() and pg_get_database_ddl(). Message #72 reports that those commits were reverted, with a thread titled "pg_get_*_ddl() needs a redesign". The thread does not reply to message #72, and the material does not say what the redesign changes. A committer would likely ask that this patch wait for, or follow, that redesign.

**Question:** Which parts of this interface depend on the reverted sibling design, and how does the patch follow the outcome of the redesign thread?

<sub>fixable, level 2, prior rejection; messages [#39](https://www.postgresql.org/message-id/CANxoLDdVEjCCKch1O-5J6ghw89b2fkWC1ADPYTUwxu8WB-hQbw%40mail.gmail.com), [#72](https://www.postgresql.org/message-id/CAB-JLwa%2BJ-sAjirhbf5j2sJLO%2B6FuR_Ok2b2UMQVEUuBmCYjfQ%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Changed pg_get_constraintdef() output breaks pg_dump --no-tablespaces

An admin dumps a database with an EXCLUDE constraint in tablespace xts using `pg_dump --no-tablespaces`, then restores it where xts does not exist. The ADD CONSTRAINT fails with `tablespace "xts" does not exist` and the constraint is lost. psql \d also shows the tablespace twice.

Message #73 shows that the patch changed the shared pg_get_constraintdef_worker(), so every existing caller now sees a different result: pg_dump, pg_restore, psql \d, and ALTER COLUMN TYPE rebuilds. The proposal does not say whether this change is intended outside pg_get_table_ddl(). The thread has no answer to message #73. A committer would likely ask that existing SQL-visible output stay byte-identical unless a separate, justified change is proposed.

**Question:** Can the tablespace clause be limited to pg_get_table_ddl()'s own wrapper, with pg_get_constraintdef() output unchanged for all existing callers?

<sub>fixable, level 2, backpatchability; messages [#73](https://www.postgresql.org/message-id/CAAAe_zD5LSqw29MLYcQrDD-9v5q7me1d5KXpxi3td0J6aXg7jg%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Second copy of pg_dump's table logic with no in-tree sync check

A later release adds a new table property, and the commit updates pg_dump's dumpTableSchema but not ddlutils.c. pg_get_table_ddl() then emits DDL that replays without error but loses the property. Users who clone schemas with it find the difference only when behaviour changes in production.

Messages #4 to #70 show about 25 rounds in which reviewers found silent fidelity gaps by diffing against pg_dump by hand. Rui's round-trip check (messages #51, #55, #64, #70) is the only thing that caught them, and it is not part of the patch. Kyotaro raised the overlap in message #42. The author's answer in message #44 that pg_dump could call this later leaves two implementations in the meantime. [P4] was stopped over the same cost of a hand-written deparser that every DDL change must also update.

Precedent: [Support logical replication of DDL commands](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com) (Withdrawn, 2024-03-28)

**Question:** Which in-tree test fails when a new table feature is added to pg_dump but not to pg_get_table_ddl()?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#42](https://www.postgresql.org/message-id/20260702.111715.277646207293239004.horikyota.ntt%40gmail.com), [#44](https://www.postgresql.org/message-id/CANxoLDceK1w3DVbCPyricQuVKjx1qeKoWUKqyx5uoMQ8%3D6yguQ%40mail.gmail.com), [#51](https://www.postgresql.org/message-id/CAHWVJhFQWH5wrjxKtHdeJa0A2of%2BfZJUhSJFGWhw%3DJsxz_ZBnA%40mail.gmail.com), [#55](https://www.postgresql.org/message-id/CAHWVJhH4A_RRarJu5Z6Sgh4y4QKyoTAjyU7Ea-Hj0s%3Dr%2B8T%3DhA%40mail.gmail.com), [#64](https://www.postgresql.org/message-id/CAHWVJhGBAtvugUF5Na%2BfLz%3Dpmeu6DzQrj42UjLz_OY2sKcAjMQ%40mail.gmail.com), [#70](https://www.postgresql.org/message-id/CAHWVJhHbW6QLe0wVdVFFt519aqAcHfGwYSe1m0Rp0TK0icCh2Q%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. Single all-or-nothing patch mixes helper changes with the new function

A committer reviewing the 320 KB patch focuses on ddlutils.c and misses a behaviour change in a ruleutils.c or tablecmds.c helper. The change ships and breaks an unrelated caller, as message #73 found for pg_get_constraintdef(). Nobody notices until a user's restore fails.

v26 changes ruleutils.c, tablecmds.c and pg_inherits.c, and adds the new function, all in one commit. The new wrappers (pg_get_indexdef_ddl, pg_get_constraintdef_body and others) and the exported helpers could go first as separate commits, each with tests showing that existing output does not change. That split would have exposed the regression in message #73 by itself. [P3] was returned partly because the qualification rules in ruleutils.c call sites were not defined.

Precedent: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com) (Returned with feedback, 2022-11-30)

**Question:** Can the ruleutils.c and tablecmds.c changes be split into a preparatory patch whose tests show no change for existing callers?

<sub>fixable, level 2, ABI / extension API; messages [#36](https://www.postgresql.org/message-id/CANxoLDcAy%2B46QesKdK95XFM3z9eT6PsiJ4Tx_GSQmtbg1O7Evw%40mail.gmail.com), [#73](https://www.postgresql.org/message-id/CAAAe_zD5LSqw29MLYcQrDD-9v5q7me1d5KXpxi3td0J6aXg7jg%40mail.gmail.com); `maintenance-fit-4`</sub>

### 5. Partial output where pg_dump --schema-only -t already gives complete DDL

A user clones a tenant schema with pg_get_table_ddl() as Marcos describes in message #3. Triggers, policies, grants, comments and the sequences behind serial columns are missing. Inserts fail on the missing sequence or bypass the trigger, and the user notices only in production.

pg_dump --schema-only -t and -n already produce complete, replayable table DDL, including the objects this function leaves out by design (messages #52, #60). What the proposal adds is a call from SQL without a client tool, and the commit message advertises a "full definition" that the function does not deliver. A committer would likely ask whether SQL-level access to a subset of pg_dump's output is worth a permanent second implementation, given that users who need complete DDL still need pg_dump.

**Question:** Which users cannot run pg_dump --schema-only -t, and is the subset this function emits enough for them without pg_dump?

<sub>fixable, level 1b, desirability; messages [#3](https://www.postgresql.org/message-id/CAB-JLwZXk95KnOs57fe2d7zaOYf5aAAY2kVF5U7cXYXJmZ9jBg%40mail.gmail.com), [#52](https://www.postgresql.org/message-id/CANxoLDfwVcVgFJ7C6W4DUYGt2_r2w81%3DQ-L8jtiTHE%3DPiFEN0A%40mail.gmail.com), [#60](https://www.postgresql.org/message-id/CANxoLDdG-YPak2O-v-T2gbB7uw7N_0HgAcfm%2BAt2uVX3dwZ%3DGg%40mail.gmail.com), [#64](https://www.postgresql.org/message-id/CAHWVJhGBAtvugUF5Na%2BfLz%3Dpmeu6DzQrj42UjLz_OY2sKcAjMQ%40mail.gmail.com); `already-exists-2`</sub>

## Questions for the author

1. Which parts of the v26 signature and option handling depend on the reverted pg_get_*_ddl() design, and how will the patch follow the redesign thread's outcome?
2. Can the USING INDEX TABLESPACE clause be limited to pg_get_table_ddl()'s own wrapper, so that pg_get_constraintdef() output stays unchanged for pg_dump, psql and ALTER TABLE?
3. Which in-tree test fails when a new table feature is added to pg_dump's dumpTableSchema but not to pg_get_table_ddl()?
4. Which users cannot run pg_dump --schema-only -t, and is the subset this function emits enough for them without triggers, policies, owned sequences and grants?
5. Why does the function accept the trigger and policy kinds as no-ops today instead of raising an error until they produce output?

## Touch points

- **catalog changes:** The patch adds a pg_proc.dat entry for pg_get_table_ddl().
- **ABI / extension API:** The patch changes ruleutils.h, tablecmds.h and pg_inherits.h and adds new deparse wrappers.
- **SQL-standard conflict or non-standard syntax:** see objection prior-attempts-1: Extends a function family that was reverted pending a redesign
- **pg_dump / restore round-trip:** The changed pg_get_constraintdef() output reaches pg_dump and breaks --no-tablespaces for EXCLUDE constraints (message #73).
- **psql UX conventions:** Message #73 reports that psql \d shows the tablespace of an EXCLUDE constraint twice.
- **security / privileges:** The function narrows search_path during deparse and applies its own permission check on the relation.
- **backpatchability:** see objection standards-compat-1: Changed pg_get_constraintdef() output breaks pg_dump --no-tablespaces
- **maintenance burden ("who maintains this in 10 years"):** A second table DDL generator in ddlutils.c must follow every future table feature that pg_dump learns.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** pg_dump --schema-only -t already produces table DDL, including objects this function omits.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows ruleutils.c and pg_dump, and who follows the pg_get_*_ddl() redesign thread. No committer speaks in the visible thread.
- Priority: needs design rethink; audience a sizeable minority; review cost high. Tool authors and users who clone schemas asked for this in the thread, but the interface follows a function family that was reverted for redesign. The patch is about 320 KiB in one commit and changes ruleutils.c and tablecmds.c helpers that existing callers share.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 8 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
