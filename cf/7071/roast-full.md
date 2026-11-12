# Design pre-review: Bug: Whole-row var in indexes corrupts indexes after DDL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7071](https://commitfest.postgresql.org/patch/7071/) (Needs review); authors: Matthias van de Meent (mmeent)
- Thread: [Bug: Whole-row var in indexes corrupts indexes after DDL](https://www.postgresql.org/message-id/flat/CAEze2WjDaDyvztdXh3Cb2J%3D11CMVRp4NWAW0E6dBRKa1T9w5ag%40mail.gmail.com), started 2026-07-29
- Messages read: 9 of 9
- Patch prose read: v5-0001-Track-whole-row-Var-dependencies-in-pg_depend.patch, v1-0001-Track-whole-row-Var-dependencies-in-pg_depend.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is not in question: the thread shows index corruption and message #2 lists reports of this class since 2020. The surviving objections are fixable design points about the catalog representation (replication-ondisk-2, maintenance-fit-1) and the missing back-branch plan (replication-ondisk-1). A committer who knows pg_depend and the object address code has to decide on the sentinel subid and the new ObjectType value.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review cannot check how aclchk.c, seclabel.c and dropcmds.c treat OBJECT_WHOLE_ROW, or whether every reader of refobjsubid handles -32768. Messages #4 to #9 were not shown in full.

## The proposal, restated

**Problem.** Indexes and constraints whose expressions use a whole-row Var are not rebuilt or revalidated when ADD COLUMN, DROP COLUMN or ALTER COLUMN TYPE changes the row value of existing rows, so the index becomes corrupt. The author ties this to a4639d64e2 and earlier reports of the same class, and the bug affects any user with such expression indexes or constraints.

**Proposed design.** The patch records whole-row references as pg_depend entries with a pseudo-subobject, WholeRowAttrNumber. Shape-changing ALTER TABLE uses these entries to find and rebuild dependent indexes and revalidate constraints without scanning all of them. v4 adds a new object type, OBJECT_WHOLE_ROW, so the object address functions can name and decode it.

**User-visible surface.**
- pg_depend rows with refobjsubid = WholeRowAttrNumber (-32768) for whole-row references.
- New object types "table whole row" and "foreign table whole row" in pg_describe_object(), pg_identify_object(), pg_identify_object_as_address() and pg_get_object_address().
- New OBJECT_WHOLE_ROW value in the ObjectType enum in parsenodes.h, visible to extensions and event triggers.
- ADD COLUMN, DROP COLUMN and ALTER COLUMN TYPE now rebuild indexes and revalidate constraints that use whole-row expressions.

**What the author claims.**
- The patch fixes index corruption from whole-row expression indexes after shape-changing DDL.
- Explicit tracking avoids O(N) dependency scans, which the author calls prohibitively expensive otherwise.
- Checking that relfilenode changes is enough testing, because a rebuild reads the table, not the old index.
- The ATPostAlterTypeCleanup failure on combined ADD COLUMN and DROP COLUMN already exists in HEAD and needs a separate fix.

**Thread state.** Andrey Borodin and Xinlei Zhang reviewed and tested; v5 addresses their points and testing shows no new correctness issues. Open: the patch depends on CF 6121 for the combined ALTER TABLE failure, and whole-row partition keys and column renames are left for later.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-29; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. New refobjsubid value breaks the assumption that a nonzero subid is a column

fixable · level 2, rubric item 9 · catalog changes, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-2`

A later patch or an extension walks pg_depend and treats any nonzero refobjsubid as an attnum. When it meets -32768 it fails with an error like "column -32768 of relation ... does not exist", or it silently skips the row. The failure shows up only for tables that have whole-row expressions.

Until now refobjsubid was either 0 or a positive attnum. Review found two consumers that missed the new value: process_function_rte_ref() in message #2 and the object address functions in messages #2 and #5. Between v1 and v5 the patch grew from three backend files to seven, adding aclchk.c, dropcmds.c, seclabel.c and event_trigger.c. Nothing makes a missed consumer fail at build time, and the material does not compare this design with other ways to mark whole-row rows in pg_depend.

Thread: [#2 (Andrey Borodin, 2026-08-23)](https://www.postgresql.org/message-id/BEE8820C-50BD-467A-BDBC-F95827D737EB%40yandex-team.ru), [#5 (Andrey Borodin, 2026-09-04)](https://www.postgresql.org/message-id/F4161B2A-79DC-453A-BCE1-4A22139B0664%40yandex-team.ru), [#7 (Matthias van de Meent, 2026-09-08)](https://www.postgresql.org/message-id/CAEze2WhuqrHBhUkt_QNYTq1YajV_unu3NVJKnOvsjizaLDo%3DMA%40mail.gmail.com)

**Question:** Which code in core reads refobjsubid, has each place been checked for WholeRowAttrNumber, and why use a magic subid instead of another marker?

Counter-check: held. Messages #2 and #5 show two consumers that missed the new value during review, and the file list grew from three backend files to seven. The thread does not compare the sentinel subid with another marker.

### 2. OBJECT_WHOLE_ROW adds a public object type that is not a real object

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years"), catalog changes · `maintenance-fit-1`

An extension or event trigger that switches over ObjectType gets OBJECT_WHOLE_ROW after upgrade and errors or ignores it. A later patch adds an ObjectType-wide feature and must decide again what a whole-row "object" means for GRANT, SECURITY LABEL, DROP or COMMENT. Nothing fails at build time if that decision is skipped.

v4 added OBJECT_WHOLE_ROW only to make the pg_identify_object_as_address() round trip work (message #5, message #7). As a result, v5 touches aclchk.c, seclabel.c, dropcmds.c and event_trigger.c, though a whole row cannot be granted, labelled or dropped on its own. Message #7 also notes that "<relkind> column" addresses already fail the round trip for indexes and views, so the round trip does not force a new enum value. A committer would likely ask for a representation that reuses the attribute path with subobject -32768.

Thread: [#5 (Andrey Borodin, 2026-09-04)](https://www.postgresql.org/message-id/F4161B2A-79DC-453A-BCE1-4A22139B0664%40yandex-team.ru), [#7 (Matthias van de Meent, 2026-09-08)](https://www.postgresql.org/message-id/CAEze2WhuqrHBhUkt_QNYTq1YajV_unu3NVJKnOvsjizaLDo%3DMA%40mail.gmail.com), [#9 (Matthias van de Meent, 2026-09-09)](https://www.postgresql.org/message-id/CAEze2WhgWxZx_1XyHE75CbxeTLWWwPUEq%3D112S-5TWaH5wMAkw%40mail.gmail.com)

**Question:** Why can the whole-row pseudo-subobject not be encoded through the existing OBJECT_COLUMN address path, without a new ObjectType value seen by extensions and event triggers?

Counter-check: held. Message #7 says the round trip already fails for some "<relkind> column" cases, so it does not by itself require a new ObjectType. The thread gives no reason for wiring a non-object into aclchk.c, seclabel.c and dropcmds.c.

### 3. Catalog-based bug fix leaves supported branches and their existing indexes uncovered

fixable · level 2, rubric item 6 · backpatchability, catalog changes, ABI / extension API · `replication-ondisk-1`

A user on a supported back branch has an index on f(t) and runs ALTER TABLE t ADD COLUMN c int. The index is still not rebuilt, so index scans keep returning wrong rows. The user finds out from inconsistent query results, possibly long after the DDL.

The entry is tagged Bugfix, but the fix depends on new pg_depend rows and a new ObjectType value in parsenodes.h. Back branches would need the new rows for every existing index and constraint, plus an ABI change. The material says nothing about back branches, and message #2 shows the same bug class reported since 2020. A committer would likely ask whether a separate fix for released branches is planned, such as a catalog scan during shape-changing DDL.

Thread: [#1 (Matthias van de Meent, 2026-07-29)](https://www.postgresql.org/message-id/CAEze2WjDaDyvztdXh3Cb2J%3D11CMVRp4NWAW0E6dBRKa1T9w5ag%40mail.gmail.com), [#2 (Andrey Borodin, 2026-08-23)](https://www.postgresql.org/message-id/BEE8820C-50BD-467A-BDBC-F95827D737EB%40yandex-team.ru)

**Question:** What fix, if any, do supported back branches get, and how does it handle indexes created before the fix?

Counter-check: held. The entry is tagged Bugfix and message #2 lists reports since 2020, but the thread never mentions back branches. The fix needs pg_depend rows that existing indexes on released branches lack, so the question stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Existing indexes and constraints get the new pg_depend rows only when pg_upgrade restores the schema. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds pg_depend rows with refobjsubid = WholeRowAttrNumber (-32768). |
| ABI / extension API | **yes** | The patch adds OBJECT_WHOLE_ROW to the ObjectType enum in parsenodes.h, visible to extensions and event triggers. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | v5 touches aclchk.c and seclabel.c to handle OBJECT_WHOLE_ROW. |
| backpatchability | **yes** | The entry is tagged Bugfix, but the fix needs new pg_depend rows and a new enum value. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every reader of refobjsubid and every ObjectType-wide feature must now handle the whole-row case. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which code in core reads refobjsubid, and has each place been checked for WholeRowAttrNumber?
2. Why does the patch use a sentinel subid instead of another marker in pg_depend, such as a distinct deptype?
3. Why can the whole-row pseudo-subobject not go through the existing column address path, without a new ObjectType value?
4. What fix, if any, do supported back branches get, and how does it cover indexes created before the fix?
5. Will a patch that rejects whole-row partition keys ship with this one, or can mispartitioned rows remain after it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows pg_depend, the object address functions and the ALTER TABLE rebuild passes in tablecmds.c would have to own it.
- **Liability:** The owner signs for a new pg_depend convention and a public ObjectType value that every future reader of refobjsubid must handle, with no end date. The owner also signs for the back-branch answer and the interaction with CF 6121.
- **Author follow-up:** The author answered review within about a week in message #3 and posted revisions up to v5 in message #9.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Few users have whole-row expression indexes or constraints, but for them the bug is silent index corruption with reports since 2020. The patch is about 37 KiB across dependency.c, objectaddress.c and tablecmds.c, and adds a lasting catalog convention.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Back branches keep the corruption bug and get no fix. Refuted (not specific): Duplicate of replication-ondisk-1. P1 concerns column aliases in whole-row Var output and must not be used as support for a back-branch plan here.
- `standards-compat-2` refobjsubid = -32768 changes the documented pg_depend contract. Refuted (not specific): Duplicate of replication-ondisk-2, which makes the same sentinel point with evidence from the thread. The inner-join case loses nothing if whole-row references have no column-level pg_depend rows today, and missing documentation hunks are out of scope.
- `maintenance-fit-2` Bug fix depends on catalog rows that existing indexes do not have. Refuted (not specific): Duplicate of replication-ondisk-1 for the minor-release case. The pg_upgrade part does not hold, because pg_upgrade recreates the schema through pg_dump and restore, which writes the new pg_depend rows.
- `maintenance-fit-3` Each row-changing DDL path must remember to call the rebuild hook. Refuted (not specific): The patch follows the per-subcommand pattern that ALTER COLUMN TYPE already uses in tablecmds.c to remember dependents, so this applies to existing code equally. The cited renames and partition keys are stated scope limits, and none is a missed call.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] reverted column aliases on whole-row Var output because blessed RECORD tuples were stored on disk; that reason does not apply to pg_depend tracking. Message #2 lists earlier reports of this bug class, including prior work on whole-row dependencies for DROP COLUMN and ALTER TYPE, but the thread does not say why that work stopped.
- standards-compat: 2 objection(s). Upgrade itself looks safe from this angle, because pg_upgrade restores the schema with CREATE INDEX, which records the new pg_depend rows. The new constraint revalidation errors on ADD COLUMN are corrections, but the release notes should mention them.
- replication-ondisk: 2 objection(s). pg_upgrade restores the schema through DDL, so new clusters get the pg_depend rows. No WAL, heap or wire protocol format changes are shown.
- security: 0 objection(s). The proposal grants no new privilege and adds no new user-callable action. The new rebuilds run inside ALTER TABLE, which only the table owner or a superuser can run, and they evaluate index expressions the same way ALTER COLUMN TYPE and REINDEX already do. The new pg_depend rows and the "table whole row" object type expose only what pg_depend and the attribute address types already show. A reviewer cannot judge from the prose alone how the aclchk.c and seclabel.c changes handle OBJECT_WHOLE_ROW, so that needs a check against the code.
- maintenance-fit: 3 objection(s). The goal itself is not in question from this angle. Costs fall only on tables with whole-row expressions, and the thread shows repeated field reports of this bug class.
- already-exists: 0 objection(s). This is a fix for index and constraint corruption inside ALTER TABLE, and no extension, contrib module or user idiom can make core rebuild those indexes after shape-changing DDL. The only alternative in the material is scanning all indexes and constraints on each such DDL; the author rejects it on cost, and that is a design trade-off for other angles. I cannot name an existing mechanism that does this job, and [P1] covers whole-row Var naming, not dependency tracking.
- demand: 0 objection(s). The proposal fixes index corruption, and message #2 lists independent reports of the same bug class from 2020 and 2023, plus a4639d64e2. Each fix costs a shape-changing ALTER TABLE on a table with whole-row index or constraint expressions, so demand is shown and the 'nobody will use it' test does not apply.

### Precedents retrieved

- P1: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - used in the review
- P2: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant; reason summarised from its thread
- P3: [Revert "Temporarily make pg_ctl and server shutdown a whole lot chattier."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c5e9b771275b93b09eec6b760677fe6c5e726ab2) (revert, Reverted, 2016-02-10) - judged not relevant
- P4: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P5: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant; reason summarised from its thread
- P6: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P7: [Improve use of expression indexes for ORDER BY](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P9: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P10: [Introduce the ability to enable/disable indexes using ALTER INDEX](https://commitfest.postgresql.org/patch/5274/) (cf, Withdrawn, 2025-09-08) - judged not relevant; reason summarised from its thread
- P11: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant; reason summarised from its thread
- P12: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P13: [Add a feature that enables row-wise comparison partition pruning](https://commitfest.postgresql.org/patch/2654/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P14: [pageinspect: Hash index support](https://commitfest.postgresql.org/patch/758/) (cf, Returned with feedback, 2016-12-20) - judged not relevant
- P15: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P16: [Avoid orphaned dependencies](https://commitfest.postgresql.org/patch/3106/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P17: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P18: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P19: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P20: [Remove dependence on integer wrapping](https://commitfest.postgresql.org/patch/5035/) (cf, Withdrawn, 2024-06-17) - judged not relevant
- P21: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P22: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant
- P23: [Indexes with Included Columns (was Covering + unique indexes)](https://commitfest.postgresql.org/patch/961/) (cf, Returned with feedback, 2017-04-08) - judged not relevant
- P24: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 136 s; cost reported by the CLI: $1.80
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 81 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:17:24.990Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
