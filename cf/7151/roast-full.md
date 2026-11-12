# Design pre-review: Let OpenSSL auto-select DH parameters by default

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7151](https://commitfest.postgresql.org/patch/7151/) (Needs review); authors: Peter Eisentraut (petere)
- Thread: [Let OpenSSL auto-select DH parameters by default](https://www.postgresql.org/message-id/flat/953bceb7-340d-4717-9682-0ebe65c4b57e%40eisentraut.org), started 2026-08-11
- Messages read: 1 of 1
- Patch prose read: 0001-Let-OpenSSL-auto-select-DH-parameters-by-default.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

One objection survived, security-1, and it is fixable. A server with a key below 2048 bits and an empty ssl_dh_params_file loses the 2048-bit DH floor on upgrade, with no warning or release note. The goal and the design are otherwise sound, and no level fails. Someone who knows the OpenSSL code must confirm the library behavior and the version coverage.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm that a failed DH file load on reload keeps the old SSL context, or how the patch behaves on LibreSSL. The sizes OpenSSL picks for common key types were also not checked.

## The proposal, restated

**Problem.** The server falls back to the compiled-in FILE_DH2048 parameters when ssl_dh_params_file is empty, and nobody knows when or how to update them. The author wants the parameter size to follow the server key size and current practice without manual upkeep.

**Proposed design.** When ssl_dh_params_file is empty, the server calls SSL_CTX_set_dh_auto() so OpenSSL picks well-known parameters sized to the certificate key, and FILE_DH2048 goes away. The docs mark ssl_dh_params_file as obsolescent and README.SSL loses its ephemeral DH section. A failed DH file load on reload now fails the reload and keeps the old SSL context, where before it fell back to built-in parameters.

**User-visible surface.**
- ssl_dh_params_file: the empty default now means library-chosen DH parameters, and the docs call the setting obsolescent.
- Reload behavior: an unreadable ssl_dh_params_file now fails the SSL reload and keeps the old context instead of using built-in parameters.
- Servers with small keys may get DH parameters smaller than the old 2048-bit default.

**What the author claims.**
- SSL_CTX_set_dh_auto() exists in all relevant OpenSSL and LibreSSL versions.
- Auto-selected parameter size grows with key size and future standards.
- The OpenSSL documentation recommends this approach.
- The old reload fallback to built-in parameters was a bug.
- Smaller DH parameters only occur with key sizes that ignore current best practice.

**Thread state.** Only the first message is visible. The entry has moved through two CommitFests with no recorded review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-11; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Silent drop below 2048-bit DH for servers with small keys

fixable · level 2, rubric item 7 · security / privileges · `security-1`

A site runs an old 1024-bit RSA certificate and upgrades Postgres. With the empty default, SSL_CTX_set_dh_auto() picks DH parameters matched to that key, so TLS 1.2 DHE sessions use a smaller group than the old 2048-bit built-in one. Nothing in the server log or the docs tells the administrator.

Message #1 accepts that small keys can now give DH parameters below the old default, and calls that the user's own fault. Today the compiled-in parameters set a 2048-bit floor for DHE sessions, whatever the certificate size. The patch removes that floor on upgrade, with no setting change, no warning and no planned release note. OpenSSL security levels may reject such keys on some builds, but the proposal does not rely on that or say which builds do.

Thread: [#1 (Peter Eisentraut, 2026-08-11)](https://www.postgresql.org/message-id/953bceb7-340d-4717-9682-0ebe65c4b57e%40eisentraut.org)

**Question:** Why not keep a 2048-bit minimum, for example by logging a warning or refusing auto-selected groups below it, and will the release notes mention the change?

Counter-check: held. Message #1 confirms the drop and proposes no warning, floor or release note. The affected case is narrow (small keys, TLS 1.2 DHE ciphers only), but a server upgraded with unchanged settings does lose the 2048-bit floor, so it stays fixable.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch removes the FILE_DH2048 macro from src/include/libpq/libpq-be.h, a backend header that extensions can include. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but the empty default of ssl_dh_params_file changes meaning and the docs mark the setting obsolescent. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch changes which DH parameters TLS 1.2 DHE sessions use, and small keys can get groups below 2048 bits. |
| backpatchability | **yes** | Message #1 calls the old reload fallback a bug but does not say whether that fix is meant for back branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch removes compiled-in parameters that had no update process and makes the server depend on SSL_CTX_set_dh_auto() in OpenSSL and LibreSSL. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why not keep a 2048-bit minimum for auto-selected DH parameters, or log a warning when the selected group is smaller?
2. What DH parameter sizes does SSL_CTX_set_dh_auto() pick for RSA 2048, RSA 3072 and ECDSA P-256 server keys?
3. Which OpenSSL and LibreSSL versions first provide SSL_CTX_set_dh_auto(), and are they at or below the minimum versions Postgres supports?
4. Is the reload behavior change for an unreadable ssl_dh_params_file meant for back-patching, and will it get a release note?
5. Does any known extension use the FILE_DH2048 macro from libpq-be.h?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Eisentraut, a committer per the review notes, wrote the patch. A second reviewer who knows the OpenSSL backend code and LibreSSL differences has not spoken in the thread.
- **Liability:** The owner signs for the default TLS 1.2 DHE parameters of every server with an empty ssl_dh_params_file, and for future changes in how OpenSSL and LibreSSL pick them. This lasts as long as TLS 1.2 is supported.
- **Author follow-up:** The author moved the entry to the next CommitFest on 2026-10-01 and asked for reviews in message #1.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** most users
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The change affects every server that leaves ssl_dh_params_file empty, which is the default, and it removes compiled-in parameters that nobody knew how to update. The patch is about 9 KiB in one backend file plus docs, with one fixable point (security-1) to settle.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Existing servers may silently negotiate smaller DH parameters after upgrade. Refuted (not specific): Duplicate of security-1. Both describe the same 1024-bit RSA upgrade case, and security-1 states the lost 2048-bit floor more exactly and fits the safety item better than item 6.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no row about the same or a similar idea, and message #1 does not point to an earlier discussion of auto-selected DH parameters. From the prior-attempts angle there is no earlier rejection reason to carry over.
- standards-compat: 1 objection(s). The reload change aligns DH file errors with other SSL reload errors and is safe. Marking the GUC obsolescent without removing it keeps existing configs and dumps working.
- replication-ondisk: 0 objection(s). From the replication and on-disk angle, I have no objection. The proposal touches no on-disk format, WAL, catalog or Postgres protocol message; it only changes how the server picks TLS 1.2 DH parameters at SSL context setup. A walsender or other client that negotiates TLS 1.2 gets library-chosen parameters, but the protocol is unchanged. The removed FILE_DH2048 macro is a backend-internal constant in a new major version, so I see no realistic extension ABI break.
- security: 1 objection(s). Only one security point found. The reload change, which keeps the old SSL context when the DH file cannot be loaded, is safer than the old fallback. Preferring well-known groups matches current TLS practice, so neither is raised.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle, the proposal lowers long-run cost. It removes the compiled-in FILE_DH2048 parameters, which nobody knew when or how to update, and hands sizing to the TLS library. It adds no GUC, no hot-path cost and no new dependency, and a committer wrote it. Points that need the code or more information: the oldest OpenSSL and LibreSSL versions that have SSL_CTX_set_dh_auto(), whether any extension uses FILE_DH2048, and whether the reload change gets a release note. No reviewer engaged in two CommitFests, which bears on committability, not on the merits.
- already-exists: 0 objection(s). Postgres has no existing way to make the default DH parameters track key size without manual upkeep. ssl_dh_params_file needs each administrator to supply a file, and the patch replaces Postgres's own compiled-in parameters with the SSL library's mechanism, so it removes a mechanism and adds none.
- demand: 0 objection(s). No demand objection applies. The patch removes code, the compiled-in FILE_DH2048 parameters, that has no documented process for keeping it current, and it adds no feature, GUC, syntax or catalog surface that needs a user group to justify it. The benefit goes to every server that leaves ssl_dh_params_file empty, which is the default, so the users who gain are the same users who carry any risk. The open points are the small-key DH size, the reload behavior change, the minimum OpenSSL and LibreSSL versions, and removing the libpq-be.h macro. These are safety and compatibility questions for other angles.

### Precedents retrieved

- P1: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant; reason summarised from its thread
- P2: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P3: [Direct SSL Connections](https://commitfest.postgresql.org/patch/4135/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P4: [Reload SSL certificates on SIGHUP](https://commitfest.postgresql.org/patch/259/) (cf, Returned with feedback, 2015-10-03) - judged not relevant
- P5: [Add test case for sslinfo](https://commitfest.postgresql.org/patch/2203/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P6: [Set host(no)ssl settings from initdb](https://commitfest.postgresql.org/patch/2378/) (cf, Returned with feedback, 2020-09-07) - judged not relevant; reason summarised from its thread
- P7: [Add external tool to auto-tune some postgresql.conf parameters](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [Opclass parameters](https://commitfest.postgresql.org/patch/1559/) (cf, Returned with feedback, 2018-03-01) - judged not relevant; reason summarised from its thread
- P9: [Opclass parameters](https://commitfest.postgresql.org/patch/2183/) (cf, Returned with feedback, 2019-12-01) - judged not relevant; reason summarised from its thread
- P10: [Simplify ExecWithoutOverlapsNotEmpty by removing unused parameter](https://commitfest.postgresql.org/patch/6556/) (cf, Withdrawn, 2026-06-06) - judged not relevant; reason summarised from its thread
- P11: [Fix incorrect Spanish translation and remove obsolete FIXME comments in es.po](https://commitfest.postgresql.org/patch/6515/) (cf, Withdrawn, 2026-03-19) - judged not relevant; reason summarised from its thread
- P12: [Remove unused function parameters, part 2: replication](https://commitfest.postgresql.org/patch/6264/) (cf, Withdrawn, 2026-02-13) - judged not relevant
- P13: [Revert "Only provide new libpq sslpasskey hook for openssl-enabled builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6741cfa5964768279a13e01e46931664fe0903a4) (revert, Reverted, 2020-04-17) - judged not relevant
- P14: [Remove unused function parameters, part 1: contrib](https://commitfest.postgresql.org/patch/6263/) (cf, Withdrawn, 2026-02-13) - judged not relevant
- P15: [Let people set host(no)ssl settings from initdb](https://commitfest.postgresql.org/patch/2916/) (cf, Rejected, 2021-03-09) - judged not relevant
- P16: [autovacuum may skip tables when session_authorization/role is set on database](https://commitfest.postgresql.org/patch/4702/) (cf, Returned with feedback, 2024-01-11) - judged not relevant
- P17: [Fallback table AM for relkinds without storage](https://commitfest.postgresql.org/patch/2986/) (cf, Returned with feedback, 2021-03-14) - judged not relevant
- P18: [Refactor compile-time assertion checks for C/C++](https://commitfest.postgresql.org/patch/2437/) (cf, Returned with feedback, 2020-03-23) - judged not relevant
- P19: [Revert "Add test case for obsoleting slot with active walsender"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96795176810b979a2bf1f4563bdcb161679d5b95) (revert, Reverted, 2021-06-20) - judged not relevant
- P20: [OpenSSL 3.0.0 compatibility](https://commitfest.postgresql.org/patch/2810/) (cf, Withdrawn, 2021-03-10) - judged not relevant
- P21: [Avoid mixing custom and OpenSSL BIO functions](https://commitfest.postgresql.org/patch/4834/) (cf, Withdrawn, 2024-03-12) - judged not relevant
- P22: [Revert "Add USER SET parameter values for pg_db_role_setting"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b9a7a822723aebb16cbe7e5fb874e5124745b07e) (revert, Reverted, 2023-05-17) - judged not relevant
- P23: [Make Intel's ICX compiler working](https://commitfest.postgresql.org/patch/6576/) (cf, Withdrawn, 2026-03-13) - judged not relevant
- P24: [Revert "Refactor compile-time assertion checks in c.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa650a99b4bfbe95e88f44070ae0df1818944a7) (revert, Reverted, 2020-03-23) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 101 s; cost reported by the CLI: $1.08
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 56 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:24:12.202Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
