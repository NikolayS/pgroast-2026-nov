# Design pre-review: Add PAM Tests and Option For Custom PAM Config Location

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7219](https://commitfest.postgresql.org/patch/7219/) (Needs review); authors: Andrew Jackson (andrewjackson)
- Thread: [Add PAM Tests and Option For Custom PAM Config Location](https://www.postgresql.org/message-id/flat/CAKK5BkHsWCMLS4B8N%3D9juny1kCzkoHFXHkesEA%3D-ng4pRAHdUw%40mail.gmail.com), started 2026-08-29
- Messages read: 3 of 3
- Patch prose read: 0001-Add-PAM-tests.patch, 0001-Add-PAM-tests.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

PAM test coverage is a sound goal and no objection to it survived. The open point is the permanent pg_hba.conf option that the commit message says was "mostly implemented for the sake of testing". already-exists-1 asks why a test-only redirect such as pam_wrapper is not enough, and demand-1 finds no production user asking for pamconfdir. The list should settle that choice before code review.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it could not be checked whether the ifdef gate covers option parsing in hba.c, so behaviour on builds without pam_start_confdir is unknown. The two level 2 objections that survived (standards-compat-2, standards-compat-3) were not evaluated.

## The proposal, restated

**Problem.** Postgres has no tests for PAM authentication. The author thinks none exist because a test would need root to write /etc/pam.d, or a test environment set up by hand. The commit message adds that sites where DBAs have no root access could use a custom PAM configuration directory.

**Proposed design.** A new pg_hba.conf option, pamconfdir, makes the server call pam_start_confdir (linux-pam 1.4.0 and later) instead of pam_start. Configure and meson probe for that function, and builds without it leave the option out. A new TAP suite in src/test/pam writes a temporary PAM configuration that uses pam_permit and pam_exec, and runs only when PG_TEST_EXTRA includes pam.

**User-visible surface.**
- New pg_hba.conf option pamconfdir for the pam method, present only when built against a PAM that has pam_start_confdir.
- New PG_TEST_EXTRA value pam that enables the src/test/pam TAP suite.
- New build-time symbol HAVE_PAM_START_CONFDIR in pg_config.h.

**What the author claims.**
- Postgres has no PAM tests today, and this suite adds coverage without root or a hand-prepared environment.
- pamconfdir is useful outside tests, for example where DBAs are not allowed root access.
- Builds against older PAM work unchanged; the author built on NixOS with PAM 1.3.1 and 1.7.2, using both make and meson.
- Testing only with pamconfdir set is acceptable because the code paths are mostly the same as with the default directory.

**Thread state.** Brian Cosgrove reviewed v2 and found 5 of 26 assertions failing under meson because of a wrong log path; he also asked for pamconfdir in pg_hba_file_rules, pam in CI and regress.sgml, and a meson fix so frontend tools do not link PAM. The author has not replied to that review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-29; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. pam_wrapper already allows rootless PAM tests without a new pg_hba option

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

Postgres ships pamconfdir as a permanent pg_hba.conf option that exists only on some builds. The new suite runs only through pam_start_confdir, so a regression in the plain pam_start path that production servers use still passes CI. The project then supports a production option that was added mostly for tests.

The pam_wrapper library from the cwrap project preloads into a process and redirects PAM service lookup to a test directory, with no root. A suite built on it would test the unchanged pam_start path, which message #1 accepts as untested. The commit message says the option was "mostly implemented for the sake of testing" and gives non-root DBAs as a secondary use, with no field report. The cost of pam_wrapper is one more optional test dependency and LD_PRELOAD limits on some platforms.

Thread: [#1 (Andrew Jackson, 2026-08-29)](https://www.postgresql.org/message-id/CAKK5BkHsWCMLS4B8N%3D9juny1kCzkoHFXHkesEA%3D-ng4pRAHdUw%40mail.gmail.com)

**Question:** Why is a server-side pamconfdir option better than running the tests under pam_wrapper, which also covers the default pam_start path?

Counter-check: held. pam_wrapper exists and redirects PAM service lookup without root, and the thread does not discuss it. The author's own statement that the option was mostly added for testing makes a test-only alternative a fair question.

### 2. User-facing pamconfdir option justified mainly by test needs

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), security / privileges · `demand-1`

Postgres ships pamconfdir as a permanent pg_hba.conf option, and every later release must document and support it. If few production sites set it, the project keeps a security-relevant knob, chosen per build, that serves mostly its own test suite. A committer would find this out only after removal is no longer possible.

The commit message says the option "was mostly implemented for the sake of testing". Its only production case is DBAs without root, given in one sentence with no field report. Such a DBA still needs a sysadmin to install PAM modules and grant access to credential stores, and a one-time /etc/pam.d/postgresql file covers that today. Message #3 asks for pg_hba_file_rules support, which grows the surface, but no one in the thread asks for the option itself.

Thread: [#1 (Andrew Jackson, 2026-08-29)](https://www.postgresql.org/message-id/CAKK5BkHsWCMLS4B8N%3D9juny1kCzkoHFXHkesEA%3D-ng4pRAHdUw%40mail.gmail.com), [#3 (Brian Cosgrove, 2026-09-22)](https://www.postgresql.org/message-id/CAGKCzNCOOFk1%3DpkUfDA0c6k%2B9_Q7C0NwAs%3DnsQ5vkJ_O17RRRA%40mail.gmail.com)

**Question:** Which production sites need a non-default PAM directory per pg_hba line, and why can their sysadmin not install a file in /etc/pam.d once?

Counter-check: held. The only production case is one sentence in the commit message, and no user in the thread asks for the option. This is a show-demand point, and evidence of sites that need a non-default PAM directory would answer it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The new option should appear in the pg_hba_file_rules.options column, and message #3 reports that it does not. |
| ABI / extension API | **yes** | The patch touches src/include/libpq/hba.h to carry the new pamconfdir value. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | pamconfdir changes which directory supplies the PAM policy that decides authentication. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A permanent pg_hba.conf option, two build probes and a Linux-only opt-in test suite depend on a linux-pam function. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | pam_wrapper may give rootless PAM tests without a server option, and the thread does not discuss it. |

## 5. Questions the author must answer before anyone reviews code

1. Why is a server-side pamconfdir option better than running the tests under pam_wrapper, which also covers the default pam_start path?
2. Which production sites need a non-default PAM directory per pg_hba.conf line, and why can their sysadmin not install a file in /etc/pam.d once?
3. What does a server built without pam_start_confdir do when pg_hba.conf contains pamconfdir: error, warning, or silent ignore?
4. Will the next version show pamconfdir in pg_hba_file_rules.options, as message #3 asks?
5. Does the server check that the pamconfdir directory exists when it loads pg_hba.conf, or does a wrong path show up only at login?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the client authentication code (auth.c, hba.c) and the configure and meson probes would have to own it. The visible thread has the author and one reviewer.
- **Liability:** The owner signs for a permanent pg_hba.conf option tied to a linux-pam function, present only on some builds. The test suite runs only on Linux and only with PG_TEST_EXTRA=pam.
- **Author follow-up:** The author posted a second version six days after the first without being asked. The review in message #3, from 2026-09-22, has no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is about 20 KiB and touches auth.c, hba.c and the build files, so review is cheap. The value is test coverage for PAM users, but the choice between a permanent pg_hba.conf option and a test-only redirect should be settled first.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) pamconfdir exists only on some builds; behaviour elsewhere is undefined. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-3` (fixable, level 2) New hba option missing from pg_hba_file_rules and the manual. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-1` Permanent pg_hba.conf option added mainly to make tests possible. Refuted (not specific): Duplicate of demand-1 (no shown production demand) and already-exists-1 (pam_wrapper alternative). Its security point is weak, because the server's OS user already controls pg_hba.conf and can set trust.
- `security-1` pamconfdir has no ownership or permission check on the PAM directory. Refuted (no failure scenario): The scenario needs the administrator to choose a directory that others can write. Postgres makes no such check for hba_file, ident_file or pg_hba.conf include files, which carry the same weight, so the option follows existing practice.
- `maintenance-fit-1` User-facing pg_hba option added mainly to serve a test suite. Refuted (not specific): Duplicate of demand-1 for the missing production demand, and of standards-compat-2 for the behaviour on builds without pam_start_confdir.
- `maintenance-fit-2` Opt-in suite skips the default pam_start path and does not run in CI. Refuted (no failure scenario): The scenario describes today's state, where PAM has no tests at all, so the patch does not make it worse. The PG_TEST_EXTRA gate copies the ldap and kerberos suites, and the CI and log-path points are test-level items already raised in message #3.

### Review angles

- prior-attempts: 0 objection(s). The one precedent, [P11], reverted a TAP test that broke on Debian because pg_config is patched there. It shares only the theme of TAP tests with this proposal, and its recorded reason does not apply to PAM tests or to pamconfdir. The thread cites no earlier attempt at PAM tests or a custom PAM configuration directory, so no precedent was found.
- standards-compat: 3 objection(s). Not applicable; objections listed above.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state: no WAL, on-disk format, catalog or wire protocol change, and nothing for pg_upgrade or standbys. The new HbaLine field changes an internal struct in a master-only feature, which raises no backpatch or extension ABI concern that I could tie to a concrete failure. The pg_hba_file_rules output gap in message #3 belongs to another angle.
- security: 1 objection(s). pamconfdir does not grant new power: anyone who can edit pg_hba.conf can already set trust. The one gap found is the missing permission check on the PAM directory.
- maintenance-fit: 2 objection(s)
- already-exists: 1 objection(s). One objection only. Nothing in core gives a non-root DBA a custom PAM directory, so that use case gets no objection from this angle beyond its thin demand.
- demand: 1 objection(s). PAM test coverage is a project need, not a user feature, so it needs no demand evidence. The demand question applies only to the production use of pamconfdir.

### Precedents retrieved

- P1: [Allow cluster owner to bypass authentication](https://commitfest.postgresql.org/patch/2244/) (cf, Returned with feedback, 2020-04-06) - judged not relevant; reason summarised from its thread
- P2: [Add inline comments to the pg_hba_file_rules view](https://commitfest.postgresql.org/patch/4546/) (cf, Withdrawn, 2023-10-10) - judged not relevant; reason summarised from its thread
- P3: [Addition of authenticated ID to pg_stat_activity](https://commitfest.postgresql.org/patch/3095/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P4: [pg_hba_lookup function to get all matching pg_hba.conf entries](https://commitfest.postgresql.org/patch/356/) (cf, Returned with feedback, 2016-04-08) - judged not relevant; reason summarised from its thread
- P5: [Support custom authentication methods using hooks](https://commitfest.postgresql.org/patch/3546/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P6: [pg_hba_file_settings](https://commitfest.postgresql.org/patch/723/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P7: [pg_stat_activity: make slow/hanging authentication more visible](https://commitfest.postgresql.org/patch/4974/) (cf, Withdrawn, 2025-03-21) - judged not relevant; reason summarised from its thread
- P8: [Revert "initdb: Change authentication defaults"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7961886580a594e519ca7ed1811b464206738be5) (revert, Reverted, 2019-07-22) - judged not relevant
- P9: [Rewriting the test of pg_upgrade as a TAP test](https://commitfest.postgresql.org/patch/1101/) (cf, Returned with feedback, 2017-09-29) - judged not relevant; reason summarised from its thread
- P10: [Support multiple password-based authentication tokens concurrently](https://commitfest.postgresql.org/patch/3748/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P11: [Revert "Add TAP test to automate the equivalent of check_guc"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=52377bb81a2c0427d357bba71e7149de02d016ce) (revert, Reverted, 2022-02-12) - used in the review
- P12: [Patch for migration of the pg_commit_ts directory](https://commitfest.postgresql.org/patch/5710/) (cf, Withdrawn, 2025-10-15) - judged not relevant
- P13: [Revert "Cannot use WL_SOCKET_WRITEABLE without WL_SOCKET_READABLE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a58db3aa10e62e4228aa409ba006014fa07a8ca2) (revert, Reverted, 2020-12-15) - judged not relevant
- P14: [Session Start/End hooks](https://commitfest.postgresql.org/patch/1318/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P15: [START/END line number for COPY FROM](https://commitfest.postgresql.org/patch/1940/) (cf, Rejected, 2019-01-30) - judged not relevant
- P16: [Revert 9.5 pgindent changes to atomics directory files](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=befa3e648ce018d84cd2a0df701927c56fe3da4e) (revert, Reverted, 2015-05-25) - judged not relevant
- P17: [delay starting WAL receiver](https://commitfest.postgresql.org/patch/4117/) (cf, Withdrawn, 2023-01-11) - judged not relevant
- P18: [Problem during Windows service start](https://commitfest.postgresql.org/patch/2087/) (cf, Withdrawn, 2019-09-05) - judged not relevant
- P19: [Rewriting the test of pg_upgrade as a TAP test - take three](https://commitfest.postgresql.org/patch/3120/) (cf, Withdrawn, 2021-11-18) - judged not relevant
- P20: [Find additional connection service files in pg_service.conf.d directory](https://commitfest.postgresql.org/patch/1480/) (cf, Returned with feedback, 2018-09-30) - judged not relevant
- P21: [Rewriting the test of pg_upgrade as a TAP test - take two](https://commitfest.postgresql.org/patch/1497/) (cf, Returned with feedback, 2018-04-07) - judged not relevant
- P22: [Create shorthand for including all extra tests](https://commitfest.postgresql.org/patch/4547/) (cf, Rejected, 2024-01-20) - judged not relevant
- P23: [Recognizing superuser in pg_hba.conf](https://commitfest.postgresql.org/patch/2397/) (cf, Returned with feedback, 2020-07-03) - judged not relevant
- P24: [pg_basebackup to adjust existing data directory permissions](https://commitfest.postgresql.org/patch/2018/) (cf, Returned with feedback, 2019-11-27) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 161 s; cost reported by the CLI: $1.80
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 87 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:30:12.044Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
