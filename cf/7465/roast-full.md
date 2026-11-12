# Design pre-review: Fix UNIQUE WITHOUT OVERLAPS + NULLS NOT DISTINCT + multiranges

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7465](https://commitfest.postgresql.org/patch/7465/) (Needs review); authors: Paul Jungwirth (pjungwir)
- Thread: [Fix UNIQUE WITHOUT OVERLAPS + NULLS NOT DISTINCT + multiranges](https://www.postgresql.org/message-id/flat/CA%2BrenyV93dvyRbjYy320WaWE7Rg-uRDV4JfFBZ2AmmUcODZEgw%40mail.gmail.com), started 2026-10-09
- Messages read: 2 of 2
- Patch prose read: v1-0001-Fix-UNIQUE-NULLS-NOT-DISTINCT-with-WITHOUT-OVERLA.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

This is a correctness fix for a shipped feature: a declared NULLS NOT DISTINCT constraint accepts duplicates on multirange columns. No objection survived at levels 1a or 1b. The one surviving objection, standards-compat-1, is fixable: the backpatch gives 18 users no guidance for tables that already hold duplicate NULL keys. That point and the C signature change in a back branch are for a committer to judge.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check whether index_recheck_constraint is exported, or whether the new parameter leaves ordinary EXCLUDE constraints unchanged as the commit message says.

## The proposal, restated

**Problem.** A unique or primary key constraint declared with WITHOUT OVERLAPS and NULLS NOT DISTINCT treats NULLs as distinct when the range column is a multirange, so it accepts duplicates. Ranges work because their GiST index is not lossy; multiranges go through index_recheck_constraint, which never treats nulls as matching. Andres Freund reported the bug.

**Proposed design.** The patch adds a parameter to index_recheck_constraint that selects the null handling, so it behaves like check_exclusion_or_unique_constraint. Ordinary EXCLUDE constraints keep their current behaviour. The fix is to be backpatched through 18.

**User-visible surface.**
- Changed behaviour: unique and primary key constraints using WITHOUT OVERLAPS and NULLS NOT DISTINCT on multirange columns now reject duplicate NULLs.
- Changed C API: index_recheck_constraint in execIndexing.c gets a new parameter for null handling.

**What the author claims.**
- Ranges already behave correctly; only lossy GiST cases such as multiranges are affected.
- Ordinary EXCLUDE constraints are not affected, since they treat all operators as strict regardless of declaration.
- EXCLUDE constraints with a non-strict operator such as IS NOT DISTINCT FROM accept two NULLs, and this assumption is documented only in a code comment.
- SP-GiST could never support non-strict operators, because it never passes nulls to the opclass.

**Thread state.** Osama Abdul Qader replied, restating the report and the open EXCLUDE question without new findings. Peter Eisentraut is listed as reviewer but has not posted; whether to allow, forbid or document non-strict operators in EXCLUDE constraints is open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-09; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Backpatched fix gives no path for 18 tables already holding duplicates

fixable · level 2, rubric item 13 · backpatchability, pg_dump / restore round-trip · `standards-compat-1`

A user on 18 has a multirange table with WITHOUT OVERLAPS and NULLS NOT DISTINCT, and it already holds duplicate NULL keys. After the minor update, a later REINDEX, or a pg_dump restore that rebuilds the constraint, may fail. The user first sees the duplicates during an upgrade or a restore.

The commit message says the bug "wrongly permitted duplicates", so some 18 tables can hold rows that the fixed constraint rejects. The patch touches only execIndexing.c and the regression tests, and it has no documentation hunk. The proposal does not say how users find such rows before or after updating. Backpatched constraint fixes usually come with a release-note query that finds violating rows and advice on whether to REINDEX.

Thread: [#1 (Paul A Jungwirth, 2026-10-09)](https://www.postgresql.org/message-id/CA%2BrenyV93dvyRbjYy320WaWE7Rg-uRDV4JfFBZ2AmmUcODZEgw%40mail.gmail.com)

**Question:** What release-note text and detection query will tell 18 users whether their multirange temporal keys already hold duplicate NULLs, and what to do about them?

Counter-check: held. The commit message says the bug "wrongly permitted duplicates", so 18 tables can hold rows that a rebuilt constraint rejects. The thread and the patch prose do not mention existing data, and the brief lists this as open.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds a parameter to index_recheck_constraint in execIndexing.c and is to be backpatched to 18. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Restoring a dump of an 18 table that already holds duplicate NULL keys may fail once the constraint is enforced. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message says "Backpatch-through: 18", and the fix changes constraint behaviour in a minor release. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What release-note text and detection query will tell 18 users whether their multirange temporal keys already hold duplicate NULLs, and how to repair them?
2. Is index_recheck_constraint visible to extensions, and if so, does the 18 backpatch keep the old signature working?
3. Do the new tests in without_overlaps.sql cover a multirange unique constraint with NULLS NOT DISTINCT for both the insert path and the update path?
4. Will the open question about non-strict operators in EXCLUDE constraints go to a separate thread, so that it does not hold up this fix?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Eisentraut is signed up as reviewer in the CommitFest entry but has not posted in the thread. The owner needs to know the executor's exclusion-constraint recheck code and the temporal constraint feature.
- **Liability:** The owner signs for a behaviour change in a minor release of 18: constraints start rejecting rows they accepted before. The owner also signs for the changed function signature in a back branch.
- **Author follow-up:** The author wrote the temporal constraint feature and offers in message #1 to send a separate patch for the EXCLUDE case.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug affects only unique constraints that combine WITHOUT OVERLAPS, NULLS NOT DISTINCT and a multirange column, but it lets a declared constraint accept duplicates. The patch is 8.4 KB and touches one executor file plus regression tests.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` No plan for multirange tables that already hold duplicate NULL keys. Refuted (not specific): Duplicate of standards-compat-1. The patch changes no on-disk format, so the pg_upgrade and safety framing adds nothing beyond the existing-duplicates point.

### Review angles

- prior-attempts: 0 objection(s). Neither listed precedent is the same idea or a close relative. [P1] reverted temporal keys over empty-range handling, a different defect, and that reason does not apply to this null-handling fix. [P18] fixed index-only scan rechecks and records an ABI concern in a different area; it does not count against this patch. The thread points to no earlier attempt at this fix, so no precedent was found.
- standards-compat: 1 objection(s). The fix makes the behaviour match what NULLS NOT DISTINCT already promises, and ranges already behave this way, so it adds no new syntax and no GUC. The only compatibility gap from this angle is the missing guidance for existing data in a backpatch. The open question about non-strict EXCLUDE operators belongs to a separate patch.
- replication-ondisk: 1 objection(s). The patch changes no on-disk format, WAL, catalog or wire protocol. The only persistent-state issue is data already admitted by the bug.
- security: 0 objection(s). The fix tightens null handling in a constraint recheck for one constraint type. It adds no privilege, role, default, exposure path or code run by maintenance commands, and no unprivileged user gains any ability or information. Backpatching the C signature change is an ABI concern for another angle.
- maintenance-fit: 0 objection(s). From the maintenance angle this is a small bugfix in one function, written by the author of the temporal constraint feature, and it leaves ordinary EXCLUDE constraints unchanged. Two points could not be checked without the code. First, whether index_recheck_constraint is exported, since a signature change in a backpatch would then need a compatibility wrapper. Second, how existing 18 tables that already hold duplicate NULLs are handled after a minor upgrade.
- already-exists: 0 objection(s). This is a correctness fix for a shipped feature, and no existing feature or extension gives the same result. Using a range column instead of a multirange avoids the bug but is not a fix. The patch extends the existing recheck function, so it adds no second mechanism and no new hook.
- demand: 0 objection(s). This is a correctness bug fix: a declared NULLS NOT DISTINCT constraint accepts duplicates on multirange columns, reported independently by Andres Freund (message #1). Demand is not the test for honouring a documented constraint, and the cost falls only on users of the feature.

### Precedents retrieved

- P1: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - used in the review
- P2: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant
- P3: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P4: [Remove redundant DISTINCT when GROUP BY already guarantees uniqueness](https://commitfest.postgresql.org/patch/6869/) (cf, Withdrawn, 2026-07-23) - judged not relevant; reason summarised from its thread
- P5: [Erase the distinctClause if the result is unique by definition](https://commitfest.postgresql.org/patch/2433/) (cf, Returned with feedback, 2022-04-05) - judged not relevant; reason summarised from its thread
- P6: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P7: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P8: [pg_dump: fix NOT NULL constraint name comparison using makeObjectName](https://commitfest.postgresql.org/patch/6601/) (cf, Withdrawn, 2026-10-04) - judged not relevant; reason summarised from its thread
- P9: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P10: [Revert "Catalog NOT NULL constraints" and fallout](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9ce04b50e120275afbc03ca0b80839dde3da8308) (revert, Reverted, 2023-04-12) - judged not relevant
- P11: [Remove deprecated exclusive backup mode](https://commitfest.postgresql.org/patch/1913/) (cf, Withdrawn, 2021-03-19) - judged not relevant; reason summarised from its thread
- P12: [SimpleLruTruncate() mutual exclusion (data loss from lack thereof)](https://commitfest.postgresql.org/patch/2025/) (cf, Withdrawn, 2020-01-25) - judged not relevant; reason summarised from its thread
- P13: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P14: [Revert "Enable fast default for domains with non-volatile constraints"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0354e29c41a9fb7491b3c7c23f079b1923c045a) (revert, Reverted, 2026-06-08) - judged not relevant
- P15: [deferred primary key and logical replication](https://commitfest.postgresql.org/patch/2631/) (cf, Returned with feedback, 2020-11-29) - judged not relevant
- P16: [Dump-restore loosing 'attnotnull' bit for DEFERRABLE PRIMARY KEY column(s)](https://commitfest.postgresql.org/patch/4729/) (cf, Rejected, 2023-12-28) - judged not relevant
- P17: [Temporal foreign key actions](https://commitfest.postgresql.org/patch/7238/) (cf, Withdrawn, 2026-09-17) - judged not relevant
- P18: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - used in the review
- P19: [\[PATCH\] reduce page overlap of GiST indexes built using sorted method](https://commitfest.postgresql.org/patch/3487/) (cf, Returned with feedback, 2022-01-09) - judged not relevant
- P20: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P21: [Fix unique join costings](https://commitfest.postgresql.org/patch/2050/) (cf, Returned with feedback, 2020-12-03) - judged not relevant
- P22: [Indexes with Included Columns (was Covering + unique indexes)](https://commitfest.postgresql.org/patch/961/) (cf, Returned with feedback, 2017-04-08) - judged not relevant
- P23: [temporary file usage report with extended protocol and unnamed portals](https://commitfest.postgresql.org/patch/5711/) (cf, Withdrawn, 2025-08-21) - judged not relevant
- P24: [unnesting multirange data types](https://commitfest.postgresql.org/patch/3178/) (cf, Returned with feedback, 2021-10-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 90 s; cost reported by the CLI: $1.12
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:58:04.676Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
