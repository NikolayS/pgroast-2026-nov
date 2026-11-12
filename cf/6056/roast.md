# Design pre-review: meson: Fix libpq search order

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6056](https://commitfest.postgresql.org/patch/6056/); [thread](https://www.postgresql.org/message-id/flat/CA%2BhUKGKispvxLyrBn3%3D3mp0BB1N%2BRBYR5eE2guCOksnwEoOcPQ%40mail.gmail.com), 16 of 16 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at level 1a or 1b: the build failure is real and patch 1 makes meson match configure. Three fixable level 2 points remain. maintenance-fit-1 shows the CI canary does not cover include paths from pkg-config dependencies. maintenance-fit-2 and standards-compat-2 concern the optional libintl guess in patch 3 and the unstated back-patch scope, with the choice between v4 and the cppflags change still open.

Not checked: The code was not read, so this review cannot confirm that every target that includes libpq-fe.h gets libpq first, or what patch 3 probes. Only 3 of 16 messages were visible; later messages are known from the brief and the objections.

## Prior attempts

- [Allow meson to find ICU in non-standard location](https://www.postgresql.org/message-id/flat/10930b0dd06d7fbba2b67c07d39bc7ace08d5e42.camel@j-davis.com) (related; Withdrawn, 2023-03-08). "Obviously we'd like a little better solution so that others don't get confused, but it's not really a problem for me any more." ([message](https://www.postgresql.org/message-id/875e58307282b963b4e8cc4671bad8a8473caca2.camel%40j-davis.com)) Different now: P5 changed how meson finds ICU when -Dextra_* options are given, and its author withdrew it because his own case already worked. Patch 3 here guesses the libintl prefix from msgfmt so that no -Dextra_* option is needed; patches 1 and 2 fix header order, which P5 did not touch.
- [Revert change of inclusion order for pg_config_os.h;](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5e04f3857c9369351edbf8f906d1d8c2665c8a1f) (related; Reverted, 2003-05-15). "If you need another include file for Windows, then add it; don't go messing with the semantics of every other port's include files." Different now: P6 reverted a global change to the inclusion order of pg_config_os.h. Patch 1 only places the in-tree libpq directory first, as configure does.

## Objections a reviewer is likely to raise

### 1. Order convention in about 30 meson.build files has only partial CI coverage

A later patch adds a frontend tool with `dependencies: [libxml, libpq]`. On FreeBSD, pkg-config for libxml adds -I/usr/local/include, where a packaged older libpq-fe.h lives. CI stays green because the canary sits only in -Dextra_include_dirs, and a developer finds out when their own build fails.

Patch 1 makes correctness depend on list order in about 30 build files, which message #9 calls a "flimsy list that is copied all over the place". Heikki's concern in message #6 is met by patch 2, but only for paths from extra_include_dirs and --with-includes. Message #16 says paths from other dependencies can expose an external libpq-fe.h again. The canary does not cover that source.

**Question:** Can the canary check also cover include paths from pkg-config dependencies, or can the ordering be set in one central place so new meson.build files cannot get it wrong?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#6](https://www.postgresql.org/message-id/b3796a7b-8f78-4a2d-ba74-24e00f2ca71e%40iki.fi), [#9](https://www.postgresql.org/message-id/CA%2BhUKG%2B%3DMPY7MDh3fv8DjbGNBTO9zUO7v_RZDs29M5fLRJD-uw%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/CA%2BhUKGLWjwa3mTP1nEySjmGwFnHXdTE79OymmQdf-1yC8%2B6nDQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Guessing the libintl prefix from msgfmt is a heuristic core must keep

A macOS user has Homebrew and MacPorts, and the msgfmt first in $PATH comes from the other prefix or from a multilib layout using lib64. Meson then takes libintl headers or libraries from a prefix the user did not choose. The user may not see the mismatch until link time or run time.

Patch 3 itself asks whether there is a reason not to do this. Tristan objects to the hard-coded "lib" directory in message #14, and the thread does not answer him. Once released, users will depend on this discovery rule, and every later packaging layout becomes a report against core. Patch 3 is not needed for the libpq fix, and the material does not say whether it would be back-patched.

**Question:** Can patches 3 and 4 move to a separate master-only thread, and what does the probe do when msgfmt and libintl come from different prefixes or use lib64?

<sub>fixable, level 2, design; messages [#12](https://www.postgresql.org/message-id/CA%2BhUKGJk85%2BsDf4D%3DJSwuvA9a4-sy6fnM3B%3DCVaU9Rk2V4NE2g%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/DDZP25P4VZ48.3LWMZBGA1K9RH%40partin.io); `maintenance-fit-2`</sub>

### 3. Back-patch scope does not say which patches go to stable branches

A distribution rebuilds a 16.x minor release with its existing meson options. If patch 3 is back-patched, which libintl Meson finds can change within the minor release. A Windows packager may also see a different header order if Andres's cppflags approach replaces v4.

Message #9 and the v4-0001 commit message plan a back-patch to 16. Only patch 1 is a plain reorder; patch 3 changes how a dependency is found. Message #16 leaves open whether v4 or the cppflags change will be committed, and that choice decides what packagers of stable branches see.

**Question:** Which of the four patches, and which of the two approaches, is meant for 16 through 19, and which only for master?

<sub>fixable, level 2, backpatchability; messages [#9](https://www.postgresql.org/message-id/CA%2BhUKG%2B%3DMPY7MDh3fv8DjbGNBTO9zUO7v_RZDs29M5fLRJD-uw%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/2q55w253wxssh42krubzta2f2giakyagcmusie5gpjqy2zjnxs%40aaarowf2ab7c), [#16](https://www.postgresql.org/message-id/CA%2BhUKGLWjwa3mTP1nEySjmGwFnHXdTE79OymmQdf-1yC8%2B6nDQ%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Which approach is proposed for commit: the v4 reorder of dependency lists, or the move of extra_include_dirs into cppflags from message #15?
2. Which of the four patches go to branches 16 and later, and which are for master only?
3. Can the canary check also cover include paths that come from pkg-config dependencies, so a wrong order next to such a dependency fails CI?
4. What does patch 3 do when msgfmt and libintl are in different prefixes, or when the library directory is lib64?
5. Does the chosen approach fix the Windows case from message #15, where extra_include_dirs comes before src/include/port/win32?

## Touch points

- **backpatchability:** Patch 1 says "Back-patch to 16"; the scope for patches 3 and 4 is not stated.
- **maintenance burden ("who maintains this in 10 years"):** Patch 1 sets a dependency order rule in about 30 meson.build files, and patch 3 adds a libintl discovery heuristic.

## Who would commit it, and when

- Owner: qualified committer engaged. The CommitFest entry lists Thomas Munro as author, and the brief reports Heikki and Andres in the thread. The owner needs to know the meson build and the CI setup.
- Priority: fix design first; audience niche; review cost low. The failure hits developers who build with meson while an older libpq package with headers is installed outside the default paths. Patch 1 is a mechanical reorder, but the thread has not settled between v4 and the cppflags change, and patch 3 is separable from the fix.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
