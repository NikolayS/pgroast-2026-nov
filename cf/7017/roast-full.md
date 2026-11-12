# Design pre-review: Soft DROP TABLE, recoverable drops for PostgreSQL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7017](https://commitfest.postgresql.org/patch/7017/) (Needs review); authors: Surya Poondla (spoondla)
- Thread: [\[DESIGN\] Soft DROP TABLE, recoverable drops for PostgreSQL](https://www.postgresql.org/message-id/flat/CAOVWO5o3TQHdEb_%2Bagn0UFHmZDN19q5cxExX3Dc_5W%2Bb0Bx_mQ%40mail.gmail.com), started 2026-06-03
- Messages read: 11 of 11
- Patch prose read: 0001-WIP-pgtrashcan-contrib-extension.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The review stops at level 1b on already-exists-1. The patch touches only contrib and doc wiring and builds with PGXS, so the same code works as an outside extension. Message #6 asked why it must be in contrib and has no direct answer. The goal of recoverable drops is not in dispute, and the level 2 objections that survived were not evaluated.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and the function reference in pgtrashcan.sgml is truncated. Lock handling during the move, search_path in the SECURITY DEFINER functions, and any per-entry privilege checks could not be judged.

## The proposal, restated

**Problem.** An accidental drop table, often from automation run against the wrong cluster, destroys data that can only come back through point-in-time recovery or a logical restore. The author wants a lighter first line of recovery for operators, like Oracle's recycle bin or Snowflake's UNDROP TABLE.

**Proposed design.** A new contrib extension, forked from Peter Eisentraut's 2014 pgtrashcan, uses a ProcessUtility hook to turn drop table into a move to a reserved pgtrashcan schema under an OID-derived name. A protected catalog records each dropped table, and SQL functions list, restore, and purge entries. Foreign keys, triggers, views, rules and RLS policies are dropped, and their definitions are kept as JSONB for manual rebuild.

**User-visible surface.**
- New contrib extension pgtrashcan; needs shared_preload_libraries and CREATE EXTENSION in each database.
- Reserved schema pgtrashcan with catalog table pgtrashcan._trash_catalog, holding original name, schema, OID, drop time, dropped_by and JSONB metadata.
- SQL functions pgtrashcan_list_dropped(), pgtrashcan_restore() by name or OID, and pgtrashcan_purge().
- GUC pgtrashcan.enabled (USERSET, default on) to opt out per session; GUC pgtrashcan.internal_operation (SUSET) for internal use.
- Changed DROP TABLE behaviour; on restore, NOTICE messages emit the captured definitions of dropped dependent objects.

**What the author claims.**
- Recovery from point-in-time recovery or backup is too heavy when an operator wants a table back minutes later.
- Oracle, Db2 and Snowflake offer similar recovery, which shows a common need.
- Indexes, TOAST and owned sequences move with the table and come back on restore.
- The OID-derived name keeps several drops of the same table name apart and never hits NAMEDATALEN truncation.
- A 58-test regression suite covers the extension.

**Thread state.** Zsolt Parragi, Andreas Karlsson, Amit Kapila and Álvaro Herrera asked why this is not an outside extension and noted that DROP SCHEMA ... CASCADE, DROP OWNED and DROP EXTENSION bypass the hook. Amit's last message asks for a hook at the right level, such as in performDeletion, before going further, and the author has not answered it.

## 2. Prior attempts

### Log LSN before table is dropped (related)

- **Link:** [https://commitfest.postgresql.org/patch/5386/](https://commitfest.postgresql.org/patch/5386/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CACLU5mSr+t+6K_Ww6z1fk_=jP=5-U0CEqMsHjii8uG_6UWQo-g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CACLU5mSr+t+6K_Ww6z1fk_=jP=5-U0CEqMsHjii8uG_6UWQo-g@mail.gmail.com)
- **Date:** decided 2025-07-25; started 2024-11-08
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Kirill Reshke, Jim Jones (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why the entry was returned with feedback. Reviewers asked for changes. Jim Jones found that temporary and unlogged tables also trigger the message. Kirill Reshke asked why an extension using an object access hook could not do the logging, and noted that the logged LSN can be earlier than the commit record of the drop.
- **Reason (quote):** "What stops us from logging all the same inside object access hook defined by extension?" ([message](https://www.postgresql.org/message-id/CALdSSPiRDvbuPgZKAB1%2BBDb3Nfe-i3PYkiy%3DScMZ7tU5f0toKQ%40mail.gmail.com))
- **What is different this time:** P1 logged the LSN at DROP TABLE so the table could come back through point-in-time recovery. This proposal keeps the table in a reserved schema, so no point-in-time recovery is needed.


## 3. Objections a reviewer is likely to raise

### 1. Contrib placement adds nothing the external pgtrashcan cannot do

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A committer commits contrib/pgtrashcan, a 343 KiB module that uses only the existing ProcessUtility_hook. From then on the project owns its behaviour across every DDL change, release after release. The same code could have shipped as an update to the external pgtrashcan, with no extra cost to core.

The patch touches only contrib and doc wiring, so every part of it already works as an external extension built with PGXS, as the README shows. Andreas asked in message #6 why it cannot update or fork the existing extension outside core, and that question has no answer. Amit in message #7 gives one reason for in-tree placement, coordinated review with core, but the patch does not handle pg_dump, publications or the other coupled areas he lists. A similar question about using an extension hook was raised for [P1].

**Precedents.**
- [Log LSN before table is dropped](https://commitfest.postgresql.org/patch/5386/) (Returned with feedback, 2025-07-25); thread: [link](https://www.postgresql.org/message-id/flat/CACLU5mSr+t+6K_Ww6z1fk_=jP=5-U0CEqMsHjii8uG_6UWQo-g@mail.gmail.com)

Thread: [#4 (Zsolt Parragi, 2026-07-15)](https://www.postgresql.org/message-id/CAN4CZFPW60AvbCTUhY6XPNJbcB2q1-1Zh6dhEpLqXh1fJuvYTg%40mail.gmail.com), [#6 (Andreas Karlsson, 2026-07-24)](https://www.postgresql.org/message-id/ffdee390-f977-4e75-ba86-36348ea6341d%40proxel.se), [#7 (Amit Kapila, 2026-07-25)](https://www.postgresql.org/message-id/CAA4eK1JrsMd-1H6tZ6%3Dj_NO%3DVqoj1s%3DoFb46Zd2VAqCXLKo33g%40mail.gmail.com), [#10 (surya poondla, 2026-07-27)](https://www.postgresql.org/message-id/CAOVWO5qmGUP_nw6naGL_rjnCmeVLaFkueNNwfQ39eZKF%2B4QsnQ%40mail.gmail.com)

**Question:** What does contrib placement give this module that a maintained external fork of pgtrashcan, built with PGXS on the same hook, cannot give?

Counter-check: held. The patch touches only contrib and doc wiring, and the README shows a PGXS build, so the same code works outside core. Message #6 has no direct answer, and message #10 confirms the coupled areas Amit named are not handled. [P1] must not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Trashed relation files stay on disk until a manual purge, and pg_upgrade of the reserved schema is not handled. |
| WAL / replication / standby | **yes** | Soft-dropped tables stay in publications, and pgtrashcan._trash_catalog is an ordinary table that FOR ALL TABLES publishes. |
| wire protocol | no |  |
| catalog changes | **yes** | The extension adds the reserved pgtrashcan schema and the protected table pgtrashcan._trash_catalog, keyed by OID-derived names. |
| ABI / extension API | **yes** | The module depends on ProcessUtility_hook, and message #11 asks for a hook in the deletion code instead. |
| SQL-standard conflict or non-standard syntax | **yes** | DROP TABLE becomes a move to another schema, and CASCADE also moves FK-referencing tables. |
| new GUC | **yes** | The patch adds pgtrashcan.enabled (USERSET, default on) and pgtrashcan.internal_operation (SUSET). |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump emits trashed tables as normal tables, and message #10 confirms the patch does not handle this. |
| psql UX conventions | no |  |
| security / privileges | **yes** | Restore and purge are SECURITY DEFINER functions, and trashed tables keep their grants while their RLS policies are dropped. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A 343.6 KB contrib module must follow every change to drop paths, dependent object types, publications and pg_dump. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The module is a fork of an outside extension and uses only an existing hook, so it can stay outside core. |

## 5. Questions the author must answer before anyone reviews code

1. What does contrib placement give this module that a maintained outside fork of pgtrashcan, built with PGXS on the same hook, cannot give?
2. Which core hook, existing or new, would let one interception point cover drop table, drop schema cascade, drop owned and drop extension, and could it be committed first?
3. What bounds the disk space and time held by trashed relations when a rotation job drops tables without setting pgtrashcan.enabled to off?
4. How are pgtrashcan._trash_catalog and the trashed tables kept out of FOR ALL TABLES publications, and what does pg_dump emit for them?
5. Which role may restore or purge a given trash entry, and what keeps a restored table closed until its RLS policies are rebuilt?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The owner would need to know the dependency and deletion code (performDeletion), and also logical replication and pg_dump. Reviewers in the thread asked design questions, but none has offered to take the patch.
- **Liability:** The owner signs for changed DROP TABLE behaviour in a shipped module, for as long as contrib carries it. Every new drop path or dependent object type needs a matching change, and nothing fails when one is missed.
- **Author follow-up:** The author posted the patch, answered reviewers in messages #5 and #10, and moved the entry to the next CommitFest on 2026-10-15. Message #11 has no reply yet.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A concrete need that only in-tree placement meets would change the answer, for example a new core deletion hook that the module depends on and tests. Reports from users of the outside pgtrashcan who cannot get this version outside core would also count. The level 2 design points still need answers after that.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Accidental drops are a real operator problem, and message #1 cites similar features in Oracle, Db2 and Snowflake. The 343.6 KB patch misses DROP SCHEMA ... CASCADE and DROP OWNED, and the thread asks for a hook at the right level and a reason for contrib before review continues.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Drop table ... cascade moves child tables that stock Postgres keeps. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) Dump and restore of the trash schema has no defined behaviour. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) The trash catalog and trashed tables leak into FOR ALL TABLES publications. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-3` (fixable, level 2) Trashed relation files are kept with no limit and no automatic purge. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-1` (fixable, level 2) Restore and purge have no per-table ownership check. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-2` (fixable, level 2) Dropped tables stay readable and replicated under their old grants. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) Statement-level hook cannot cover all drop paths, so contrib would freeze it. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-3` (fixable, level 2) Hand-kept list of dependent object types will drift silently. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` No reason given to move a 2014 out-of-tree extension into contrib. Refuted (not specific): Duplicate of already-exists-1, which makes the same point with more concrete support from the patch. [P1] proposed a core log message and its recorded reason is not known, so it gives no support.
- `standards-compat-3` Default-on USERSET GUC changes what DROP TABLE means. Refuted (not specific): Duplicate of replication-ondisk-3 for disk space and replication-ondisk-1 for publications. The remaining point fails because the operator opts in through shared_preload_libraries and CREATE EXTENSION, and changing DROP TABLE is the stated purpose of that choice.
- `replication-ondisk-2` OID-derived trash names do not survive a logical dump and restore. Refuted (not specific): Duplicate of standards-compat-2. Its claim about pg_extension_config_dump also cannot be checked, because the material does not show the extension script.
- `security-3` Unbounded trash retention lets any table creator fill the disk. Refuted (no failure scenario): A role that can create tables can already fill the disk without this extension, and pgtrashcan.enabled is USERSET, so that role can opt out. The remaining retention concern is a duplicate of replication-ondisk-3.
- `maintenance-fit-2` No reason given why this belongs in contrib, not outside core. Refuted (not specific): Duplicate of already-exists-1. [P1] gives no support, because its recorded reason for being returned is not known.
- `maintenance-fit-4` Default-on trash keeps disk space with no retention limit. Refuted (not specific): Duplicate of replication-ondisk-3, which states the same retention problem with the thread's own partition-rotation case.
- `already-exists-2` The core gap is a deletion hook, which this patch does not add. Refuted (not specific): Duplicate of maintenance-fit-1 for the missing deletion hook, and of already-exists-1 for the point that contrib adds little.
- `demand-1` No demand shown for contrib over the existing external extension. Refuted (not specific): Duplicate of already-exists-1 on why contrib is needed over an outside extension. On demand for the feature, the rubric counts other databases that have it as evidence, and Amit calls the use cases reasonable in message #7.

### Review angles

- prior-attempts: 1 objection(s). The thread names pgtrashcan (2014, outside core) as the earlier form of this idea in messages #1 and #2. The material gives no reason why that extension stayed out of core.
- standards-compat: 3 objection(s). Objections listed; desirability and the hook-placement question raised in message #11 are covered by other angles.
- replication-ondisk: 3 objection(s). The design adds no WAL records, on-disk format, wire messages or core ABI. Physical standbys replay ordinary catalog changes, so the objections cover only logical replication, dump and restore, and space reclamation.
- security: 3 objection(s). Objections focus on privileges and exposure only. Without the code, I could not check search_path handling in the SECURITY DEFINER functions or in the generated DDL strings.
- maintenance-fit: 4 objection(s). Objections found; see list.
- already-exists: 2 objection(s). Two objections raised from the already-exists angle.
- demand: 1 objection(s). Demand for undoing accidental drops is plausible, given the incidents in message #1 and similar features in other systems. What the material lacks is evidence that users want this module in contrib rather than as an external extension.

### Precedents retrieved

- P1: [Log LSN before table is dropped](https://commitfest.postgresql.org/patch/5386/) (cf, Returned with feedback, 2025-07-25) - used in the review; reason summarised from its thread
- P2: [Improve recovery performance of dropping database having many tablespaces](https://commitfest.postgresql.org/patch/1658/) (cf, Returned with feedback, 2018-09-30) - judged not relevant; reason summarised from its thread
- P3: [Add callback table access method to reset filenode when dropping relation](https://commitfest.postgresql.org/patch/3073/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P4: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P5: [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://commitfest.postgresql.org/patch/5992/) (cf, Returned with feedback, 2026-04-02) - judged not relevant; reason summarised from its thread
- P6: [Call RelationDropStorage() for broader range of object drops.](https://commitfest.postgresql.org/patch/1295/) (cf, Rejected, 2018-01-22) - judged not relevant; reason summarised from its thread
- P7: [Allow reads to proceed during FK/trigger drops by reducing relation-level lock from AccessExclusive to ShareRowExclusive](https://commitfest.postgresql.org/patch/6124/) (cf, Withdrawn, 2025-11-25) - judged not relevant; reason summarised from its thread
- P8: [Close race in DROP TABLESPACE on Windows](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Tablespaces) (do_not_want, Not worth doing, date unknown) - judged not relevant; reason summarised from its thread
- P9: [Prepare Transaction support for ON COMMIT DROP temporary tables](https://commitfest.postgresql.org/patch/1954/) (cf, Returned with feedback, 2019-02-01) - judged not relevant
- P10: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - judged not relevant
- P11: [DROP relation IF EXISTS Docs and Tests - Bug Fix](https://commitfest.postgresql.org/patch/2603/) (cf, Withdrawn, 2021-08-11) - judged not relevant
- P12: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P13: [Prevent extension creation in temporary schemas](https://commitfest.postgresql.org/patch/1969/) (cf, Returned with feedback, 2019-03-07) - judged not relevant
- P14: [Add CREATE SCHEMA ... LIKE support](https://commitfest.postgresql.org/patch/6481/) (cf, Returned with feedback, 2026-04-06) - judged not relevant
- P15: [Tab completion for CREATE SCHEMA AUTHORIZATION](https://commitfest.postgresql.org/patch/3252/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P16: [Identifying user-created objects](https://commitfest.postgresql.org/patch/2444/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P17: [Requiring recovery.signal or standby.signal when recovering with a backup_label](https://commitfest.postgresql.org/patch/4244/) (cf, Rejected, 2023-11-10) - judged not relevant
- P18: [improve large object handling in pg_upgrade](https://commitfest.postgresql.org/patch/3841/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P19: [Large files for relations](https://commitfest.postgresql.org/patch/4305/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P20: [Various small unrelated doc patches: plpgsql, schemas, permissions, oidvector](https://commitfest.postgresql.org/patch/4581/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P21: [recovery modules](https://commitfest.postgresql.org/patch/4082/) (cf, Withdrawn, 2024-07-19) - judged not relevant
- P22: [vacuumlo: report the number of large objects going to be removed](https://commitfest.postgresql.org/patch/2148/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P23: [psql: add schema size information to \dn+](https://commitfest.postgresql.org/patch/2010/) (cf, Withdrawn, 2019-02-25) - judged not relevant
- P24: [Add recovery_min_apply_delay_reconnect recovery option](https://commitfest.postgresql.org/patch/1325/) (cf, Returned with feedback, 2017-11-15) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 187 s; cost reported by the CLI: $2.77
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 143 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:15:24.945Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
