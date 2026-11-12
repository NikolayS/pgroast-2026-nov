# Design pre-review: Parameterized append subpaths

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7339](https://commitfest.postgresql.org/patch/7339/) (Needs review); authors: Alexander Pyhalov (a.pyhalov), Gleb Kashkin (glebka)
- Thread: [Parameterized append subpaths](https://www.postgresql.org/message-id/flat/588d117fb042bbae3a6e7b4379ab8ec7%40postgrespro.ru), started 2026-09-09
- Messages read: 3 of 3
- Patch prose read: 0001-Preserve-all-differently-parameterized-paths.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at level 1: message #1 shows a hash join of about 1739 ms chosen over a 2.7 ms parameterized nested loop because a path was lost. Two fixable design points remain. maintenance-fit-1 applies only to the add_path() patch, which message #2 already moves away from. maintenance-fit-2 asks for the contract of the new FDW callback, and message #3 supports that design.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The patch prose for the two patches in message #2 is not in the material, so the callback's contract and the cache's key and lifetime could not be checked. Whether the reparameterized path gets correct costs and quals needs the code.

## The proposal, restated

**Problem.** add_path() can discard a parameterized foreign scan path when its cost and row count tie with a non-parameterized one. If one partition loses its parameterized path, the planner cannot build a parameterized Append, so it picks a hash join over a cheaper parameterized nested loop. This hits partitioned tables with postgres_fdw partitions and use_remote_estimate on.

**Proposed design.** Patch 0001 (Pyhalov, WIP) changes add_path() in pathnode.c to keep all differently parameterized paths. The alternative (Kashkin) keeps add_path() as is and adds an FDW callback so reparameterize_path() can rebuild a parameterized foreign scan path when needed. An optional postgres_fdw patch caches parameterized path cost estimates to avoid repeat remote EXPLAIN round trips.

**User-visible surface.**
- New FDW API callback to reparameterize foreign scan paths, called from reparameterize_path() (alternative design).
- postgres_fdw implements the new callback and caches remote cost estimates for parameterized paths (optional patch).
- Plan choice changes: queries may get a parameterized nested loop where they got a hash join before.

**What the author claims.**
- In the example, the hash join takes about 1739 ms and the parameterized nested loop plan takes about 2.7 ms.
- The planner chose the hash join because a parameterized path was missing, not because of cost.
- Reparameterization fits the current path dominance model and builds extra foreign paths only when needed.
- The cache avoids repeated remote EXPLAIN calls for paths already costed and rejected.

**Thread state.** Etsuro Fujita gave +1 to the reparameterization approach and pointed to his own similar FDW API proposal from the foreign-table-inheritance work. Nobody has answered him, and nobody has reviewed the first patch.

## 2. Prior attempts

### A hook for path-removal decision on add_path (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2234/](https://commitfest.postgresql.org/patch/2234/)
- **Date:** decided 2021-10-05; started 2019-08-12
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P2 proposed an extension hook to decide path removal in add_path(). Patch 0001 changes the dominance rule in core, and the alternative design leaves add_path() alone and reparameterizes foreign paths through a new FDW callback.


## 3. Objections a reviewer is likely to raise

### 1. Keeping all parameterized paths in add_path() costs every query

fixable · level 2, rubric item 8 · performance on the hot path, planner behaviour change · `maintenance-fit-1`

A user runs a 10-way join on local tables with many indexes and no foreign tables. Every base and join rel now keeps every differently parameterized path, so pathlists grow and join search does more work. The user sees higher planning time and memory after upgrade, with no plan change.

Patch 0001 changes add_path() in pathnode.c, which every rel of every query goes through. The problem shown in message #1 is specific to postgres_fdw partitions with use_remote_estimate, but this fix puts the cost on all planning. No planning-time or memory numbers are given. Message #2 already offers a narrower design, which suggests 0001 is not needed for the stated problem.

Thread: [#1 (Alexander Pyhalov, 2026-09-09)](https://www.postgresql.org/message-id/588d117fb042bbae3a6e7b4379ab8ec7%40postgrespro.ru), [#2 (Gleb Kashkin, 2026-09-23)](https://www.postgresql.org/message-id/98f84e7438c5b78a5a1c77539ffc7e0b%40postgrespro.ru)

**Question:** What planning-time and memory change does 0001 cause on join-heavy queries with no foreign tables, such as a many-way join on indexed local tables?

Counter-check: held. Patch 0001 touches only pathnode.c, and add_path() runs for every rel of every query. The thread gives no planning-time or memory numbers, and message #2 itself calls the narrower design a better fit for the current model.

### 2. New FDW callback is a permanent API with no shown contract

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

An FDW author implements the callback and returns a path whose cost or restrictions do not match the new required_outer. reparameterize_path() trusts it, and the Append is built on wrong costs or wrong quals. Nothing in core checks the result, so the bug shows up as wrong plans or wrong results in the field.

Message #2 adds an FDW callback called from reparameterize_path(), but the material shows no name, signature, documentation, or rules on what the FDW must return or may refuse. Once released, FdwRoutine members are hard to change or remove. Message #3 points to an earlier proposal of the same API from the foreign-table-inheritance work, and no comparison is given. A committer would likely want one agreed contract before adding a member to FdwRoutine.

Thread: [#2 (Gleb Kashkin, 2026-09-23)](https://www.postgresql.org/message-id/98f84e7438c5b78a5a1c77539ffc7e0b%40postgrespro.ru), [#3 (Etsuro Fujita, 2026-09-23)](https://www.postgresql.org/message-id/CAPmGK15wGDUj_gnzaS8JrL7xyDt19FGsfDs8nrSUii%2B1enKcCg%40mail.gmail.com)

**Question:** What are the callback's signature and documented contract, and how does it differ from the earlier FDW API proposal mentioned in message #3?

Counter-check: held. The failure scenario is weak, because core trusts every FDW path callback the same way. The request stands: the material shows no signature or contract, and message #3 asks for a comparison with the earlier API that nobody has answered.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The alternative design adds a new FDW callback that reparameterize_path() calls. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | Both designs change which paths survive, so some queries get a parameterized nested loop where they got a hash join. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | A new FDW callback or a changed add_path() dominance rule limits the fix to master. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new FDW API member and a cost cache in postgres_fdw stay for good. |
| performance on the hot path | **yes** | Patch 0001 from message #1 changes add_path() in pathnode.c, which every query's planning goes through. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which design do you plan to pursue: the add_path() change from message #1 or the FDW reparameterization callback from message #2?
2. If the add_path() patch stays, what planning-time and memory change does it cause on a many-way join of indexed local tables with no foreign tables?
3. What are the callback's signature and documented contract, including when an FDW may decline to reparameterize a path?
4. How does the callback differ from the FDW API proposed in the foreign-table-inheritance work that message #3 points to?
5. What is the postgres_fdw cost cache keyed on, and is it freed at the end of each planning run?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Etsuro Fujita gave +1 to the callback design in message #3 and says he proposed this FDW API before. The add_path() patch would need someone who knows planner path dominance, and nobody has reviewed it.
- **Liability:** The owner signs for a new FDW API member that external FDWs may implement and that is hard to change after release. The owner also signs for plan changes in postgres_fdw queries and for the correctness of the cost cache.
- **Author follow-up:** Two authors from the same company are active: the second posted a new design two weeks after the first message and created the CommitFest entry. Fujita's question in message #3 has no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The defect affects partitioned tables with postgres_fdw partitions and use_remote_estimate, but the shown plan is about 600 times slower than the lost one. The callback design keeps the cost on FDW users and has support in message #3. The patches from message #2 were not read, so review cost is an estimate.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-3` postgres_fdw cost cache has no stated key or lifetime. Refuted (no failure scenario): The scenario depends on a cache that outlives one planning run, and the proposal does not say it does. Message #2 describes reuse for paths "already costed and rejected", which points to the same planning run, and the objection concedes it then goes away.

### Review angles

- prior-attempts: 0 objection(s). No precedent records a reason that applies here. P2 and P9 touch add_path() but have no recorded reason, and P1 was returned only because it bitrotted. Message #3 says Fujita proposed a similar FDW reparameterize API during the foreign-table-inheritance work, but the thread does not say why it was not adopted. A committer would likely ask for a comparison with that proposal, but the proposal gives no reason against this design.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to: no SQL syntax, GUC, catalog object, pg_dump or psql change is proposed. Plan changes for existing queries are the intended fix, and an optional FDW callback follows existing FdwRoutine practice. The callback signature and the cache keying and invalidation in postgres_fdw are not shown, so the client-visible effect of cached remote estimates could not be checked.
- replication-ondisk: 0 objection(s). Both designs only change planner path selection and add an in-memory FDW callback and cost cache. Neither touches on-disk format, WAL, catalogs or the wire protocol. A new FdwRoutine member would limit the alternative to master, which is normal for a new feature, and existing FDWs leave it unset.
- security: 0 objection(s). The proposal changes path retention in add_path() and adds an FDW callback for reparameterization. Neither touches privileges, catalogs, authentication or user-visible data, so I see no security objection. One point cannot be checked without the code: the postgres_fdw estimate cache should be keyed by user mapping, so a role that switches users with SET ROLE in a session never reuses remote estimates taken under another role's credentials.
- maintenance-fit: 3 objection(s). Committability: Etsuro Fujita, who knows postgres_fdw, is engaged and supports the reparameterization design. This lowers the maintenance concern for the callback approach. The objections above are about 0001's cost and the unspecified callback and cache contracts.
- already-exists: 0 objection(s). Postgres has no existing way to get this plan reliably. Settings such as enable_hashjoin = off and hint extensions only force a plan, and a set_rel_pathlist_hook path would still be discarded by add_path(). The alternative design extends reparameterize_path(), which is the existing mechanism, so it adds no second way to do the same job.
- demand: 0 objection(s). Message #1 shows a real plan-quality defect: a hash join of about 1739 ms is chosen over a parameterized nested loop of about 2.7 ms. The cause is a lost parameterized path, not cost. The trigger is common in postgres_fdw sharding, where an empty or near-empty partition sits next to large ones under use_remote_estimate. The interest has lasted: in message #3, Fujita supports the reparameterization approach and says he proposed the same FDW API years ago. Costs mostly fall on FDW users, except that patch 0001 changes add_path() for every query, which is a design question for other angles. One demand gap is small: the optional postgres_fdw cache comes with no numbers showing that repeated remote EXPLAIN calls cost real planning time. That gap does not justify a rejection.

### Precedents retrieved

- P1: [postgres_fdw: support parameterized foreign joins](https://commitfest.postgresql.org/patch/1042/) (cf, Returned with feedback, 2017-11-29) - used in the review; reason summarised from its thread
- P2: [A hook for path-removal decision on add_path](https://commitfest.postgresql.org/patch/2234/) (cf, Returned with feedback, 2021-10-05) - used in the review
- P3: [Async execution of postgres_fdw.](https://commitfest.postgresql.org/patch/32/) (cf, Rejected, 2015-05-09) - judged not relevant; reason summarised from its thread
- P4: [Asynchronous execution on postgres-fdw](https://commitfest.postgresql.org/patch/291/) (cf, Rejected, 2015-08-10) - judged not relevant; reason summarised from its thread
- P5: [Push down more UPDATEs/DELETEs in postgres_fdw](https://commitfest.postgresql.org/patch/791/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P6: [postgres_fdw: evaluate placeholdervars on remote server](https://commitfest.postgresql.org/patch/1055/) (cf, Returned with feedback, 2017-09-27) - judged not relevant; reason summarised from its thread
- P7: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2409/) (cf, Returned with feedback, 2020-01-28) - judged not relevant; reason summarised from its thread
- P8: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [Refactor add_path for clarity and extensibility](https://commitfest.postgresql.org/patch/3599/) (cf, Returned with feedback, 2022-10-12) - used in the review
- P10: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant; reason summarised from its thread
- P11: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2499/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P12: [Consider incremental sort for fractional paths in generate_orderappend_paths](https://commitfest.postgresql.org/patch/3513/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P13: [postgres_fdw super user checks](https://commitfest.postgresql.org/patch/824/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P14: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/167/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P15: [apply_scanjoin_target_to_paths and partitionwise join](https://commitfest.postgresql.org/patch/4929/) (cf, Returned with feedback, 2025-01-13) - judged not relevant
- P16: [Refactoring postgres_fdw/connection.c](https://commitfest.postgresql.org/patch/3782/) (cf, Withdrawn, 2023-05-23) - judged not relevant
- P17: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/20/) (cf, Rejected, 2015-01-16) - judged not relevant
- P18: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P19: [\[(known) BUG\] DELETE/UPDATE more than one row in partitioned foreign table](https://commitfest.postgresql.org/patch/7211/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P20: [postgres_fdw - make cached connection functions tests meaningful with debug_invalidate_system_caches_always](https://commitfest.postgresql.org/patch/3112/) (cf, Rejected, 2021-07-21) - judged not relevant
- P21: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P22: [Go back to suppressing foreign_data DETAIL test output.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09963cedced7ffb98a06298cc16305767fd2b4dd) (revert, Partially reverted, 2019-03-22) - judged not relevant
- P23: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P24: [Make Append Cost aware of some run time partition prune case](https://commitfest.postgresql.org/patch/2829/) (cf, Returned with feedback, 2021-11-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 119 s; cost reported by the CLI: $0.99
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 61 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:41:47.792Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
