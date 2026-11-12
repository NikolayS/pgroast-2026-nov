# Design pre-review: psql: Fix REPACK index completion after a column list

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7611](https://commitfest.postgresql.org/patch/7611/) (Needs review); authors: Sho Ito (sean0628)
- Thread: [\[PATCH\] psql: Fix REPACK index completion after a column list](https://www.postgresql.org/message-id/flat/CAG7L5%2B7Xmus%2BJLa0m%3DLpw49wU%3DsecPm5dzNd2ibmfvhdSYEfqA%40mail.gmail.com), started 2026-11-08
- Messages read: 5 of 5
- Patch prose read: v3-0001-psql-Fix-REPACK-tab-completion-around-USING-INDEX.patch, v3-0002-psql-Add-regression-tests-for-REPACK-tab-completi.patch, v1-0001-psql-Fix-REPACK-index-completion-after-a-column-list.patch, v1-0001-psql-Test-REPACK-index-completion-after-a-column-list.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible example in message #1, the fix is confined to psql tab completion for an existing command, and it adds no syntax, GUC, catalog state or server code. The remaining points are small questions about completion behaviour, not design objections. Confidence is medium because the code was not read and nobody has replied to v3.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the word-position matching in v3 could not be checked against multi-column lists, schema-qualified table names or option lists without ANALYZE. Messages #4 and #5 were not shown in full.

## The proposal, restated

**Problem.** With a column list, psql completes `REPACK (ANALYZE) tab1 (c1) USING INDEX` with indexes from unrelated tables. The completion code takes `prev3_wd` as the table name, but that word is the column list, so the table query returns nothing and completion falls back to all indexes. The patch also merges a fix for completing INDEX after a typed `USING`.

**Proposed design.** v3 skips the column list to find the target table when it completes an index name. It also completes INDEX after an explicitly typed USING, with or without command options or a column list. It adds no completion directly after a column list, and no column-list continuations when the command has no option list, because those forms cannot enable ANALYZE.

**User-visible surface.**
- psql tab completion: `REPACK ... (cols) USING INDEX <Tab>` offers indexes of the named table only.
- psql tab completion: `REPACK ... USING <Tab>` completes INDEX, also with options or a column list.

**What the author claims.**
- Without a command option list, column-list forms cannot enable ANALYZE and are always invalid, so completing them is wrong.
- Checking whether the option list enables ANALYZE adds enough complexity to justify a separate patch.

**Thread state.** Chao Li reviewed v1 and v2 and found that v2 completed `USING INDEX` after `repack t (a)`, which the server rejects without ANALYZE. Sho posted v3 with that completion removed, and nobody has replied to v3 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

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
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | The patch changes which words and index names psql offers for REPACK in src/bin/psql/tab-complete.in.c. |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry is tagged Bugfix, and the thread does not say which release first shipped REPACK. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds word-position rules for REPACK that must follow any later change to the REPACK grammar. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. With v3, does `REPACK (VERBOSE) t (a) USING INDEX <Tab>` still offer index names, although the server rejects a column list without ANALYZE?
2. What does v3 offer for `REPACK (ANALYZE) t (a) <Tab>`: the same as before the patch, or nothing?
3. Which release first shipped REPACK, and do you ask for a backpatch to it?
4. Does the table lookup still find the table when the column list has several columns or spaces, such as `(c1, c2)`?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows psql tab completion and the REPACK grammar would have to own it. No committer is visible in the thread; the only reviewer is Chao Li.
- **Liability:** Low: the owner signs for a few completion rules in tab-complete.in.c and their TAP tests, which must be kept in step with the REPACK grammar.
- **Author follow-up:** Good sign: the author posted v2 and v3 within days of each review and withdrew the sibling entry #7610 as agreed in message #3.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix helps psql users who type REPACK with a column list or a typed USING. The two patches are under 3 KB each, touch one psql source file and one TAP test, and add no new user-visible surface beyond completion.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] is a different set of psql completion fixes, and its reason (keep IF NOT EXISTS out of the same patch) does not apply, because the reviewer in message #2 asked for the two related REPACK fixes to be merged. [P11] and [P18] cover COPY option completion and record no reason. The thread cites only the withdrawn sibling entry from message #3, which this patch absorbs.
- standards-compat: 0 objection(s). From the standards and compatibility angle this is a small psql tab-completion fix. It adds no syntax, no GUC, no catalog state and no dump output, and existing scripts behave the same. Completing INDEX after USING follows the existing CLUSTER pattern. v3 leaves completion directly after a column list unchanged, which settles the issue from message #4. One inconsistency is left: v3 can still look up indexes after an option list without ANALYZE, such as `(VERBOSE)`, which the server rejects. psql completion is best-effort elsewhere too, so this is not worth an objection. Whether a backpatch is needed depends on the release that first shipped REPACK, which the material does not say.
- replication-ondisk: 0 objection(s). The patch changes only psql tab completion in tab-complete.in.c and its TAP test. It adds no on-disk format, WAL record, catalog, wire protocol message or extension API, so nothing persistent or cross-node is affected. Backpatching would not be limited by ABI.
- security: 0 objection(s). The patch has no privilege or exposure aspect. It changes which index names psql tab completion offers for REPACK, and those names come from catalog queries that the connected user can already run. It adds no new privilege, default, server code path or data exposure.
- maintenance-fit: 0 objection(s). From the maintenance angle there is no objection. The patch is a small fix in tab-complete.in.c, built on the same word-position matching that the existing completion rules use. It adds no hot-path cost, no API and no new mechanism, and it comes with TAP tests. The open point is the deferred check for whether the option list enables ANALYZE, and that is a later improvement. Nobody has reviewed v3 yet, and no committer has joined the thread.
- already-exists: 0 objection(s). This is a bug fix to existing psql tab completion for REPACK, and no existing feature, extension or tool offers the corrected completion. Tab completion lives only in psql's tab-complete.in.c, so this angle has no in-core or out-of-core alternative to name.
- demand: 0 objection(s). This is a small bug fix to existing psql tab completion for REPACK, a command that already ships. Completing indexes from unrelated tables is wrong behaviour that any REPACK user can hit, so it does not need separate evidence of demand. The fix touches only tab-complete.in.c and its TAP test, and adds no UI, GUC or hot-path cost for anyone else.

### Precedents retrieved

- P1: [Tab completion for SELECT in psql](https://commitfest.postgresql.org/patch/1376/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P2: [Tab-completion refactoring and word-removal feature (was IF-NOT_EXISTS for tab-completion)](https://commitfest.postgresql.org/patch/855/) (cf, Returned with feedback, 2017-03-17) - judged not relevant; reason summarised from its thread
- P3: [Fix psql-completion for CREATE INDEX](https://commitfest.postgresql.org/patch/452/) (cf, Returned with feedback, 2016-02-03) - used in the review; reason summarised from its thread
- P4: [REPACK (CONCURRENTLY) doesn't handle invalid indexes](https://commitfest.postgresql.org/patch/7039/) (cf, Returned with feedback, 2026-09-13) - judged not relevant
- P5: [IF (NOT) EXISTS in psql-completion](https://commitfest.postgresql.org/patch/518/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P6: [psql: Add tab-complete for optional view parameters](https://commitfest.postgresql.org/patch/4053/) (cf, Returned with feedback, 2023-07-06) - judged not relevant; reason summarised from its thread
- P7: [REPACK (CONCURRENTLY) doesn't handle invalid indexes](https://commitfest.postgresql.org/patch/7040/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P8: [remove VersionedQuery support from psql tab completion](https://commitfest.postgresql.org/patch/6977/) (cf, Withdrawn, 2026-08-11) - judged not relevant; reason summarised from its thread
- P9: [Multi-column list partitioning](https://commitfest.postgresql.org/patch/3383/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P10: [Table AM modifications to accept column projection lists](https://commitfest.postgresql.org/patch/2922/) (cf, Withdrawn, 2021-06-30) - judged not relevant; reason summarised from its thread
- P11: [\[PATCH v1\] psql: Do not complete JSON format for COPY FROM](https://commitfest.postgresql.org/patch/7584/) (cf, Withdrawn, 2026-11-04) - used in the review
- P12: [psql: add tab completion for \df slash command suffixes](https://commitfest.postgresql.org/patch/2268/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P13: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P14: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P15: [tab-completion debug print](https://commitfest.postgresql.org/patch/1917/) (cf, Returned with feedback, 2019-02-02) - judged not relevant
- P16: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P17: [improve tab-completion behavior when parse text in single quote](https://commitfest.postgresql.org/patch/3267/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P18: [\[PATCH v1\] psql: Do not complete HEADER MATCH for COPY TO](https://commitfest.postgresql.org/patch/7585/) (cf, Withdrawn, 2026-11-04) - used in the review
- P19: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant
- P20: [Revert analyze support for partitioned tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8127b7390119c21479f5ce495b7d2168930e82) (revert, Reverted, 2021-08-16) - judged not relevant
- P21: [\d+ tablename, \d+ indexname about subpartition](https://commitfest.postgresql.org/patch/4398/) (cf, Returned with feedback, 2024-01-27) - judged not relevant
- P22: [default_index_tablespace](https://commitfest.postgresql.org/patch/216/) (cf, Returned with feedback, 2015-06-14) - judged not relevant
- P23: [Remove unused include from analyze.c](https://commitfest.postgresql.org/patch/6987/) (cf, Withdrawn, 2026-10-05) - judged not relevant
- P24: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 70 s; cost reported by the CLI: $0.66
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 41 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:07:08.930Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
