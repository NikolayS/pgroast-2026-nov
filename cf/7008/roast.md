# Design pre-review: Separation of OpenSSL and LibreSSL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7008](https://commitfest.postgresql.org/patch/7008/); [thread](https://www.postgresql.org/message-id/flat/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se), 9 of 9 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread, but the design is not agreed. maintenance-fit-1 is decisive: the patch copies about 1900 common lines, and nothing fails when a change reaches only one copy. Message #3 offers a shared file, and the thread has not settled it. demand-2, standards-compat-1 and standards-compat-2 are fixable points on order of work, build options and the USE_OPENSSL define.

Not checked: The code was not read, so the real size of the difference between the file pairs and the behaviour of the build discovery are unchecked. Only 3 of 9 thread messages were visible.

## Prior attempts

- [Revert "Add notBefore and notAfter to SSL cert info display"](https://www.postgresql.org/message-id/flat/F0DF7102-192D-4C21-96AE-9A01AE153AD1@yesql.se) (related; Reverted, 2024-03-22). "LibreSSL didn't support ASN1_TIME_diff until OpenBSD 7.1, leaving the older OpenBSD animals in the buildfarm complaining." Different now: P5 reverted one feature because old LibreSSL lacked an API. This proposal changes no feature; it moves the LibreSSL differences into separate files.

## Objections a reviewer is likely to raise

### 1. Two copies of about 1900 shared lines with no check against drift

A committer fixes a TLS bug or a security issue in be-secure-openssl.c and backpatches it. The same code in be-secure-libressl.c is left unchanged, and every test on OpenSSL builds passes. LibreSSL users keep the bug until someone diffs the two files by hand.

Message #3 counts 1889 common lines and only about 640 differing lines in the backend pair, and it reports that fe-secure-libressl.c may already miss 7d45a6dc1974 in the PoC itself. Message #4 says testing twice is not new, and that is true, but applying every change twice is new, and message #1 agrees backpatching gets harder. A missed change in the copy fails only if a LibreSSL buildfarm animal runs a test for it, and the patch names no such coverage.

**Question:** Which mechanism, such as a shared file for common routines or a required LibreSSL CI run, makes a change applied to only one copy fail loudly?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se), [#2](https://www.postgresql.org/message-id/f3806b11-7d89-4888-8cdd-4ec453b5abe3%40eisentraut.org), [#3](https://www.postgresql.org/message-id/anxRgMpS_NuO3Kyl%40alvherre.pgsql), [#4](https://www.postgresql.org/message-id/56FD9443-85A3-4EB6-BD07-F41A894EFC3F%40yesql.se); `maintenance-fit-1`</sub>

### 2. New USE_LIBRESSL define can silently drop SSL code in extensions

An extension that reads SSL connection details guards its code with `#ifdef USE_OPENSSL`, as is common today. Built against a v20 LibreSSL server, it compiles without error but leaves out its SSL code. Users see missing SSL information at run time, with no build error.

The brief lists USE_LIBRESSL as a new pg_config.h define next to USE_OPENSSL, and message #1 mentions `USE_OPENSSL || USE_LIBRESSL` checks. The touched files include no contrib module, so the proposal does not say how pgcrypto and sslinfo, which depend on OpenSSL, are enabled in a libressl build. The proposal also does not say whether USE_OPENSSL stays defined for LibreSSL builds, or what extension authors should test for.

**Question:** In a libressl build, is USE_OPENSSL still defined, are pgcrypto and sslinfo built, and what should out-of-tree code check?

<sub>fixable, level 2, ABI / extension API; messages [#1](https://www.postgresql.org/message-id/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se); `standards-compat-2`</sub>

### 3. Existing LibreSSL builds using ssl=openssl get no defined behavior

An OpenBSD packager or buildfarm owner keeps `--with-ssl=openssl` or `-Dssl=openssl` against LibreSSL, as works today, and upgrades to v20. The openssl files no longer carry the compatibility ifdefs, so the build fails late or compiles against APIs LibreSSL lacks. The packager finds out at build or run time.

Today LibreSSL is used through the openssl option, so every existing LibreSSL build script names `openssl`. The patch adds a `libressl` choice to meson_options.txt. The proposal does not say whether the openssl choice will detect LibreSSL and stop with a clear message, or pick libressl by itself. It also does not say which library `ssl=auto` picks when both are installed, and the autoconf spelling is not shown.

Precedent: [Revert "Add notBefore and notAfter to SSL cert info display"](https://www.postgresql.org/message-id/flat/F0DF7102-192D-4C21-96AE-9A01AE153AD1@yesql.se) (Reverted, 2024-03-22)

**Question:** What does a v20 build with ssl=openssl against LibreSSL headers do, and what does ssl=auto pick when both libraries are present?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se); `standards-compat-1`</sub>

### 4. The split has no payoff until OpenSSL 1.1.1 support is dropped

The split is committed for v20, but the OpenSSL 1.1.1 drop slips or is reduced in scope. The project then carries two near-identical TLS files with no 3.0+ API code to justify them, and every TLS backpatch to v20 has to be applied twice.

Message #4 says there is no point in the patch until 1.1.1 is deprecated, and message #1 offers to keep it rebased until a v3+ API is needed. Message #7 leaves the order of the two steps open. Serverside SNI for LibreSSL is the only concrete near-term user named, and it is planned work, not code.

**Question:** Has the OpenSSL 1.1.1 drop for v20 been agreed, and which committed or posted patch needs a 3.0+ API that the shared file cannot carry?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se), [#4](https://www.postgresql.org/message-id/56FD9443-85A3-4EB6-BD07-F41A894EFC3F%40yesql.se), [#7](https://www.postgresql.org/message-id/anz6JQf4L8Wo-rk8%40paquier.xyz); `demand-2`</sub>

## Questions for the author

1. Which mechanism, such as a shared file for common routines or a required LibreSSL CI run, makes a change applied to only one copy fail loudly?
2. Has the OpenSSL 1.1.1 drop for v20 been agreed, and which posted patch needs a 3.0+ API that the shared file cannot carry?
3. In a libressl build, is USE_OPENSSL still defined, are pgcrypto and sslinfo built, and what should out-of-tree code check?
4. What does a v20 build with ssl=openssl against LibreSSL headers do, and what does ssl=auto pick when both libraries are installed?
5. Does fe-secure-libressl.c contain the change from 7d45a6dc1974, which message #3 suspects is missing?

## Touch points

- **ABI / extension API:** The patch adds USE_LIBRESSL to pg_config.h and changes libpq-be.h, which extensions that test USE_OPENSSL compile against.
- **security / privileges:** The patch duplicates the TLS code of backend and libpq, so a security fix must reach two files.
- **backpatchability:** Message #1 says the split makes backpatching harder, since back branches keep one file and master has two.
- **maintenance burden ("who maintains this in 10 years"):** Two near-identical file pairs, a second build option and LibreSSL test infrastructure need care in every later TLS change.

## Who would commit it, and when

- Owner: qualified committer engaged. Someone who knows the TLS code in libpq and the backend, and the build system, must own it. Daniel Gustafsson wrote the patch, and Peter Eisentraut and Álvaro Herrera reviewed the approach.
- Priority: fix design first; audience niche; review cost medium. Only LibreSSL builds gain directly, though the split also clears the way for OpenSSL 3.0+ APIs. The patch is a PoC of about 143 KiB, mostly copied code, and the thread must first choose between a full copy and a shared file.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
