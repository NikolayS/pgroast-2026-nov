# Design pre-review: Disallow whole-row index references with virtual generated columns?

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6749](https://commitfest.postgresql.org/patch/6749/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [Disallow whole-row index references with virtual generated columns?](https://www.postgresql.org/message-id/flat/CAJTYsWVmfHyooKCj30suW8%3DY6b8be7WcyYmuAKMD%3D2mTPFpDkg%40mail.gmail.com), started 2026-05-08
- Messages read: 8 of 8
- Patch prose read: v1-0001-Disallow-whole-row-index-references-with-virtual-generated-columns.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection disputes the goal: message #3 shows a unique index that keys on (1,) while SQL returns (1,2). Two fixable design gaps remain. The check runs only in DefineIndex, so `alter table ... add column ... virtual` on a table with a whole-row index reaches the same state (replication-ondisk-1). The proposal also does not say what pg_upgrade or a restore does with indexes of this kind that already exist (standards-compat-1).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check whether the check catches every form of whole-row reference, or whether alter table add column already rejects this case elsewhere. The error message and error code are also not shown.

## The proposal, restated

**Problem.** DefineIndex rejects virtual generated columns named directly in index expressions and predicates, but it accepts whole-row Vars that include them. Index build and maintenance evaluate against the physical heap tuple, where the virtual column is missing. So a whole-row expression index keys on (1,) while SQL sees (1,2), and a partial unique index with WHERE t IS NOT NULL can leave rows out and fail to enforce uniqueness.

**Proposed design.** The patch extends the existing check in DefineIndex to reject whole-row Vars (varattno = 0) in index expressions and predicates when the relation has any virtual generated column. Whole-row references stay allowed on relations without virtual generated columns. The author calls this a conservative stopgap until RelationGetIndexExpressions() and RelationGetIndexPredicate() support virtual generated columns.

**User-visible surface.**
- create index with a whole-row reference in an expression or a where predicate now raises an error when the table has a virtual generated column.

**What the author claims.**
- A whole-row Var on such a relation bypasses the existing restriction on indexing virtual generated columns.
- A whole-row expression index stores keys without the virtual column, for example (1,) where select returns (1,2).
- A partial unique index with a whole-row predicate can silently skip rows and leave uniqueness unenforced.
- Existing whole-row indexes on tables without virtual generated columns are not affected.

**Thread state.** Álvaro Herrera questioned a virtual-column-specific fix and doubted the use of whole-row indexes; Matthias van de Meent showed STORED and VIRTUAL behave differently and suggested whole-row IS [NOT] NULL in indexes may be unsafe in general. The CF reviewer instead points to CF 5667, which adds index support for virtual generated columns, and nobody has reviewed this patch directly.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-05-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Adding a virtual generated column later still creates the broken index

fixable · level 2, rubric item 7 · design, catalog changes · `replication-ondisk-1`

A user has `create unique index ... on t (a) where t is not null` on a table with no virtual column, which passes the new check. They then run `alter table t add column b int generated always as (a * 2) virtual`. The index now evaluates the predicate on the heap tuple without b, and uniqueness gaps appear exactly as in message #1.

The patch touches only indexcmds.c, so the check runs at DefineIndex time. The invariant the patch protects is that no index evaluates a whole-row value that is missing a virtual column. ALTER TABLE ADD COLUMN reaches the same state without passing through DefineIndex. Matthias in message #4 already noted that adding columns can change whole-row IS NOT NULL results, so this path is known to matter.

Thread: [#1 (Ayush Tiwari, 2026-05-08)](https://www.postgresql.org/message-id/CAJTYsWVmfHyooKCj30suW8%3DY6b8be7WcyYmuAKMD%3D2mTPFpDkg%40mail.gmail.com), [#3 (Ayush Tiwari, 2026-05-08)](https://www.postgresql.org/message-id/CAJTYsWV09r2OTJq690rm4CE1Qtg%3DfqDpWNo0PfHCfAbK199qqg%40mail.gmail.com), [#4 (Matthias van de Meent, 2026-05-08)](https://www.postgresql.org/message-id/CAEze2WhQWJgvjBCq0t4MDa1cscuO3NPO%3D7tt3fRJEDyLdhbvzQ%40mail.gmail.com)

**Question:** Does alter table add column of a virtual generated column fail when the table already has an index with a whole-row reference, and where is that enforced?

Counter-check: held. The patch touches only indexcmds.c, and alter table add column does not go through DefineIndex. The material does not say that this path is rejected, so the wrong-key state from message #3 stays reachable.

### 2. Existing whole-row indexes on such tables block dump restore and pg_upgrade

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, on-disk format / pg_upgrade, backpatchability · `standards-compat-1`

A Postgres 18 user has a whole-row expression or partial index on a table with a virtual generated column. pg_upgrade to 19 replays that CREATE INDEX from the schema dump, the new check rejects it, and the upgrade stops partway. A plain pg_dump restore also fails at that index, and the user learns this only during the restore.

Virtual generated columns already shipped, so these indexes can exist in the field today, and the patch is tagged Bugfix. The proposal does not mention upgrade, restore, or a pg_upgrade pre-check. Message #1 says existing whole-row indexes "should not be affected", but that covers only tables without virtual generated columns. If the fix is back-patched, a minor release would also start rejecting a dump that the same major version produced.

Thread: [#1 (Ayush Tiwari, 2026-05-08)](https://www.postgresql.org/message-id/CAJTYsWVmfHyooKCj30suW8%3DY6b8be7WcyYmuAKMD%3D2mTPFpDkg%40mail.gmail.com), [#3 (Ayush Tiwari, 2026-05-08)](https://www.postgresql.org/message-id/CAJTYsWV09r2OTJq690rm4CE1Qtg%3DfqDpWNo0PfHCfAbK199qqg%40mail.gmail.com)

**Question:** What does pg_upgrade or a restore do with an existing index of this kind, and should pg_upgrade --check report it before the upgrade starts?

Counter-check: held. Virtual generated columns shipped in 18 and DefineIndex accepts these indexes today, so they can exist in the field. The proposal does not say how pg_upgrade or a restore handles them, and the thread does not discuss it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade replays create index for existing whole-row indexes on tables with virtual generated columns, and the new check rejects them. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | see objection replication-ondisk-1: Adding a virtual generated column later still creates the broken index |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | A dump that contains such an index fails to restore at the create index statement. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CF entry is tagged Bugfix and virtual generated columns already shipped, so a back-patch would add a new create index error in a minor release. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The check must be removed or rewritten when index support for virtual generated columns arrives, as message #3 says. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does `alter table ... add column ... generated always as (...) virtual` fail when the table already has an index with a whole-row reference, and where is that enforced?
2. What do pg_upgrade and a pg_dump restore do with an existing index of this kind, and should pg_upgrade --check report it?
3. Is the fix meant for back branches, and what should users who already have such an index do?
4. If CF 5667 is committed, is this patch dropped for master and kept for released branches only?
5. Does the patch address Matthias's point that whole-row IS [NOT] NULL predicates in indexes may be unsafe on any table when columns are added?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera replied in message #2 but questioned the virtual-column-specific scope and did not offer to take the patch. The owner needs to know DefineIndex and the generated-column code.
- **Liability:** A new create index error, possibly in a released branch, and the restore and pg_upgrade failures it can cause for existing indexes. The check must be removed when index support for virtual generated columns arrives.
- **Author follow-up:** The author answered the first review the same day in message #3, with new test cases.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 4.3 KB in indexcmds.c plus regression tests, and it affects only whole-row indexes on tables with virtual generated columns. It fixes a silent uniqueness failure, but the alter table path and the upgrade path need an answer first.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Adding a virtual generated column later bypasses the new check. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same alter table path under the safety item.
- `replication-ondisk-2` Existing indexes break pg_upgrade and dump restore with no prior check. Refuted (not specific): Duplicate of standards-compat-1. Its extra question about the recommended fix for users of 18 belongs with that objection.
- `maintenance-fit-1` Check at create index only; alter table add column is not covered. Refuted (not specific): Duplicate of replication-ondisk-1. The maintainability framing adds no separate failure beyond the same alter table add column gap.
- `maintenance-fit-2` Stopgap overlaps CF 5667, which removes the same restriction. Refuted (no failure scenario): The new check extends the existing virtual-column check in DefineIndex, which CF 5667 must remove anyway, so no extra hidden cleanup arises. A removable error check is not a permanent commitment, and released branches need a fix that a feature patch cannot provide.
- `maintenance-fit-3` New error can make existing dumps fail to restore. Refuted (not specific): Duplicate of standards-compat-1, which covers the same restore and pg_upgrade failure, including the back-patch case.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] shares only the generated-column topic, and its reason, that the COPY WHERE clause filters input data while generated columns belong to the target table, does not apply to index definitions. The thread names no earlier attempt at this fix; message #7 points to CF 5667, which is a parallel proposal to add support rather than an earlier rejection.
- standards-compat: 2 objection(s). The SQL surface is a new error on create index and adds no syntax, so no SQL standard conflict appears.
- replication-ondisk: 2 objection(s). Two objections from the persistence angle. The new error adds no WAL, catalog or wire changes. The risks are a gap in the check, which ALTER TABLE ADD COLUMN can bypass, and existing indexes that will not restore.
- security: 0 objection(s). The proposal only adds an error in create index, and only the table owner can create the indexes it affects. The wrong keys and the unenforced uniqueness in messages #1 and #3 are data-integrity issues on the owner's own table, and no other role gains access or visibility through them. Nothing here changes privileges, defaults, authentication or exposure, so the security angle has no objection.
- maintenance-fit: 3 objection(s). The cost for users who do not use the feature looks negligible: one extra check at create index time, and no hot-path or ABI impact.
- already-exists: 0 objection(s). Postgres has no existing feature that stops a whole-row index on a table with a virtual generated column from storing keys without that column. The proposal extends the check DefineIndex already makes, so it adds no second mechanism. CF 5667 is an uncommitted alternative and does not already meet the need. The workarounds in message #2 (IS DISTINCT FROM NULL, or a STORED column) avoid the bug but do not prevent it.
- demand: 0 objection(s). The patch is a bugfix, so demand is not the right test. Message #3 shows a unique index that stores keys without the virtual column, and a partial unique index that lets duplicate rows in without any error. Álvaro's doubt in message #5 that whole-row indexes have real users counts in favour of the restriction, because few users would hit the new error. Whether this fix or CF 5667 should go in is a design question for other angles.

### Precedents retrieved

- P1: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P2: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant; reason summarised from its thread
- P3: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - used in the review; reason summarised from its thread
- P4: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P5: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P6: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P7: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - judged not relevant; reason summarised from its thread
- P8: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P9: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant; reason summarised from its thread
- P10: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P11: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant
- P12: [Predicate locking in SP-Gist index](https://commitfest.postgresql.org/patch/1215/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P13: [Improve use of expression indexes for ORDER BY](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P14: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P15: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P16: [Indexes with Included Columns (was Covering + unique indexes)](https://commitfest.postgresql.org/patch/961/) (cf, Returned with feedback, 2017-04-08) - judged not relevant
- P17: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant
- P18: [Revert badly-broken patch to dump comments on composite-type columns.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=18661f282bae5599abb68f53bc8b451b2d67bd75) (revert, Reverted, 2004-08-06) - judged not relevant
- P19: [Add "\gi file" to psql to allow COPY with variable expansion](https://commitfest.postgresql.org/patch/5674/) (cf, Returned with feedback, 2025-06-17) - judged not relevant
- P20: [Revert per-index collation version tracking feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721) (revert, Reverted, 2021-05-07) - judged not relevant
- P21: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P22: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant
- P23: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P24: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 110 s; cost reported by the CLI: $1.27
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 78 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:02:00.936Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
