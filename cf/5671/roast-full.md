# Design pre-review: Produce LLVM bitcode files on meson builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5671](https://commitfest.postgresql.org/patch/5671/) (Needs review); authors: Andres Freund (andresfreund), Diego Fronza (dfronza), Nazir Bilal Yavuz (nbyavuz)
- Thread: [meson vs. llvm bitcode files](https://www.postgresql.org/message-id/flat/206b001d-1884-4081-bd02-bed5c92f02ba%40eisentraut.org), started 2024-09-05
- Messages read: 32 of 32
- Patch prose read: v1-0001-meson-Add-generated-header-stamps.patch, v1-0002-meson-Add-postgresql-extension.pc-for-building-ex.patch, v1-0003-meson-Test-building-extensions-by-using-postgresq.patch, v1-0004-meson-WIP-Add-docs-for-postgresql-extension.pc.patch, v1-0005-meson-Add-LLVM-bitcode-emission.patch, v1-0006-meson-Generate-bitcode-files-of-contrib-extension.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is against the core bitcode patches. They restore what the autoconf build already does, and messages #1, #25 and #27 show demand. All ranked objections concern the general extension .pc file in the last patch: no extension project is shown to need it (demand-1), its name and Requires list are not agreed (standards-compat-2, standards-compat-3), and autoconf builds do not install it (standards-compat-1). That patch is last in the series, so the rest can go in without it.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check that the meson bitcode flags match the makefile build, what the irlink script does, or what the .pc file holds with -Dllvm=disabled. Only 3 of 32 messages were read directly.

## The proposal, restated

**Problem.** The meson build does not produce LLVM bitcode (.bc) files, so JIT inlining of server and contrib functions does not work with meson builds. Peter Eisentraut called this the last major regression for using meson in production. Christoph Berg says it blocked meson for package builds on apt.postgresql.org.

**Proposed design.** Meson compiles selected server sources and five contrib modules (hstore, citext, intarray, ltree, pg_trgm) to bitcode and installs them in $pkglibdir/bitcode/ with ThinLTO indexes. A generated pkg-config file supplies the include paths and compiler flags. A last patch renames it to a general extension build interface, and another excludes Windows resource files from bitcode source lists.

**User-visible surface.**
- New installed files: .bc bitcode files and indexes under $pkglibdir/bitcode/ in meson builds with LLVM.
- New pkg-config file postgresql-$version-llvm-jit-bitcode.pc, renamed in the last patch to postgresql-$version-extension.pc.
- pkg-config variables such as dir_mod, dir_data, dir_include, dir_bitcode, pkglibdir and bindir.
- New documentation section on building extensions with the .pc file, with an example meson.build.
- New build helper script src/tools/irlink.

**What the author claims.**
- Bitcode loading and JIT inlining work for backend and contrib modules, per Diego Fronza's tests with intarray.
- Bitcode only helps contrib modules with lightweight SQL operators or functions, so the patch limits the contrib list to five modules.
- Removing an explicit dependency and using extracted objects as input avoids a 4x slowdown of build.ninja generation, about 22s versus 5.1s.
- The .pc file can build extensions with other build systems as well as meson.
- Christoph Berg reports v15 works in Debian postgresql-20 snapshot packages for months.

**Thread state.** Patch is at v17, with review from Peter Eisentraut, Andres Freund, Zsolt Parragi, Christoph Berg and Harrison Booth. Christoph says the core bitcode part could be committed, while naming and scope of the general extension .pc file stay open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-09-05; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. General extension .pc file has no demonstrated users yet

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

Postgres 20 installs postgresql-20-extension.pc as a public build interface, and extension authors start to depend on its variables and Requires list. A later release changes the Requires list or the compiler flags still disputed in message #19 and #20. Their builds break, and they find out on upgrade.

Demand for the bitcode part is shown in message #1, #25 and #27, but nobody asks for a general extension interface. Christoph proposed the rename in message #25 and says in message #27 that there are no extension builds yet. Andres said in message #21 that there is not enough agreement for a name this general. Committing patch 0006 makes it a public interface that must be kept stable, with no extension author shown to need it.

Thread: [#19 (Peter Eisentraut, 2026-03-31)](https://www.postgresql.org/message-id/c6b26dee-bd62-44d9-b4f3-730b6ecf95c6%40eisentraut.org), [#20 (Andres Freund, 2026-04-01)](https://www.postgresql.org/message-id/3whtmw6twe442o3r44gsagheq5cs2clcghzjeo7rljstpdemrh%40oojczkhk2mbt), [#21 (Andres Freund, 2026-04-06)](https://www.postgresql.org/message-id/blfq7kk5bylebetudvyslef5j2bvigrqikmwy66i4znvkv63uy%40smypdulfe7gb), [#25 (Christoph Berg, 2026-07-15)](https://www.postgresql.org/message-id/ald4zLd-nes2CjuZ%40msg.df7cb.de), [#27 (Christoph Berg, 2026-09-18)](https://www.postgresql.org/message-id/aqzmJ7-5202qXuzx%40msg.df7cb.de)

**Question:** Which extension projects outside core have built against postgresql-$version-extension.pc, and what did they need from it that PGXS or pg_config does not give them?

Counter-check: held. Christoph says in message #27 that there are no extension builds yet. Peter's remark in message #19 that extensions should switch eventually shows intent, but the thread shows no extension project asking for the file.

### 2. Installed .pc name and variables become a contract before agreement

fixable · level 2, rubric item 6 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

Version 20 ships postgresql-20-extension.pc with dir_mod, dir_bitcode and the full Requires list, and extensions start to depend on it. A later release changes the name, a variable or the Requires semantics. Extension builds then fail against the new server until every extension's build file is changed.

In message #21 Andres said there is not enough agreement yet for a name this general. In message #13 Peter asked whether "extension" is the right term. The disagreement between Peter (message #19) and Andres (message #20) on whether Requires should list every server dependency is still open, and Christoph in message #27 has no extension builds to report. The .pc file is installed even with -Dllvm=disabled (message #24), so a meson build with LLVM disabled also publishes the interface.

Thread: [#13 (Peter Eisentraut, 2026-03-12)](https://www.postgresql.org/message-id/4286824f-40c3-4716-ad71-2085b83f3736%40eisentraut.org), [#19 (Peter Eisentraut, 2026-03-31)](https://www.postgresql.org/message-id/c6b26dee-bd62-44d9-b4f3-730b6ecf95c6%40eisentraut.org), [#20 (Andres Freund, 2026-04-01)](https://www.postgresql.org/message-id/3whtmw6twe442o3r44gsagheq5cs2clcghzjeo7rljstpdemrh%40oojczkhk2mbt), [#21 (Andres Freund, 2026-04-06)](https://www.postgresql.org/message-id/blfq7kk5bylebetudvyslef5j2bvigrqikmwy66i4znvkv63uy%40smypdulfe7gb), [#24 (Nazir Bilal Yavuz, 2026-07-06)](https://www.postgresql.org/message-id/CAN55FZ2V%2BX%3DWgyr2U0C-%3DhH1RWELmuQcqV523o0R1Yi22K4FCg%40mail.gmail.com), [#27 (Christoph Berg, 2026-09-18)](https://www.postgresql.org/message-id/aqzmJ7-5202qXuzx%40msg.df7cb.de)

**Question:** Can the core bitcode patches be committed under the internal llvm-jit-bitcode name while the general extension .pc file waits for agreed names, variables and Requires semantics?

Counter-check: held. Andres's concern in message #21 about a general name was never withdrawn, and the Requires disagreement in messages #19 and #20 is open. Patch 0006 is last in the series, so the bitcode patches can go in without it.

### 3. General extension .pc file exists only in meson builds

fixable · level 2, rubric item 12 · pg_dump / restore round-trip, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `standards-compat-1`

An extension author follows the new docs and writes a meson.build that calls dependency('postgresql-20-extension'). A user or packager whose server was built with autoconf then cannot build that extension, because the .pc file is not installed. The author finds out from bug reports, platform by platform.

In message #13 Peter said that installed .pc files must also be built by the makefiles, or else not installed. The autoconf patch was dropped after Andres in message #21 proposed a narrow llvm-jit-bitcode name, so it would not be needed. The last patch (v17-0006) brings back the general postgresql-$version-extension.pc name and documents it as an extension build interface, but the autoconf part is not back. A user-facing interface that depends on how the server was built departs from PGXS and pg_config, which both build systems provide.

Thread: [#13 (Peter Eisentraut, 2026-03-12)](https://www.postgresql.org/message-id/4286824f-40c3-4716-ad71-2085b83f3736%40eisentraut.org), [#21 (Andres Freund, 2026-04-06)](https://www.postgresql.org/message-id/blfq7kk5bylebetudvyslef5j2bvigrqikmwy66i4znvkv63uy%40smypdulfe7gb), [#22 (Nazir Bilal Yavuz, 2026-07-01)](https://www.postgresql.org/message-id/CAN55FZ0Y0J29rUqjpX8BMU7HwSgeSz2RxcJm%2B4PLhCeGjtWmhg%40mail.gmail.com), [#25 (Christoph Berg, 2026-07-15)](https://www.postgresql.org/message-id/ald4zLd-nes2CjuZ%40msg.df7cb.de), [#26 (Nazir Bilal Yavuz, 2026-07-17)](https://www.postgresql.org/message-id/CAN55FZ26ohMftt20NUtVAxU8Feqjhz-jEfUvFKATYEaRR%2Bxmqg%40mail.gmail.com), [#30 (Nazir Bilal Yavuz, 2026-11-02)](https://www.postgresql.org/message-id/CAN55FZ3MWpX%2B6FddixNW7NEoYNnHVktsDzsbV060EPO3gxzsnw%40mail.gmail.com)

**Question:** Will autoconf builds install the same postgresql-$version-extension.pc file, or will the general rename wait until they do?

Counter-check: held. Message #22 removed the autoconf patch, and v17-0006 brings back the general name without it. Peter's condition in message #13 is unanswered for the renamed file.

### 4. Requires list makes extension builds need every server -dev package

fixable · level 2, rubric item 10 · design, ABI / extension API · `standards-compat-3`

A packager builds a small extension against a distribution server package through the .pc file. pkg-config --cflags fails unless krb5, ldap, icu, libxml2, lz4, openssl, zlib and zstd development packages are all installed. Today the same extension builds with PGXS without most of these packages.

Peter in message #19 calls this a regression for extensions moving to the .pc file. Andres answers in message #20 that server headers include some of these dependencies, and that static linking needs them. That holds for headers such as libpq-be.h, but the proposal does not show which server headers need ldap, lz4 or zstd. It also does not say why Requires.private would not cover the static-linking case. The thread leaves this open.

Thread: [#13 (Peter Eisentraut, 2026-03-12)](https://www.postgresql.org/message-id/4286824f-40c3-4716-ad71-2085b83f3736%40eisentraut.org), [#14 (Nazir Bilal Yavuz, 2026-03-18)](https://www.postgresql.org/message-id/CAN55FZ0KxMphg%2BM%2B4Shm%3DHmNV%2BXhC7LUKc6hrC%2B-QiovhzvdVA%40mail.gmail.com), [#19 (Peter Eisentraut, 2026-03-31)](https://www.postgresql.org/message-id/c6b26dee-bd62-44d9-b4f3-730b6ecf95c6%40eisentraut.org), [#20 (Andres Freund, 2026-04-01)](https://www.postgresql.org/message-id/3whtmw6twe442o3r44gsagheq5cs2clcghzjeo7rljstpdemrh%40oojczkhk2mbt)

**Question:** Which listed dependencies do installed server headers actually include, and why would Requires.private not cover the rest?

Counter-check: held. Andres's reply in message #20 covers headers such as libpq-be.h, but it does not show that every listed package is needed. Peter did not accept it, and the thread has no later answer.

### 5. Nothing tests that meson-built bitcode is still used for inlining

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A later patch changes include paths, compiler flags or a contrib source layout. The .bc files still build but no longer match, or are no longer installed, and JIT inlining quietly falls back. Packagers learn of it only through slower queries, if they notice.

In messages #4 and #26, inlining was confirmed only by adding debug logging to llvmjit_inline.cpp, and message #25 says it is hard to tell whether the .bc files are used. Meson now keeps its own bitcode flags and a hand-picked contrib list, separate from the makefile path, so each build-system change has a second place to keep right. Full server JIT on Windows is untested (message #28).

Thread: [#4 (Diego Fronza, 2025-03-10)](https://www.postgresql.org/message-id/CA%2BhiPX8R69OeApj55Fy1VXOhuHUKfdvp2f80icLy65z76Jy%3DJA%40mail.gmail.com), [#25 (Christoph Berg, 2026-07-15)](https://www.postgresql.org/message-id/ald4zLd-nes2CjuZ%40msg.df7cb.de), [#26 (Nazir Bilal Yavuz, 2026-07-17)](https://www.postgresql.org/message-id/CAN55FZ26ohMftt20NUtVAxU8Feqjhz-jEfUvFKATYEaRR%2Bxmqg%40mail.gmail.com), [#28 (harrisontbooth(at)gmail(dot)com, 2026-10-13)](https://www.postgresql.org/message-id/CAAH-eSX_OniTafzLjiis%2BLsgnicS5JSuiJriQw_B2wEHM_q4mA%40mail.gmail.com)

**Question:** Which automated test, run in CI, fails when a meson build installs bitcode that the server cannot load or inline?

Counter-check: held. Messages #4, #25 and #26 confirm that inlining was checked only by hand-added debug logging. The makefile build may have the same gap, but the meson path adds a second flag set and contrib list that can drift silently.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The last patch installs postgresql-$version-extension.pc and documents it as an extension build interface. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | see objection standards-compat-1: General extension .pc file exists only in meson builds |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Meson gets its own bitcode flags, a hand-picked contrib list, the irlink script and an installed .pc file to keep stable. |
| performance on the hot path | **yes** | Meson builds start to inline server and contrib functions in JIT, and the thread shows no query-time measurement. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The general extension .pc file overlaps with pg_config and PGXS, which both build systems already provide. |

## 5. Questions the author must answer before anyone reviews code

1. Can the core bitcode patches be committed under the llvm-jit-bitcode name while the general extension .pc rename waits for agreement?
2. Which extension projects outside core need the installed .pc file, and what does it give them that pg_config or PGXS does not?
3. Will autoconf builds install the same postgresql-$version-extension.pc file, or will the rename wait until they do?
4. Which dependencies in the Requires list do installed server headers include, and why would Requires.private not cover the rest?
5. Which automated CI test fails when a meson build installs bitcode that the server cannot load or inline?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund co-authored the series and reviewed it in the thread. Peter Eisentraut reviewed it but removed himself as reviewer in April 2026.
- **Liability:** The owner signs for a second set of bitcode flags and a contrib list that must follow the makefile build. If the last patch goes in, the .pc name, variables and Requires list must stay stable for every later release.
- **Author follow-up:** The series reached v17 and the CommitFest history shows rebases through October 2026, so the authors are still active.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Packagers who build with meson and LLVM need the bitcode files, and Christoph Berg reports v15 working in Debian snapshot packages for months. The patch touches about 40 meson.build files but no server code. Review cost is an estimate because no patch code was read.

## Appendix

### Held up in the counter-check, not listed above

- `already-exists-1` (fixable, level 1b) General extension .pc file duplicates pg_config and PGXS. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `maintenance-fit-1` Last patch commits to a public extension build interface without agreement. Refuted (not specific): Duplicate of standards-compat-2, which states the same contract risk and the same request to hold back the rename. This one also says Andres dropped the tests and autoconf parts, but Nazir removed them in message #22.

### Review angles

- prior-attempts: 0 objection(s). No precedent found in the list. The thread names one earlier attempt, a 2022 meson bitcode patch by Andres Freund (message #2), and this series continues that work rather than reviving a rejected idea.
- standards-compat: 3 objection(s). The core bitcode emission restores autoconf behaviour and raises no compatibility issue from this angle. The objections concern only the general extension .pc file.
- replication-ondisk: 0 objection(s). The proposal changes only the meson build and installed build artifacts (.bc files, ThinLTO indexes, a .pc file). It adds no on-disk cluster format, WAL record, catalog change, wire protocol message or server ABI change. The bitcode is not persistent cluster state, and the autoconf build already installs the same kind of bitcode files, so this angle has nothing to object to.
- security: 0 objection(s). The proposal changes only the meson build and its install layout, so it has no privilege or exposure aspect. The .bc files go under $pkglibdir/bitcode/ with the same ownership as the shared libraries, and the server loads them the same way as in autoconf builds today. The .pc file holds only build paths and flags. The proposal gives a non-superuser no new abilities and no new visibility.
- maintenance-fit: 2 objection(s). The core bitcode emission fixes a stated meson regression for packagers and adds no hot-path cost. Andres, a committer who knows this area, co-authored the patch and reviewed it.
- already-exists: 1 objection(s). The bitcode part restores what the autoconf build already does, so it adds no second mechanism. The only overlap is the general extension .pc file in the last patch.
- demand: 1 objection(s). Demand for the core bitcode part is real: Peter calls it the last major meson regression, and Christoph uses it in apt.postgresql.org packages. Only the general extension .pc file in the last patch lacks shown demand.

### Precedents retrieved

- P1: [Meson build updates](https://commitfest.postgresql.org/patch/4410/) (cf, Withdrawn, 2023-09-01) - judged not relevant; reason summarised from its thread
- P2: [Improvements to Meson docs](https://commitfest.postgresql.org/patch/4208/) (cf, Returned with feedback, 2024-01-20) - judged not relevant; reason summarised from its thread
- P3: [Speed up clean meson builds by ~25%](https://commitfest.postgresql.org/patch/4914/) (cf, Rejected, 2024-05-20) - judged not relevant; reason summarised from its thread
- P4: [Invalid postgres version detection in meson.build](https://commitfest.postgresql.org/patch/5177/) (cf, Rejected, 2024-08-20) - judged not relevant; reason summarised from its thread
- P5: [Allow meson to find ICU in non-standard location](https://commitfest.postgresql.org/patch/4203/) (cf, Withdrawn, 2023-03-08) - judged not relevant; reason summarised from its thread
- P6: [Enable jitlink as an alternative jit linker of legacy Rtdyld and add riscv jitting support](https://commitfest.postgresql.org/patch/3857/) (cf, Returned with feedback, 2023-09-01) - judged not relevant; reason summarised from its thread
- P7: [Revert "Add USER SET parameter values for pg_db_role_setting"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b9a7a822723aebb16cbe7e5fb874e5124745b07e) (revert, Reverted, 2023-05-17) - judged not relevant
- P8: [JIT compiling expressions/tuple deforming](https://commitfest.postgresql.org/patch/1285/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P9: [Lazy JIT IR code generation to increase JIT speed with partitions](https://commitfest.postgresql.org/patch/3071/) (cf, Withdrawn, 2023-01-27) - judged not relevant; reason summarised from its thread
- P10: [Configurable location for extension .control files](https://commitfest.postgresql.org/patch/170/) (cf, Returned with feedback, 2015-07-30) - judged not relevant
- P11: [Sample configuration files](https://commitfest.postgresql.org/patch/745/) (cf, Returned with feedback, 2016-09-30) - judged not relevant
- P12: [Add inline comments to the pg_hba_file_rules view](https://commitfest.postgresql.org/patch/4546/) (cf, Withdrawn, 2023-10-10) - judged not relevant
- P13: [SQL Inlining Behavior](https://commitfest.postgresql.org/patch/1965/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P14: [Revert "Convert *GetDatum() and DatumGet*() macros to inline functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e8d78581bb864369035c736a82ecfb57482a1a09) (revert, Reverted, 2022-09-12) - judged not relevant
- P15: [Configurable file mode mask](https://commitfest.postgresql.org/patch/1069/) (cf, Returned with feedback, 2017-03-21) - judged not relevant
- P16: [Revert inadvertant change in pg_config behavior](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fc8a81e3e73e30ac083cc60837ecf98b33d5cc7e) (revert, Reverted, 2016-02-17) - judged not relevant
- P17: [jit_warn_above_fraction parameter](https://commitfest.postgresql.org/patch/3573/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [Revert "For all ppc compilers, implement pg_atomic_fetch_add_ with inline asm."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e9fae0696d9e3ff70a1438775ad9f786b854a5) (revert, Reverted, 2019-09-15) - judged not relevant
- P19: [snapshot too old, configured by time](https://commitfest.postgresql.org/patch/348/) (cf, Returned with feedback, 2016-02-01) - judged not relevant
- P20: [Add ssl_protocols configuration option](https://commitfest.postgresql.org/patch/11/) (cf, Rejected, 2015-02-13) - judged not relevant
- P21: [Fix PGXS so built control files work with VPATH builds](https://commitfest.postgresql.org/patch/2461/) (cf, Returned with feedback, 2020-07-31) - judged not relevant
- P22: [Mingw: Fix import library extension, build actual static libraries](https://commitfest.postgresql.org/patch/2082/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P23: [Revert "Make configure prefer python3 to plain python."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=512fc2dd790e3e7c35b3af8cd7586aeb25248c6e) (revert, Reverted, 2022-01-20) - judged not relevant
- P24: [Revert all new recent changes to add PGXS options for TAP and isolation](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d7dd18686d0373289a3bf2b3669edae20875cbc) (revert, Reverted, 2018-11-26) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 20; wall time: 153 s; cost reported by the CLI: $3.68
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 8 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 84 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:35:21.050Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
