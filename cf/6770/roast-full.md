# Design pre-review: [BUG] DELETE/UPDATE more than one row in partitioned foreign table

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6770](https://commitfest.postgresql.org/patch/6770/) (Needs review); authors: Nikita Malakhov (nmalakh), ajitpostgres awekar (ajit.awekar)
- Thread: [\[(known) BUG\] DELETE/UPDATE more than one row in partitioned foreign table](https://www.postgresql.org/message-id/flat/20250718175314.4513c00a%40karst), started 2025-07-18
- Messages read: 37 of 37
- Patch prose read: v3-0001-Test-exposing-bug-when-foreign-table-points-to-a-.patch, v3-0002-Error-out-if-one-iteration-of-non-direct-DML-affe.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is not in question: the bug silently changes wrong rows on the remote side, and messages #1 and #6 report production hits. The surviving objections are all fixable level 2 points. The decisive one is maintenance-fit-1: v5-0001 passes an attribute number outside min_attr..max_attr through the planner, and no planner committer has reviewed it. prior-attempts-1 leaves the stable-branch fix open, which the list must settle before commit.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check how v5-0001 handles every planner array indexed by attno, or whether it extends the existing row-identity path or duplicates it. I also could not check the ABI effect on planner structs.

## The proposal, restated

**Problem.** When a foreign table points to a remote partitioned table or inheritance parent, a non-direct UPDATE or DELETE identifies the remote row by ctid alone. Rows in different partitions can share a ctid, so postgres_fdw changes or deletes rows that do not match the query. Users of postgres_fdw on remote partitioned tables get silent logical corruption on the remote side, a bug known since 2018.

**Proposed design.** The current patch set (v5) has two parts. v5-0001 lets an FDW add a row-identity pseudo-column outside the normal attribute range to the planner; v5-0002 makes postgres_fdw fetch the remote tableoid this way and send "WHERE ctid = $1 AND tableoid = $2". It also fetches columns one by one and builds the whole row locally, in place of shipping ROW().

**User-visible surface.**
- Remote SQL for every non-direct UPDATE/DELETE in postgres_fdw now selects tableoid and adds "AND tableoid = $2", visible in EXPLAIN VERBOSE.
- C API: planner support for FDW-injected row-identity pseudo-columns, with attribute number RemoteTableOidAttributeNumber (MaxHeapAttributeNumber + 1).
- Former Assert(fdwroutine != NULL) checks in the planner become runtime errors for FDWs.
- postgres-fdw.sgml: new notes on no partition pruning for per-row commands and on chained foreign tables not being protected.
- EXPLAIN shows (t.*)::t IS NULL in some whole-row filters on foreign partitions.

**What the author claims.**
- The patch fixes the wrong-row UPDATE/DELETE for a foreign table on a remote partitioned or inherited table.
- v5-0001 has no effect unless an FDW uses it.
- The tableoid condition is cheap: about 10% more per remote statement (reviewer measurement).
- On a plain remote table there is no measurable slowdown; with 100 partitions per-row commands cost about ten times more, because they go through the root.
- Chained foreign tables (ft -> ft -> partitioned) are not detected and stay wrong, as documented.

**Thread state.** Manuel Reyes Bravo tested v5 and set the entry to Ready for Committer; Jakub Wartak also reviewed. Back-patching is still open: Etsuro Fujita and Michael Paquier favoured a back-patchable remotely_inherited option, and Jakub calls the v5 planner changes invasive.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-07-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Out-of-range pseudo attribute number adds a silent planner invariant

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change · `maintenance-fit-1`

A later planner patch indexes per-attribute arrays such as attr_needed or attr_widths by attno, assuming the range min_attr..max_attr. It does not handle RemoteTableOidAttributeNumber. On a production build, postgres_fdw UPDATE/DELETE on a remote partitioned table then reads out of bounds or plans wrongly, and nobody notices until a user reports it.

v5-0001 touches costsize.c, initsplan.c, setrefs.c, relnode.c and ruleutils.c so that one attribute number outside the normal range can pass through. Message #36 turns some existing Asserts into runtime errors, but only at places that already had a check. Message #27 asks for a planner expert and message #34 says the reviewer can no longer follow the edge cases. No planner committer has reviewed v5-0001 in the thread, although message #29 says Robert Haas suggested the approach off-list.

Thread: [#27 (Jakub Wartak, 2026-09-16)](https://www.postgresql.org/message-id/CAKZiRmwr4zEZ2rFWv54GVh7FfAstQTdbA0HAv2M4E4dpLsX1-Q%40mail.gmail.com), [#29 (Ajit Awekar, 2026-09-22)](https://www.postgresql.org/message-id/CAER375NQEqXLQ5eqFJ8QpnpnYKCHeyi8YhG6CdAm6m%3DQGc6unw%40mail.gmail.com), [#34 (Jakub Wartak, 2026-10-28)](https://www.postgresql.org/message-id/CAKZiRmzpABcNuk%2BZrhmV3%3DGtaA2S%3DC3%3DvKw_ggXCUwQUv8D4ag%40mail.gmail.com), [#36 (Ajit Awekar, 2026-11-03)](https://www.postgresql.org/message-id/CAER375NOcbQyDGxfRDM0wuRmJx8j5TBmkXtC0QY0j2Doye%3DHSA%40mail.gmail.com)

**Question:** Which planner code paths assume attno lies within min_attr..max_attr, and what makes a future patch that misses the pseudo-column fail in tests instead of in the field?

Counter-check: held. The planner sizes per-attribute arrays by min_attr..max_attr, so an attribute number above that range needs special handling wherever they are indexed. Messages #27 and #34 say no planner expert has reviewed this, and the proposal does not show a loud failure if a later patch misses it.

### 2. Same non-back-patchable shape that reportedly stalled the earlier fix

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API, planner behaviour change · `prior-attempts-1`

A user on Postgres 16 to 19 runs a non-direct UPDATE on a foreign table over a remote partitioned table. Rows with a shared ctid in other partitions are silently changed. The user finds out later from wrong data, because v5 changes the planner and only lands in 20.

Message #1 reports that the 2018 fixes lost attention partly because they needed new FDW features and could not be backported. Message #9 says this line of work builds on that earlier design and is not back-patchable. v5 keeps that shape: planner changes that message #34 calls invasive, and varnullingrels use that limits it to 16 and later (message #27). Etsuro's remotely_inherited patch (messages #9, #10, #14) is the stable-branch answer, but the thread does not say whether it is still planned alongside v5.

Thread: [#1 (Jehan-Guillaume de Rorthais, 2025-07-18)](https://www.postgresql.org/message-id/20250718175314.4513c00a%40karst), [#9 (Etsuro Fujita, 2026-06-01)](https://www.postgresql.org/message-id/CAPmGK166P%2Bngd2ehady%3D_f-L4MePgBdBNxN5gi5_gSAfmV82QA%40mail.gmail.com), [#13 (Michael Paquier, 2026-06-10)](https://www.postgresql.org/message-id/aij4NSMQYpQxtRMF%40paquier.xyz), [#14 (Etsuro Fujita, 2026-06-10)](https://www.postgresql.org/message-id/CAPmGK15vOW11uodFuSEDcYaUxLX4egOgwGq4BZasgG_pNOa_7Q%40mail.gmail.com), [#27 (Jakub Wartak, 2026-09-16)](https://www.postgresql.org/message-id/CAKZiRmwr4zEZ2rFWv54GVh7FfAstQTdbA0HAv2M4E4dpLsX1-Q%40mail.gmail.com), [#34 (Jakub Wartak, 2026-10-28)](https://www.postgresql.org/message-id/CAKZiRmzpABcNuk%2BZrhmV3%3DGtaA2S%3DC3%3DvKw_ggXCUwQUv8D4ag%40mail.gmail.com)

**Question:** What fix goes to stable branches, and is the remotely_inherited patch still planned to be committed alongside v5?

Counter-check: held. Messages #9, #27 and #34 say this line of work is not back-patchable, and the thread does not settle what stable branches get. v5 does not cause the stable-branch bug, so this is a question to settle before commit and stays fixable.

### 3. Chained foreign tables keep silent wrong-row updates behind a protective-looking plan

fixable · level 2, rubric item 7 · design, planner behaviour change · `standards-compat-2`

An administrator points a foreign table at a remote foreign table that sits on a partitioned table. EXPLAIN VERBOSE shows "WHERE ctid = $1 AND tableoid = $2", so the administrator trusts the result. Non-direct UPDATE then changes rows in other partitions, and nothing reports it until someone audits the data.

Message #28 shows this case still corrupts data under v5, and that the new condition makes the plan look protected. Message #29 declines a remote relkind check because of an extra round trip per query and documents the gap instead. A documentation note does not reach users who never read it, and the rest of the patch changes this behaviour from silently wrong to correct. The relkind could be checked only on the non-direct UPDATE/DELETE path, once per foreign table and connection, at much lower cost than the patch assumes.

Thread: [#28 (Manuel Reyes Bravo, 2026-09-16)](https://www.postgresql.org/message-id/CA%2BbCEdCo-w5_34AAm6BTZ0PucNg3sycpM8-7L6Qn_KCwxSR30Q%40mail.gmail.com), [#29 (Ajit Awekar, 2026-09-22)](https://www.postgresql.org/message-id/CAER375NQEqXLQ5eqFJ8QpnpnYKCHeyi8YhG6CdAm6m%3DQGc6unw%40mail.gmail.com), [#30 (Manu, 2026-09-22)](https://www.postgresql.org/message-id/179008834802.2666470.8606898268993687598%40gmail.com), [#37 (Manu, 2026-11-03)](https://www.postgresql.org/message-id/179371008812.1245613.18345043610354071644%40gmail.com)

**Question:** Why could postgres_fdw not check the remote relkind once per foreign table and connection, only for non-direct UPDATE/DELETE, and refuse when it is a foreign table?

Counter-check: held. Message #28 shows the chained case still changes wrong rows while EXPLAIN shows the tableoid condition. Message #29 rejects a relkind check for its cost on every query, which does not answer a check limited to the non-direct path.

### 4. New FDW planner API is shaped for postgres_fdw and undocumented

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

An author of another FDW, for example one for a remote sharded store, sees RemoteTableOidAttributeNumber and uses it for a different row identity. fdwhandler.sgml says nothing about its rules, so the FDW gets whole-row or setrefs errors like those in message #34. Core then has to keep the undocumented behaviour stable in every later release.

The material shows one fixed attribute number named for the remote tableoid, plus the rule that the scan must go through fdw_scan_tlist (message #21). The proposal does not say whether a second pseudo-column, or an FDW other than postgres_fdw, is supported. It also does not add any documentation for FDW authors. Once released, this becomes extension-facing behaviour that the project must keep working.

Thread: [#21 (Ajit Awekar, 2026-08-27)](https://www.postgresql.org/message-id/CAER375N8yyZhdvJMMU1fx2ZUcQmMbgj%3D8Vf9Qj3Qv_KFeczFZQ%40mail.gmail.com), [#34 (Jakub Wartak, 2026-10-28)](https://www.postgresql.org/message-id/CAKZiRmzpABcNuk%2BZrhmV3%3DGtaA2S%3DC3%3DvKw_ggXCUwQUv8D4ag%40mail.gmail.com), [#36 (Ajit Awekar, 2026-11-03)](https://www.postgresql.org/message-id/CAER375NOcbQyDGxfRDM0wuRmJx8j5TBmkXtC0QY0j2Doye%3DHSA%40mail.gmail.com)

**Question:** Is v5-0001 meant as a general FDW API, and if so, where are its rules documented and how does an FDW add more than one row-identity column?

Counter-check: held. The material shows one fixed attribute number named for the remote tableoid and no documentation for FDW authors. The C API may change between major versions, so the stability claim is overstated, but the open question about scope and documentation stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v5-0001 adds planner support for FDW row-identity pseudo-columns and the attribute number RemoteTableOidAttributeNumber. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | v5-0001 changes costsize.c, initsplan.c, setrefs.c and relnode.c, and turns planner Asserts into runtime errors. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | v5 relies on varnullingrels, so it cannot go below Postgres 16, and the thread calls its planner changes too invasive to back-patch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Later planner patches must handle one attribute number outside min_attr..max_attr, and nothing shown fails loudly if they miss it. |
| performance on the hot path | **yes** | Every non-direct UPDATE/DELETE in postgres_fdw gains a tableoid fetch and condition; with 100 remote partitions per-row commands cost about ten times more. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which planner code paths index per-attribute arrays by attno within min_attr..max_attr, and what makes a later patch that misses the pseudo-column fail in tests?
2. What fix goes to stable branches, and is the remotely_inherited option still planned alongside v5?
3. Why can postgres_fdw not check the remote relkind once per foreign table and connection, only for non-direct UPDATE/DELETE, and refuse a remote foreign table?
4. Is v5-0001 meant as a general FDW API, and if so, where are its rules documented and can an FDW add more than one row-identity column?
5. Has a committer who knows the planner agreed to review v5-0001, given that message #29 says the approach was suggested off-list?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. v5-0001 needs a committer who knows the planner's per-attribute bookkeeping and setrefs; none has reviewed it in the thread. Etsuro Fujita takes part and knows postgres_fdw, but favoured the remotely_inherited option.
- **Liability:** The owner signs for a new planner invariant, an attribute number above the normal range, for as long as the FDW pseudo-column support exists. The owner also signs for the choice of what stable branches get.
- **Author follow-up:** The authors posted revisions up to v5, answered review in messages #29 and #36, and kept the entry rebased across three CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The bug silently corrupts remote data and has two independent production reports, so it is worth attention now. The entry is Ready for Committer by a tester, but v5-0001 changes several planner files and no planner committer has reviewed it.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Back-branch fix and the remotely_inherited option are not settled with v5. Refuted (no failure scenario): The scenario needs remotely_inherited in stable branches but not in HEAD. Message #13 discusses the option for "HEAD and the stable branches", and the project commits a fix to master before back-patching it, so the restore failure has no path.
- `maintenance-fit-3` Two different fixes may live on across branches. Refuted (not specific): Duplicate of prior-attempts-1. Its added point, that back-patched fixes need different versions per branch, holds for any change that is not back-patched.

### Review angles

- prior-attempts: 1 objection(s). No precedent found in the list. Inside the thread, message #1 points to the 2018 discussion: its fix was returned with feedback, and the tableoid junk-column approach was reportedly rejected. Message #9 reports a comment that the tableoid condition was added even for non-inherited tables. Message #30 measures no slowdown on plain tables, which answers that point. The chained foreign table limitation (message #15) is documented, and message #17 accepts leaving it out.
- standards-compat: 2 objection(s). Nothing else from the standards and compatibility angle. The patch adds no SQL syntax and no GUC. The remote SQL change uses tableoid, which every supported remote server has.
- replication-ondisk: 0 objection(s). The proposal changes no persistent or replicated state: no on-disk format, WAL record, catalog, pg_upgrade step or wire protocol message. The remote SQL gains a tableoid column and condition, which any remote Postgres server already supports. The ABI effect of v5-0001 on planner structs cannot be checked without the code, and back-patching is already debated in the thread.
- security: 0 objection(s). The proposal has no privilege or exposure aspect worth objecting to. It adds a tableoid fetch and an "AND tableoid = $2" condition to remote per-row UPDATE/DELETE under the existing user mapping and remote privileges. The fix narrows the rows affected, so it closes a path where a user with UPDATE or DELETE on one remote partition could change rows elsewhere, and it creates no new role, setting or way for a non-superuser to see data. The extra per-row cost through the root is already open to any user allowed to run the statement. Turning planner Asserts into runtime errors makes a misbehaving FDW fail more safely. Without the code, I could not check whether the remote tableoid pseudo-column can show up in user-visible output beyond EXPLAIN VERBOSE.
- maintenance-fit: 3 objection(s). Hot-path cost on plain remote tables was measured in message #30 with no visible difference, so no performance objection is raised.
- already-exists: 0 objection(s). Postgres has no existing feature, option or extension that makes non-direct UPDATE/DELETE through postgres_fdw hit the right row on a remote partitioned table, so a committer cannot point to one. The workarounds in the thread (direct modify, one foreign table per remote partition in message #4, the remotely_inherited error option in messages #2 and #9) avoid or refuse the case and do not fix it. Without the code I could not check whether v5-0001 duplicates the existing add_row_identity_var path or extends it.
- demand: 0 objection(s). The material shows real demand for a silent data-corruption bug. Independent users report production hits: Dalibo in message #1 and Postgres Professional in message #6. The bug has been known since 2018, and several committers and reviewers treat it as needing a fix. Concerns about cost and backpatching belong to other angles, not demand.

### Precedents retrieved

- P1: [Push down more UPDATEs/DELETEs in postgres_fdw](https://commitfest.postgresql.org/patch/791/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P2: [Asynchronous execution on postgres-fdw](https://commitfest.postgresql.org/patch/291/) (cf, Rejected, 2015-08-10) - judged not relevant; reason summarised from its thread
- P3: [Async execution of postgres_fdw.](https://commitfest.postgresql.org/patch/32/) (cf, Rejected, 2015-05-09) - judged not relevant; reason summarised from its thread
- P4: [Revert misguided change to postgres_fdw FOR UPDATE/SHARE code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8ec8760fc87ecde0516e511f1c55aec627b01ea7) (revert, Reverted, 2014-12-12) - judged not relevant
- P5: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [postgres_fdw: support parameterized foreign joins](https://commitfest.postgresql.org/patch/1042/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P7: [Optimization for updating foreign tables in Postgres FDW](https://commitfest.postgresql.org/patch/162/) (cf, Returned with feedback, 2015-05-13) - judged not relevant
- P8: [postgres_fdw: evaluate placeholdervars on remote server](https://commitfest.postgresql.org/patch/1055/) (cf, Returned with feedback, 2017-09-27) - judged not relevant; reason summarised from its thread
- P9: [Kerberos delegation support in libpq and postgres_fdw](https://commitfest.postgresql.org/patch/3265/) (cf, Returned with feedback, 2021-11-26) - judged not relevant; reason summarised from its thread
- P10: [postgres_fdw super user checks](https://commitfest.postgresql.org/patch/824/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P11: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P12: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P13: [Avoid computing ORDER BY junk columns unnecessarily](https://commitfest.postgresql.org/patch/4717/) (cf, Returned with feedback, 2024-02-10) - judged not relevant
- P14: [Output affected rows in EXPLAIN](https://commitfest.postgresql.org/patch/4551/) (cf, Rejected, 2023-11-22) - judged not relevant
- P15: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P16: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P17: [Refactoring postgres_fdw/connection.c](https://commitfest.postgresql.org/patch/3782/) (cf, Withdrawn, 2023-05-23) - judged not relevant
- P18: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P19: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P20: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P21: [A contrib extension to help prevent common errors in DML](https://commitfest.postgresql.org/patch/948/) (cf, Returned with feedback, 2017-03-16) - judged not relevant
- P22: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2409/) (cf, Returned with feedback, 2020-01-28) - judged not relevant
- P23: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2499/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P24: [Add and report the new "session_read_only" GUC pseudo-variable.](https://commitfest.postgresql.org/patch/1090/) (cf, Returned with feedback, 2019-02-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 147 s; cost reported by the CLI: $3.07
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 78 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:03:16.330Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
