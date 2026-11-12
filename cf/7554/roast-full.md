# Design pre-review: Assert failure in parallel query due to SubPlan double visit for hash join

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7554](https://commitfest.postgresql.org/patch/7554/) (Needs review); authors: Lukas Fittl (lfittl)
- Thread: [Assert failure in parallel query due to SubPlan double visit for hash join](https://www.postgresql.org/message-id/flat/CAP53PkzYGGESHX%2Bdsy04rcO-7PpBjGg-QNzfLcAC%2BagydukTcA%40mail.gmail.com), started 2026-10-26
- Messages read: 8 of 8
- Patch prose read: v1-0001-Process-each-PlanState-only-once-during-parallel-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is real: message #1 gives a reproducer and messages #2 and #3 confirm it. No objection survived at levels 1a or 1b. Two fixable design points remain at level 2: standards-compat-1 (no fix shape for the stable branches, where the bug has existed since Postgres 10) and maintenance-fit-2 (the Bitmapset-first context rule is not checked at compile time for out-of-core callers). Neither would stop the fix; both need a decision from someone who knows the executor.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and the favored walker-level design has no posted patch. Messages #4 to #8 were not shown, so the context-struct details and the walker cost could not be checked.

## The proposal, restated

**Problem.** When one SubPlan has several SubPlanStates, parallel query walks its subtree once per reference. With hash joins, EXPLAIN ANALYZE then fails an assertion on a duplicate shm_toc key. Non-assert builds show inflated loop counts and lose per-worker details such as "Sort Method", and users have no easy way to notice.

**Proposed design.** The patch adds planstate_tree_collect(), which returns the distinct PlanStates of a tree. The parallel DSM setup and instrumentation code then process each list entry once and no longer recurse per reference. ExecShutdownNode() is left unchanged.

**User-visible surface.**
- C API: new helper planstate_tree_collect() in nodes/nodeFuncs.h.
- Changed exported functions in executor/execParallel.h; the prose does not say which signatures change.
- EXPLAIN ANALYZE of affected parallel plans shows correct loop counts and per-worker Sort Method lines.

**What the author claims.**
- The bug dates from b8d7f053c5c (Postgres 10), where ExecInitHashJoin started creating a second SubPlanState; the author has not tested older branches.
- The fix removes the assertion failure and corrects loops and Sort Method output, shown with a reproducer and a regression test.
- All in-core shutdown callbacks are idempotent, so ExecShutdownNode() is not a practical problem today.
- The affected plan shape is uncommon, and users of non-assert builds would rarely notice wrong output.
- Dedup inside each existing walker would need context changes in four places.

**Thread state.** Sami Imseih approved v1 with naming and comment notes. David Geier and the author now favor making planstate_tree_walker() skip visited nodes with a Bitmapset on plan_node_id carried in a common context struct, which would also fix ExecShutdownNode(); no new patch is posted yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Proposed walker change cannot reach the back branches that have the bug

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `standards-compat-1`

A packager ships a minor release with the fix after it is back-patched to stable branches. An extension that calls planstate_tree_walker() or the changed execParallel.h functions, built against the earlier minor release, now passes a context of the wrong shape. It misbehaves or crashes, and users find out only after the minor upgrade.

The bug dates from Postgres 10, so it is present in every supported branch (message #1). v1 changes exported signatures in execParallel.h. The design now favored in messages #7 and #8 changes the context contract of planstate_tree_walker(), which is part of the extension API. Neither version says whether there will be a separate fix for the back branches that keeps the ABI stable.

Thread: [#1 (Lukas Fittl, 2026-10-26)](https://www.postgresql.org/message-id/CAP53PkzYGGESHX%2Bdsy04rcO-7PpBjGg-QNzfLcAC%2BagydukTcA%40mail.gmail.com), [#7 (Lukas Fittl, 2026-11-04)](https://www.postgresql.org/message-id/CAP53PkwgWzwBbY8mYmeqGqMSKURD07H_vQbqjkdtcROttBpn9w%40mail.gmail.com), [#8 (David Geier, 2026-11-04)](https://www.postgresql.org/message-id/b0fa92db-7fc1-4452-9f5c-54fe721d331e%40gmail.com)

**Question:** What fix ships to the supported back branches, and does it leave the signatures of planstate_tree_walker() and the execParallel.h functions unchanged?

Counter-check: held. The bug is in every supported branch, v1 changes exported execParallel.h functions, and the favored design changes the planstate_tree_walker() context contract. No message says which fix, if any, goes to stable branches.

### 2. Required Bitmapset-first context changes walker contract silently for extensions

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

An extension calls planstate_tree_walker() with its own context struct, which has no Bitmapset as its first member. The code still compiles against the new release. On a production build the walker reads the first field as a Bitmapset and crashes or skips nodes. The Bitmapset node-tag assertion only catches this on assert builds.

Message #8 proposes requiring "the first member of any context is a Bitmapset" behind the existing void * context. That is a convention the compiler cannot check. Message #8 also counts only in-core call sites, while planstate_tree_walker() is exported and used by out-of-core code. Option 2 in message #7, an explicit extra argument, breaks callers at compile time, and message #8 rejects it only for convenience at the call sites.

Thread: [#7 (Lukas Fittl, 2026-11-04)](https://www.postgresql.org/message-id/CAP53PkwgWzwBbY8mYmeqGqMSKURD07H_vQbqjkdtcROttBpn9w%40mail.gmail.com), [#8 (David Geier, 2026-11-04)](https://www.postgresql.org/message-id/b0fa92db-7fc1-4452-9f5c-54fe721d331e%40gmail.com)

**Question:** How will out-of-core callers of planstate_tree_walker() learn about the new context requirement at compile time rather than through a crash on non-assert builds?

Counter-check: held. planstate_tree_walker() is exported with a void pointer context, and message #8 lists only in-core call sites. An old extension context would still compile under option 1, and only an assertion would catch the mismatch.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v1 adds planstate_tree_collect() to nodeFuncs.h and changes exported functions in execParallel.h; the favored design changes the planstate_tree_walker() context contract. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bug dates from Postgres 10, and the thread does not say which fix goes to stable branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every future planstate_tree_walker() caller must follow the new visit-once rule or context layout. |
| performance on the hot path | **yes** | Dedup in planstate_tree_walker() adds a visited check to every PlanState walk; no one has measured it. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which supported branches get the fix, and what change goes there that leaves planstate_tree_walker() and the execParallel.h functions unchanged?
2. How does an out-of-core caller of planstate_tree_walker() learn at compile time that its context must start with a Bitmapset?
3. Why is the Bitmapset-first context preferred over an explicit extra argument, which would make old callers fail to compile?
4. What does the visited check cost per walk for callers that never meet a shared SubPlan?
5. Is a CustomScan shutdown callback that is not idempotent a real case that the stable-branch fix must also cover?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows the executor's parallel query setup (execParallel.c) and the nodeFuncs.c walkers. The visible thread shows the author and two reviewers; the author of b8d7f053c5c was CCed and has not replied.
- **Liability:** The owner signs for a changed planstate_tree_walker() contract that every in-core and out-of-core caller must follow, and for a stable-branch fix that may differ from the master fix.
- **Author follow-up:** The author is engaged with reviewers and plans a reworked design per the brief; no new patch is posted yet. The fix also unblocks the author's stack-based instrumentation series.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The plan shape is uncommon, but the assertion failure and the wrong EXPLAIN ANALYZE output are confirmed by two reviewers. The v1 patch is about 31 KiB in execParallel.c and nodeFuncs.c, and the walker contract change needs care for extension callers.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` New walker context rule breaks extension callers without a compile error. Refuted (not specific): Duplicate of maintenance-fit-2.
- `replication-ondisk-1` Bitmapset-first walker context breaks out-of-core planstate_tree_walker callers. Refuted (not specific): Duplicate of maintenance-fit-2.
- `replication-ondisk-2` Back-branch fix shape is undefined while exported signatures change. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-1` Fix for a bug present since Postgres 10 has no back-patch shape. Refuted (not specific): Duplicate of standards-compat-1. Its claim that a new nodeFuncs.h helper cannot go into a stable branch is also too strong, because adding a function keeps the ABI.
- `maintenance-fit-3` v1 fixes four callers and leaves the generic walker unsafe. Refuted (already handled): Message #3 and message #6 raise this point, and the author agrees in message #7 to move the dedup into planstate_tree_walker(), which covers ExecShutdownNode(). The thread has already set v1's collect helper aside.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P15] reverts a 2017 fix for a different instrumentation bug, accumulation across worker rescans, and records only a generic reason. The thread does point to a64846f3ada as a related old problem, but says nothing more about it (message #1). The open design question is v1's list-based helper versus dedup inside planstate_tree_walker(), and it is being settled in messages #6-#8.
- standards-compat: 2 objection(s). The user-visible change, corrected EXPLAIN ANALYZE loop counts and Sort Method lines, fixes wrong output and needs no compatibility handling. Both objections concern the extension API and back-patching.
- replication-ondisk: 2 objection(s). The patch adds no on-disk, WAL, catalog or wire-protocol state. Its DSM state lives only for one query in one server, so pg_upgrade, standbys and subscribers are unaffected. Both objections are about the extension API and back-patching.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes how the executor walks PlanState trees during parallel setup and instrumentation; it adds no SQL surface, privilege, setting or new information exposure, and it fixes inflated EXPLAIN ANALYZE output instead of revealing more. An unprivileged user can already trigger the assertion only on assert-enabled builds, which the fix removes.
- maintenance-fit: 3 objection(s). The bug is real and confirmed by two reviewers, so the objections concern back-patch shape and API contract, not desirability. [P15] is a reverted instrumentation fix, but its recorded reason is too vague to apply here.
- already-exists: 0 objection(s). This is a bug fix for wrong results and an assertion failure, and no extension or existing setting can do that job. The second mechanism next to planstate_tree_walker(), and the per-node guards like checking shared_info, were raised in messages #3 and #6, and the thread already plans to put dedup into the walker itself.
- demand: 0 objection(s). This is a bug fix, so the demand test does not apply in the usual way. Message #1 gives a reproducer for an assertion failure on master, and messages #2 and #3 confirm it independently. On non-assert builds, EXPLAIN ANALYZE shows wrong loop counts and drops Sort Method lines. A second plan shape in message #5 shows the bug is not limited to hash joins. Wrong output is a defect even if few users notice it.

### Precedents retrieved

- P1: [Fixes a trivial bug in dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5996/) (cf, Withdrawn, 2025-09-02) - judged not relevant; reason summarised from its thread
- P2: [Add OID descriptions to dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5987/) (cf, Withdrawn, 2025-08-25) - judged not relevant; reason summarised from its thread
- P3: [Fix TOCTOU race in ReplicationSlotsComputeRequiredLSN()](https://commitfest.postgresql.org/patch/6682/) (cf, Withdrawn, 2026-04-17) - judged not relevant; reason summarised from its thread
- P4: [pg_upgrade allows itself to be run twice (initdb)](https://commitfest.postgresql.org/patch/3872/) (cf, Returned with feedback, 2022-12-01) - judged not relevant; reason summarised from its thread
- P5: [KNN for B-tree](https://commitfest.postgresql.org/patch/1804/) (cf, Returned with feedback, 2020-03-16) - judged not relevant; reason summarised from its thread
- P6: [Improve pg_restore toc file parsing and format for better performances](https://commitfest.postgresql.org/patch/4509/) (cf, Returned with feedback, 2024-01-14) - judged not relevant; reason summarised from its thread
- P7: [Subplan result cache](https://commitfest.postgresql.org/patch/1646/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P8: [Safer hash table initialization macro](https://commitfest.postgresql.org/patch/6276/) (cf, Withdrawn, 2026-03-09) - judged not relevant; reason summarised from its thread
- P9: [Disable parallel query by default](https://commitfest.postgresql.org/patch/5751/) (cf, Rejected, 2026-02-11) - judged not relevant
- P10: [Fix assertion failure in SnapBuildInitialSnapshot()](https://commitfest.postgresql.org/patch/4149/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P11: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [Revert "Show index search count in EXPLAIN ANALYZE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d00107cd63e780753aa25563fa37603369997d0c) (revert, Reverted, 2025-03-05) - judged not relevant
- P13: [Fix slotsync worker busy loop causing repeated log messages](https://commitfest.postgresql.org/patch/6546/) (cf, Withdrawn, 2026-05-01) - judged not relevant
- P14: [Sort leaf pages by ctid for gist indexes built using sorting method](https://commitfest.postgresql.org/patch/3455/) (cf, Withdrawn, 2021-12-16) - judged not relevant
- P15: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - used in the review
- P16: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P17: [Hashjoin fall back to block nested hashloop join](https://commitfest.postgresql.org/patch/2588/) (cf, Withdrawn, 2020-11-10) - judged not relevant
- P18: [Minimizing calls to SubTransSetParent()](https://commitfest.postgresql.org/patch/3806/) (cf, Withdrawn, 2023-03-22) - judged not relevant
- P19: [sort leaf pages by ctid for gist indexes built using sorted method](https://commitfest.postgresql.org/patch/3454/) (cf, Returned with feedback, 2022-03-22) - judged not relevant
- P20: [Optionally record Plan IDs to track plan changes for a query](https://commitfest.postgresql.org/patch/5486/) (cf, Returned with feedback, 2025-03-24) - judged not relevant
- P21: [CALL optional in PL/pgSQL](https://commitfest.postgresql.org/patch/1596/) (cf, Rejected, 2018-04-05) - judged not relevant
- P22: [Parallel INSERT (INTO ... SELECT ...)](https://commitfest.postgresql.org/patch/2844/) (cf, Returned with feedback, 2021-04-01) - judged not relevant
- P23: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [Parallel INSERT SELECT take 2](https://commitfest.postgresql.org/patch/3143/) (cf, Returned with feedback, 2022-07-28) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 111 s; cost reported by the CLI: $1.65
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 81 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:09:35.199Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
