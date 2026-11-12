# Design pre-review: Add pg_get_table_ddl() to reconstruct CREATE TABLE statements

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6837](https://commitfest.postgresql.org/patch/6837/) (Needs review); authors: Akshay Joshi (akshay.joshi)
- Thread: [\[PATCH\] Add pg_get_table_ddl() to reconstruct CREATE TABLE statements](https://www.postgresql.org/message-id/flat/CANxoLDfjQnhM%3DE6JSyYo9s9OdjqoN8s_3wE5yL%3DkaDu_X8j-dA%40mail.gmail.com), started 2026-06-03
- Messages read: 73 of 73
- Patch prose read: v26-0001-Add-pg_get_table_ddl-to-reconstruct-CREATE-TABLE.patch, v1-0001-Add-pg_get_table_ddl-to-reconstruct-CREATE-TABLE.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has shown interest from several users, so level 1b does not stop the review. The design does: prior-attempts-1 shows the signature follows sibling functions that were reverted pending a redesign, and the thread has no answer on how this patch follows it. standards-compat-1 shows that v26 changes pg_get_constraintdef() output and breaks pg_dump --no-tablespaces. maintenance-fit-1 adds that no in-tree test keeps this function in sync with pg_dump.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 73 messages and no code were read, and the content of the redesign thread is not in the material. The permission check, the search_path reset on error and the full set of changed ruleutils.c helpers could not be checked.

## The proposal, restated

**Problem.** Postgres has no server-side function that returns the full DDL for a table. The author extends the pg_get_database/role/tablespace_ddl family to tables, for tools and for users who clone schemas, for example in a multi-tenant setup.

**Proposed design.** A set-returning function returns the CREATE TABLE statement and the follow-up ALTER TABLE, CREATE INDEX, CREATE RULE and CREATE STATISTICS statements, one per row. It calls existing ruleutils.c deparse helpers plus new thin wrappers, and omits any clause whose value equals the default. Named boolean parameters control formatting, and only_kinds or except_kinds arrays filter which object classes it emits.

**User-visible surface.**
- SQL function pg_get_table_ddl(relation regclass, pretty, owner, tablespace, schema_qualified boolean, only_kinds text[], except_kinds text[]) returning setof text
- Kind vocabulary: table, index, primary_key, unique, check, foreign_key, exclusion, rule, statistics, trigger, policy, rls, replica_identity, partition
- schema_qualified parameter, which narrows search_path to pg_catalog or to the table's own schema during deparse
- Changed output of pg_get_constraintdef(): EXCLUDE constraints now include USING INDEX TABLESPACE (reported in message #73)

**What the author claims.**
- Reusing the ruleutils.c helpers avoids new pg_get_index_ddl or pg_get_constraint_ddl wrappers.
- A pg_regress test with exact .out diffs checks a deparse function better than TAP.
- Sharing code with pg_dump's dumpTableSchema is impractical; pg_dump could later call this function instead.
- A reviewer's round-trip against pg_dump on v25 found 509 of 563 regression tables identical, with the rest by design or deferred.

**Thread state.** Zsolt Parragi, Rui Zhao, Kyotaro Horiguchi, Chao Li and Marcos Pegoraro reviewed through v26, and Rui reports no open items. Marcos then pointed out that the sibling pg_get_*_ddl commits were reverted pending a redesign, and Henson Choi reported a pg_dump --no-tablespaces regression from the changed pg_get_constraintdef(); the author has answered neither.

## 2. Prior attempts

### psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2664/](https://commitfest.postgresql.org/patch/2664/)
- **Thread:** [https://www.postgresql.org/message-id/flat/191ef0d76e30a36f7d6326a3ce58bd0e@postgrespro.ru](https://www.postgresql.org/message-id/flat/191ef0d76e30a36f7d6326a3ce58bd0e@postgrespro.ru)
- **Date:** decided 2020-11-24; started 2020-07-28
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P14 put CREATE output for tables and other objects in psql commands. This proposal is a server-side SQL function. P14's reason is not recorded.

### Support logical replication of DDL commands (related)

- **Link:** [https://commitfest.postgresql.org/patch/3595/](https://commitfest.postgresql.org/patch/3595/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com)
- **Date:** decided 2024-03-28; started 2022-02-21
- **Outcome:** Withdrawn (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier objected that the hand-written deparser, with its one-to-one mapping of node enums to hardcoded text, makes every change to the parse node structures of replicated DDL also require a change here. He asked for generated code instead, and the authors argued that much of the deparsing cannot be generated. Work stopped after July 2023, and in March 2024 Amit Kapila said the entry should be closed because it still needs a lot of work.
- **Reason (quote):** "As presented, this stuff has an extremely high maintenance cost long-term because it requires anybody doing a change in the parsed tree structures of the DDLs replicated to also change the code paths introduced by this patch set." ([message](https://www.postgresql.org/message-id/ZIgf3qBvFoTsMhIc%40paquier.xyz))
- **What is different this time:** P4 deparsed DDL commands from parse trees for replication. This proposal deparses catalog state for one table, but it is also a hand-written deparser that must follow every DDL change.

### pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled (related)

- **Link:** [https://commitfest.postgresql.org/patch/3811/](https://commitfest.postgresql.org/patch/3811/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com)
- **Date:** decided 2022-11-30; started 2022-08-10
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Alvaro Herrera (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane said pg_get_indexdef is the odd one out, since most generate_relation_name calls in ruleutils.c ignore the pretty flag, and Alvaro Herrera judged the patch too small for making PRETTYFLAG_SCHEMA work consistently. Tom asked for a defined rule on which call sites qualify names, with a documented helper function. The author did not reply to these reviews, and Michael Paquier returned the patch after six weeks without a reply.
- **Reason (quote):** "In other words, I think your patch is too small." ([message](https://www.postgresql.org/message-id/20220810085850.ghwvrkcqzimif5ok%40alvherre.pgsql))
- **What is different this time:** P3 changed schema qualification in pg_get_constraintdef call by call. This patch narrows search_path during deparse and adds thin ruleutils.c wrappers.


## 3. Objections a reviewer is likely to raise

### 1. Extends a function family that was reverted pending a redesign

fixable · level 2, rubric item 6 · prior rejection, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

The patch is committed with the named-boolean signature copied from pg_get_role_ddl() and its siblings. The redesign of the reverted siblings then picks a different option model or output contract. Either pg_get_table_ddl() ships out of line with its family, or its interface changes again before or after release.

Message #39 bases the v14+ signature on a commit to pg_get_role_ddl(), pg_get_tablespace_ddl() and pg_get_database_ddl(). Message #72 reports that those commits were reverted, with a thread titled "pg_get_*_ddl() needs a redesign". The thread does not reply to message #72, and the material does not say what the redesign changes. A committer would likely ask that this patch wait for, or follow, that redesign.

Thread: [#39 (Akshay Joshi, 2026-07-01)](https://www.postgresql.org/message-id/CANxoLDdVEjCCKch1O-5J6ghw89b2fkWC1ADPYTUwxu8WB-hQbw%40mail.gmail.com), [#72 (Marcos Pegoraro, 2026-09-23)](https://www.postgresql.org/message-id/CAB-JLwa%2BJ-sAjirhbf5j2sJLO%2B6FuR_Ok2b2UMQVEUuBmCYjfQ%40mail.gmail.com)

**Question:** Which parts of this interface depend on the reverted sibling design, and how does the patch follow the outcome of the redesign thread?

Counter-check: held. Message #39 ties the signature to the sibling functions, and message #72 reports those commits were reverted pending a redesign. The thread has no reply, so the material offers nothing to refute this with.

### 2. Changed pg_get_constraintdef() output breaks pg_dump --no-tablespaces

fixable · level 2, rubric item 12 · backpatchability, pg_dump / restore round-trip, psql UX conventions · `standards-compat-1`

An admin dumps a database with an EXCLUDE constraint in tablespace xts using `pg_dump --no-tablespaces`, then restores it where xts does not exist. The ADD CONSTRAINT fails with `tablespace "xts" does not exist` and the constraint is lost. psql \d also shows the tablespace twice.

Message #73 shows that the patch changed the shared pg_get_constraintdef_worker(), so every existing caller now sees a different result: pg_dump, pg_restore, psql \d, and ALTER COLUMN TYPE rebuilds. The proposal does not say whether this change is intended outside pg_get_table_ddl(). The thread has no answer to message #73. A committer would likely ask that existing SQL-visible output stay byte-identical unless a separate, justified change is proposed.

Thread: [#73 (Henson Choi, 2026-10-02)](https://www.postgresql.org/message-id/CAAAe_zD5LSqw29MLYcQrDD-9v5q7me1d5KXpxi3td0J6aXg7jg%40mail.gmail.com)

**Question:** Can the tablespace clause be limited to pg_get_table_ddl()'s own wrapper, with pg_get_constraintdef() output unchanged for all existing callers?

Counter-check: held. Message #73 gives a reproduction on v26 and names the affected callers. The author has not answered, and the commit message does not list a change to pg_get_constraintdef() output.

### 3. Second copy of pg_dump's table logic with no in-tree sync check

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), pg_dump / restore round-trip · `maintenance-fit-1`

A later release adds a new table property, and the commit updates pg_dump's dumpTableSchema but not ddlutils.c. pg_get_table_ddl() then emits DDL that replays without error but loses the property. Users who clone schemas with it find the difference only when behaviour changes in production.

Messages #4 to #70 show about 25 rounds in which reviewers found silent fidelity gaps by diffing against pg_dump by hand. Rui's round-trip check (messages #51, #55, #64, #70) is the only thing that caught them, and it is not part of the patch. Kyotaro raised the overlap in message #42. The author's answer in message #44 that pg_dump could call this later leaves two implementations in the meantime. [P4] was stopped over the same cost of a hand-written deparser that every DDL change must also update.

**Precedents.**
- [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (Withdrawn, 2024-03-28); thread: [link](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com)

Thread: [#42 (Kyotaro Horiguchi, 2026-07-02)](https://www.postgresql.org/message-id/20260702.111715.277646207293239004.horikyota.ntt%40gmail.com), [#44 (Akshay Joshi, 2026-07-02)](https://www.postgresql.org/message-id/CANxoLDceK1w3DVbCPyricQuVKjx1qeKoWUKqyx5uoMQ8%3D6yguQ%40mail.gmail.com), [#51 (Rui Zhao, 2026-07-07)](https://www.postgresql.org/message-id/CAHWVJhFQWH5wrjxKtHdeJa0A2of%2BfZJUhSJFGWhw%3DJsxz_ZBnA%40mail.gmail.com), [#55 (Rui Zhao, 2026-07-10)](https://www.postgresql.org/message-id/CAHWVJhH4A_RRarJu5Z6Sgh4y4QKyoTAjyU7Ea-Hj0s%3Dr%2B8T%3DhA%40mail.gmail.com), [#64 (Rui Zhao, 2026-07-24)](https://www.postgresql.org/message-id/CAHWVJhGBAtvugUF5Na%2BfLz%3Dpmeu6DzQrj42UjLz_OY2sKcAjMQ%40mail.gmail.com), [#70 (Rui Zhao, 2026-09-08)](https://www.postgresql.org/message-id/CAHWVJhHbW6QLe0wVdVFFt519aqAcHfGwYSe1m0Rp0TK0icCh2Q%40mail.gmail.com)

**Question:** Which in-tree test fails when a new table feature is added to pg_dump but not to pg_get_table_ddl()?

Counter-check: held. Message #44 defers the pg_dump switch, so two generators remain, and the round-trip check that found the gaps is not in the patch. [P4] concerns a parse-tree deparser for replication, but its recorded maintenance reason applies here too.

### 4. Single all-or-nothing patch mixes helper changes with the new function

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-4`

A committer reviewing the 320 KB patch focuses on ddlutils.c and misses a behaviour change in a ruleutils.c or tablecmds.c helper. The change ships and breaks an unrelated caller, as message #73 found for pg_get_constraintdef(). Nobody notices until a user's restore fails.

v26 changes ruleutils.c, tablecmds.c and pg_inherits.c, and adds the new function, all in one commit. The new wrappers (pg_get_indexdef_ddl, pg_get_constraintdef_body and others) and the exported helpers could go first as separate commits, each with tests showing that existing output does not change. That split would have exposed the regression in message #73 by itself. [P3] was returned partly because the qualification rules in ruleutils.c call sites were not defined.

**Precedents.**
- [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (Returned with feedback, 2022-11-30); thread: [link](https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com)

Thread: [#36 (Akshay Joshi, 2026-06-29)](https://www.postgresql.org/message-id/CANxoLDcAy%2B46QesKdK95XFM3z9eT6PsiJ4Tx_GSQmtbg1O7Evw%40mail.gmail.com), [#73 (Henson Choi, 2026-10-02)](https://www.postgresql.org/message-id/CAAAe_zD5LSqw29MLYcQrDD-9v5q7me1d5KXpxi3td0J6aXg7jg%40mail.gmail.com)

**Question:** Can the ruleutils.c and tablecmds.c changes be split into a preparatory patch whose tests show no change for existing callers?

Counter-check: held. v26 is one patch that touches ruleutils.c, tablecmds.c and pg_inherits.c, and message #73 shows a shared-helper change reached existing callers unnoticed through 26 versions. [P3] is about qualification rules and must not be used as support.

### 5. Partial output where pg_dump --schema-only -t already gives complete DDL

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved) · `already-exists-2`

A user clones a tenant schema with pg_get_table_ddl() as Marcos describes in message #3. Triggers, policies, grants, comments and the sequences behind serial columns are missing. Inserts fail on the missing sequence or bypass the trigger, and the user notices only in production.

pg_dump --schema-only -t and -n already produce complete, replayable table DDL, including the objects this function leaves out by design (messages #52, #60). What the proposal adds is a call from SQL without a client tool, and the commit message advertises a "full definition" that the function does not deliver. A committer would likely ask whether SQL-level access to a subset of pg_dump's output is worth a permanent second implementation, given that users who need complete DDL still need pg_dump.

Thread: [#3 (Marcos Pegoraro, 2026-06-08)](https://www.postgresql.org/message-id/CAB-JLwZXk95KnOs57fe2d7zaOYf5aAAY2kVF5U7cXYXJmZ9jBg%40mail.gmail.com), [#52 (Akshay Joshi, 2026-07-08)](https://www.postgresql.org/message-id/CANxoLDfwVcVgFJ7C6W4DUYGt2_r2w81%3DQ-L8jtiTHE%3DPiFEN0A%40mail.gmail.com), [#60 (Akshay Joshi, 2026-07-14)](https://www.postgresql.org/message-id/CANxoLDdG-YPak2O-v-T2gbB7uw7N_0HgAcfm%2BAt2uVX3dwZ%3DGg%40mail.gmail.com), [#64 (Rui Zhao, 2026-07-24)](https://www.postgresql.org/message-id/CAHWVJhGBAtvugUF5Na%2BfLz%3Dpmeu6DzQrj42UjLz_OY2sKcAjMQ%40mail.gmail.com)

**Question:** Which users cannot run pg_dump --schema-only -t, and is the subset this function emits enough for them without pg_dump?

Counter-check: held. Messages #3, #15 and #35 show interest from independent users, which lowers the weight of this objection. But the thread never says why pg_dump --schema-only -t does not serve them, and the function omits objects pg_dump emits.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds a pg_proc.dat entry for pg_get_table_ddl(). |
| ABI / extension API | **yes** | The patch changes ruleutils.h, tablecmds.h and pg_inherits.h and adds new deparse wrappers. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection prior-attempts-1: Extends a function family that was reverted pending a redesign |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | The changed pg_get_constraintdef() output reaches pg_dump and breaks --no-tablespaces for EXCLUDE constraints (message #73). |
| psql UX conventions | **yes** | Message #73 reports that psql \d shows the tablespace of an EXCLUDE constraint twice. |
| security / privileges | **yes** | The function narrows search_path during deparse and applies its own permission check on the relation. |
| backpatchability | **yes** | see objection standards-compat-1: Changed pg_get_constraintdef() output breaks pg_dump --no-tablespaces |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second table DDL generator in ddlutils.c must follow every future table feature that pg_dump learns. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | pg_dump --schema-only -t already produces table DDL, including objects this function omits. |

## 5. Questions the author must answer before anyone reviews code

1. Which parts of the v26 signature and option handling depend on the reverted pg_get_*_ddl() design, and how will the patch follow the redesign thread's outcome?
2. Can the USING INDEX TABLESPACE clause be limited to pg_get_table_ddl()'s own wrapper, so that pg_get_constraintdef() output stays unchanged for pg_dump, psql and ALTER TABLE?
3. Which in-tree test fails when a new table feature is added to pg_dump's dumpTableSchema but not to pg_get_table_ddl()?
4. Which users cannot run pg_dump --schema-only -t, and is the subset this function emits enough for them without triggers, policies, owned sequences and grants?
5. Why does the function accept the trigger and policy kinds as no-ops today instead of raising an error until they produce output?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows ruleutils.c and pg_dump, and who follows the pg_get_*_ddl() redesign thread. No committer speaks in the visible thread.
- **Liability:** The owner signs for a permanent SQL-callable interface, for changed ruleutils.c helpers shared with pg_dump and psql, and for keeping a second table DDL generator correct in every later release.
- **Author follow-up:** The author posted 26 versions through message #71 in response to review. Messages #72 and #73 have no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Tool authors and users who clone schemas asked for this in the thread, but the interface follows a function family that was reverted for redesign. The patch is about 320 KiB in one commit and changes ruleutils.c and tablecmds.c helpers that existing callers share.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-3` (fixable, level 2) Reserved trigger and policy kinds change default output later. Not listed: not among the strongest objections.
- `standards-compat-4` (fixable, level 2) schema_qualified adds a qualification rule unlike other deparse functions. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-2` Second hand-written table deparser with nothing to keep it in sync. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same point and names the missing in-tree check more directly.
- `standards-compat-2` Signature follows a sibling family that was reverted for redesign. Refuted (not specific): Duplicate of prior-attempts-1.
- `replication-ondisk-1` Shared pg_get_constraintdef() change alters pg_dump output and index placement. Refuted (not specific): Duplicate of standards-compat-1. Both rest on message #73 and ask for the same confinement of the tablespace clause.
- `maintenance-fit-2` Shared ruleutils change breaks pg_dump --no-tablespaces for EXCLUDE. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-3` Interface built on a function family that was reverted for redesign. Refuted (not specific): Duplicate of prior-attempts-1.
- `already-exists-1` Second table-DDL generator alongside pg_dump's dumpTableSchema. Refuted (not specific): Duplicate of maintenance-fit-1.
- `demand-1` Use case not separated from what pg_dump --schema-only -t already gives. Refuted (not specific): Duplicate of already-exists-2.
- `demand-2` Every future table feature must also update a second deparser. Refuted (not specific): Duplicate of maintenance-fit-1. The who-pays framing adds no separate failure scenario.

### Review angles

- prior-attempts: 2 objection(s). No precedent in the list rejects a server-side table DDL function. The schema-qualification concern from P3 was settled in the thread.
- standards-compat: 4 objection(s). Objections found; see list.
- replication-ondisk: 1 objection(s). Apart from the pg_get_constraintdef() side effect, the proposal is a read-only SQL function. It adds a pg_proc entry and no on-disk format, WAL, replication or wire protocol state.
- security: 0 objection(s). No security objection found. Everything the function emits (columns, defaults, reloptions, constraints, indexes, rules, statistics) is readable today through world-readable catalogs and the existing pg_get_*def functions, and the proposal does not emit policies, grants or security labels. Partition-tree recursion has check_stack_depth (message #65), and the default schema_qualified => true is the safe setting. Not checkable without code: which aclcheck the function applies (message #70 mentions aclcheck_error) and whether narrowing search_path during deparse is undone on error.
- maintenance-fit: 4 objection(s). Objections listed above.
- already-exists: 2 objection(s). Two objections from this angle: the mechanism duplicates pg_dump, and pg_dump already covers the need more completely.
- demand: 2 objection(s). Some demand exists: a production interest in message #35, a multi-tenant cloning workflow in messages #3 and #15, and existing sibling pg_get_*_ddl functions. So the objections ask for more evidence and do not claim the user group is empty.

### Precedents retrieved

- P1: [Fix NULL dereference in pg_get_database_ddl()](https://commitfest.postgresql.org/patch/6669/) (cf, Withdrawn, 2026-04-13) - judged not relevant; reason summarised from its thread
- P2: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P3: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - used in the review; reason summarised from its thread
- P4: [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (cf, Withdrawn, 2024-03-28) - used in the review; reason summarised from its thread
- P5: [Add callback table access method to reset filenode when dropping relation](https://commitfest.postgresql.org/patch/3073/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P6: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P7: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P8: [xpath_table needs to be implemented/implementable to get rid of contrib/xml2 \[1\]](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant; reason summarised from its thread
- P10: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P11: [Use JOIN USING aliases in ruleutils.c](https://commitfest.postgresql.org/patch/3538/) (cf, Returned with feedback, 2022-03-25) - used in the review; reason summarised from its thread
- P12: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant
- P13: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - judged not relevant
- P14: [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://commitfest.postgresql.org/patch/2664/) (cf, Returned with feedback, 2020-11-24) - used in the review
- P15: [server side variables](https://commitfest.postgresql.org/patch/916/) (cf, Returned with feedback, 2017-02-01) - judged not relevant
- P16: [Doc: Fix definition of asynchronous replication](https://commitfest.postgresql.org/patch/6005/) (cf, Withdrawn, 2025-10-03) - judged not relevant
- P17: [Make deparsing of column defaults faster](https://commitfest.postgresql.org/patch/1657/) (cf, Returned with feedback, 2018-09-24) - judged not relevant
- P18: [Move tablespace path re-creation from the makefiles to pg_regress (test for mkdir function in the backend)](https://commitfest.postgresql.org/patch/3067/) (cf, Withdrawn, 2021-04-16) - judged not relevant
- P19: [Parallel Inserts in CREATE TABLE AS](https://commitfest.postgresql.org/patch/2841/) (cf, Withdrawn, 2021-03-20) - judged not relevant
- P20: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant
- P21: [Push down time-related SQLValue functions to foreign server](https://commitfest.postgresql.org/patch/3289/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P22: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P23: [Reduce the log spam by excluding certain SQLSTATEs](https://commitfest.postgresql.org/patch/4964/) (cf, Withdrawn, 2025-04-03) - judged not relevant
- P24: [pg_dumpall --exclude-database option](https://commitfest.postgresql.org/patch/1743/) (cf, Returned with feedback, 2018-10-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 183 s; cost reported by the CLI: $6.40
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 128 text field(s) checked, 4 finding(s), 4 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:05:02.475Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
