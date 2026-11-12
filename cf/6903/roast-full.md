# Design pre-review: Two bugs around ALTER TYPE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6903](https://commitfest.postgresql.org/patch/6903/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Two bugs around ALTER TYPE](https://www.postgresql.org/message-id/flat/39977E94-40DD-4176-A191-CE3D120968B2%40yandex-team.ru), started 2026-04-14
- Messages read: 5 of 5
- Patch prose read: v2-0001-Add-regression-test-for-SETOF-composite-invalidat.patch, v2-0005-Address-Surya-s-feedback.patch, v2-0004-Lock-SETOF-composite-return-type-during-plpgsql-e.patch, v2-0003-Add-TAP-test-for-plpgsql-RETURN-QUERY-DDL-race.patch, v2-0002-Track-RTE_FUNCTION-composite-rowtype-dependencies.patch, 0001-Add-regression-test-for-SETOF-composite-invalidation.patch, 0002-Track-RTE_FUNCTION-composite-rowtype-dependencies.patch, 0003-Add-TAP-test-for-plpgsql-RETURN-QUERY-DDL-race.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

Both bugs are reported by users and reproduced by a reviewer, and 0002 uses the existing relationOids invalidation channel. The surviving objections are fixable design points about the 0004 lock: its duration and back-branch status (standards-compat-1), its PL/pgSQL-only scope (standards-compat-2) and its effect on hot standby replay (replication-ondisk-1). None questions the goal. A committer who knows locking and the plan cache has to judge the new blocking behavior.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it is not possible to check that 0002 covers all paths that build function RTE dependencies, or exactly when 0004 takes the lock relative to the cached descriptor.

## The proposal, restated

**Problem.** PL/pgSQL functions that use RETURN QUERY over a non-inlined SETOF function returning a named composite fail with "structure of query does not match function result type" after ALTER TYPE ... ADD ATTRIBUTE or ALTER TABLE ... ADD COLUMN. Users reported it. A second form of the failure happens when a concurrent transaction alters the type while RETURN QUERY is running, and CREATE OR REPLACE FUNCTION does not help there.

**Proposed design.** The planner records the typrelid of a named composite returned by each RTE_FUNCTION in the plan's relation dependencies, so the relcache invalidation that ALTER TYPE sends marks cached plans stale. PL/pgSQL takes AccessShareLock on that typrelid at function entry for SETOF functions returning a named composite, and holds it to transaction end. A concurrent ALTER TYPE or ALTER TABLE then waits behind the running call.

**User-visible surface.**
- Cached and prepared plans that scan SETOF functions returning a composite (also through a domain) are replanned after ALTER TYPE, as the new ALTER TYPE doc paragraph states.
- A running SETOF PL/pgSQL call holds AccessShareLock on its composite return type until transaction end, so ALTER TYPE/ALTER TABLE on that type blocks.

**What the author claims.**
- Bug 1 reproduces in a single session and the setrefs.c change makes the second call return the new 3-column shape.
- Bug 2 reproduces reliably only with injection points; an included TAP test covers it.
- The lock mirrors how table scans hold AccessShareLock to pin the schema for the statement.
- Both bugs still reproduce on the newest minor releases.

**Thread state.** Surya Poondla reproduced both bugs, confirmed relationOids is the right invalidation channel, and sent test and comment fixes that v2 addresses. Heikki Linnakangas is a listed reviewer but has not posted; the author asked again on 2026-11-10 to move the fixes forward.

## 2. Prior attempts

### Support prepared statement invalidation when result or argument types change (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4518/](https://commitfest.postgresql.org/patch/4518/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAGECzQQ3nPt5joW50FygoM_7YU9QjKO06say4c-sCceqh8vnsg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAGECzQQ3nPt5joW50FygoM_7YU9QjKO06say4c-sCceqh8vnsg@mail.gmail.com)
- **Date:** decided 2024-07-25; started 2023-08-25
- **Outcome:** Withdrawn (cf)
- **Who objected:** Tom Lane, Shay Rojansky (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane called the change a wire protocol break: a client that already ran Describe Statement has no way to learn that the result description is now wrong, and the error exists to prevent that. Shay Rojansky noted that the Npgsql driver caches RowDescriptions. The author agreed and withdrew the patch until the protocol has a way to signal a new RowDescription, for which he needs the separate protocol-version work.
- **Reason (quote):** "What if the client has previously done a Describe Statement on the prepared statement? We have no mechanism for notifying it that that information is now falsified." ([message](https://www.postgresql.org/message-id/1238027.1721835596%40sss.pgh.pa.us))
- **What is different this time:** P2 removed the "cached plan must not change result type" check so clients would get a new RowDescription. This proposal only adds the composite typrelid to plan dependencies and does not change what is reported to clients, so the reason P2 was withdrawn does not apply.


## 3. Objections a reviewer is likely to raise

### 1. New lock changes ALTER TYPE blocking for existing users, mainly if backpatched

fixable · level 2, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

An application keeps a transaction open after calling a SETOF PL/pgSQL function. A deploy script runs ALTER TYPE on the standalone composite type and waits on its AccessExclusiveLock. Every new call to any SETOF function returning that type then queues behind it, and the script that used to finish at once now stalls the application.

Patch 0004 takes AccessShareLock at function entry and holds it to transaction end. Message #1 justifies it as keeping the row shape stable within one statement, which is a shorter span than the lock covers. Today, ALTER TYPE on a standalone composite does not wait for function callers, so this is a new blocking contract. The material does not say whether the fix targets back branches, where such a change in locking behavior would surprise users after a minor upgrade.

Thread: [#1 (Andrey Borodin, 2026-04-14)](https://www.postgresql.org/message-id/39977E94-40DD-4176-A191-CE3D120968B2%40yandex-team.ru), [#5 (Andrey Borodin, 2026-11-10)](https://www.postgresql.org/message-id/69E6F43A-76A1-4A68-B03E-B6CC769944FC%40yandex-team.ru)

**Question:** Is the lock meant for back branches, and why must it last to transaction end instead of the end of the calling statement?

Counter-check: held. Message #1 states that the lock lasts to transaction end, while its stated goal is a stable row shape within one statement. The thread does not say whether 0004 targets back branches, and no message weighs the DDL wait.

### 2. Race fix covers only PL/pgSQL, other function languages keep the race

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A user ports the reported function to a SQL-language or PL/Python SETOF function returning the same composite. A concurrent ALTER TYPE plus CREATE OR REPLACE FUNCTION commits mid-call. The user hits the same row-shape mismatch error that the release notes say is fixed.

Message #1 places the root cause in plpgsql_estate_setup(), and patch 0004 touches only pl_exec.c. The condition it guards against is a tuple descriptor captured at call start and then replanned against a changed type. That condition is not specific to PL/pgSQL. A committer would likely ask for a fix in the common function-call or SRF path, or for a stated reason why other languages are safe.

Thread: [#1 (Andrey Borodin, 2026-04-14)](https://www.postgresql.org/message-id/39977E94-40DD-4176-A191-CE3D120968B2%40yandex-team.ru)

**Question:** Do SQL-language functions and other PLs returning SETOF a named composite have the same race, and if so, where would a common fix go?

Counter-check: held. Patch 0004 touches only pl_exec.c, and the thread does not say whether SQL-language functions or other PLs have the same race. A reason for a PL/pgSQL-only fix is a fair request.

### 3. New lock on typrelid adds hot standby replay conflicts

fixable · level 2, rubric item 8 · WAL / replication / standby, backpatchability · `replication-ondisk-1`

A reporting standby runs a long transaction that calls a SETOF PL/pgSQL function returning planinv_ct. The primary runs ALTER TYPE planinv_ct ADD ATTRIBUTE. Replay of the logged AccessExclusiveLock on typrelid now waits behind the standby's new AccessShareLock, so replay stalls up to max_standby_streaming_delay and then cancels the query.

0004 takes AccessShareLock on typrelid at function entry and holds it to transaction end (message #1). Standbys replay AccessExclusiveLock records, so this creates a conflict that did not exist before, on standbys that never run the DDL. This copies how table scans behave, which lowers the concern, but the thread covers only the primary. If the fix goes to back branches, standby operators would see new replay lag or new query cancels after a minor update.

Thread: [#1 (Andrey Borodin, 2026-04-14)](https://www.postgresql.org/message-id/39977E94-40DD-4176-A191-CE3D120968B2%40yandex-team.ru), [#4 (Andrey Borodin, 2026-07-21)](https://www.postgresql.org/message-id/95599FF6-EDE9-42B2-8940-7856F4A3A199%40yandex-team.ru)

**Question:** How does the new lock behave on a hot standby during replay of ALTER TYPE, and is that change acceptable in a minor release if this is backpatched?

Counter-check: held. Standbys replay AccessExclusiveLock records, so a standby transaction holding the new lock on a standalone composite's typrelid is a new replay conflict. Table scans behave the same way, which lowers the concern, but the thread does not discuss standbys or back branches.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | A standby transaction holding the new AccessShareLock on typrelid conflicts with replay of ALTER TYPE's AccessExclusiveLock. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: New lock changes ALTER TYPE blocking for existing users, mainly if backpatched |
| new GUC | no |  |
| planner behaviour change | **yes** | 0002 changes setrefs.c so plans scanning composite-returning functions gain a relation dependency and are replanned after ALTER TYPE. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, and 0004 adds a lock that makes ALTER TYPE wait where it did not before. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The lock lives only in pl_exec.c, and alter_type.sgml documents a new invalidation rule the project must keep. |
| performance on the hot path | **yes** | 0004 takes a lock on every call of a SETOF PL/pgSQL function returning a named composite; no measurement is shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Are 0002 and 0004 meant for back branches, or only for master?
2. Why must the 0004 lock last to transaction end instead of the end of the calling statement?
3. Do SQL-language functions and other PLs returning SETOF a named composite have the same race, and if so, where would a common fix go?
4. How does the new lock behave on a hot standby when replay reaches ALTER TYPE on the same composite type?
5. What do the extra plan dependency and the per-call lock cost for a workload that calls such functions often?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows plan cache invalidation and heavyweight lock behavior in PL/pgSQL; Heikki Linnakangas is a listed reviewer but has not posted.
- **Liability:** The owner signs for a new lock that makes ALTER TYPE and ALTER TABLE wait behind open transactions, on primaries and standbys, and for the documented invalidation rule. If backpatched, this covers all supported branches.
- **Author follow-up:** The author sent v2 addressing the reviewer's comments and asked again on 2026-11-10 to move the fixes forward. CFbot has reported a rebase need since 2026-09-08.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patches are small and fix reported bugs, but the entry has sat in 3 CommitFests with no committer posting. The 0004 lock needs judgment on blocking behavior; 0002 could go ahead separately.

## Appendix

### Refuted in the counter-check

- `security-1` New lock lets any caller block DDL on a table they cannot read. Refuted (wrong facts): Parse analysis already takes AccessShareLock on a table before any privilege check and holds it to transaction end. A role without SELECT can hold the same lock today with `begin; prepare p as select * from t;`, so 0004 adds no new capability.
- `maintenance-fit-1` Race fix covers PL/pgSQL only, not other SETOF callers. Refuted (not specific): Duplicate of standards-compat-2, which states the same PL/pgSQL-only concern with a cleaner scenario.
- `maintenance-fit-2` New lock held to transaction end changes DDL wait behavior. Refuted (not specific): Duplicate of standards-compat-1, which covers the same lock duration and back-branch question.
- `maintenance-fit-3` Point fixes may conflict with ongoing tuple descriptor versioning work. Refuted (no failure scenario): Message #5 says the versioning work is already released and both bugs still reproduce, and nothing in the thread shows a conflict with it. The setrefs.c change uses the existing relationOids mechanism, and reviewer silence belongs under committability.
- `demand-1` Bug 2 lock puts a cost on every caller to fix a rare race. Refuted (not specific): Duplicate of standards-compat-1; the failure scenario is the same DDL wait. Message #1 shows users hit the race in production even after adding CREATE OR REPLACE FUNCTION, so the demand part does not hold.

### Review angles

- prior-attempts: 0 objection(s). The only listed precedent, P2, was withdrawn because it would change the RowDescription that a client had already cached. This proposal keeps the result-type check, so that reason does not apply. The thread cites no earlier discussion of the same fix.
- standards-compat: 2 objection(s). The plan invalidation in patch 0002 follows existing relcache dependency practice, and the thread checked that it uses the right invalidation channel. No standards objection to that part.
- replication-ondisk: 1 objection(s). The plan-dependency fix (0002) changes only in-memory plan cache state. It does not touch on-disk format, WAL records, catalogs, the wire protocol or struct layouts, so it raises no concern from this angle. [P2] is about RowDescription on the wire and does not apply to SPI plans inside PL/pgSQL.
- security: 1 objection(s). The plan invalidation change in 0002 has no privilege or exposure aspect that I can see.
- maintenance-fit: 3 objection(s). The bug and the planner fix in 0002 look sound in broad strokes. A reviewer reproduced both bugs and confirmed relationOids is the right invalidation channel.
- already-exists: 0 objection(s). Postgres has no existing feature that does this job, so this angle raises nothing. For bug 1, running create or replace function on the inner SRF works around the stale plan by hand, but a manual step after every alter type is not a fix for wrong cached-plan behaviour. For bug 2, message #1 shows that create or replace function does not help, and lock table cannot lock a standalone composite type. [P2] covers result-type changes of prepared statements at the protocol level, not this RETURN QUERY case.
- demand: 1 objection(s). Bug 1 shows real demand: users reported it, it reproduces in one session, and a reviewer confirmed it (messages #1, #2). The only demand concern is the cost of the bug 2 lock compared with its narrow benefit.

### Precedents retrieved

- P1: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5974/) (cf, Withdrawn, 2025-08-19) - judged not relevant; reason summarised from its thread
- P2: [Support prepared statement invalidation when result or argument types change](https://commitfest.postgresql.org/patch/4518/) (cf, Withdrawn, 2024-07-25) - used in the review; reason summarised from its thread
- P3: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5975/) (cf, Withdrawn, 2025-12-31) - judged not relevant; reason summarised from its thread
- P4: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P5: [Revert badly-broken patch to dump comments on composite-type columns.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=18661f282bae5599abb68f53bc8b451b2d67bd75) (revert, Reverted, 2004-08-06) - judged not relevant
- P6: [Change the bool member of the Query structure to bits](https://commitfest.postgresql.org/patch/4844/) (cf, Withdrawn, 2024-02-20) - judged not relevant; reason summarised from its thread
- P7: [postgres_fdw - make cached connection functions tests meaningful with debug_invalidate_system_caches_always](https://commitfest.postgresql.org/patch/3112/) (cf, Rejected, 2021-07-21) - judged not relevant; reason summarised from its thread
- P8: [Implicit prepare of statements: replace literals with parameters and store cached plans](https://commitfest.postgresql.org/patch/1150/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P9: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant; reason summarised from its thread
- P10: [Modify the planner to better estimate caching effects](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P11: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P12: [Avoid orphaned dependencies](https://commitfest.postgresql.org/patch/3106/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P13: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P14: [Revert: Avoid looping over all type cache entries in TypeCacheRelCallback()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8daa62a10c911c851f7e9ec5ef7b90cfd4b73212) (revert, Reverted, 2024-08-25) - judged not relevant
- P15: [Remove dependence on integer wrapping](https://commitfest.postgresql.org/patch/5035/) (cf, Withdrawn, 2024-06-17) - judged not relevant
- P16: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant
- P17: [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (cf, Withdrawn, 2024-03-28) - judged not relevant
- P18: [Fix NULL dereference in pg_get_database_ddl()](https://commitfest.postgresql.org/patch/6669/) (cf, Withdrawn, 2026-04-13) - judged not relevant
- P19: [Revert "Change mbbisearch to return the character range"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f8c8a8bccc23f6ca38f7a92c9a614e73fa1fcfb6) (revert, Reverted, 2021-08-26) - judged not relevant
- P20: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P21: [Revert "Stabilize output of new regression test case".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cbadba8dd632fc0d4162f7d686fec631bce7dfd0) (revert, Reverted, 2018-02-14) - judged not relevant
- P22: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant
- P23: [revert "Throw error for ALTER TABLE RESET of an invalid option"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cb74a67e26523eb2408f441bfc589c80f76c465) (revert, Reverted, 2014-08-26) - judged not relevant
- P24: [Revert my ill-considered change that made formrdesc not insert the correct relation rowtype OID into the relcache entries it builds.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca70c3cfda609d3b817c16b48f25c086c11b9e4c) (revert, Reverted, 2009-09-26) - judged not relevant

### Warnings

- attachment limit (8) reached; 0004-Lock-SETOF-composite-return-type-during-plpgsql-exec.patch not read
- review angle "standards-compat" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 146 s; cost reported by the CLI: $1.53
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 86 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:07:31.593Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
