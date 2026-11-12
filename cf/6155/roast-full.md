# Design pre-review: Add a warning on stats manipulation functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6155](https://commitfest.postgresql.org/patch/6155/) (Needs review); authors: Yugo Nagata (yugo.nagata)
- Thread: [Can we use Statistics Import and Export feature to perforamance testing?](https://www.postgresql.org/message-id/flat/TY3PR01MB118914FCCBBDC126C34AA287382B52%40TY3PR01MB11891.jpnprd01.prod.outlook.com), started 2025-04-08
- Messages read: 13 of 13
- Patch prose read: doc_add_warning_on_stats_manipulation_funcs.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The patch adds one paragraph to an existing docs warning and changes no code. Message #1 shows demand from several PGEcons member companies, and message #2 supports each factor the paragraph names. The only surviving objection, standards-compat-1, is about where the note is placed and is fixable. Wording and the missing rebase are documentation review, outside levels 1-2.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only the first 3 of 13 messages and the v1 docs hunk were read. The exact wording, the SGML placement after a rebase, and any agreement in later messages could not be checked.

## The proposal, restated

**Problem.** Users want to copy production statistics to a staging server with pg_dump --statistics-only and expect the same plans there. The planner also uses inputs that statistics do not carry, so plans can differ. The patch is meant to stop users from asking this question again.

**Proposed design.** The patch adds one paragraph to the existing warning box on statistics manipulation functions in func-admin.sgml. The paragraph says manually restored statistics do not guarantee the same plans as the source, because relation sizes, index OIDs and configuration parameters affect the planner. There is no code change.

**User-visible surface.**
- New paragraph in the warning on statistics manipulation functions (doc/src/sgml/func/func-admin.sgml).

**What the author claims.**
- The name "statistics manipulation functions" suggests they can control generated plans, so a warning helps.
- Relation sizes, index OIDs and configuration parameters can make plans differ after restoring statistics.
- Ryohei Takahashi says the patch prevents user confusion.

**Thread state.** Ryohei Takahashi approved v1, and Kenichiro Tanaka proposed rewording: "actual relation file sizes", "the creation order of indexes", and adding the actual data range read from indexes. The author has not answered, and a cross-reference patch from vellaipandiyan sm never reached the list.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-04-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Warning sits on the functions page, not where pg_dump users look

fixable · level 1b, rubric item 2 · pg_dump / restore round-trip · `standards-compat-1`

A user follows the workflow from message #1 and runs pg_dump --statistics-only, then restores the output on staging. They read the pg_dump and pg_restore pages, which do not carry the new warning. They find out only when staging plans differ from production.

The question in message #1 is about pg_dump --statistics-only, and message #3 asked for the note on the pg_dump page. The patch in message #8 changes only func-admin.sgml, which pg_dump users do not need to read, because pg_dump calls pg_restore_relation_stats() for them. No later message settles where the note belongs, so the patch may not stop the repeat question it was written for.

Thread: [#1 ("Ryohei Takahashi (Fujitsu)", 2025-04-08)](https://www.postgresql.org/message-id/TY3PR01MB118914FCCBBDC126C34AA287382B52%40TY3PR01MB11891.jpnprd01.prod.outlook.com), [#3 ("Ryohei Takahashi (Fujitsu)", 2025-04-08)](https://www.postgresql.org/message-id/TY3PR01MB11891C649F6B4406994EC972982B52%40TY3PR01MB11891.jpnprd01.prod.outlook.com), [#8 (Yugo Nagata, 2025-10-23)](https://www.postgresql.org/message-id/20251023192753.165621c1ae24b04b718cebf4%40sraoss.co.jp)

**Question:** Why does the warning not also belong in, or link from, the pg_dump and pg_restore pages that describe --statistics-only?

Counter-check: held. Message #3 asked for the note on the pg_dump page, and the patch changes only func-admin.sgml. Takahashi accepted v1 in message #9, but no message says why the pg_dump and pg_restore pages need no note or link.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | The warning is about statistics restored from pg_dump --statistics-only output, but the pg_dump and pg_restore pages get no note. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should the pg_dump and pg_restore pages that describe --statistics-only carry the same note, or a link to it, as message #3 asked?
2. Do you accept Tanaka's wording from message #13: "actual relation file sizes", "the creation order of indexes", and the actual data range read from indexes?
3. Should the warning also name work_mem and the other real-data effects that Corey Huinker raised, or say that the list of factors is not complete?
4. Can you post a rebased version, since CFbot reported in 2026-03 that the patch needs a rebase?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the statistics import and export documentation would have to pick this up; the visible messages show no committer review of the patch.
- **Liability:** One docs paragraph with a short list of factors. The owner signs for its accuracy and for keeping the list in line with planner behaviour.
- **Author follow-up:** The author moved the entry to each new CommitFest through 2026-10, but has not answered the wording suggestions or posted a rebase.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is a 738-byte docs change with demand shown in message #1 and almost no cost after commit. It needs a rebase and a revision that settles the wording and the placement before a committer can take it.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P10] adds input functions for extended statistics types and was withdrawn over version and environment portability of stats formats, which does not apply to a docs-only warning. The thread points to no earlier discussion of this warning, so the prior-attempts angle has nothing to raise.
- standards-compat: 1 objection(s). The patch changes no syntax, catalog, GUC or behaviour, so the standards and compatibility angle has little to object to. The only point is where the warning is placed. Wording issues from message #13 are documentation detail and fall outside this angle.
- replication-ondisk: 0 objection(s). The patch adds one paragraph to func-admin.sgml and changes no code, so it touches no on-disk format, WAL, replication, catalog, wire protocol or extension ABI. This angle has nothing to object to. [P10] is about restoring extended statistics, not about documentation, so it does not apply.
- security: 0 objection(s). The patch adds one documentation paragraph saying that restored statistics do not guarantee the same plans. It changes no code, privilege, default or exposure, so it raises no security concern.
- maintenance-fit: 0 objection(s). The patch adds one non-exhaustive paragraph to an existing docs warning. It touches no hot path, no API and no catalog, and adds no dependency, so its maintenance cost is close to zero. The wording points raised in message #13 are documentation review, which is out of scope here. [P10] is about extended statistics input functions and its reason does not apply.
- already-exists: 0 objection(s). No existing feature or document already gives this warning. The planner docs explain the actual-size scaling (message #6), but they do not say that restored statistics will not reproduce plans. An extension cannot replace a documentation note, and get_relation_info_hook (message #2) only helps users who want to force sizes, so a docs patch remains a reasonable place for the note.
- demand: 0 objection(s). Demand is shown in the material. Message #1 reports that several PGEcons member companies want production statistics on staging servers, and the confusion came up on the list itself. A docs-only paragraph costs almost nothing and adds no burden for non-users. Wording questions in message #13 are for other angles.

### Precedents retrieved

- P1: [Adaptive query optimization: create extented statistics using auto_explain extension](https://commitfest.postgresql.org/patch/2958/) (cf, Returned with feedback, 2021-11-04) - judged not relevant; reason summarised from its thread
- P2: [Revert "vacuumdb: Add option for analyzing only relations missing stats."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19e57f4f78e4354d9a21c284868373d28bb0d368) (revert, Reverted, 2025-03-07) - judged not relevant
- P3: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P4: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P5: [Fix pg_stat_statements query jumbling to use relation name instead of relation oid](https://commitfest.postgresql.org/patch/365/) (cf, Rejected, 2015-11-15) - judged not relevant; reason summarised from its thread
- P6: [proposal of adding 2 new diag items related to parsing the dynamic sql statement](https://commitfest.postgresql.org/patch/3258/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P7: [Revert "Allow units to be specified in relation option setting value."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9df492664a74a948c87232f523eea637920a516b) (revert, Reverted, 2014-08-28) - judged not relevant
- P8: [multivariate statistics](https://commitfest.postgresql.org/patch/120/) (cf, Returned with feedback, 2015-08-25) - judged not relevant; reason summarised from its thread
- P9: [multivariate statistics](https://commitfest.postgresql.org/patch/450/) (cf, Returned with feedback, 2016-04-08) - judged not relevant; reason summarised from its thread
- P10: [Implement type input functions for extended statistics types](https://commitfest.postgresql.org/patch/4114/) (cf, Withdrawn, 2023-03-01) - used in the review; reason summarised from its thread
- P11: [multivariate statistics](https://commitfest.postgresql.org/patch/58/) (cf, Rejected, 2015-01-16) - judged not relevant; reason summarised from its thread
- P12: [Optionally record Plan IDs to track plan changes for a query](https://commitfest.postgresql.org/patch/5486/) (cf, Returned with feedback, 2025-03-24) - judged not relevant
- P13: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P14: [Restore lock level to update statusFlags](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dcfff74fb16622898a9d1df33e530f477caab095) (revert, Partially reverted, 2020-11-26) - judged not relevant
- P15: [Extending with C: doc on how to call internal functions](https://commitfest.postgresql.org/patch/5259/) (cf, Withdrawn, 2024-09-18) - judged not relevant
- P16: [Refuse setting toast.* reloptions when TOAST relation does not exist](https://commitfest.postgresql.org/patch/1486/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P17: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P18: [psql export import bytea data](https://commitfest.postgresql.org/patch/913/) (cf, Rejected, 2017-09-07) - judged not relevant
- P19: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant
- P20: [More stable query plans via more predictable column statistics](https://commitfest.postgresql.org/patch/434/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P21: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P22: [Add --include-table-data-where option to pg_dump, to export only a subset of table data](https://commitfest.postgresql.org/patch/1644/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P23: [Logging plan of the currently running query](https://commitfest.postgresql.org/patch/3142/) (cf, Withdrawn, 2023-06-06) - judged not relevant
- P24: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant

### Warnings

- synthesis said NO-FUNDAMENTAL-OBJECTION but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 92 s; cost reported by the CLI: $1.34
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 46 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:43:50.845Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
