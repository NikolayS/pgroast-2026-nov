# Design pre-review: Asynchronous MergeAppend Execution

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5129](https://commitfest.postgresql.org/patch/5129/) (Needs review); authors: Alexander Pyhalov (a.pyhalov)
- Thread: [Asynchronous MergeAppend](https://www.postgresql.org/message-id/flat/59be194c5a409fb9fc9f2031581b8a44%40postgrespro.ru), started 2024-07-17
- Messages read: 64 of 64
- Patch prose read: 0001-Make-Append-subplans-async-capable-in-more-cases.patch, v1-0001-mark_async_capable-subpath-should-match-subplan.patch, v1-0002-MergeAppend-should-support-Async-Foreign-Scan-subpla.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal, and message #4 reports demand from production. Two fixable design points remain at level 2. maintenance-fit-1: an FDW that casts the ForeignAsyncConfigureWait requestor to AppendState compiles unchanged and fails only at run time. standards-compat-2: v25 does not say which setting turns off async MergeAppend. A committer who knows the async executor code is engaged and can settle both.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v25 code and its benchmarks were not seen. Without it, nobody can judge whether async heap filling keeps the merge order and cleans up pending requests on error; the withdrawn ordered Append side patch returned wrong results.

## The proposal, restated

**Problem.** When the planner picks MergeAppend instead of Append, async foreign scans become synchronous, so remote partitions run one after another. The author reports large slowdowns on partitioned, distributed tables with 20 or more postgres_fdw partitions when plans switch from async Append to sync MergeAppend.

**Proposed design.** MergeAppend gains async support for Foreign Scan children, but only for the initial heap filling, where it fetches one tuple from each child. After that, children are read synchronously. The current version (v25, by Etsuro Fujita) keeps MergeAppend separate from Append, shares only the event-wait code, drops the earlier AppendBase refactoring, and changes postgres_fdw's wait configuration.

**User-visible surface.**
- New GUC enable_async_merge_append (boolean, default on), shown in early versions; the material does not say if v25 keeps it.
- EXPLAIN shows "Async Foreign Scan" under Merge Append.
- New wait event WAIT_EVENT_MERGE_APPEND_READY, visible in pg_stat_activity (added in v17).
- FDW C API change: ForeignAsyncConfigureWait may now get a MergeAppendState as requestor; documented in fdwhandler.sgml.
- postgres-fdw.sgml documentation updated for async MergeAppend.

**What the author claims.**
- Up to two times faster with remote servers in the author's tests.
- Single-client pgbench: about 202 tps versus 149 tps with async_capable on versus off.
- Later version: +78% single-client and +4% with 16 clients; the older version lost 5% with 16 clients.
- A reviewer measured about 4302 tps patched versus 3847 tps on master.
- Async after the initial heap filling gave no benefit and cost some speed, so the patch now drops it.

**Thread state.** Etsuro Fujita rewrote the patch as v25 and plans to push it after about a week if nobody objects; Pyhalov agrees and Korotkov's opinion is pending. A side patch to make ordered Append async-capable was withdrawn after Kirill Reshke showed it returns wrong results.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-07-17; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Async FDWs now receive MergeAppendState with no opt-in or loud failure

fixable · level 2, rubric item 9 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

An out-of-tree FDW that is async-capable casts the ForeignAsyncConfigureWait requestor to AppendState, as the API has allowed until now. After upgrading, a query plans Merge Append over its scans and the FDW reads MergeAppendState fields as Append fields. The result can be wrong waits, a hang, or a crash, and nothing fails at compile time or at load time.

Message #54 states that the patch changes the current semantics of ForeignAsyncConfigureWait for out-of-tree FDWs. The answer in message #55 was to update fdwhandler.sgml. Documentation does not stop an existing binary from receiving a node type it never expected, and the default GUC is on, so the new path is taken without any action by the FDW author. A committer would likely ask whether async MergeAppend should apply only to FDWs that say they support it.

Thread: [#54 (Alexander Pyhalov, 2026-10-28)](https://www.postgresql.org/message-id/a44e9ead6205f575da3df01b8c964e73%40postgrespro.ru), [#55 (Etsuro Fujita, 2026-11-01)](https://www.postgresql.org/message-id/CAPmGK15bYO_A-fTcBQnmjicAn%3DNPyOUVygDqjByXS%3Dga16p4Qw%40mail.gmail.com)

**Question:** How does an existing async-capable FDW that assumes an AppendState requestor avoid being used under Merge Append until its author updates it?

Counter-check: held. Extensions are rebuilt for a major version, so the "existing binary" part does not hold. But an FDW that casts the requestor to AppendState still compiles unchanged and fails only at run time, and message #55 answers this with documentation only.

### 2. The GUC that controls async MergeAppend is not settled in v25

fixable · level 2, rubric item 12 · new GUC, planner behaviour change · `standards-compat-2`

A site runs with enable_async_append = off to avoid async foreign scans. After upgrade, ordered queries switch to async MergeAppend because that path is gated by a separate enable_async_merge_append GUC that defaults to on, or by no GUC at all. The admin finds out from new wait events or a regression in production.

The early patches add enable_async_merge_append next to the existing enable_async_append (v1 config.sgml hunk). Message #36 still mentions it, but the v25 rewrite in messages #53 and #57 does not say whether it stays. A second enable_* setting for one feature departs from how enable_async_append is used today. If the GUC is dropped, the patch also does not say whether enable_async_append covers MergeAppend.

Thread: [#1 (Alexander Pyhalov, 2024-07-17)](https://www.postgresql.org/message-id/59be194c5a409fb9fc9f2031581b8a44%40postgrespro.ru), [#36 (Alexander Pyhalov, 2026-04-14)](https://www.postgresql.org/message-id/fa7ae004ec282fde84c29f36e0c85979%40postgrespro.ru), [#53 (Etsuro Fujita, 2026-10-27)](https://www.postgresql.org/message-id/CAPmGK15%3DKctsePnvKMNoYm3JOpuiZRWfWEfMs1GaBk9Rj45c5w%40mail.gmail.com), [#57 (Etsuro Fujita, 2026-11-03)](https://www.postgresql.org/message-id/CAPmGK14YATtpoUk6X54u1sW97mj-tccf4oc0hMX%3DmzG5mz02VQ%40mail.gmail.com)

**Question:** In v25, which setting turns off async MergeAppend, and does enable_async_append = off still disable all async foreign scans?

Counter-check: held. Messages #53 to #57 do not say which setting turns off async MergeAppend in v25, or whether enable_async_append covers it. A separate enable_* setting per plan type follows existing practice, so only the open question stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | ForeignAsyncConfigureWait may now get a MergeAppendState as requestor, and execnodes.h changes. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Early versions add enable_async_merge_append; the material does not say if v25 keeps it. |
| planner behaviour change | **yes** | createplan.c and costsize.c change, and Foreign Scan children of MergeAppend become async by default. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | MergeAppend gets its own async code path next to Append's, plus a new wait event and FDW API text. |
| performance on the hot path | **yes** | nodeMergeAppend.c changes, and no v25 benchmark shows the cost for MergeAppend plans without async children. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. In v25, which setting turns off async MergeAppend, and does enable_async_append = off still disable all async foreign scans?
2. How does an async-capable FDW that casts the ForeignAsyncConfigureWait requestor to AppendState fail loudly, or decline use under Merge Append, until its author updates it?
3. What does v25 itself gain on the motivating case, with 20 or more postgres_fdw partitions and slow remote servers?
4. What does v25 cost a MergeAppend plan that has no async-capable children, compared with master?
5. Does v25 change the cost model, and if so, when does the planner now prefer MergeAppend over async Append?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Etsuro Fujita reviews the entry, wrote v25, and plans to push it; the owner must know the async Append executor code and postgres_fdw.
- **Liability:** The owner signs for a second async code path in the executor, the changed ForeignAsyncConfigureWait contract for out-of-tree FDWs, and postgres_fdw's wait configuration, with no end date.
- **Author follow-up:** Pyhalov has kept the entry moving through 11 CommitFests, moved it forward himself in 2026-08 and 2026-10, and agrees with v25.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The gain goes to users of ordered queries over postgres_fdw partitions, and message #4 reports a production slowdown there. v25 dropped the AppendBase refactoring and a committer is engaged; the v25 patch was not read, so review cost is an estimate.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` ForeignAsyncConfigureWait gets a new requestor type without any opt-in. Refuted (not specific): Duplicate of maintenance-fit-1. Its request that an FDW keep working "without being rebuilt" does not apply, because extensions are recompiled for each major version.
- `replication-ondisk-1` Existing async-capable FDWs get a MergeAppendState requestor without opting in. Refuted (not specific): Duplicate of maintenance-fit-1. The concern about already built binaries does not apply, because extensions are recompiled for each major version.
- `already-exists-1` A second planner GUC next to enable_async_append. Refuted (not specific): Duplicate of standards-compat-2, which asks the same question about v25 more directly. The per-server async_capable option still turns off async execution for both node types.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The listed rows cover async Append and ForeignScan, postgres_fdw async and Custom Scan async, and none of their recorded reasons apply here. The [P2] concern about UNION ALL is answered by message #51, which shows async scans under a UNION ALL Merge Append. The ordered-Append side patch in message #59 was withdrawn in message #64 after the wrong result in message #63.
- standards-compat: 2 objection(s). The SQL surface, pg_dump and psql are not touched. EXPLAIN reuses the existing "Async Foreign Scan" label. The new wait event follows existing practice.
- replication-ondisk: 1 objection(s). The proposal is executor and planner only. It has no on-disk format, WAL, catalog, replication or wire protocol change. The only cross-boundary surface is the FDW async API.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. Async MergeAppend changes only how the executor waits on foreign scans that a user can already run, under the same user mapping and permissions. It adds no new access path to data, files, other sessions or credentials. The new wait event appears in pg_stat_activity under the usual visibility rules. One unbounded resource is possible: an async scan holds one pending remote request per partition. But plain async Append already does this for the same users and servers, so the risk is not new.
- maintenance-fit: 1 objection(s). Other maintenance concerns are weak. The async Append author is now the committer and owns the code. v25 dropped the invasive AppendBase refactoring. Any GUC follows the existing enable_async_append practice.
- already-exists: 1 objection(s). Postgres has no existing feature, extension or documented method that runs MergeAppend children asynchronously. The executor change cannot be made outside core, so the only point from this angle is the possible duplicate GUC.
- demand: 0 objection(s). The material shows real demand. Message #4 reports a production slowdown when plans over distributed tables with 20+ postgres_fdw partitions switched from async Append to sync MergeAppend. Independent reviewers measured gains (message #6) or called it important (message #33), and the cost falls mainly on async foreign scan users. The missing v25 benchmark is a measurement question for another angle.

### Precedents retrieved

- P1: [Asynchronous execution of ForeignScans](https://commitfest.postgresql.org/patch/751/) (cf, Returned with feedback, 2016-12-02) - used in the review
- P2: [Asynchronous execution for postgres_fdw](https://commitfest.postgresql.org/patch/856/) (cf, Returned with feedback, 2017-03-16) - used in the review; reason summarised from its thread
- P3: [Improve Append/MergeAppend EXPLAIN output when all subplans are pruned.](https://commitfest.postgresql.org/patch/2102/) (cf, Returned with feedback, 2019-12-01) - judged not relevant; reason summarised from its thread
- P4: [Make Append Cost aware of some run time partition prune case](https://commitfest.postgresql.org/patch/2829/) (cf, Returned with feedback, 2021-11-04) - judged not relevant; reason summarised from its thread
- P5: [Parallel Append implementation](https://commitfest.postgresql.org/patch/929/) (cf, Returned with feedback, 2017-02-01) - judged not relevant; reason summarised from its thread
- P6: [libpq should append error messages, not overwrite](https://commitfest.postgresql.org/patch/1752/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P7: [Asynchronous execution on postgres-fdw](https://commitfest.postgresql.org/patch/291/) (cf, Rejected, 2015-08-10) - used in the review; reason summarised from its thread
- P8: [asynchronous execution support for Custom Scan](https://commitfest.postgresql.org/patch/3813/) (cf, Returned with feedback, 2024-02-01) - used in the review; reason summarised from its thread
- P9: [doc: Add some OID/TOAST-related limitations to the limits appendix.](https://commitfest.postgresql.org/patch/4320/) (cf, Withdrawn, 2023-06-13) - judged not relevant; reason summarised from its thread
- P10: [Async execution of postgres_fdw.](https://commitfest.postgresql.org/patch/32/) (cf, Rejected, 2015-05-09) - used in the review
- P11: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [Pattern based listeners for asynchronous messaging (LISTEN/NOTIFY)](https://commitfest.postgresql.org/patch/1218/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P13: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant
- P14: [Allow ordered partition scans in more cases](https://commitfest.postgresql.org/patch/4198/) (cf, Withdrawn, 2023-06-24) - judged not relevant
- P15: [Remove dead macro exec_subplan_get_plan](https://commitfest.postgresql.org/patch/3894/) (cf, Rejected, 2023-04-04) - judged not relevant
- P16: [Subplan result cache](https://commitfest.postgresql.org/patch/1646/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P17: [Revert "Common SQL/JSON clauses"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1460fc5942591fdb6bee0bc8342019ede31ff3b6) (revert, Reverted, 2022-03-22) - judged not relevant
- P18: [Fix collation handling in postgres_fdw](https://commitfest.postgresql.org/patch/2685/) (cf, Returned with feedback, 2021-09-25) - judged not relevant
- P19: [Make the optimiser aware of partitions ordering](https://commitfest.postgresql.org/patch/1093/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P20: [Improving the heapgetpage function improves performance in common scenarios](https://commitfest.postgresql.org/patch/4524/) (cf, Withdrawn, 2023-09-06) - judged not relevant
- P21: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P22: [calculate and use most common type for polymorphic arguments](https://commitfest.postgresql.org/patch/258/) (cf, Rejected, 2015-07-22) - judged not relevant
- P23: [Kerberos delegation support in libpq and postgres_fdw](https://commitfest.postgresql.org/patch/3265/) (cf, Returned with feedback, 2021-11-26) - judged not relevant
- P24: [Push down more UPDATEs/DELETEs in postgres_fdw](https://commitfest.postgresql.org/patch/791/) (cf, Returned with feedback, 2017-11-29) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 174 s; cost reported by the CLI: $4.44
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 73 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:29:08.597Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
