# Design pre-review: Use-after-free of a shared Path in add_path()/add_partial_path()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7498](https://commitfest.postgresql.org/patch/7498/) (Needs review); authors: Jeevan Chalke (jeevan.chalke)
- Thread: [Use-after-free of a shared Path in add_path()/add_partial_path()](https://www.postgresql.org/message-id/flat/CAM2%2B6%3DXaYtMGwh27O7oDTfAw7kV4VdUVbxC2RVzWtKz_ysekKw%40mail.gmail.com), started 2026-09-09
- Messages read: 5 of 5
- Patch prose read: v2-0001-Don-t-pfree-a-Path-that-belongs-to-a-different-re.patch, v2-0002-Add-regression-test-for-shared-partial-path-use-a.patch, v1-0001-Don-t-pfree-a-Path-that-belongs-to-a-different-re.patch, v1-0002-Add-regression-test-for-shared-partial-path-use-a.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with two reproducers, Richard Guo confirmed the bug on v19 and master in message #3, and the fix changes no signature or user-visible behaviour. The remaining points are the choice among the parent_rel check, a flat copy and reference counting, and the back-patch range, which a planner committer has to settle.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody here could check that every path constructor sets path->parent as the fix assumes, or whether the patch updates the add_path() header comment and optimizer/README. The create_ordered_paths() case has no regression test, so its coverage rests on the author's manual check.

## The proposal, restated

**Problem.** add_path() and add_partial_path() free dominated or rejected paths, assuming no other rel references them. grouping_planner() and create_ordered_paths() pass in paths that still belong to another rel, so a freed path can leave a Gather subpath dangling. Planning then fails with "ERROR: unrecognized node type: N" for some UNION queries with a set-returning function under parallel query.

**Proposed design.** add_path() and add_partial_path() skip the pfree() when path->parent differs from parent_rel, next to the existing IndexPath exemption. The author relies on every create_*_path() constructor setting path->parent before the path reaches add_path(), so a mismatch marks a path borrowed from another rel. No caller changes; 0002 adds two regression tests to select_parallel.sql.

**What the author claims.**
- The same defect shows up independently in grouping_planner() and create_ordered_paths(), so a general check beats fixing each call site.
- The fix also covers the 2023 postgres_fdw report, which had no standalone reproducer.
- The bug exists on all branches; the author first said v18 only and corrected this after Richard's v19 reproducer.
- The fix is small and has no API impact, so it suits back-patching better than copying paths or reference counting.
- The postgres_fdw leak it causes is a node or two per query, freed with the planning memory context.

**Thread state.** Richard Guo reproduced the bug on v19 and master and thinks the parent_rel check works, while naming flat copy and reference counting as alternatives. Ashutosh Bapat points to an earlier discussion that found reference counting invasive; the author asks whether Richard will take the patch over, with no reply yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-09; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | add_path() and add_partial_path() keep their signatures, but FDW and custom paths whose parent differs from parent_rel are no longer freed. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | The patch changes when add_path() and add_partial_path() free a dominated or rejected path; plan choice is meant to stay the same. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bug is on all branches and the author asks for a back-patch; no reviewer has said how far back. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The fix relies on every path constructor setting path->parent, a rule future path types must keep. |
| performance on the hot path | **yes** | add_path() runs for every candidate path; the patch adds one pointer compare and keeps some paths until planning ends. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which create_*_path() constructors, including FDW and custom scan paths, were checked to set path->parent to the rel later passed to add_path()?
2. Can a path be shared across rels while path->parent equals parent_rel, so that the check still frees it?
3. How far back should the fix go, and does Richard Guo's v19 reproducer also fail on the older supported branches?
4. Why is the parent_rel check preferred over a flat copy of the path at the two call sites, which Richard Guo says also fixes the bug?
5. How much extra planner memory do the unfreed paths cost on a query with many partitions or many postgres_fdw upper paths?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows the planner's path generation and upper-rel code. Richard Guo reviewed in message #3; the visible thread does not show whether he will commit it.
- **Liability:** The owner signs for a back-patched change to add_path() memory rules on all supported branches, including extension paths that the regression suite does not cover, for the life of those branches.
- **Author follow-up:** The author posted a v2 with a second regression test after review and asked whether Richard Guo will take the patch over, which suggests continued engagement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** This is a planner failure reachable from plain SQL under parallel query, reproduced by a reviewer on v19 and master. The patch is about 7 KiB in two planner files plus tests, with no API change.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent records a reason that applies here. [P3] proposed an extension hook for the add_path() removal decision, a different idea with no recorded reason. Message #1 says the 2023 report stalled for lack of a reproducer, and this proposal adds two. Message #4 says reference counting was found invasive, which supports the smaller parent_rel check.
- standards-compat: 0 objection(s). The patch adds no syntax, GUC, catalog object, dump content or psql change, and query results stay the same, so nothing touches the user-facing contract. The extension-visible effect, borrowed FDW paths no longer being freed, is an API and memory question for the design angle.
- replication-ondisk: 0 objection(s). The patch changes only when the planner frees in-memory Path nodes during planning, so it adds no on-disk, WAL, catalog, wire-protocol or replication state. add_path() and add_partial_path() keep their signatures and no structs change, so back-patching raises no ABI concern from this angle.
- security: 0 objection(s). The fix exposes nothing new to unprivileged users and adds no privilege, GUC or monitoring change. The skipped pfree() keeps a few extra Path nodes until the planning memory context is reset, which an unprivileged user cannot grow into a denial of service, and removing a use-after-free reachable from plain SQL helps security.
- maintenance-fit: 0 objection(s). No objection from the maintenance angle: this is a crash fix that Richard Guo reproduced on v19 and master and supports (message #3), it changes no API, and it costs nothing on the hot path beyond one pointer compare in add_path(). The postgres_fdw leak is a node or two per query, freed with the planning context. Two points could not be checked without the code: whether every custom and FDW path sets path->parent consistently, and whether the patch updates the optimizer/README rule on freeing paths. [P3] is a hook proposal and does not apply here.
- already-exists: 0 objection(s). This is a bug fix for a planner use-after-free, so no existing feature or extension meets the need. The patch adds its check next to the existing IndexPath exemption in add_path() rather than building a second mechanism. [P3] proposed a path-removal hook, and the corpus records no reason for its outcome, so it does not apply here.
- demand: 0 objection(s). This is a bug fix for a planner use-after-free that a customer hit in production, a teammate reported independently, and a 2023 postgres_fdw report showed earlier. Richard Guo reproduced it on v19 and master (message #3), so demand is shown, and the benefit goes to every user of parallel query.

### Precedents retrieved

- P1: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P2: [Consider incremental sort for fractional paths in generate_orderappend_paths](https://commitfest.postgresql.org/patch/3513/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P3: [A hook for path-removal decision on add_path](https://commitfest.postgresql.org/patch/2234/) (cf, Returned with feedback, 2021-10-05) - used in the review
- P4: [Fix comment about generate_gather_paths](https://commitfest.postgresql.org/patch/2876/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P5: [Partial aggregates push down](https://commitfest.postgresql.org/patch/3369/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P6: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4019/) (cf, Returned with feedback, 2024-01-27) - judged not relevant; reason summarised from its thread
- P7: [Partial sort](https://commitfest.postgresql.org/patch/409/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P8: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4857/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P9: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - judged not relevant; reason summarised from its thread
- P10: [Optimize Numeric using fixed buffer reducing pallocs for small numbers](https://commitfest.postgresql.org/patch/4196/) (cf, Returned with feedback, 2023-09-13) - judged not relevant; reason summarised from its thread
- P11: [Optimizing Node Files Support](https://commitfest.postgresql.org/patch/4034/) (cf, Rejected, 2023-03-21) - judged not relevant; reason summarised from its thread
- P12: [Optimize SnapBuild by maintaining committed.xip in sorted order](https://commitfest.postgresql.org/patch/6187/) (cf, Withdrawn, 2026-04-29) - judged not relevant
- P13: [Docs: make source file path references consistent](https://commitfest.postgresql.org/patch/6380/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P14: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant
- P15: [Fix autovacuum freeze bug where weight can lower a table's freeze score](https://commitfest.postgresql.org/patch/7242/) (cf, Withdrawn, 2026-09-03) - judged not relevant
- P16: [Refactor add_path for clarity and extensibility](https://commitfest.postgresql.org/patch/3599/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [Fix the statistics gathering in case of commit and chain](https://commitfest.postgresql.org/patch/3657/) (cf, Returned with feedback, 2022-10-06) - judged not relevant
- P18: [quick and dirty fix: Quadratic planning time for ordered paths over partitioned tables](https://commitfest.postgresql.org/patch/5516/) (cf, Rejected, 2025-02-03) - judged not relevant
- P19: [Asynchronous execution on postgres-fdw](https://commitfest.postgresql.org/patch/291/) (cf, Rejected, 2015-08-10) - judged not relevant
- P20: [New hooks in the connection path](https://commitfest.postgresql.org/patch/3735/) (cf, Returned with feedback, 2023-04-04) - judged not relevant
- P21: [Don't shut down Gather\[Merge\] early under Limit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=080313f8296fb0bcc74bd70fc8e15cd64f45945e) (revert, Partially reverted, 2019-11-26) - judged not relevant
- P22: [Problem in 'ORDER BY' of a column using a created collation?](https://commitfest.postgresql.org/patch/5990/) (cf, Returned with feedback, 2025-10-21) - judged not relevant
- P23: [Async execution of postgres_fdw.](https://commitfest.postgresql.org/patch/32/) (cf, Rejected, 2015-05-09) - judged not relevant
- P24: [Push down more UPDATEs/DELETEs in postgres_fdw](https://commitfest.postgresql.org/patch/791/) (cf, Returned with feedback, 2017-11-29) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 86 s; cost reported by the CLI: $1.07
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:01:17.470Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
