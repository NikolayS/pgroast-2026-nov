# Design pre-review: Produce LLVM bitcode files on meson builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5671](https://commitfest.postgresql.org/patch/5671/); [thread](https://www.postgresql.org/message-id/flat/206b001d-1884-4081-bd02-bed5c92f02ba%40eisentraut.org), 32 of 32 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is against the core bitcode patches. They restore what the autoconf build already does, and messages #1, #25 and #27 show demand. All ranked objections concern the general extension .pc file in the last patch: no extension project is shown to need it (demand-1), its name and Requires list are not agreed (standards-compat-2, standards-compat-3), and autoconf builds do not install it (standards-compat-1). That patch is last in the series, so the rest can go in without it.

Not checked: Without the code, this review could not check that the meson bitcode flags match the makefile build, what the irlink script does, or what the .pc file holds with -Dllvm=disabled. Only 3 of 32 messages were read directly.

## Prior attempts

None found before 2024-09-05.

## Objections a reviewer is likely to raise

### 1. General extension .pc file has no demonstrated users yet

Postgres 20 installs postgresql-20-extension.pc as a public build interface, and extension authors start to depend on its variables and Requires list. A later release changes the Requires list or the compiler flags still disputed in message #19 and #20. Their builds break, and they find out on upgrade.

Demand for the bitcode part is shown in message #1, #25 and #27, but nobody asks for a general extension interface. Christoph proposed the rename in message #25 and says in message #27 that there are no extension builds yet. Andres said in message #21 that there is not enough agreement for a name this general. Committing patch 0006 makes it a public interface that must be kept stable, with no extension author shown to need it.

**Question:** Which extension projects outside core have built against postgresql-$version-extension.pc, and what did they need from it that PGXS or pg_config does not give them?

<sub>fixable, level 1b, demand; messages [#19](https://www.postgresql.org/message-id/c6b26dee-bd62-44d9-b4f3-730b6ecf95c6%40eisentraut.org), [#20](https://www.postgresql.org/message-id/3whtmw6twe442o3r44gsagheq5cs2clcghzjeo7rljstpdemrh%40oojczkhk2mbt), [#21](https://www.postgresql.org/message-id/blfq7kk5bylebetudvyslef5j2bvigrqikmwy66i4znvkv63uy%40smypdulfe7gb), [#25](https://www.postgresql.org/message-id/ald4zLd-nes2CjuZ%40msg.df7cb.de), [#27](https://www.postgresql.org/message-id/aqzmJ7-5202qXuzx%40msg.df7cb.de); `demand-1`</sub>

### 2. Installed .pc name and variables become a contract before agreement

Version 20 ships postgresql-20-extension.pc with dir_mod, dir_bitcode and the full Requires list, and extensions start to depend on it. A later release changes the name, a variable or the Requires semantics. Extension builds then fail against the new server until every extension's build file is changed.

In message #21 Andres said there is not enough agreement yet for a name this general. In message #13 Peter asked whether "extension" is the right term. The disagreement between Peter (message #19) and Andres (message #20) on whether Requires should list every server dependency is still open, and Christoph in message #27 has no extension builds to report. The .pc file is installed even with -Dllvm=disabled (message #24), so a meson build with LLVM disabled also publishes the interface.

**Question:** Can the core bitcode patches be committed under the internal llvm-jit-bitcode name while the general extension .pc file waits for agreed names, variables and Requires semantics?

<sub>fixable, level 2, ABI / extension API; messages [#13](https://www.postgresql.org/message-id/4286824f-40c3-4716-ad71-2085b83f3736%40eisentraut.org), [#19](https://www.postgresql.org/message-id/c6b26dee-bd62-44d9-b4f3-730b6ecf95c6%40eisentraut.org), [#20](https://www.postgresql.org/message-id/3whtmw6twe442o3r44gsagheq5cs2clcghzjeo7rljstpdemrh%40oojczkhk2mbt), [#21](https://www.postgresql.org/message-id/blfq7kk5bylebetudvyslef5j2bvigrqikmwy66i4znvkv63uy%40smypdulfe7gb), [#24](https://www.postgresql.org/message-id/CAN55FZ2V%2BX%3DWgyr2U0C-%3DhH1RWELmuQcqV523o0R1Yi22K4FCg%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/aqzmJ7-5202qXuzx%40msg.df7cb.de); `standards-compat-2`</sub>

### 3. General extension .pc file exists only in meson builds

An extension author follows the new docs and writes a meson.build that calls dependency('postgresql-20-extension'). A user or packager whose server was built with autoconf then cannot build that extension, because the .pc file is not installed. The author finds out from bug reports, platform by platform.

In message #13 Peter said that installed .pc files must also be built by the makefiles, or else not installed. The autoconf patch was dropped after Andres in message #21 proposed a narrow llvm-jit-bitcode name, so it would not be needed. The last patch (v17-0006) brings back the general postgresql-$version-extension.pc name and documents it as an extension build interface, but the autoconf part is not back. A user-facing interface that depends on how the server was built departs from PGXS and pg_config, which both build systems provide.

**Question:** Will autoconf builds install the same postgresql-$version-extension.pc file, or will the general rename wait until they do?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#13](https://www.postgresql.org/message-id/4286824f-40c3-4716-ad71-2085b83f3736%40eisentraut.org), [#21](https://www.postgresql.org/message-id/blfq7kk5bylebetudvyslef5j2bvigrqikmwy66i4znvkv63uy%40smypdulfe7gb), [#22](https://www.postgresql.org/message-id/CAN55FZ0Y0J29rUqjpX8BMU7HwSgeSz2RxcJm%2B4PLhCeGjtWmhg%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/ald4zLd-nes2CjuZ%40msg.df7cb.de), [#26](https://www.postgresql.org/message-id/CAN55FZ26ohMftt20NUtVAxU8Feqjhz-jEfUvFKATYEaRR%2Bxmqg%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/CAN55FZ3MWpX%2B6FddixNW7NEoYNnHVktsDzsbV060EPO3gxzsnw%40mail.gmail.com); `standards-compat-1`</sub>

### 4. Requires list makes extension builds need every server -dev package

A packager builds a small extension against a distribution server package through the .pc file. pkg-config --cflags fails unless krb5, ldap, icu, libxml2, lz4, openssl, zlib and zstd development packages are all installed. Today the same extension builds with PGXS without most of these packages.

Peter in message #19 calls this a regression for extensions moving to the .pc file. Andres answers in message #20 that server headers include some of these dependencies, and that static linking needs them. That holds for headers such as libpq-be.h, but the proposal does not show which server headers need ldap, lz4 or zstd. It also does not say why Requires.private would not cover the static-linking case. The thread leaves this open.

**Question:** Which listed dependencies do installed server headers actually include, and why would Requires.private not cover the rest?

<sub>fixable, level 2, design; messages [#13](https://www.postgresql.org/message-id/4286824f-40c3-4716-ad71-2085b83f3736%40eisentraut.org), [#14](https://www.postgresql.org/message-id/CAN55FZ0KxMphg%2BM%2B4Shm%3DHmNV%2BXhC7LUKc6hrC%2B-QiovhzvdVA%40mail.gmail.com), [#19](https://www.postgresql.org/message-id/c6b26dee-bd62-44d9-b4f3-730b6ecf95c6%40eisentraut.org), [#20](https://www.postgresql.org/message-id/3whtmw6twe442o3r44gsagheq5cs2clcghzjeo7rljstpdemrh%40oojczkhk2mbt); `standards-compat-3`</sub>

### 5. Nothing tests that meson-built bitcode is still used for inlining

A later patch changes include paths, compiler flags or a contrib source layout. The .bc files still build but no longer match, or are no longer installed, and JIT inlining quietly falls back. Packagers learn of it only through slower queries, if they notice.

In messages #4 and #26, inlining was confirmed only by adding debug logging to llvmjit_inline.cpp, and message #25 says it is hard to tell whether the .bc files are used. Meson now keeps its own bitcode flags and a hand-picked contrib list, separate from the makefile path, so each build-system change has a second place to keep right. Full server JIT on Windows is untested (message #28).

**Question:** Which automated test, run in CI, fails when a meson build installs bitcode that the server cannot load or inline?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#4](https://www.postgresql.org/message-id/CA%2BhiPX8R69OeApj55Fy1VXOhuHUKfdvp2f80icLy65z76Jy%3DJA%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/ald4zLd-nes2CjuZ%40msg.df7cb.de), [#26](https://www.postgresql.org/message-id/CAN55FZ26ohMftt20NUtVAxU8Feqjhz-jEfUvFKATYEaRR%2Bxmqg%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAAH-eSX_OniTafzLjiis%2BLsgnicS5JSuiJriQw_B2wEHM_q4mA%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Can the core bitcode patches be committed under the llvm-jit-bitcode name while the general extension .pc rename waits for agreement?
2. Which extension projects outside core need the installed .pc file, and what does it give them that pg_config or PGXS does not?
3. Will autoconf builds install the same postgresql-$version-extension.pc file, or will the rename wait until they do?
4. Which dependencies in the Requires list do installed server headers include, and why would Requires.private not cover the rest?
5. Which automated CI test fails when a meson build installs bitcode that the server cannot load or inline?

## Touch points

- **ABI / extension API:** The last patch installs postgresql-$version-extension.pc and documents it as an extension build interface.
- **pg_dump / restore round-trip:** see objection standards-compat-1: General extension .pc file exists only in meson builds
- **maintenance burden ("who maintains this in 10 years"):** Meson gets its own bitcode flags, a hand-picked contrib list, the irlink script and an installed .pc file to keep stable.
- **performance on the hot path:** Meson builds start to inline server and contrib functions in JIT, and the thread shows no query-time measurement.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The general extension .pc file overlaps with pg_config and PGXS, which both build systems already provide.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund co-authored the series and reviewed it in the thread. Peter Eisentraut reviewed it but removed himself as reviewer in April 2026.
- Priority: proceed; audience a sizeable minority; review cost medium. Packagers who build with meson and LLVM need the bitcode files, and Christoph Berg reports v15 working in Debian snapshot packages for months. The patch touches about 40 meson.build files but no server code. Review cost is an estimate because no patch code was read.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
