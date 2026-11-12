# Design pre-review: Fix race with LLVM and bison.

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7206](https://commitfest.postgresql.org/patch/7206/) (Needs review); authors: Andres Freund (andresfreund)
- Thread: [Fix race with LLVM and bison.](https://www.postgresql.org/message-id/flat/f070a61e-1dd3-4fd9-bbc9-18d100b6722a%40postgrespro.ru), started 2026-03-27
- Messages read: 5 of 5
- Patch prose read: v3-0001-Fix-race-with-LLVM-and-bison.patch, v1-0002-Patch-simplify-reproducing-of-race-with-LLVM-and-.patch, v1-0001-Fix-race-with-LLVM-compilation-and-bison-code-gen.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the build failure is reported with a reproduction, and the fix copies the rule src/backend/common.mk already uses. The one surviving objection, maintenance-fit-1, is a fixable scope point at level 2. The remaining decision is the backpatch risk for out-of-tree extensions, which a committer who knows the make build system has to judge.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it cannot be confirmed which pgxs.mk bitcode targets get the new dependency, or what the change to src/backend/common.mk does. The commit message names only MODULES.

## The proposal, restated

**Problem.** With --with-llvm and parallel make, contrib/cube and contrib/seg can fail because bitcode for cubescan.c or segscan.c compiles before bison generates cubeparse.h or segparse.h. Zsolt Parragi reports that pg_plan_advice is affected too. The failure hits anyone building contrib or pgxs extensions with LLVM enabled.

**Proposed design.** The patch makes each .bc file for MODULES in pgxs.mk depend on its .o file, as src/backend/common.mk already does for backend objects. The .o file already carries the dependencies on generated headers, so the .bc build waits for them. An earlier alternative added per-module rules only to the cube and seg Makefiles.

**User-visible surface.**
- Changed make dependency rules in src/makefiles/pgxs.mk, which out-of-tree extensions use to build bitcode.

**What the author claims.**
- Andres Freund says `make -C contrib/cube cubescan.bc` shows the problem without the sleep-based reproduction patch.
- Andres Freund says the backend avoids the race because each .bc file already depends on the .o file.
- Andres Freund fears that backpatching could break some extensions, so he leans toward fixing master only.
- Zsolt Parragi says a generic fix in pgxs.mk covers pg_plan_advice and any future affected module.

**Thread state.** Andres Freund posted the generic fix, and Maksim Melnikov and Zsolt Parragi agreed with it; Maksim added it to the CommitFest as v3. Nobody has decided on backpatching, and Maksim's idea of moving the rule into src/Makefile.global.in got no reply.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-03-27; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Stated scope covers MODULES only, not MODULE_big or PROGRAM objects

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

An out-of-tree extension builds with MODULE_big and a bison-generated header under --with-llvm and make -j. Its .bc files carry no dependency on the .o files, so the build fails at random. The author finds out from a packager's failed build.

The v3 commit message says the .bc files "for MODULES" now depend on their .o files. pgxs also builds bitcode for OBJS under MODULE_big, and the thread does not say whether those targets get the same rule. Message #4 counts on the fix to cover pg_plan_advice "and all future cases", which holds only if every bitcode path in pgxs.mk gets the dependency. If one path is left out, the next module with a generated header hits the same race, and nothing in the build warns about the gap.

Thread: [#2 (Andres Freund, 2026-03-27)](https://www.postgresql.org/message-id/v67jke2iv6kaywjnvwhxc3u5x3ttvrz555k5ngcxu2qwu435rl%405wpsxnrsz23k), [#4 (Zsolt Parragi, 2026-03-30)](https://www.postgresql.org/message-id/CAN4CZFP-HxkwSDsFuBAekdO6C3V13CS%3D1b0oPRz_TB-EPnrRKQ%40mail.gmail.com), [#5 ("Maksim(dot)Melnikov", 2026-08-27)](https://www.postgresql.org/message-id/03a56df5-9925-44bf-9c45-0fbf3fb1f311%40postgrespro.ru)

**Question:** Does v3 add the .bc-on-.o dependency for every bitcode target pgxs.mk builds, including MODULE_big OBJS, and is pg_plan_advice fixed by it?

Counter-check: held. contrib/cube and contrib/seg build with MODULE_big and OBJS, so a fix for the reported failure has to cover that path. But the v3 commit message names only MODULES, and no message confirms a test of v3, so the question stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes dependency rules in src/makefiles/pgxs.mk, which out-of-tree extensions use to build bitcode. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #2 leans toward master only for fear of breaking extensions; nobody answered the per-module backpatch idea in message #3. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The .bc-on-.o rule would exist in both common.mk and pgxs.mk, and each bitcode path in pgxs.mk needs it. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does v3 add the .bc-on-.o dependency for every bitcode target pgxs.mk builds, including MODULE_big OBJS, which contrib/cube and contrib/seg use?
2. Has anyone run `make -C contrib/cube cubescan.bc` and a parallel build of pg_plan_advice with v3 applied, and did both succeed?
3. Which extension setup could break if the pgxs.mk change were backpatched, as message #2 fears?
4. Should back branches get the per-module v2 fix for cube and seg, or no fix at all?
5. Does the meson build have the same race between bitcode compilation and bison-generated headers?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund wrote the generic fix in message #2 and says he wrote the original backend rule; the owner needs to know the make build system and LLVM bitcode rules.
- **Liability:** The owner signs for build behaviour of every pgxs extension built with --with-llvm, including out-of-tree ones, for as long as the make build is supported.
- **Author follow-up:** Maksim Melnikov replied within three days and registered v3 in the CommitFest; no sign either way from Andres Freund after message #2 in the visible thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 2 KiB, touches two makefiles, and fixes a build failure reported for cube, seg and pg_plan_advice. It copies an existing rule, so the project carries little new after commit.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P9] reverted a dtrace makefile dependency change because it broke parallel builds, but it only shares the area. This patch adds an ordering edge from each .bc file to its .o file, and src/backend/common.mk already uses that pattern, so the [P9] reason does not carry over. The thread cites no earlier attempt at this fix.
- standards-compat: 0 objection(s). The change brings pgxs.mk in line with the rule src/backend/common.mk already uses, and adds no syntax, GUC, catalog object or dump state. Message #2 names a backpatch risk for extensions but gives no concrete setup that breaks, and the proposal targets master only, so no failure scenario can be written. Open points that need the code: whether MODULE_big OBJS bitcode gets the same rule, and whether meson has the same race. [P9] is about dtrace dependencies and does not apply here.
- replication-ondisk: 0 objection(s). The proposal changes only make dependency rules in pgxs.mk and src/backend/common.mk. It touches no on-disk format, WAL, catalog, replication, wire protocol or C ABI, so this angle has nothing to object to; the pgxs effect on out-of-tree extension builds belongs to another angle.
- security: 0 objection(s). The change only adds make dependencies so that .bc files wait for their .o files in pgxs builds. It touches no privilege checks, roles, data access paths, authentication, or server runtime behaviour, so it has no security or privilege aspect to object to.
- maintenance-fit: 1 objection(s). The core idea fits core well. It copies the rule src/backend/common.mk already uses, adds no hot-path cost, and a committer who knows this area wrote it. The one objection is about scope, and a new version can fix it.
- already-exists: 0 objection(s). This is a build-system bug fix. Nothing in Postgres or in an extension already makes .bc files for pgxs MODULES wait for generated headers. The patch copies the existing rule in src/backend/common.mk instead of adding a second mechanism, so the existing-feature and extension angle has nothing to raise. [P9] is about dtrace dependencies and does not apply.
- demand: 0 objection(s). Demand is shown. This is a build bug that fails `make world-bin -j3` with --with-llvm, and two people reported it independently for cube and seg (message #1) and for pg_plan_advice (message #4). The fix mirrors existing practice in src/backend/common.mk, and the cost falls only on LLVM builds of pgxs modules.

### Precedents retrieved

- P1: [Add llvm version into the version string](https://commitfest.postgresql.org/patch/5202/) (cf, Returned with feedback, 2024-09-24) - judged not relevant; reason summarised from its thread
- P2: [Fix race condition in SSI when reading PredXact->SxactGlobalXmin](https://commitfest.postgresql.org/patch/6145/) (cf, Withdrawn, 2025-10-22) - judged not relevant; reason summarised from its thread
- P3: [Revert "Fix race in Parallel Hash Join batch cleanup."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7f7f25f15edb6eacec58179ef5285e874aa4435b) (revert, Reverted, 2021-03-17) - judged not relevant
- P4: [Avoid orphaned dependencies](https://commitfest.postgresql.org/patch/3106/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P5: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant; reason summarised from its thread
- P6: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - judged not relevant
- P7: [Revert ill-fated patch to release exclusive lock early after vacuum truncates a table. Introduces race condition, as shown by buildfarm failures.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=eb5f4d6c5cece3e3fa80a506153e5202b093b7ca) (revert, Reverted, 2007-09-20) - judged not relevant
- P8: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - judged not relevant
- P9: [Revert "Have dtrace depend on object files directly, not objfiles.txt"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=08fbad0afd62690cc82990c0504529ef238ac24d) (revert, Reverted, 2015-10-15) - used in the review
- P10: [Allow COPY "text" to output a header and add header matching mode to COPY FROM](https://commitfest.postgresql.org/patch/2504/) (cf, Returned with feedback, 2020-09-29) - judged not relevant; reason summarised from its thread
- P11: [Revert addition of pg_terminate_backend() because of race conditions.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=76365960d229dca0ad8454040233e3eaaeb413bb) (revert, Reverted, 2008-04-15) - judged not relevant
- P12: [Revert haphazard pgxs makefile changes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e4b5a070b4d25da2c2f80af306fa1d19044370ae) (revert, Reverted, 2014-12-04) - judged not relevant
- P13: [Close race in DROP TABLESPACE on Windows](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Tablespaces) (do_not_want, Not worth doing, date unknown) - judged not relevant; reason summarised from its thread
- P14: [create index failed. PARALLEL UNSAFE function does not prevent parallel index build](https://commitfest.postgresql.org/patch/4860/) (cf, Rejected, 2024-03-13) - judged not relevant; reason summarised from its thread
- P15: [Enable jitlink as an alternative jit linker of legacy Rtdyld and add riscv jitting support](https://commitfest.postgresql.org/patch/3857/) (cf, Returned with feedback, 2023-09-01) - judged not relevant; reason summarised from its thread
- P16: [Revert "Allow parallel workers to cope with a newly-created session user ID."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e6a9637488e2673efb87f8ead657789e9889fb17) (revert, Reverted, 2024-08-01) - judged not relevant
- P17: [Lazy JIT IR code generation to increase JIT speed with partitions](https://commitfest.postgresql.org/patch/3071/) (cf, Withdrawn, 2023-01-27) - judged not relevant
- P18: [List TAP test files in makefiles](https://commitfest.postgresql.org/patch/6011/) (cf, Rejected, 2025-12-03) - judged not relevant
- P19: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant
- P20: [Fix PGXS so built control files work with VPATH builds](https://commitfest.postgresql.org/patch/2461/) (cf, Returned with feedback, 2020-07-31) - judged not relevant
- P21: [Revert 29854ee8d1 due to buildfarm failures](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=817bb0a7d1e02df4643d37304aed8574cf43f629) (revert, Reverted, 2021-06-15) - judged not relevant
- P22: [Revert "Fix bogus %name-prefix option syntax in all our Bison files."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71ed8b3ca76cd720f4013c3c20f0d2706583ab9a) (revert, Reverted, 2014-05-28) - judged not relevant
- P23: [Revert "Replace PostmasterRandom() with a stronger way of generating randomness."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faae1c918e8aaae034eaf3ea103fcb6ba9adc5ab) (revert, Reverted, 2016-10-18) - judged not relevant
- P24: [Parse dates that include CE and BCE](https://commitfest.postgresql.org/patch/3682/) (cf, Returned with feedback, 2022-08-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 92 s; cost reported by the CLI: $0.72
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:29:17.857Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
