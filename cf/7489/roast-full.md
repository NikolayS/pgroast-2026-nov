# Design pre-review: GIN jsonb_ops misses root strings in strict jsonpath queries

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7489](https://commitfest.postgresql.org/patch/7489/) (Needs review); authors: Rui Zhao (zhaorui)
- Thread: [GIN jsonb_ops misses root strings in strict jsonpath queries](https://www.postgresql.org/message-id/flat/CAHWVJhHH0k3PxWmS%3D%3DV1p_PONOMUaSm6LjUV487fwnOqd0jTQw%40mail.gmail.com), started 2026-10-20
- Messages read: 3 of 3
- Patch prose read: v1-0001-Fix-GIN-lookups-for-root-strings-in-strict-jsonpa.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at any level. The proposal fixes wrong query results: a jsonb_ops index scan misses a root string that a sequential scan returns, and two people hit it independently (messages #1 and #2). The fix changes query-side key extraction only, so existing indexes match without REINDEX. Message #3 reports that it was pushed.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the review could not check root non-string scalars in strict mode or jsonb_path_ops. Message #3 does not say what the editorialization changed.

## The proposal, restated

**Problem.** A GIN jsonb_ops index scan returns no rows for strict-mode jsonpath queries that match a root string, such as @@ 'strict $ == "a scalar"'. A sequential scan returns the row. Users of jsonb_ops indexes with strict jsonpath get wrong results.

**Proposed design.** gin_extract_jsonb() indexes a root string with JGINFLAG_KEY because a root scalar is stored as a one-element raw-scalar array. In strict mode, jsonb_ops__extract_nodes() searches for JGINFLAG_STR at the root, so the patch sets key_entry to GIN_TRUE for that case. Regression tests for @@ and @? are added to jsonb.sql.

**User-visible surface.**
- Strict-mode @@ and @? queries on a jsonb_ops GIN index now return matching root strings; no syntax or API change.

**What the author claims.**
- Existing indexes match after the fix; no REINDEX is needed.
- Only strict-mode queries using jsonb_ops are affected.
- make check passed all 239 tests; without the code fix only the two new indexed queries fail.
- Reproduced on master at e75fcc9867f.

**Thread state.** Shihao Zhong hit the same bug, reached the same one-line fix, and gave +1. Tom Lane replied that he pushed it with minor editorialization, so the CommitFest status "Needs review" is out of date.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-20; 24 candidates retrieved, none is the same or a similar idea).

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
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest target is "stable", and the thread does not say which back branches received the fix. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which back branches received the fix, given that the CommitFest target is "stable"?
2. Do root numbers, booleans and null in strict mode have the same mismatch between gin_extract_jsonb() and jsonb_ops__extract_nodes()?
3. Does jsonb_path_ops return the matching root string for the same two strict-mode queries?
4. Should the CommitFest entry be marked committed, since message #3 says the fix was pushed?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane pushed the fix with minor editorialization (message #3).
- **Liability:** One changed line in the query-side key extraction of jsonb_ops, plus two regression queries. It adds no on-disk format, catalog, WAL or API surface to carry.
- **Author follow-up:** No sign either way; the fix was pushed four days after the report without a further message from the author.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is a one-line wrong-results fix in jsonb_gin.c with two regression queries, and it affects only strict-mode jsonpath queries on jsonb_ops indexes. Message #3 says it was pushed, so the remaining work is to update the CommitFest entry, which still says "Needs review".

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the retrieved candidates share keywords only, and the thread points to no earlier discussion of this bug. It is a narrow wrong-results fix in jsonb_ops strict-mode extraction, a second reporter reached the same fix (message #2), and message #3 reports it was pushed.
- standards-compat: 0 objection(s). No objection from the standards and compatibility angle. The fix makes index scans return the rows a sequential scan already returns, and adds no syntax, GUC, catalog object or psql change. Existing indexes need no REINDEX and pg_dump output does not change. Not checked: root numbers, booleans and null in strict mode, and jsonb_path_ops; the thread does not cover either.
- replication-ondisk: 0 objection(s). The fix changes only query-side key extraction in jsonb_ops__extract_nodes(), so existing jsonb_ops index entries match without REINDEX. It adds no WAL record, catalog change, on-disk format change or ABI change. A standby still on an older minor release keeps the old wrong results until it is updated, which is the existing bug rather than a new hazard.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes which GIN key a strict-mode jsonpath query looks up for a root string, so an index scan returns rows a sequential scan already returns. It adds no object, privilege, function or setting, and it does not change what a user can read through RLS, column privileges or leakproofness. It adds no new memory, lock or worker cost. The fix was already pushed per message #3.
- maintenance-fit: 0 objection(s). This is a one-line bug fix for wrong query results in jsonb_ops GIN lookups. It adds no new state, GUC, API, on-disk change or hot-path cost, and needs no REINDEX. Tom Lane, a committer who knows the area, pushed it (message #3), so the area has an engaged owner. From the maintenance and fit angle there is nothing to object to. Not checked without the code: whether root non-string scalars in strict mode, or jsonb_path_ops, have the same mismatch.
- already-exists: 0 objection(s). This is a wrong-results bugfix in jsonb_ops GIN lookups, and no existing feature, setting or extension gives correct results for these indexed queries. Avoiding the index or using jsonb_path_ops changes query plans or index choice, so it is not a substitute for fixing the bug.
- demand: 0 objection(s). This is a wrong-results bug fix, and two people hit it independently (messages #1 and #2). A fix for wrong query results needs no further evidence of demand, and the change adds no surface or hot-path cost. It was pushed in message #3.

### Precedents retrieved

- P1: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - judged not relevant; reason summarised from its thread
- P2: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P3: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant; reason summarised from its thread
- P4: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P5: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant; reason summarised from its thread
- P6: [Fillfactor for GIN indexes](https://commitfest.postgresql.org/patch/13/) (cf, Rejected, 2015-01-16) - judged not relevant; reason summarised from its thread
- P7: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P8: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P9: [Add jsonpath string method $.replace(from, to)](https://commitfest.postgresql.org/patch/5261/) (cf, Withdrawn, 2024-09-25) - judged not relevant; reason summarised from its thread
- P10: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant
- P11: [Revert "Add gitignore entries for jsonpath_gram.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7b084b38310cfe9c8b58cc615a81df625c771f5d) (revert, Reverted, 2019-03-22) - judged not relevant
- P12: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P13: [jsonb array-style subscription](https://commitfest.postgresql.org/patch/485/) (cf, Returned with feedback, 2016-03-21) - judged not relevant
- P14: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P15: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant
- P16: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P17: [Revert "Reorganise jsonpath operators and methods"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d) (revert, Reverted, 2024-01-03) - judged not relevant
- P18: [Optimize numeric comparison and aggregation via packed-datum extraction](https://commitfest.postgresql.org/patch/6651/) (cf, Rejected, 2026-05-10) - judged not relevant
- P19: [Key management with tests](https://commitfest.postgresql.org/patch/2925/) (cf, Withdrawn, 2021-07-15) - judged not relevant
- P20: [Temporal foreign key actions](https://commitfest.postgresql.org/patch/7238/) (cf, Withdrawn, 2026-09-17) - judged not relevant
- P21: [Optimizing Node Files Support](https://commitfest.postgresql.org/patch/4034/) (cf, Rejected, 2023-03-21) - judged not relevant
- P22: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/1252/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P23: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/2966/) (cf, Returned with feedback, 2021-09-14) - judged not relevant
- P24: [COPY RAW](https://commitfest.postgresql.org/patch/223/) (cf, Returned with feedback, 2016-04-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 52 s; cost reported by the CLI: $0.45
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:00:35.459Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
