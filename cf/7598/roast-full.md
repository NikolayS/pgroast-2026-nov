# Design pre-review: Improve row mark handling in self-join elimination

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7598](https://commitfest.postgresql.org/patch/7598/) (Needs review); authors: Tender Wang (tndrwang)
- Thread: [Self-join elimination discards differing row lock wait policies](https://www.postgresql.org/message-id/flat/CAHewXNn989Q4kP8Ewaeo8cBp-8jK6KX8CRs5UX6rAPMBtx54Pg%40mail.gmail.com), started 2026-10-24
- Messages read: 7 of 7
- Patch prose read: v3-0001-Clarify-row-mark-handling-in-self-join-eliminatio.patch, v3-0002-Allow-self-join-elimination-with-different-row-lo.patch, 0001-Preserve-row-lock-semantics-during-self-join-elimina.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

Only demand-1 survived, and it applies to 0002 alone. The thread shows no user query that self-joins one table with different lock strengths per alias, and message #7 does not answer Tom Lane's doubt in message #4 that the merging is worth the trouble. 0001 removes unreachable code and expands comments, and no objection to it survived. The verdict rests on missing evidence of demand, so confidence is low.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check that the assertions in 0001 hold for every path into self-join elimination. I also could not check that the merge rule in 0002 matches a legal plan without self-join elimination for every pair of strength and wait policy.

## The proposal, restated

**Problem.** The thread started as a bug report: SJE drops one wait policy when two aliases use NOWAIT and SKIP LOCKED. Tom Lane argued this is not a reachable bug, and the author agreed. What remains is unreachable code and thin comments in analyzejoins.c, and the fact that differing lock strengths block SJE, so the outcome depends on join order and the weaker lock taken first can deadlock on upgrade.

**Proposed design.** 0001 removes the unreachable code that moves the removed relation's row mark when the kept relation has none, adds assertions for the invariant, and expands comments. 0002 lets SJE merge locking row marks of different strengths. It keeps the stronger lock and its wait policy, and for equal strengths keeps the stronger wait policy, as applyLockingClause() does.

**User-visible surface.**
- Plan change: queries with FOR UPDATE / FOR SHARE variants of different strength on two self-joined aliases can now get self-join elimination.

**What the author claims.**
- preprocess_rowmarks() gives every non-target base relation a row mark when any are needed, so SJE candidates either both have marks or both lack them.
- Merging into the stronger lock avoids the lock-upgrade deadlock risk of taking the weaker lock first.
- The wait-policy rule follows applyLockingClause().
- The original NOWAIT versus SKIP LOCKED example is not a correctness bug.

**Thread state.** Tom Lane analyzed the case, shared a WIP patch that 0002 builds on, and doubted the strength merging is worth the trouble. No one has reviewed v3, and a separate idea about constant-restricted self-joins was judged mostly cosmetic.

## 2. Prior attempts

### Revert: Remove useless self-joins (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d286d83c0eed695910cb20d970ea9bea2e5001](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d286d83c0eed695910cb20d970ea9bea2e5001)
- **Thread:** [https://www.postgresql.org/message-id/flat/2422119.1714691974@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/2422119.1714691974@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/flat%2F64486b0b-0404-e39e-322d-0801154901f3@postgrespro.ru](https://www.postgresql.org/message-id/flat/flat%2F64486b0b-0404-e39e-322d-0801154901f3@postgrespro.ru), [https://www.postgresql.org/message-id/flat/flat%2F1189851.1698340331@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/flat%2F1189851.1698340331@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/CAMbWs4_STsG1PKQBuvQC8W4sPo3KvML3=jOTjKLUYQuK3g8cpQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAMbWs4_STsG1PKQBuvQC8W4sPo3KvML3=jOTjKLUYQuK3g8cpQ@mail.gmail.com), [https://www.postgresql.org/message-id/flat/flat%2F18187-831da249cbd2ff8e@postgresql.org](https://www.postgresql.org/message-id/flat/flat%2F18187-831da249cbd2ff8e@postgresql.org), [https://www.postgresql.org/message-id/flat/CAMbWs4-ns73VF9gi37q61G3dS6Xuos+HtryMaBh37WQn=BsaJw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAMbWs4-ns73VF9gi37q61G3dS6Xuos+HtryMaBh37WQn=BsaJw@mail.gmail.com)
- **Date:** decided 2024-05-06; started 2023-10-25
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This commit reverts d3d55ce5713 and subsequent fixes 2b26a694554, 93c85db3b5b, b44a1708abe, b7f315c9d7d, 8a8ed916f73, b5fb6736ed3, 0a93f803f45, e0477837ce4, a7928a57b9f, 5ef34a8fc38, 30b4955a466, 8c441c08279, 028b15405b4, fe093994db4, 489072ab7a9, and 466979ef031. We are quite late in the release cycle and new bugs continue to appear. Even though we have fixes for all known bugs, there is a risk of throwing many bugs to end users. The plan for self-join elimination would be to do more review and testing, then re-commit in the early v18 cycle.
- **Reason (quote):** "We are quite late in the release cycle and new bugs continue to appear."
- **What is different this time:** P2 reverted the whole self-join elimination feature late in a release cycle because bugs kept appearing. This proposal is a narrow change to row mark handling in the re-committed feature, and 0001 only removes code.


## 3. Objections a reviewer is likely to raise

### 1. No user is shown writing self-joins with differing lock strengths

fixable · level 1b, rubric item 3 · demand, planner behaviour change, maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer pushes 0002, which adds merge logic to the row-mark handling in analyzejoins.c, an area where SJE bugs have been found. Later fixes to that logic cost committer time and may need backpatching. The thread names no query that gains from the change, so the payoff for that cost is not known.

The only queries in the thread come from developers: message #1's example uses equal strengths, and its supposed bug was withdrawn in message #5. Tom Lane raised strength merging as a possible improvement in message #3, then wrote in message #4 that he was 'not really convinced that it's worth the trouble'. Message #7 does not answer that doubt; it ports his WIP code. The deadlock-on-upgrade argument is sound in principle, but no report shows a user who hit it.

Thread: [#3 (Tom Lane, 2026-10-24)](https://www.postgresql.org/message-id/142952.1792872550%40sss.pgh.pa.us), [#4 (Tom Lane, 2026-10-25)](https://www.postgresql.org/message-id/281282.1792889795%40sss.pgh.pa.us), [#5 (Tender Wang, 2026-10-26)](https://www.postgresql.org/message-id/CAHewXN%3Dh3DmQ9Fc_aJN9dD%2BAbE90v%3Df4zmhSt_1tj1aVGTTzaQ%40mail.gmail.com), [#7 (Tender Wang, 2026-10-26)](https://www.postgresql.org/message-id/CAHewXN%3DofNonQSvzJ5a0csWtFJ%3DN5SAY_nJGPuNhkgKCs8svSg%40mail.gmail.com)

**Question:** Which real applications or ORMs emit self-joins on a unique key with different FOR UPDATE/FOR SHARE strengths per alias, and has anyone reported the lock-upgrade deadlock?

Counter-check: held. The thread shows no user query with mixed lock strengths on self-joined aliases, and message #7 does not answer Tom Lane's doubt in message #4. The deadlock and join-order points in message #3 give a motive but no evidence of demand.

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
| planner behaviour change | **yes** | 0002 lets self-join elimination apply when two aliases carry locking row marks of different strength, so those plans change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The thread does not say whether 0002, a plan and locking behaviour change, targets master only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0002 adds a merge rule for lock strength and wait policy to the row mark handling in analyzejoins.c. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which real applications or ORMs emit self-joins on a unique key with different FOR UPDATE/FOR SHARE strengths per alias, and has anyone reported the lock-upgrade deadlock?
2. When the two aliases differ in both lock strength and wait policy, does the merged row mark always behave like some legal join order of the plan without self-join elimination?
3. Is 0002 meant for master only, and is 0001 meant to be committed alone if 0002 waits?
4. Which combinations of lock strength and wait policy do the new tests in join.sql cover?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane analyzed the case and wrote the WIP patch that 0002 builds on, but he has not reviewed v3 and doubted in message #4 that 0002 is worth it. The owner needs to know planner row marks and self-join elimination.
- **Liability:** For 0001, the new assertions that preprocess_rowmarks() gives both candidates a row mark or neither. For 0002, the correctness of merged lock strength and wait policy under concurrency, for as long as self-join elimination exists.
- **Author follow-up:** The author revised the patch to v3 after agreeing with the committer's analysis and created the CommitFest entry. There is no explicit statement about post-commit fixes.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: For 0002: a field report, a -general or -bugs thread, or an ORM-generated query that self-joins a table with different lock strengths per alias and hits the lock-upgrade deadlock or join-order dependence. A statement from Tom Lane that he now wants the merging would also change it. 0001 needs no such evidence.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** 0001 is a small cleanup in one file that follows the committer's own analysis and can go ahead alone. 0002 is also small but serves a query shape no user has reported, so it should wait for evidence of demand.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Lock-strength merging in SJE adds planner code with no shown demand. Refuted (not specific): Duplicate of demand-1. [P2] records a late-cycle revert of the whole SJE feature, which was planned for re-commit, so it must not be used as support against 0002.
- `maintenance-fit-2` Second copy of the locking-clause merge rule outside applyLockingClause. Refuted (no failure scenario): applyLockingClause() combines by Max() over ordered enums, so a new value placed in order reaches any comparison-based copy. By message #3, the SJE rule only needs to match a legal non-SJE plan, so drift from the parser is not a failure.
- `already-exists-1` Users can already get SJE by writing one lock strength on both aliases. Refuted (not specific): Duplicate of demand-1. The rewrite argument applies to all of SJE, since a user can always remove the self-join by hand, so it adds nothing beyond the demand question.

### Review angles

- prior-attempts: 0 objection(s). The only precedent, [P2], reverted the whole SJE feature because of late-cycle bug risk, and that reason does not carry over to a small row mark change posted early in a cycle. The thread points to no earlier attempt at merging lock strengths, apart from the WIP patch in message #6. Tom Lane's doubt about whether 0002 is worth it (message #4) is a desirability question for the other angles.
- standards-compat: 0 objection(s). No objection from the standards and compatibility angle. 0002 adds no syntax, keyword, GUC, catalog object or dump content. The visible change is that some queries with locking clauses of different strength on two self-joined aliases now get self-join elimination. Per Tom Lane's argument in message #3, the merged lock strength and wait policy still behave like a legal non-SJE join order. The rule also matches the existing rule for repeated locking clauses on one table in applyLockingClause(), and enable_self_join_elimination remains an escape hatch. Points not checkable from the material: whether 0002 is meant for master only, since a plan change in a minor release would be unwelcome; and whether the SELECT locking-clause documentation should say that the strongest-clause rule now also covers aliases merged by SJE. The precedent [P2] concerns the revert of SJE as a whole for late-cycle bug risk, so it does not apply to this contract question.
- replication-ondisk: 0 objection(s). Both patches change only planner logic in analyzejoins.c plus regression tests. They add no catalog column, WAL record, on-disk format, wire message or struct change. The row locks taken are ordinary tuple locks under existing rules, and FOR UPDATE is not allowed on a hot standby, so this angle finds nothing persistent or cross-node to object to.
- security: 0 objection(s). No security objection. The merged lock is a strength the same query already asked for on the same table, so Postgres checked the needed privilege at parse time and no user gets a lock or data they could not get before. The proposal adds no new privilege, default or exposure. Not checked without the code: whether a view-derived alias with a different permission owner keeps its own checks.
- maintenance-fit: 2 objection(s). 0001 (dead code removal, assertions, comments) has no objection from this angle and follows Tom Lane's analysis in message #4.
- already-exists: 1 objection(s). 0001 is a cleanup of unreachable code and comments, and no existing feature or extension replaces it. The only objection from this angle concerns 0002.
- demand: 1 objection(s). 0001 is a cleanup of unreachable code and comments that Tom Lane asked for, so it needs no evidence of demand. The objection applies to 0002 only.

### Precedents retrieved

- P1: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant; reason summarised from its thread
- P2: [Revert: Remove useless self-joins](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d286d83c0eed695910cb20d970ea9bea2e5001) (revert, Reverted, 2024-05-06) - used in the review
- P3: [Document efficient self-joins / UPDATE LIMIT techniques.](https://commitfest.postgresql.org/patch/4539/) (cf, Returned with feedback, 2024-02-13) - judged not relevant; reason summarised from its thread
- P4: [Revert "Put enable_self_join_elimination into postgresql.conf.sample"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a1a7c5a7071c75676c15b26e242c7df17560bd1) (revert, Reverted, 2025-04-06) - judged not relevant
- P5: [Adding a { LOCK ROWS | LOCK KEY INDEX } clause to FK definitions](https://commitfest.postgresql.org/patch/6923/) (cf, Withdrawn, 2026-09-09) - judged not relevant
- P6: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant; reason summarised from its thread
- P7: [Allow multiple recursive self-references](https://commitfest.postgresql.org/patch/3046/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P8: [Self contradictory examining on rel's baserestrictinfo](https://commitfest.postgresql.org/patch/5404/) (cf, Returned with feedback, 2025-01-23) - judged not relevant; reason summarised from its thread
- P9: [Redundant qualifier elimination](https://commitfest.postgresql.org/patch/7127/) (cf, Returned with feedback, 2026-08-14) - judged not relevant; reason summarised from its thread
- P10: [LockHasWaiters() crashes on fast-path locks](https://commitfest.postgresql.org/patch/6732/) (cf, Withdrawn, 2026-09-03) - judged not relevant; reason summarised from its thread
- P11: [Remove LEFT JOINs in more cases](https://commitfest.postgresql.org/patch/1364/) (cf, Returned with feedback, 2018-09-28) - judged not relevant; reason summarised from its thread
- P12: [INNER JOIN removals](https://commitfest.postgresql.org/patch/130/) (cf, Returned with feedback, 2015-05-11) - judged not relevant
- P13: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P14: [Free list same_input_transnos in preprocess_aggref](https://commitfest.postgresql.org/patch/3920/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P15: [BRIN - SK_SEARCHARRAY and scan key preprocessing](https://commitfest.postgresql.org/patch/4187/) (cf, Returned with feedback, 2024-01-15) - judged not relevant
- P16: [NOT IN to ANTI JOIN transformation](https://commitfest.postgresql.org/patch/2023/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P17: [Adjust pg_stat_get_lock() prorows to match lock types](https://commitfest.postgresql.org/patch/6740/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P18: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P19: [Revert misguided change to postgres_fdw FOR UPDATE/SHARE code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8ec8760fc87ecde0516e511f1c55aec627b01ea7) (revert, Reverted, 2014-12-12) - judged not relevant
- P20: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1133/) (cf, Returned with feedback, 2018-03-30) - judged not relevant
- P21: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1666/) (cf, Returned with feedback, 2018-12-01) - judged not relevant
- P22: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P23: [Batch processing of RI checks](https://commitfest.postgresql.org/patch/2581/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P24: [Better estimate merging for duplicate vars in clausesel.c](https://commitfest.postgresql.org/patch/1036/) (cf, Returned with feedback, 2017-10-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 21; wall time: 170 s; cost reported by the CLI: $1.89
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 63 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:03:56.087Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
