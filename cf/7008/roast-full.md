# Design pre-review: Separation of OpenSSL and LibreSSL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7008](https://commitfest.postgresql.org/patch/7008/) (Needs review); authors: Daniel Gustafsson (d_gustafsson)
- Thread: [LibreSSL and OpenSSL separation in libpq to support 1.1.1 deprecation](https://www.postgresql.org/message-id/flat/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se), started 2026-07-09
- Messages read: 9 of 9
- Patch prose read: vPoC-0001-ssl-Split-OpenSSL-and-LibreSSL-into-separate-mo.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread, but the design is not agreed. maintenance-fit-1 is decisive: the patch copies about 1900 common lines, and nothing fails when a change reaches only one copy. Message #3 offers a shared file, and the thread has not settled it. demand-2, standards-compat-1 and standards-compat-2 are fixable points on order of work, build options and the USE_OPENSSL define.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the real size of the difference between the file pairs and the behaviour of the build discovery are unchecked. Only 3 of 9 thread messages were visible.

## The proposal, restated

**Problem.** Postgres supports LibreSSL through the OpenSSL code path, with ifdefs for API differences. LibreSSL stays on the 1.1.1-style API, so moving libpq and the backend to OpenSSL 3.0+ APIs after dropping 1.1.1 would add more ifdefs. The author calls the result unreadable and a CVE hazard; serverside SNI is already unavailable with LibreSSL for this reason.

**Proposed design.** The patch copies the OpenSSL code into new be-secure-libressl.c and fe-secure-libressl.c and removes the compatibility ifdefs from both copies. It adds a separate build option and extends the SSL test infrastructure for LibreSSL. It is a PoC: autoconf support is untested and build documentation is missing.

**User-visible surface.**
- Meson option ssl gains the choice 'libressl'; the description no longer lists only openssl.
- A matching configure (autoconf) option for LibreSSL, whose name the material does not show.
- New build define USE_LIBRESSL next to USE_OPENSSL, visible to code that checks pg_config.h.

**What the author claims.**
- The patch adds or removes no features and changes no API usage.
- Diffing the openssl and libressl files makes the differences easy to check.
- Maintenance burden is not new, since LibreSSL is already supported and must be tested, but backpatching gets harder.
- LibreSSL lacks the manpower to follow OpenSSL and will likely stay on the 1.1.1 API.
- Serverside SNI for LibreSSL, planned for v20, needs this split to keep files readable.

**Thread state.** Peter Eisentraut objects that most code is shared, so changes would be made and tested twice and the LibreSSL copy may degrade; Álvaro Herrera suggests a shared file for common routines. Jacob Champion and Michael Paquier support the split together with dropping OpenSSL 1.1.1 in v20, and the order of the two steps is open.

## 2. Prior attempts

### Revert "Add notBefore and notAfter to SSL cert info display" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=697f8d266cfb33409f7ccf3319f4448477066329](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=697f8d266cfb33409f7ccf3319f4448477066329)
- **Thread:** [https://www.postgresql.org/message-id/flat/F0DF7102-192D-4C21-96AE-9A01AE153AD1@yesql.se](https://www.postgresql.org/message-id/flat/F0DF7102-192D-4C21-96AE-9A01AE153AD1@yesql.se), [https://www.postgresql.org/message-id/flat/182b8565486.10af1a86f158715.2387262617218380588@highgo.ca](https://www.postgresql.org/message-id/flat/182b8565486.10af1a86f158715.2387262617218380588@highgo.ca)
- **Date:** decided 2024-03-22; started 2024-03-22
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 6acb0a628eccab8764e0306582c2b7e2a1441b9b since LibreSSL didn't support ASN1_TIME_diff until OpenBSD 7.1, leaving the older OpenBSD animals in the buildfarm complaining. Per plover in the buildfarm.
- **Reason (quote):** "LibreSSL didn't support ASN1_TIME_diff until OpenBSD 7.1, leaving the older OpenBSD animals in the buildfarm complaining."
- **What is different this time:** P5 reverted one feature because old LibreSSL lacked an API. This proposal changes no feature; it moves the LibreSSL differences into separate files.


## 3. Objections a reviewer is likely to raise

### 1. Two copies of about 1900 shared lines with no check against drift

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), security / privileges, backpatchability · `maintenance-fit-1`

A committer fixes a TLS bug or a security issue in be-secure-openssl.c and backpatches it. The same code in be-secure-libressl.c is left unchanged, and every test on OpenSSL builds passes. LibreSSL users keep the bug until someone diffs the two files by hand.

Message #3 counts 1889 common lines and only about 640 differing lines in the backend pair, and it reports that fe-secure-libressl.c may already miss 7d45a6dc1974 in the PoC itself. Message #4 says testing twice is not new, and that is true, but applying every change twice is new, and message #1 agrees backpatching gets harder. A missed change in the copy fails only if a LibreSSL buildfarm animal runs a test for it, and the patch names no such coverage.

Thread: [#1 (Daniel Gustafsson, 2026-07-09)](https://www.postgresql.org/message-id/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se), [#2 (Peter Eisentraut, 2026-08-12)](https://www.postgresql.org/message-id/f3806b11-7d89-4888-8cdd-4ec453b5abe3%40eisentraut.org), [#3 (Álvaro Herrera, 2026-08-12)](https://www.postgresql.org/message-id/anxRgMpS_NuO3Kyl%40alvherre.pgsql), [#4 (Daniel Gustafsson, 2026-08-12)](https://www.postgresql.org/message-id/56FD9443-85A3-4EB6-BD07-F41A894EFC3F%40yesql.se)

**Question:** Which mechanism, such as a shared file for common routines or a required LibreSSL CI run, makes a change applied to only one copy fail loudly?

Counter-check: held. Message #4 answers the testing cost but concedes that applying changes twice is a real cost, and message #1 concedes harder backpatching. The thread has not settled the shared file from message #3, and the patch names no LibreSSL coverage.

### 2. New USE_LIBRESSL define can silently drop SSL code in extensions

fixable · level 2, rubric item 13 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

An extension that reads SSL connection details guards its code with `#ifdef USE_OPENSSL`, as is common today. Built against a v20 LibreSSL server, it compiles without error but leaves out its SSL code. Users see missing SSL information at run time, with no build error.

The brief lists USE_LIBRESSL as a new pg_config.h define next to USE_OPENSSL, and message #1 mentions `USE_OPENSSL || USE_LIBRESSL` checks. The touched files include no contrib module, so the proposal does not say how pgcrypto and sslinfo, which depend on OpenSSL, are enabled in a libressl build. The proposal also does not say whether USE_OPENSSL stays defined for LibreSSL builds, or what extension authors should test for.

Thread: [#1 (Daniel Gustafsson, 2026-07-09)](https://www.postgresql.org/message-id/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se)

**Question:** In a libressl build, is USE_OPENSSL still defined, are pgcrypto and sslinfo built, and what should out-of-tree code check?

Counter-check: held. The touched files include no contrib module, and the material does not say whether USE_OPENSSL stays defined in a libressl build. The author's claim that no features are removed depends on that answer for pgcrypto, sslinfo and out-of-tree code.

### 3. Existing LibreSSL builds using ssl=openssl get no defined behavior

fixable · level 2, rubric item 12 · backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An OpenBSD packager or buildfarm owner keeps `--with-ssl=openssl` or `-Dssl=openssl` against LibreSSL, as works today, and upgrades to v20. The openssl files no longer carry the compatibility ifdefs, so the build fails late or compiles against APIs LibreSSL lacks. The packager finds out at build or run time.

Today LibreSSL is used through the openssl option, so every existing LibreSSL build script names `openssl`. The patch adds a `libressl` choice to meson_options.txt. The proposal does not say whether the openssl choice will detect LibreSSL and stop with a clear message, or pick libressl by itself. It also does not say which library `ssl=auto` picks when both are installed, and the autoconf spelling is not shown.

**Precedents.**
- [Revert "Add notBefore and notAfter to SSL cert info display"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=697f8d266cfb33409f7ccf3319f4448477066329) (Reverted, 2024-03-22); thread: [#1](https://www.postgresql.org/message-id/flat/F0DF7102-192D-4C21-96AE-9A01AE153AD1@yesql.se), [#2](https://www.postgresql.org/message-id/flat/182b8565486.10af1a86f158715.2387262617218380588@highgo.ca)

Thread: [#1 (Daniel Gustafsson, 2026-07-09)](https://www.postgresql.org/message-id/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se)

**Question:** What does a v20 build with ssl=openssl against LibreSSL headers do, and what does ssl=auto pick when both libraries are present?

Counter-check: held. Message #1 lists meson discovery and autoconf as unfinished, and the material does not say what ssl=openssl does against LibreSSL headers or what ssl=auto picks. [P5] concerns a missing API in old LibreSSL and must not be used as support.

### 4. The split has no payoff until OpenSSL 1.1.1 support is dropped

fixable · level 1b, rubric item 2 · desirability, maintenance burden ("who maintains this in 10 years"), backpatchability · `demand-2`

The split is committed for v20, but the OpenSSL 1.1.1 drop slips or is reduced in scope. The project then carries two near-identical TLS files with no 3.0+ API code to justify them, and every TLS backpatch to v20 has to be applied twice.

Message #4 says there is no point in the patch until 1.1.1 is deprecated, and message #1 offers to keep it rebased until a v3+ API is needed. Message #7 leaves the order of the two steps open. Serverside SNI for LibreSSL is the only concrete near-term user named, and it is planned work, not code.

Thread: [#1 (Daniel Gustafsson, 2026-07-09)](https://www.postgresql.org/message-id/2AEF44BB-6414-40F7-BB8D-4FDB66199A81%40yesql.se), [#4 (Daniel Gustafsson, 2026-08-12)](https://www.postgresql.org/message-id/56FD9443-85A3-4EB6-BD07-F41A894EFC3F%40yesql.se), [#7 (Michael Paquier, 2026-08-12)](https://www.postgresql.org/message-id/anz6JQf4L8Wo-rk8%40paquier.xyz)

**Question:** Has the OpenSSL 1.1.1 drop for v20 been agreed, and which committed or posted patch needs a 3.0+ API that the shared file cannot carry?

Counter-check: held. The author states the same condition in messages #1 and #4, but message #7 leaves the order open and prefers the split first. The material does not show that the 1.1.1 drop is agreed, so the scenario can happen.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds USE_LIBRESSL to pg_config.h and changes libpq-be.h, which extensions that test USE_OPENSSL compile against. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch duplicates the TLS code of backend and libpq, so a security fix must reach two files. |
| backpatchability | **yes** | Message #1 says the split makes backpatching harder, since back branches keep one file and master has two. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Two near-identical file pairs, a second build option and LibreSSL test infrastructure need care in every later TLS change. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which mechanism, such as a shared file for common routines or a required LibreSSL CI run, makes a change applied to only one copy fail loudly?
2. Has the OpenSSL 1.1.1 drop for v20 been agreed, and which posted patch needs a 3.0+ API that the shared file cannot carry?
3. In a libressl build, is USE_OPENSSL still defined, are pgcrypto and sslinfo built, and what should out-of-tree code check?
4. What does a v20 build with ssl=openssl against LibreSSL headers do, and what does ssl=auto pick when both libraries are installed?
5. Does fe-secure-libressl.c contain the change from 7d45a6dc1974, which message #3 suspects is missing?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Someone who knows the TLS code in libpq and the backend, and the build system, must own it. Daniel Gustafsson wrote the patch, and Peter Eisentraut and Álvaro Herrera reviewed the approach.
- **Liability:** The owner signs for keeping two TLS file pairs in step on master for as long as LibreSSL is supported, and for backpatches from two files into single-file back branches.
- **Author follow-up:** The author offers in message #1 to keep the patch rebased, plans LibreSSL SNI work for v20, and moved the entry to the next CommitFest in October 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Only LibreSSL builds gain directly, though the split also clears the way for OpenSSL 3.0+ APIs. The patch is a PoC of about 143 KiB, mostly copied code, and the thread must first choose between a full copy and a shared file.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Separate LibreSSL module has no stated minimum LibreSSL version. Refuted (wrong facts): The Postgres installation documentation already states a minimum LibreSSL version, and the commit message says only that support began without one. The revert in [P5] predates that floor, and the same risk exists today with the shared file.
- `replication-ondisk-1` New USE_LIBRESSL define silently changes what extensions compile against. Refuted (not specific): Duplicate of standards-compat-2, which covers the same USE_OPENSSL guard problem and also the contrib modules.
- `security-1` Duplicated TLS code lets security fixes miss the LibreSSL copy. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same drift between the two copies with the line counts from message #3.
- `maintenance-fit-2` Extract the shared code first, before forking the files. Refuted (not specific): Duplicate of maintenance-fit-1: the failure scenario is the same drift cost, and the shared file is already the remedy that objection asks about.
- `maintenance-fit-3` USE_LIBRESSL next to USE_OPENSSL may break code that tests USE_OPENSSL. Refuted (not specific): Duplicate of standards-compat-2. Its added question about what ssl_library reports can be asked under that objection.
- `demand-1` No evidence of LibreSSL users large enough to justify a second copy. Refuted (already handled): LibreSSL is already supported and buildfarm tested, so the patch adds no new user group, and the alternative is removing existing support. [P4] concerned adding a new library, and three TLS-area committers in the thread prefer the split to deprecation.

### Review angles

- prior-attempts: 1 objection(s). The NSS and Secure Transport entries ([P4], [P6], [P8], [P12]) added new TLS libraries and stopped for reasons specific to those libraries or to stalled patches. This proposal splits a library that is already supported, so those reasons do not apply.
- standards-compat: 2 objection(s). From the standards angle, the SQL surface, catalogs, pg_dump and psql are unchanged. Only the build options and the pg_config.h define are user-facing.
- replication-ondisk: 1 objection(s). Beyond the build define, the proposal touches no on-disk format, WAL, catalog or replication state, and it does not change the TLS behaviour on the wire.
- security: 1 objection(s). Apart from the risk of drift between the duplicated TLS files, the split adds no privilege, role or exposure change. It is a build-time choice of library with no new features.
- maintenance-fit: 3 objection(s). Objections found; the split's goal has support from Jacob Champion and Michael Paquier, so these concern the design and its cost, not the aim.
- already-exists: 0 objection(s). This is an internal restructuring of the TLS code, so no existing feature, extension or external tool could stand in for it; the status quo is the shared OpenSSL file with ifdefs. Whether to split fully, use a shared common file as in message #3, or keep the ifdefs is a design question for other angles.
- demand: 2 objection(s). The work is aimed at developers and maintainability, and three committers in the thread support it. My objections ask for evidence of LibreSSL demand and for a firm 1.1.1 decision. They do not reject the idea.

### Precedents retrieved

- P1: [OpenSSL 3.0.0 compatibility](https://commitfest.postgresql.org/patch/2810/) (cf, Withdrawn, 2021-03-10) - used in the review; reason summarised from its thread
- P2: [Revert "Only provide new libpq sslpasskey hook for openssl-enabled builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6741cfa5964768279a13e01e46931664fe0903a4) (revert, Reverted, 2020-04-17) - judged not relevant
- P3: [Avoid mixing custom and OpenSSL BIO functions](https://commitfest.postgresql.org/patch/4834/) (cf, Withdrawn, 2024-03-12) - judged not relevant; reason summarised from its thread
- P4: [Support for NSS as a libpq TLS backend](https://commitfest.postgresql.org/patch/3138/) (cf, Rejected, 2022-02-23) - used in the review; reason summarised from its thread
- P5: [Revert "Add notBefore and notAfter to SSL cert info display"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=697f8d266cfb33409f7ccf3319f4448477066329) (revert, Reverted, 2024-03-22) - used in the review
- P6: [Support for NSS as TLS backend](https://commitfest.postgresql.org/patch/2592/) (cf, Returned with feedback, 2021-04-08) - used in the review; reason summarised from its thread
- P7: [Support tls-exporter as channel binding for TLSv1.3](https://commitfest.postgresql.org/patch/3850/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P8: [libpq: Support for Apple Secure Transport SSL library](https://commitfest.postgresql.org/patch/1228/) (cf, Returned with feedback, 2017-11-28) - used in the review; reason summarised from its thread
- P9: [Revert "Change SHA2 implementation based on OpenSSL to use EVP digest routines"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fe0a1dc52c7332a65b44db8e8408a5fd1d8fc8fb) (revert, Reverted, 2020-09-29) - judged not relevant
- P10: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P11: [Avoid use deprecated Windows Memory API](https://commitfest.postgresql.org/patch/3893/) (cf, Rejected, 2023-03-20) - judged not relevant; reason summarised from its thread
- P12: [macOS Secure Transport SSL Support](https://commitfest.postgresql.org/patch/1490/) (cf, Returned with feedback, 2019-02-04) - used in the review; reason summarised from its thread
- P13: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P14: [Remove redundant initializations](https://commitfest.postgresql.org/patch/3229/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P15: [libpq compression](https://commitfest.postgresql.org/patch/3499/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P16: [Remove unused #include's](https://commitfest.postgresql.org/patch/6112/) (cf, Withdrawn, 2025-10-15) - judged not relevant
- P17: [Send duration output to separate files](https://commitfest.postgresql.org/patch/5125/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P18: [Libpq Compression](https://commitfest.postgresql.org/patch/4746/) (cf, Withdrawn, 2024-05-14) - judged not relevant
- P19: [Direct SSL Connections](https://commitfest.postgresql.org/patch/4135/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P20: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P21: [Separate memory contexts for relcache and catcache](https://commitfest.postgresql.org/patch/4554/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P22: [Add option --drop-cascade for pg_dump/restore](https://commitfest.postgresql.org/patch/3065/) (cf, Returned with feedback, 2021-12-01) - judged not relevant
- P23: [INNER JOIN removals](https://commitfest.postgresql.org/patch/130/) (cf, Returned with feedback, 2015-05-11) - judged not relevant
- P24: [Remove some unnecessary if-condition](https://commitfest.postgresql.org/patch/2760/) (cf, Withdrawn, 2020-10-22) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 21; wall time: 159 s; cost reported by the CLI: $2.82
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 6 of 24 kept
- Tone check: 97 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:14:16.292Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
