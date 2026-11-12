# Design pre-review: Fix: VACUUM FULL or CREATE INDEX fails with error "missing chunk number 0 for toast value XXX"

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6824](https://commitfest.postgresql.org/patch/6824/) (Needs review); authors: Michael Paquier (michael-kun), Ekaterina Testova (e.testova)
- Thread: [VACUUM FULL or CREATE INDEX fails with error: missing chunk number 0 for toast value XXX](https://www.postgresql.org/message-id/flat/1779078055.630771314%40fmail2.qdit), started 2026-05-18
- Messages read: 7 of 7
- Patch prose read: caching_oldest_xmin.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is reproducible, demand is shown, and no surviving objection questions the goal. Two fixable design points remain. standards-compat-1 asks what the fix reports when a chunk is missing, so that real corruption stays visible. maintenance-fit-1 asks where the tolerant handling lives, since CLUSTER, VACUUM FULL and REPACK still need it. Both need someone who knows heap visibility and TOAST to sign.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The aggressive-detoasting patch is in another thread and was not read. Its error reporting, its cost on wide toasted columns and its effect on the table AM interface could not be judged.

## The proposal, restated

**Problem.** CREATE INDEX and VACUUM FULL can fail with "missing chunk number 0 for toast value" when the TOAST table removed chunks of a tuple that the main table still treats as RECENTLY_DEAD. The thread links this to bug #18351, which was reported earlier and not resolved. The error carries SQLSTATE XX001 (data corrupted), so operators with corruption monitoring get false alerts.

**Proposed design.** The first prototype stores the parent table's OldestXmin in VacuumParams and reuses it when VACUUM processes the TOAST table, so both use the same horizon. Matthias van de Meent replied that on-access pruning and btree index tuple removal can still remove the TOAST data. Michael Paquier then posted, in another thread linked to this CF entry, a patch that detoasts attributes more aggressively during index builds.

**What the author claims.**
- The bug comes from different horizons: ComputeXidHorizons filters by database, but GetSnapshotData does not.
- The OldestXmin prototype fails make check, with freezing cutoffs too far in the past and assertion failures.
- A definitely_needed check in CREATE INDEX could slow it down.
- Michael Paquier says aggressive detoasting is fine for index builds, which need the detoasted values anyway.
- Andrey Borodin says each occurrence triggers corruption alerts in his fleet monitoring.

**Thread state.** All 7 messages are visible. Discussion moved to a separate thread where Michael Paquier posted an aggressive-detoasting patch for index builds; the CLUSTER, VACUUM FULL and REPACK paths still need a fix, and the OldestXmin prototype is effectively set aside.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-05-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Skipping tuples with missing TOAST data hides real corruption from CREATE INDEX

fixable · level 2, rubric item 12 · design, backpatchability · `standards-compat-1`

A table has a real lost TOAST chunk on a RECENTLY_DEAD tuple. Today CREATE INDEX or VACUUM FULL stops with XX001, and monitoring alerts. With the skip-on-missing design the command succeeds silently, and the operator learns of the damage only if a later read of a live tuple hits it.

Message #2 proposes ignoring TOAST visibility errors during index builds, and message #3 links a patch based on aggressive detoasting. Message #4 raises the masking concern, and no message answers it. Message #6 shows operators depend on XX001 meaning corruption. The proposal does not say whether a skipped chunk is logged, gets another SQLSTATE, or is limited to tuples the code can prove were removed by the race, and that choice changes what existing monitoring sees after upgrade or a back-patch.

Thread: [#2 (Matthias van de Meent, 2026-07-06)](https://www.postgresql.org/message-id/CAEze2WgRUfoR%3DtSQQm8zexVNeBbEYi9ZSPztrK63ObyfPxKpBw%40mail.gmail.com), [#3 (Michael Paquier, 2026-07-08)](https://www.postgresql.org/message-id/ak2eB2XgrvDw3VGN%40paquier.xyz), [#4 (Тестова Екатерина, 2026-07-08)](https://www.postgresql.org/message-id/1783496056.499330176%40fmail1.qdit), [#6 (Andrey Borodin, 2026-07-14)](https://www.postgresql.org/message-id/4D42CDAE-C881-4AE4-886B-C83EF2468C08%40yandex-team.ru)

**Question:** When a RECENTLY_DEAD tuple's TOAST data is missing, what does the fix report, and how does it tell this race apart from real corruption?

Counter-check: held. Message #4 raises the masking concern and no later message answers it. Message #6 shows monitoring depends on XX001, and the visible material does not say what the fix reports when a chunk is missing.

### 2. Missing-chunk tolerance lives in each caller, so new paths can regress

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

Someone later adds or changes a path that reads TOAST data of RECENTLY_DEAD tuples, such as a REPACK variant, a table AM's index build scan, or an extension's rewrite. That path does not repeat the detoast-and-tolerate step, and no test or assertion flags it. Users then get XX001 "missing chunk" errors again, found only through corruption alerts in production.

Message #2 says missing TOAST data is to be expected for RECENTLY_DEAD tuples, so every reader of such tuples must handle it. The fix in message #3 covers only the heap index build. It says CLUSTER, VACUUM FULL and REPACK need separate, deeper work. The material shows no shared helper or table AM contract, so each table AM and each future maintenance path must find out about this race on its own.

Thread: [#2 (Matthias van de Meent, 2026-07-06)](https://www.postgresql.org/message-id/CAEze2WgRUfoR%3DtSQQm8zexVNeBbEYi9ZSPztrK63ObyfPxKpBw%40mail.gmail.com), [#3 (Michael Paquier, 2026-07-08)](https://www.postgresql.org/message-id/ak2eB2XgrvDw3VGN%40paquier.xyz)

**Question:** Will the tolerant handling sit in one shared helper or a documented table AM rule that future copy and build paths must use, and what test fails if a path skips it?

Counter-check: held. Message #3 confirms the fix covers only index builds and that CLUSTER, VACUUM FULL and REPACK need separate work, so at least two paths need the same handling. The table AM and extension part is speculative, because other AMs manage their own TOAST storage.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The prototype adds fields to VacuumParams in vacuum.h, and the detoasting fix changes what the heap index build scan passes to callbacks. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix and the bug reproduces on 17.10, but the target version is 20. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each path that reads TOAST data of RECENTLY_DEAD tuples needs the tolerant handling, and CLUSTER, VACUUM FULL and REPACK are not yet covered. |
| performance on the hot path | **yes** | Aggressive detoasting runs in every index build scan, and its cost on large toasted columns is not shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. When a RECENTLY_DEAD tuple's TOAST data is missing, what does the fix report, and how does it tell this race apart from real corruption?
2. Will the tolerant handling sit in one shared helper that index build, CLUSTER, VACUUM FULL and REPACK all use, and what test fails if a path skips it?
3. How will copy_for_cluster() handle a RECENTLY_DEAD tuple whose external TOAST value is copied by reference and is already gone?
4. Which patch should reviewers of this CF entry read: the OldestXmin prototype, the aggressive-detoasting patch, or both?
5. Will the fix be back-patched to the supported branches where the bug reproduces, and does it change any struct or callback that extensions use?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier is listed as author and posted the detoasting patch in message #3. The owner must know heap visibility horizons, TOAST access and the table rewrite code.
- **Liability:** The owner signs for the rule that missing TOAST data of RECENTLY_DEAD tuples is tolerated without hiding real corruption, in every maintenance path and in each back-patched branch, for as long as those branches are supported.
- **Author follow-up:** Michael Paquier attached his patch thread to this entry in July and set the target version. The entry has needed a rebase since 2026-10-17, and the VACUUM FULL part has no patch yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The race raises XX001 corruption alerts in a production fleet and was reported twice, so the fix has clear value. The detoasting patch was not read; it touches heap index builds, and the rewrite paths still need a design.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. Message #1 says bug #18351 was reported earlier but not resolved, without giving a reason that could be weighed here. Matthias van de Meent rejected the OldestXmin prototype in message #2, and the discussion then moved to the aggressive-detoasting patch, so no earlier rejection reason applies to the current approach.
- standards-compat: 1 objection(s). The proposal adds no syntax, GUC, catalog object or psql change. The only contract issue in this angle is how errors are reported, covered above.
- replication-ondisk: 0 objection(s). No on-disk format, WAL, catalog or wire protocol change is proposed. The set-aside prototype only adds VacuumParams fields, and the open questions about copy_for_cluster() and back-patch ABI cannot be judged without the detoasting patch's interface.
- security: 0 objection(s). The proposal has no privilege or exposure aspect: it changes no privilege, role, default or visible data, and CREATE INDEX and VACUUM FULL keep their ownership checks. Skipping a missing chunk could hide real corruption, but message #4 already raises that, and it is an integrity question for another angle.
- maintenance-fit: 1 objection(s). The bug fix is plainly desirable, and the people working on it know the area: a committer wrote the current patch, and two experienced reviewers are engaged. Without the detoasting patch I could not check its cost for wide toasted columns or its effect on the table AM interface.
- already-exists: 0 objection(s). This is a bug fix for a race that makes CREATE INDEX and VACUUM FULL fail with an XX001 error. No existing feature, setting or extension prevents it, and the allowlist workaround in message #6 does not fix the race. Other angles cover design gaps, such as copy_for_cluster() and possible masking of real corruption.
- demand: 0 objection(s). Demand is shown. This is a bug fix for a reproducible error first reported as bug #18351 and reported again in a second thread (message #7). Message #6 describes a production fleet whose corruption monitoring pages operators on each occurrence, and several committers and reviewers are engaged.

### Precedents retrieved

- P1: [Support for 8-byte TOAST values (aka the TOAST infinite loop problem)](https://commitfest.postgresql.org/patch/5830/) (cf, Returned with feedback, 2026-02-10) - judged not relevant; reason summarised from its thread
- P2: [Refuse setting toast.* reloptions when TOAST relation does not exist](https://commitfest.postgresql.org/patch/1486/) (cf, Returned with feedback, 2018-04-10) - judged not relevant; reason summarised from its thread
- P3: [Add MAIN_RELATION_CLEANUP and TOAST_TABLE_CLEANUP options to VACUUM](https://commitfest.postgresql.org/patch/2426/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P4: [Avoiding deadlock errors in CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/2141/) (cf, Returned with feedback, 2019-11-25) - judged not relevant; reason summarised from its thread
- P5: [Full support for index LP_DEAD hint bits on standby](https://commitfest.postgresql.org/patch/2947/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P6: [create index failed. PARALLEL UNSAFE function does not prevent parallel index build](https://commitfest.postgresql.org/patch/4860/) (cf, Rejected, 2024-03-13) - judged not relevant; reason summarised from its thread
- P7: [Pluggable toaster](https://commitfest.postgresql.org/patch/3490/) (cf, Rejected, 2023-04-03) - judged not relevant; reason summarised from its thread
- P8: [Doc: Fix definition of asynchronous replication](https://commitfest.postgresql.org/patch/6005/) (cf, Withdrawn, 2025-10-03) - judged not relevant; reason summarised from its thread
- P9: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P10: [\[doc\] vacuum full, fillfactor, and "extra space"](https://commitfest.postgresql.org/patch/2412/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P11: [64-bit TOAST value ID](https://commitfest.postgresql.org/patch/4296/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P12: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P13: [Parallel CREATE INDEX for GiST indexes](https://commitfest.postgresql.org/patch/5026/) (cf, Returned with feedback, 2025-02-06) - judged not relevant
- P14: [Revert recent changes to 002_pg_upgrade.pl.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e22f675fd81ba1d96b0b9a34bbf26d5ec532d5) (revert, Reverted, 2022-08-04) - judged not relevant
- P15: [Erase the distinctClause if the result is unique by definition](https://commitfest.postgresql.org/patch/2433/) (cf, Returned with feedback, 2022-04-05) - judged not relevant
- P16: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P17: [Revert "vacuumdb: Add option for analyzing only relations missing stats."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19e57f4f78e4354d9a21c284868373d28bb0d368) (revert, Reverted, 2025-03-07) - judged not relevant
- P18: [fix and document CLUSTER privileges](https://commitfest.postgresql.org/patch/4054/) (cf, Returned with feedback, 2023-01-14) - judged not relevant
- P19: [CLUSTER command progress monitor](https://commitfest.postgresql.org/patch/1268/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P20: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant
- P21: [Add TOAST support for more system tables](https://commitfest.postgresql.org/patch/4449/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P22: [Revert MAINTAIN privilege and pg_maintain predefined role.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=151c22deee66a3390ca9a1c3675e29de54ae73fc) (revert, Reverted, 2023-07-07) - judged not relevant
- P23: [Close race in DROP TABLESPACE on Windows](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Tablespaces) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P24: [Vacuuming relations on full tablespaces (Add EMERGENCY option)](https://commitfest.postgresql.org/patch/649/) (cf, Returned with feedback, 2016-09-28) - judged not relevant

### Warnings

- CommitFest entry has 2 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/1779078055.630771314%40fmail2.qdit). Others: https://www.postgresql.org/message-id/flat/19519-fe02d8ff679d834d%40postgresql.org

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 124 s; cost reported by the CLI: $1.74
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:04:35.337Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
