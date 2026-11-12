# Design pre-review: meson: Fix libpq search order

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6056](https://commitfest.postgresql.org/patch/6056/) (Needs review); authors: Thomas Munro (macdice)
- Thread: [meson's in-tree libpq header search order vs -Dextra_include_dirs](https://www.postgresql.org/message-id/flat/CA%2BhUKGKispvxLyrBn3%3D3mp0BB1N%2BRBYR5eE2guCOksnwEoOcPQ%40mail.gmail.com), started 2025-02-03
- Messages read: 16 of 16
- Patch prose read: v4-0001-meson-Fix-libpq-header-include-order.patch, v4-0002-ci-Test-include-path-order-with-canary-libpq-fe.h.patch, v4-0003-meson-Try-to-find-libintl-without-Dextra_XXX.patch, v4-0004-ci-Remove-Dextra_XXX-on-FreeBSD-and-macOS.patch, wip-reorder-include.diff (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at level 1a or 1b: the build failure is real and patch 1 makes meson match configure. Three fixable level 2 points remain. maintenance-fit-1 shows the CI canary does not cover include paths from pkg-config dependencies. maintenance-fit-2 and standards-compat-2 concern the optional libintl guess in patch 3 and the unstated back-patch scope, with the choice between v4 and the cppflags change still open.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so this review cannot confirm that every target that includes libpq-fe.h gets libpq first, or what patch 3 probes. Only 3 of 16 messages were visible; later messages are known from the brief and the objections.

## The proposal, restated

**Problem.** With meson and -Dextra_include_dirs, an installed older libpq-fe.h is found before the in-tree header, so the build fails (for example, unknown type PGcancelConn). configure/make puts the in-tree libpq directory first, so it is not affected. The author says this hits developers on macOS, the BSDs and some Linux distributions that install optional packages outside the default search paths.

**Proposed design.** Patch 1 declares libpq as the first dependency in each meson.build that uses it, so libpq_inc comes before extra_include_dirs, c_flags and directories Meson finds. Patch 2 adds a CI check that puts a canary libpq-fe.h with #error into the include path, for both meson and configure. Patches 3 and 4 find libintl from the msgfmt prefix and drop -Dextra_XXX from the FreeBSD and macOS CI tasks.

**User-visible surface.**
- Changed meaning of meson option -Dextra_include_dirs: its directories now come after the in-tree libpq headers.
- New libintl discovery in meson (patch 3) that guesses the prefix from the location of msgfmt in $PATH.

**What the author claims.**
- The fix is mostly a mechanical reorder of dependency lists and makes meson match the configure build.
- Any non-Linux system with an installed libpq-with-headers package could hit the failure; the author may be the only one affected so far.
- The fix should be back-patched to 16, where meson arrived.
- With patch 3, FreeBSD and macOS with MacPorts build without -Dextra_XXX, and all CI operating systems pass.
- Removing the catch-all include path exposed missing dependency declarations, such as gssapi for libpq_oauth_st.

**Thread state.** Heikki does not object but fears the order rule will break again; Tristan calls patch 1 ready and doubts patch 3. Andres proposed moving extra_include_dirs into cppflags, and Thomas answered that paths from other dependencies can still expose an external libpq-fe.h, so the approach is still open.

## 2. Prior attempts

### Allow meson to find ICU in non-standard location (related)

- **Link:** [https://commitfest.postgresql.org/patch/4203/](https://commitfest.postgresql.org/patch/4203/)
- **Thread:** [https://www.postgresql.org/message-id/flat/10930b0dd06d7fbba2b67c07d39bc7ace08d5e42.camel@j-davis.com](https://www.postgresql.org/message-id/flat/10930b0dd06d7fbba2b67c07d39bc7ace08d5e42.camel@j-davis.com)
- **Date:** decided 2023-03-08; started 2023-02-22
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew it because setting -Dextra_lib_dirs, as Jacob Champion suggested, already worked with ICU 72 without the patch. Andres Freund had also pointed to -Dpkg_config_path, which finds ICU but did not add the RPATH in the author's test. The author said a clearer solution or doc updates would still help others.
- **Reason (quote):** "Obviously we'd like a little better solution so that others don't get confused, but it's not really a problem for me any more." ([message](https://www.postgresql.org/message-id/875e58307282b963b4e8cc4671bad8a8473caca2.camel%40j-davis.com))
- **What is different this time:** P5 changed how meson finds ICU when -Dextra_* options are given, and its author withdrew it because his own case already worked. Patch 3 here guesses the libintl prefix from msgfmt so that no -Dextra_* option is needed; patches 1 and 2 fix header order, which P5 did not touch.

### Revert change of inclusion order for pg_config_os.h; (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5e04f3857c9369351edbf8f906d1d8c2665c8a1f](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5e04f3857c9369351edbf8f906d1d8c2665c8a1f)
- **Date:** decided 2003-05-15; started 2003-05-15
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Revert change of inclusion order for pg_config_os.h; this broke plperl on Linux and would have who knows what unpleasant effects on other platforms. If you need another include file for Windows, then add it; don't go messing with the semantics of every other port's include files.
- **Reason (quote):** "If you need another include file for Windows, then add it; don't go messing with the semantics of every other port's include files."
- **What is different this time:** P6 reverted a global change to the inclusion order of pg_config_os.h. Patch 1 only places the in-tree libpq directory first, as configure does.


## 3. Objections a reviewer is likely to raise

### 1. Order convention in about 30 meson.build files has only partial CI coverage

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a frontend tool with `dependencies: [libxml, libpq]`. On FreeBSD, pkg-config for libxml adds -I/usr/local/include, where a packaged older libpq-fe.h lives. CI stays green because the canary sits only in -Dextra_include_dirs, and a developer finds out when their own build fails.

Patch 1 makes correctness depend on list order in about 30 build files, which message #9 calls a "flimsy list that is copied all over the place". Heikki's concern in message #6 is met by patch 2, but only for paths from extra_include_dirs and --with-includes. Message #16 says paths from other dependencies can expose an external libpq-fe.h again. The canary does not cover that source.

Thread: [#6 (Heikki Linnakangas, 2025-10-31)](https://www.postgresql.org/message-id/b3796a7b-8f78-4a2d-ba74-24e00f2ca71e%40iki.fi), [#9 (Thomas Munro, 2025-11-03)](https://www.postgresql.org/message-id/CA%2BhUKG%2B%3DMPY7MDh3fv8DjbGNBTO9zUO7v_RZDs29M5fLRJD-uw%40mail.gmail.com), [#16 (Thomas Munro, 2025-11-05)](https://www.postgresql.org/message-id/CA%2BhUKGLWjwa3mTP1nEySjmGwFnHXdTE79OymmQdf-1yC8%2B6nDQ%40mail.gmail.com)

**Question:** Can the canary check also cover include paths from pkg-config dependencies, or can the ordering be set in one central place so new meson.build files cannot get it wrong?

Counter-check: held. Message #16 itself says paths from other dependencies can expose an external libpq-fe.h. The canary in patch 2 sits only under --with-includes and -Dextra_include_dirs, so a wrong order next to a pkg-config dependency passes CI.

### 2. Guessing the libintl prefix from msgfmt is a heuristic core must keep

fixable · level 2, rubric item 6 · design, maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-2`

A macOS user has Homebrew and MacPorts, and the msgfmt first in $PATH comes from the other prefix or from a multilib layout using lib64. Meson then takes libintl headers or libraries from a prefix the user did not choose. The user may not see the mismatch until link time or run time.

Patch 3 itself asks whether there is a reason not to do this. Tristan objects to the hard-coded "lib" directory in message #14, and the thread does not answer him. Once released, users will depend on this discovery rule, and every later packaging layout becomes a report against core. Patch 3 is not needed for the libpq fix, and the material does not say whether it would be back-patched.

Thread: [#12 (Thomas Munro, 2025-11-04)](https://www.postgresql.org/message-id/CA%2BhUKGJk85%2BsDf4D%3DJSwuvA9a4-sy6fnM3B%3DCVaU9Rk2V4NE2g%40mail.gmail.com), [#14 ("Tristan Partin", 2025-11-04)](https://www.postgresql.org/message-id/DDZP25P4VZ48.3LWMZBGA1K9RH%40partin.io)

**Question:** Can patches 3 and 4 move to a separate master-only thread, and what does the probe do when msgfmt and libintl come from different prefixes or use lib64?

Counter-check: held. The author's reply in message #12 covers a guess that finds nothing, but not a guess that finds a libintl in a prefix the user did not choose. Tristan's point about the hardcoded "lib" has no answer in the thread.

### 3. Back-patch scope does not say which patches go to stable branches

fixable · level 2, rubric item 6 · backpatchability · `standards-compat-2`

A distribution rebuilds a 16.x minor release with its existing meson options. If patch 3 is back-patched, which libintl Meson finds can change within the minor release. A Windows packager may also see a different header order if Andres's cppflags approach replaces v4.

Message #9 and the v4-0001 commit message plan a back-patch to 16. Only patch 1 is a plain reorder; patch 3 changes how a dependency is found. Message #16 leaves open whether v4 or the cppflags change will be committed, and that choice decides what packagers of stable branches see.

Thread: [#9 (Thomas Munro, 2025-11-03)](https://www.postgresql.org/message-id/CA%2BhUKG%2B%3DMPY7MDh3fv8DjbGNBTO9zUO7v_RZDs29M5fLRJD-uw%40mail.gmail.com), [#15 (Andres Freund, 2025-11-04)](https://www.postgresql.org/message-id/2q55w253wxssh42krubzta2f2giakyagcmusie5gpjqy2zjnxs%40aaarowf2ab7c), [#16 (Thomas Munro, 2025-11-05)](https://www.postgresql.org/message-id/CA%2BhUKGLWjwa3mTP1nEySjmGwFnHXdTE79OymmQdf-1yC8%2B6nDQ%40mail.gmail.com)

**Question:** Which of the four patches, and which of the two approaches, is meant for 16 through 19, and which only for master?

Counter-check: held. Only v4-0001 says "Back-patch to 16", and message #9 planned patches 1 and 2 for 16+. The thread leaves open the choice between v4 and the cppflags change, and that choice decides header order in stable branches.

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
| security / privileges | no |  |
| backpatchability | **yes** | Patch 1 says "Back-patch to 16"; the scope for patches 3 and 4 is not stated. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Patch 1 sets a dependency order rule in about 30 meson.build files, and patch 3 adds a libintl discovery heuristic. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which approach is proposed for commit: the v4 reorder of dependency lists, or the move of extra_include_dirs into cppflags from message #15?
2. Which of the four patches go to branches 16 and later, and which are for master only?
3. Can the canary check also cover include paths that come from pkg-config dependencies, so a wrong order next to such a dependency fails CI?
4. What does patch 3 do when msgfmt and libintl are in different prefixes, or when the library directory is lib64?
5. Does the chosen approach fix the Windows case from message #15, where extra_include_dirs comes before src/include/port/win32?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The CommitFest entry lists Thomas Munro as author, and the brief reports Heikki and Andres in the thread. The owner needs to know the meson build and the CI setup.
- **Liability:** The owner signs for include order in about 30 meson.build files on branches 16 and later, and for any later file that gets the order wrong. If patch 3 goes in, the owner also answers reports about libintl found in an unexpected prefix.
- **Author follow-up:** The author carries the patch in all his development branches (message #3) and posted four versions. CFbot asked for a rebase in November 2025 and no rebased patch is visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The failure hits developers who build with meson while an older libpq package with headers is installed outside the default paths. Patch 1 is a mechanical reorder, but the thread has not settled between v4 and the cppflags change, and patch 3 is separable from the fix.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Libintl guess from msgfmt prefix adds a heuristic where an option already works. Refuted (not specific): Duplicate of maintenance-fit-2. P5 must not be used as support: its author withdrew it because his own case already worked, which is not a ruling against discovery heuristics.
- `standards-compat-1` Libintl prefix guessed from msgfmt in PATH can pick the wrong library. Refuted (not specific): Duplicate of maintenance-fit-2. It also misreads message #16: the libintl path exposes an external libpq-fe.h again under the cppflags approach, while patch 1 puts libpq first to cover dependency paths.
- `already-exists-1` Patch 3 adds a guessing path where -Dextra_XXX already works. Refuted (not specific): Duplicate of maintenance-fit-2, which states the same wrong-guess case and the same open question from message #14.
- `demand-1` Libintl guessing from msgfmt location has no stated user demand. Refuted (already handled): Messages #11 and #12 show the need is common: builds on BSD-derived systems and macOS need -Dextra_XXX today, and with NLS only for libintl. The remaining concern about the heuristic is covered by maintenance-fit-2.

### Review angles

- prior-attempts: 1 objection(s). No prior attempt matches patches 1-2, the libpq header order fix and the CI canary. The thread treats them as a parity fix with configure, so this angle raises nothing against them.
- standards-compat: 2 objection(s). Patch 1 brings meson in line with configure and changes no SQL, GUC, pg_dump or psql surface. On the main fix, this angle has no objection.
- replication-ondisk: 0 objection(s). The proposal changes only meson build scripts and CI configuration, so it has no on-disk, WAL, replication, catalog or wire protocol state. The one related point is that pg_config --cppflags may gain extra include dirs under Andres's variant, which affects extension builds only and is outside this angle.
- security: 0 objection(s). The proposal changes the build system only: include path order in meson, a CI canary header, and libintl discovery at build time. It adds no runtime privilege, role, setting or data path. Who runs the build already controls every include path, so no new trust boundary appears. Patch 3 guesses the libintl prefix from msgfmt in $PATH, and $PATH is the builder's own environment.
- maintenance-fit: 2 objection(s). The libpq reorder itself is a bug fix that matches configure, and Heikki and Tristan support it. The objections above are about how long the fix lasts and about the optional patch 3.
- already-exists: 1 objection(s). Patch 1 fixes a build-order bug whose only workaround is uninstalling libpq, and message #11 discussed CFLAGS and set it aside. Only the optional patch 3 is open to a convenience objection.
- demand: 1 objection(s). Patch 1 and its CI check fix a real build failure and match configure behaviour. Tom confirms that BSD-derived platforms need the include option, so a demand objection to them would not hold.

### Precedents retrieved

- P1: [Improvements to Meson docs](https://commitfest.postgresql.org/patch/4208/) (cf, Returned with feedback, 2024-01-20) - judged not relevant; reason summarised from its thread
- P2: [Meson build updates](https://commitfest.postgresql.org/patch/4410/) (cf, Withdrawn, 2023-09-01) - judged not relevant; reason summarised from its thread
- P3: [Speed up clean meson builds by ~25%](https://commitfest.postgresql.org/patch/4914/) (cf, Rejected, 2024-05-20) - judged not relevant; reason summarised from its thread
- P4: [Invalid postgres version detection in meson.build](https://commitfest.postgresql.org/patch/5177/) (cf, Rejected, 2024-08-20) - judged not relevant; reason summarised from its thread
- P5: [Allow meson to find ICU in non-standard location](https://commitfest.postgresql.org/patch/4203/) (cf, Withdrawn, 2023-03-08) - used in the review; reason summarised from its thread
- P6: [Revert change of inclusion order for pg_config_os.h;](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5e04f3857c9369351edbf8f906d1d8c2665c8a1f) (revert, Reverted, 2003-05-15) - used in the review
- P7: [preserve timestamps when installing headers](https://commitfest.postgresql.org/patch/3363/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P8: [Allow COPY "text" to output a header and add header matching mode to COPY FROM](https://commitfest.postgresql.org/patch/2504/) (cf, Returned with feedback, 2020-09-29) - judged not relevant; reason summarised from its thread
- P9: [Create shorthand for including all extra tests](https://commitfest.postgresql.org/patch/4547/) (cf, Rejected, 2024-01-20) - judged not relevant; reason summarised from its thread
- P10: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P11: [Consider incremental sort for fractional paths in generate_orderappend_paths](https://commitfest.postgresql.org/patch/3513/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P12: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P13: [Revert "Temporarily add some information about python include paths to configure."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e2217706619520fc70cb4fb41abf7535956e7bef) (revert, Reverted, 2022-01-25) - judged not relevant
- P14: [Avoid hiding shared filesets in pg_ls_tmpdir (pg_ls_* functions for showing metadata ...)](https://commitfest.postgresql.org/patch/2377/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P15: [Revert inadvertant change in pg_config behavior](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fc8a81e3e73e30ac083cc60837ecf98b33d5cc7e) (revert, Reverted, 2016-02-17) - judged not relevant
- P16: [pgbench log file headers](https://commitfest.postgresql.org/patch/4660/) (cf, Returned with feedback, 2024-04-07) - judged not relevant
- P17: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P18: [Revert per-index collation version tracking feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721) (revert, Reverted, 2021-05-07) - judged not relevant
- P19: [Detection and resolution of logical replication conflict](https://commitfest.postgresql.org/patch/5021/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P20: [Avoid orphaned dependencies](https://commitfest.postgresql.org/patch/3106/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P21: [Revert "Blind attempt to fix _configthreadlocale() failures on MinGW."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a62d90f2e5cb55d0c8998b63120d6db3c55df866) (revert, Reverted, 2024-11-27) - judged not relevant
- P22: [A hook for path-removal decision on add_path](https://commitfest.postgresql.org/patch/2234/) (cf, Returned with feedback, 2021-10-05) - judged not relevant
- P23: [Remove dependence on integer wrapping](https://commitfest.postgresql.org/patch/5035/) (cf, Withdrawn, 2024-06-17) - judged not relevant
- P24: [Allow ordered partition scans in more cases](https://commitfest.postgresql.org/patch/4198/) (cf, Withdrawn, 2023-06-24) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 152 s; cost reported by the CLI: $2.26
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 83 text field(s) checked, 5 finding(s), 5 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:41:29.105Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
