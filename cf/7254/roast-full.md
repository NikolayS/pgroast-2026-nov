# Design pre-review: Allow bare library names for non-superuser LOAD

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7254](https://commitfest.postgresql.org/patch/7254/) (Needs review); authors: ChangAo Chen (chenchangao)
- Thread: [\[PATCH\] Allow bare library names for non-superuser LOAD](https://www.postgresql.org/message-id/flat/tencent_E6968C151EDE818A1D4CA85FA1772D668A06%40qq.com), started 2026-09-03
- Messages read: 8 of 8
- Patch prose read: v1-0001-Allow-bare-library-names-for-non-superuser-LOAD.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is modest and no objection is fatal. The open point is standards-compat-1: after the patch, `LOAD 'demo'` loads `$libdir/demo` for a superuser and `$libdir/plugins/demo` for other roles, with no message. Message #6 shows the case, and the answer in message #7 was not accepted in message #8. demand-1 is a secondary point: convenience is the only stated motive, and the explicit prefix works today.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check how the patch detects a bare name, including Windows path separators, or whether dynamic_library_path is ever consulted. Only 3 of 8 messages were read directly.

## The proposal, restated

**Problem.** Non-superusers may LOAD only libraries in $libdir/plugins, and they must write that prefix explicitly, as in LOAD '$libdir/plugins/mylib'. The author finds this inconvenient and inconsistent with local_preload_libraries, which prepends $libdir/plugins to bare names.

**Proposed design.** The patch moves the $libdir/plugins expansion from load_libraries() into load_file(), so every restricted load handles bare names the same way. When a non-superuser runs LOAD with a bare file name, Postgres prepends $libdir/plugins/. Names with a path must still begin with exactly that string.

**User-visible surface.**
- LOAD 'name' run by a non-superuser now resolves a bare name to $libdir/plugins/name instead of failing.
- The same LOAD 'name' resolves to different files for a superuser and a non-superuser when both directories hold a library of that name.
- A non-superuser LOAD of a bare name not present in plugins now fails with "could not access file" instead of "access to library ... is not allowed".
- The LOAD reference page documents the automatic prefix.

**What the author claims.**
- Writing the $libdir/plugins/ prefix is inconvenient for non-superusers.
- The change makes LOAD consistent with local_preload_libraries.
- Documenting the expansion is enough to justify the changed error message.
- Admins should avoid installing different libraries with the same name, which removes the ambiguity concern.

**Thread state.** Japin Li and solai v found the patch fine, and solai v tested that path escapes are still rejected. Jim Jones objected that the same command loads different files per role, accepted the error-message change if documented, and left the role-dependent semantics open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-03; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Same LOAD string resolves to different files depending on role

fixable · level 2, rubric item 12 · design, security / privileges · `standards-compat-1`

An admin tests a setup script as superuser, where `LOAD 'demo'` loads `$libdir/demo`. The same script run by an application role, or after `SET ROLE`, loads `$libdir/plugins/demo` without any message. Before the patch that run failed with an error, as message #6 shows with auto_explain.

local_preload_libraries prepends `$libdir/plugins` for every role, so it does not depend on the role. The patch makes one LOAD string mean two paths depending on `superuser()`, so the consistency claim holds only for the restricted branch. Message #7 answers that admins should not install same-named libraries, and message #8 calls this trading a server guarantee for admin discipline. The thread did not settle the point, and nothing in the proposal detects or reports the ambiguity.

Thread: [#6 (Jim Jones, 2026-09-09)](https://www.postgresql.org/message-id/b08c5b95-f36d-412b-a936-7e710df6d24e%40uni-muenster.de), [#7 (cca5507, 2026-09-09)](https://www.postgresql.org/message-id/tencent_170EF0BA0E2EEC98439B0B536AA352006409%40qq.com), [#8 (Jim Jones, 2026-09-09)](https://www.postgresql.org/message-id/fbf5df8b-2c09-4153-bcad-c7682686d632%40uni-muenster.de)

**Question:** Why not resolve bare names the same way for all roles, or raise an error when the name exists in both `$libdir` and `$libdir/plugins`?

Counter-check: held. Message #6 reproduces the case, and message #8 leaves it open after the author's reply in #7. local_preload_libraries is restricted for every role, so the existing practice the patch copies does not cover role-dependent resolution in LOAD.

### 2. No user demand shown for dropping the $libdir/plugins/ prefix

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A deployment where the same library name exists in $libdir and $libdir/plugins sees LOAD 'name' pick a different file per role, as message #6 shows. Admins learn this when a script that worked as superuser loads another module under a normal role. The users who gain only skip typing a fixed prefix.

The only stated need is the author's view that the prefix is inconvenient (message #3), and Japin Li called it "a bit inconvenient" (message #4). No field report or -general request is cited, and no non-superuser workflow is shown that the prefix blocks. The workaround is to write '$libdir/plugins/name', which works today and is documented. The cost of role-dependent resolution and a changed error falls on admins and scripts that never asked for the shortcut.

Thread: [#3 (cca5507, 2026-09-03)](https://www.postgresql.org/message-id/tencent_A25AB53229C6323A784407EA76DAA1AB8508%40qq.com), [#4 (Japin Li, 2026-09-03)](https://www.postgresql.org/message-id/SY7PR01MB10921D6FCB0C8E7BBBF111DD8B6B62%40SY7PR01MB10921.ausprd01.prod.outlook.com), [#6 (Jim Jones, 2026-09-09)](https://www.postgresql.org/message-id/b08c5b95-f36d-412b-a936-7e710df6d24e%40uni-muenster.de), [#8 (Jim Jones, 2026-09-09)](https://www.postgresql.org/message-id/fbf5df8b-2c09-4153-bcad-c7682686d632%40uni-muenster.de)

**Question:** Which users or tools run non-superuser LOAD of plugins today, and where has the explicit prefix caused them real trouble?

Counter-check: held. The thread gives convenience as the only motive (messages #3 and #4), and the explicit prefix does the job. The cost is small and falls mostly on users of bare names, so this is a show-demand point and no reason to reject.

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
| security / privileges | **yes** | The patch changes the path check in load_file() that limits non-superuser LOAD to `$libdir/plugins`. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Role-dependent resolution of a bare LOAD name becomes documented behavior that later changes must keep. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | `LOAD '$libdir/plugins/name'` already gives non-superusers the same result today. |

## 5. Questions the author must answer before anyone reviews code

1. Why not resolve bare names the same way for all roles, or raise an error when the name exists in both `$libdir` and `$libdir/plugins`?
2. Which users or tools run non-superuser LOAD of plugins today, and where has the explicit prefix caused them real trouble?
3. Does the new "could not access file" error for a missing plugin carry a different SQLSTATE than the old "access to library ... is not allowed" error?
4. Does moving the expansion into load_file() change superuser LOAD or session_preload_libraries in any way?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the dynamic loader code in dfmgr.c and the preload library settings would have to own it. No committer is identified in the visible thread.
- **Liability:** The owner signs for LOAD semantics that depend on the role and for a changed error on a missing plugin. Both are user-visible and hard to change once scripts use bare names.
- **Author follow-up:** The author answered review comments on the same day in message #3 and again in message #7. Nothing is said about fixes after commit.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 3.8 KB across load.sgml, dfmgr.c and miscinit.c, and it helps only sites where non-superusers LOAD plugins. The role-dependent resolution from message #6 needs an agreed answer before code review is useful.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Role-dependent resolution of LOAD becomes a permanent contract. Refuted (not specific): Duplicate of standards-compat-1. The added claim that the project could never later make the ambiguous case an error has no support in the material.
- `maintenance-fit-2` Demand rests on one convenience report. Refuted (not specific): Duplicate of demand-1. This version also says few sites allow non-superuser LOAD and that the rule is documented "on purpose", and the thread supports neither statement.
- `already-exists-1` Explicit $libdir/plugins/ prefix already does the job. Refuted (not specific): Duplicate of demand-1, which already states that the explicit prefix works today and asks the same question. Its failure scenario repeats standards-compat-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus search returned no earlier attempt at this idea. Message #4 points to the commit that introduced $libdir/plugins, but it does not say that a bare-name LOAD was proposed or rejected then. Without a recorded earlier rejection, this angle has no reason to apply. The role-dependent resolution concern in message #6 belongs to the design angle.
- standards-compat: 1 objection(s). The error-message and SQLSTATE change was raised in message #6 and accepted with documentation in message #8, so it is not raised here. The proposal does not touch pg_dump, psql, GUCs or SQL syntax.
- replication-ondisk: 0 objection(s). The proposal changes only how a non-superuser LOAD resolves a bare file name at run time. It adds no on-disk, WAL, catalog, wire protocol or replication state, and the material does not show a changed load_file() signature that would affect extensions. The role-dependent resolution and the changed error message belong to other angles.
- security: 0 objection(s). The patch gives non-superusers no new privilege. A bare name expands to $libdir/plugins/name, which they could already LOAD with the explicit prefix, and message #5 reports that /tmp, $libdir and ../ paths are still rejected. The new "could not access file" error leaks nothing new, because the explicit-prefix form already gave the same answer. The role-dependent resolution in message #6 is a semantics issue, not an escalation: superusers can load either file anyway. Without the code, two points could not be checked: how a "bare" name is detected, including Windows separators, and whether dynamic_library_path is ever consulted.
- maintenance-fit: 2 objection(s). No hot-path, ABI or external-dependency concern is visible from the prose. The change is small and touches only dfmgr.c and miscinit.c.
- already-exists: 1 objection(s). One objection from this angle: the explicit prefix already gives non-superusers the same result, and no extension or other mechanism is needed.
- demand: 1 objection(s). One demand objection. The rest of the thread concerns semantics and error codes, which other angles cover.

### Precedents retrieved

- P1: [document an example to set multiple libraries in shared_preload_libraries](https://commitfest.postgresql.org/patch/3437/) (cf, Withdrawn, 2022-01-10) - judged not relevant; reason summarised from its thread
- P2: [warn if GUC set to an invalid shared library](https://commitfest.postgresql.org/patch/3482/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P3: [Mingw: Fix import library extension, build actual static libraries](https://commitfest.postgresql.org/patch/2082/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P4: [pg_regress.c: Fix "make check" on Mac OS X: Pass DYLD_LIBRARY_PATH](https://commitfest.postgresql.org/patch/4340/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P5: [Revert "Tidy up locale thread safety in ECPG library."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3c8e463b0d885e0d976f6a13a1fb78187b25c86f) (revert, Reverted, 2025-03-28) - judged not relevant
- P6: [function pg_setting_value_split() to parse shared_preload_libraries etc.](https://commitfest.postgresql.org/patch/2782/) (cf, Withdrawn, 2020-11-29) - judged not relevant; reason summarised from its thread
- P7: [libpq: Support for Apple Secure Transport SSL library](https://commitfest.postgresql.org/patch/1228/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P8: [pglogical_output - a general purpose logical decoding output plugin](https://commitfest.postgresql.org/patch/418/) (cf, Returned with feedback, 2016-02-03) - judged not relevant; reason summarised from its thread
- P9: [pg_stat_statements: improve loading and saving routines for the dump file.](https://commitfest.postgresql.org/patch/5512/) (cf, Rejected, 2025-04-04) - judged not relevant; reason summarised from its thread
- P10: [system load consideration hook for parallel workers](https://commitfest.postgresql.org/patch/725/) (cf, Rejected, 2016-09-22) - judged not relevant
- P11: [Revert "Replace pg_restrict by standard restrict"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fa16e7fd84886643f480c36614fa11d45fc6d26f) (revert, Reverted, 2026-01-14) - judged not relevant
- P12: [Delegating superuser tasks to new security roles](https://commitfest.postgresql.org/patch/3223/) (cf, Withdrawn, 2021-11-15) - judged not relevant
- P13: [Fix search_path for maintenance commands](https://commitfest.postgresql.org/patch/4367/) (cf, Withdrawn, 2024-01-18) - judged not relevant
- P14: [Add document is_superuser](https://commitfest.postgresql.org/patch/3889/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P15: [Add "\gi file" to psql to allow COPY with variable expansion](https://commitfest.postgresql.org/patch/5674/) (cf, Returned with feedback, 2025-06-17) - judged not relevant
- P16: [Add table expansion hook](https://commitfest.postgresql.org/patch/2614/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P17: [Restrict pg_rewind to specified/needed directories](https://commitfest.postgresql.org/patch/1343/) (cf, Returned with feedback, 2018-02-01) - judged not relevant
- P18: [Recognizing superuser in pg_hba.conf](https://commitfest.postgresql.org/patch/2397/) (cf, Returned with feedback, 2020-07-03) - judged not relevant
- P19: [Orphaned users in PG16 and above can only be managed by Superusers](https://commitfest.postgresql.org/patch/5608/) (cf, Withdrawn, 2026-03-05) - judged not relevant
- P20: [Revert refactoring of restore command code to shell_restore.c](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f6e15ac93c58c1140e4a4affe61e78f7346497a) (revert, Reverted, 2023-02-05) - judged not relevant
- P21: [Restricting maximum keep segments by repslots](https://commitfest.postgresql.org/patch/1260/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P22: [Allow placeholders in ALTER ROLE w/o superuser](https://commitfest.postgresql.org/patch/3749/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P23: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant
- P24: [Revert addition of -lpgport, which breaks the build on platforms that are sticky about non-PIC code in shared libraries.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9bfee3adeeb5eb08bdfe192371a4db67fb6a216f) (revert, Reverted, 2004-05-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 144 s; cost reported by the CLI: $1.29
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 70 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:32:53.446Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
