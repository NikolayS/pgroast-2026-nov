# Design pre-review: Fix race with LLVM and bison.

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7206](https://commitfest.postgresql.org/patch/7206/); [thread](https://www.postgresql.org/message-id/flat/f070a61e-1dd3-4fd9-bbc9-18d100b6722a%40postgrespro.ru), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the build failure is reported with a reproduction, and the fix copies the rule src/backend/common.mk already uses. The one surviving objection, maintenance-fit-1, is a fixable scope point at level 2. The remaining decision is the backpatch risk for out-of-tree extensions, which a committer who knows the make build system has to judge.

Not checked: Without the code, it cannot be confirmed which pgxs.mk bitcode targets get the new dependency, or what the change to src/backend/common.mk does. The commit message names only MODULES.

## Prior attempts

None found before 2026-03-27.

## Objections a reviewer is likely to raise

### 1. Stated scope covers MODULES only, not MODULE_big or PROGRAM objects

An out-of-tree extension builds with MODULE_big and a bison-generated header under --with-llvm and make -j. Its .bc files carry no dependency on the .o files, so the build fails at random. The author finds out from a packager's failed build.

The v3 commit message says the .bc files "for MODULES" now depend on their .o files. pgxs also builds bitcode for OBJS under MODULE_big, and the thread does not say whether those targets get the same rule. Message #4 counts on the fix to cover pg_plan_advice "and all future cases", which holds only if every bitcode path in pgxs.mk gets the dependency. If one path is left out, the next module with a generated header hits the same race, and nothing in the build warns about the gap.

**Question:** Does v3 add the .bc-on-.o dependency for every bitcode target pgxs.mk builds, including MODULE_big OBJS, and is pg_plan_advice fixed by it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/v67jke2iv6kaywjnvwhxc3u5x3ttvrz555k5ngcxu2qwu435rl%405wpsxnrsz23k), [#4](https://www.postgresql.org/message-id/CAN4CZFP-HxkwSDsFuBAekdO6C3V13CS%3D1b0oPRz_TB-EPnrRKQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/03a56df5-9925-44bf-9c45-0fbf3fb1f311%40postgrespro.ru); `maintenance-fit-1`</sub>

## Questions for the author

1. Does v3 add the .bc-on-.o dependency for every bitcode target pgxs.mk builds, including MODULE_big OBJS, which contrib/cube and contrib/seg use?
2. Has anyone run `make -C contrib/cube cubescan.bc` and a parallel build of pg_plan_advice with v3 applied, and did both succeed?
3. Which extension setup could break if the pgxs.mk change were backpatched, as message #2 fears?
4. Should back branches get the per-module v2 fix for cube and seg, or no fix at all?
5. Does the meson build have the same race between bitcode compilation and bison-generated headers?

## Touch points

- **ABI / extension API:** The patch changes dependency rules in src/makefiles/pgxs.mk, which out-of-tree extensions use to build bitcode.
- **backpatchability:** Message #2 leans toward master only for fear of breaking extensions; nobody answered the per-module backpatch idea in message #3.
- **maintenance burden ("who maintains this in 10 years"):** The .bc-on-.o rule would exist in both common.mk and pgxs.mk, and each bitcode path in pgxs.mk needs it.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund wrote the generic fix in message #2 and says he wrote the original backend rule; the owner needs to know the make build system and LLVM bitcode rules.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is about 2 KiB, touches two makefiles, and fixes a build failure reported for cube, seg and pg_plan_advice. It copies an existing rule, so the project carries little new after commit.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
