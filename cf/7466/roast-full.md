# Design pre-review: Fix WITHOUT OVERLAPS PKs used for functional grouping

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7466](https://commitfest.postgresql.org/patch/7466/) (Needs review); authors: Paul Jungwirth (pjungwir)
- Thread: [Fix WITHOUT OVERLAPS PKs used for functional grouping](https://www.postgresql.org/message-id/flat/CA%2BrenyWoOf6dM_%2Be6Zath%2B0i5Lr%2Buzn1hz7DAuSaV_A5Tz6XhA%40mail.gmail.com), started 2026-10-09
- Messages read: 6 of 6
- Patch prose read: v1-0001-Fix-bad-functional-grouping-inference-with-a-temp.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The patch fixes a wrong-results bug that a committer reported, and the reporter backed the narrow fix in message #4. One fixable design point survived, replication-ondisk-1: views stored on 18.x that rely on a WITHOUT OVERLAPS key are not re-checked and fail later at restore or pg_upgrade. That is a backpatch decision for a committer who knows the area, and it does not question the fix itself.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that the check accepts only btree indexes and leaves ordinary primary keys alone is unverified. Whether stored views are in fact not re-checked after the update rests on the surviving objection's description.

## The proposal, restated

**Problem.** The planner accepts a WITHOUT OVERLAPS primary key as proof of functional dependency for GROUP BY, but grouping uses the default btree equality and the key uses GiST equality. When they differ, as with citext, queries that select ungrouped columns are accepted and can return wrong results. Andres Freund reported it; it affects users of temporal primary keys.

**Proposed design.** check_functional_grouping accepts only btree indexes, so a WITHOUT OVERLAPS primary key no longer proves functional dependency. The author defers a finer check that compares each key column's equality operator with the grouping equality, to a later release unless reviewers accept it for backpatch.

**User-visible surface.**
- Queries that group by a WITHOUT OVERLAPS primary key and select other columns now fail with an error instead of returning rows.

**What the author claims.**
- The fix is simple enough to backpatch through 18.
- citext is the only built-in case found that shows the bug, because btree_gist has no citext opclass and GiST uses gist_text_ops by binary coercion.
- Matching GiST and btree equality procs could later allow functional grouping again.

**Thread state.** Yilin Zhang showed that a previously working query now errors; Andres Freund replied that this is the intended effect and backed the simpler fix. The rest of the thread is about whether bug reports belong in separate threads; Peter Eisentraut is the listed reviewer but has not posted.

## 2. Prior attempts

### Revert "Add GROUP BY ALL". (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a32733d8f10f0e35f7e4ee772ddbef552296577f](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a32733d8f10f0e35f7e4ee772ddbef552296577f)
- **Thread:** [https://www.postgresql.org/message-id/flat/5243308F-8E5C-45AA-828C-FAD96C4F34DA@gmail.com](https://www.postgresql.org/message-id/flat/5243308F-8E5C-45AA-828C-FAD96C4F34DA@gmail.com), [https://www.postgresql.org/message-id/flat/CAHM0NXjz0kDwtzoe-fnHAqPB1qA8_VJN0XAmCgUZ+iPnvP5LbA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAHM0NXjz0kDwtzoe-fnHAqPB1qA8_VJN0XAmCgUZ+iPnvP5LbA@mail.gmail.com)
- **Date:** decided 2026-07-17; started 2025-09-29
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit ef38a4d9756db9ae1d20f40aa39f3cf76059b81a which implemented the GROUP BY ALL syntax, as well as 2ce745836 which made some comment improvements therein. A postcommit review discovered that GROUP BY ALL missed our special handling of entries that also appear in an ORDER BY in the query. This caused the query to return wrong results when ORDER BY specifies non-default equality semantics. While this should be fixable with some refactoring, doing it cleanly seems like too much code churn for late beta. We'll revert and try again in v20. The reverted commit also included some additional comment wordsmithing and docs cleanup, which are retained as they weren't connected to the reverted feature. catversion bump needed due to change in struct Query.
- **Reason (quote):** "This caused the query to return wrong results when ORDER BY specifies non-default equality semantics."
- **What is different this time:** P11 reverted a new grouping feature that ignored non-default equality semantics. This patch removes an existing inference with the same class of flaw and also takes the narrow route.


## 3. Objections a reviewer is likely to raise

### 1. Stored views that use the temporal key break later dump and upgrade

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, on-disk format / pg_upgrade, catalog changes, backpatchability · `replication-ondisk-1`

On 18.0, a user creates a view that groups by a WITHOUT OVERLAPS primary key and selects other columns. After the minor update, the stored view still runs and can still return wrong results for citext. The failure appears later, when pg_dump restore or pg_upgrade to 19 recreates the view and hits the new error partway through.

Functional grouping is checked only at parse analysis, and the view records a pg_depend entry on the primary key constraint, so existing views are not re-validated. The commit message and message #1 cover new queries only, and the brief notes nothing about release notes. With Backpatch-through: 18, this catalog state is already in released clusters. Message #4 backs the simpler fix, but no message covers objects that already exist.

Thread: [#1 (Paul A Jungwirth, 2026-10-09)](https://www.postgresql.org/message-id/CA%2BrenyWoOf6dM_%2Be6Zath%2B0i5Lr%2Buzn1hz7DAuSaV_A5Tz6XhA%40mail.gmail.com), [#4 (Andres Freund, 2026-10-09)](https://www.postgresql.org/message-id/nwsx2isaffevptmws7anxv2xsc66rlmarom6a2bvc6jug3hsqh%40mtwkrt5r3psf)

**Question:** How will users on 18.x find views and rules that already depend on a WITHOUT OVERLAPS key for functional grouping, before a dump or pg_upgrade fails on them?

Counter-check: held. Postgres checks functional grouping only at parse analysis, so a view stored on 18.x keeps working until a restore or pg_upgrade re-parses it and fails. The thread and commit message do not cover stored views or release notes.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade re-creates stored views that group by a WITHOUT OVERLAPS key, and the new error can stop it. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch changes check_functional_grouping in pg_constraint.c; existing pg_depend entries from views on the temporal key stay. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The set of queries accepted under functional dependency for GROUP BY becomes smaller. |
| new GUC | no |  |
| planner behaviour change | **yes** | A WITHOUT OVERLAPS primary key no longer proves functional dependency, so some queries now fail at parse analysis. |
| pg_dump / restore round-trip | **yes** | A dump of a view created on 18.x that relies on the temporal key fails to restore on a patched server. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message says Backpatch-through: 18, and working queries start to fail after a minor update. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How will users on 18.x find views and rules that already depend on a WITHOUT OVERLAPS key for functional grouping, before a dump or pg_upgrade fails on them?
2. Will the release notes warn that queries grouping by a WITHOUT OVERLAPS primary key can start to fail after the minor update, and say how to rewrite them?
3. Should existing views that rely on a WITHOUT OVERLAPS key keep running after the minor update, given that they can still return wrong results for citext?
4. Does a citext column that gets case-sensitive GiST comparisons through gist_text_ops need a separate fix outside WITHOUT OVERLAPS?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund reported the bug and backed the simpler fix in the thread; Peter Eisentraut is the listed reviewer but has not posted. The owner needs to know parse analysis of GROUP BY and the temporal key code.
- **Liability:** The owner signs for a behaviour change in a minor release of 18: working queries start to fail, and stored views can fail at a later restore or pg_upgrade. The code change itself is small and adds nothing to maintain.
- **Author follow-up:** The author offers in message #1 to update the patch with the per-column equality check and to send follow-up patches for v20.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** This is a wrong-results fix for users of temporal primary keys, in one backend file plus regression tests. The open point is how the backpatch treats views already stored on 18.x, which a release note and a catalog query can cover.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Existing views on temporal keys may fail dump and restore after minor upgrade. Refuted (not specific): Duplicate of replication-ondisk-1, which describes the same stored-view failure more precisely.
- `standards-compat-2` Backpatch breaks correct range-type queries to fix a citext case. Refuted (already handled): Message #2 raised this exact case. In message #4 Andres Freund weighed it and backed the simpler fix because the feature is new, and the commit message calls the per-column check too ambitious for a backpatch.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposed the same idea. P11 shows a grouping feature reverted because it mixed equality semantics, which supports this fix rather than arguing against it, and P6 and P24 do not apply. Two points could not be checked here: the size of the backpatch behaviour change (message #2), and whether release notes will warn about queries that start to fail.
- standards-compat: 2 objection(s). Both objections concern how the fix is backpatched. Neither is against fixing the wrong-results bug.
- replication-ondisk: 1 objection(s). The patch adds no WAL, on-disk format or wire protocol change, and no new catalog column. The objection covers only catalog state that already exists in 18.x clusters.
- security: 0 objection(s). The patch has no privilege or exposure aspect. It makes the planner stricter about functional dependency, adds no object, privilege or setting, and gives no user access to data, sessions or resources they lacked before. Backpatch compatibility and pg_dump restore of existing views are other angles.
- maintenance-fit: 0 objection(s). From the maintenance angle, the patch narrows a feature to btree indexes in one file, and adds no GUC, API or hot-path cost. Yilin Zhang raised the queries that now fail (message #2), and Andres Freund answered that the simpler fix is intended (message #4). The backpatch break remains unsized, and the patch does not yet say whether release notes will warn about it.
- already-exists: 0 objection(s). This is a wrong-results bugfix that narrows check_functional_grouping to btree indexes, and no existing feature, extension or setting gives correct grouping results without it. Users who lose functional grouping can list the extra columns in GROUP BY, which is the usual alternative but does not replace the fix.
- demand: 0 objection(s). This is a wrong-results fix for a bug a committer reported, so demand is not in question. The cost to working temporal-key queries was raised in message #2, and the reporter accepted it in message #4.

### Precedents retrieved

- P1: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P2: [deferred primary key and logical replication](https://commitfest.postgresql.org/patch/2631/) (cf, Returned with feedback, 2020-11-29) - judged not relevant; reason summarised from its thread
- P3: [Dump-restore loosing 'attnotnull' bit for DEFERRABLE PRIMARY KEY column(s)](https://commitfest.postgresql.org/patch/4729/) (cf, Rejected, 2023-12-28) - judged not relevant; reason summarised from its thread
- P4: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P5: [Add sortsupport for range types and btree_gist](https://commitfest.postgresql.org/patch/3686/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P6: [Remove useless GROUP BY columns considering unique index](https://commitfest.postgresql.org/patch/4751/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P7: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P8: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant; reason summarised from its thread
- P9: [Temporal foreign key actions](https://commitfest.postgresql.org/patch/7238/) (cf, Withdrawn, 2026-09-17) - judged not relevant; reason summarised from its thread
- P10: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P11: [Revert "Add GROUP BY ALL".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a32733d8f10f0e35f7e4ee772ddbef552296577f) (revert, Reverted, 2026-07-17) - used in the review
- P12: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P13: [Indeterminate behavior for the GROUP BY clause](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P14: [remove_useless_groupby_columns is too enthusiastic](https://commitfest.postgresql.org/patch/3766/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P15: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P16: [Invalid permission check in pg_stats for functional indexes](https://commitfest.postgresql.org/patch/2274/) (cf, Returned with feedback, 2020-07-05) - judged not relevant
- P17: [Parallel CREATE INDEX for GiST indexes](https://commitfest.postgresql.org/patch/5026/) (cf, Returned with feedback, 2025-02-06) - judged not relevant
- P18: [Add GiST support for mixed-width integer operators](https://commitfest.postgresql.org/patch/5113/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P19: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P20: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P21: [\[PATCH\] reduce page overlap of GiST indexes built using sorted method](https://commitfest.postgresql.org/patch/3487/) (cf, Returned with feedback, 2022-01-09) - judged not relevant
- P22: [ANALYZE: hash-accelerate MCV tracking for equality-only types](https://commitfest.postgresql.org/patch/6422/) (cf, Withdrawn, 2026-05-03) - judged not relevant
- P23: [Remove extra Sort node above a btree-compatible Index Scan](https://commitfest.postgresql.org/patch/5606/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P24: [Fix GROUP BY ALL handling of ORDER BY operator semantics](https://commitfest.postgresql.org/patch/6952/) (cf, Withdrawn, 2026-07-16) - used in the review

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 87 s; cost reported by the CLI: $0.99
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 57 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:58:07.063Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
