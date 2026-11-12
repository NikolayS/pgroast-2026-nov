# Design pre-review: Add PAM Tests and Option For Custom PAM Config Location

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7219](https://commitfest.postgresql.org/patch/7219/); [thread](https://www.postgresql.org/message-id/flat/CAKK5BkHsWCMLS4B8N%3D9juny1kCzkoHFXHkesEA%3D-ng4pRAHdUw%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). PAM test coverage is a sound goal and no objection to it survived. The open point is the permanent pg_hba.conf option that the commit message says was "mostly implemented for the sake of testing". already-exists-1 asks why a test-only redirect such as pam_wrapper is not enough, and demand-1 finds no production user asking for pamconfdir. The list should settle that choice before code review.

Not checked: Without the code, it could not be checked whether the ifdef gate covers option parsing in hba.c, so behaviour on builds without pam_start_confdir is unknown. The two level 2 objections that survived (standards-compat-2, standards-compat-3) were not evaluated.

## Prior attempts

None found before 2026-08-29.

## Objections a reviewer is likely to raise

### 1. pam_wrapper already allows rootless PAM tests without a new pg_hba option

Postgres ships pamconfdir as a permanent pg_hba.conf option that exists only on some builds. The new suite runs only through pam_start_confdir, so a regression in the plain pam_start path that production servers use still passes CI. The project then supports a production option that was added mostly for tests.

The pam_wrapper library from the cwrap project preloads into a process and redirects PAM service lookup to a test directory, with no root. A suite built on it would test the unchanged pam_start path, which message #1 accepts as untested. The commit message says the option was "mostly implemented for the sake of testing" and gives non-root DBAs as a secondary use, with no field report. The cost of pam_wrapper is one more optional test dependency and LD_PRELOAD limits on some platforms.

**Question:** Why is a server-side pamconfdir option better than running the tests under pam_wrapper, which also covers the default pam_start path?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAKK5BkHsWCMLS4B8N%3D9juny1kCzkoHFXHkesEA%3D-ng4pRAHdUw%40mail.gmail.com); `already-exists-1`</sub>

### 2. User-facing pamconfdir option justified mainly by test needs

Postgres ships pamconfdir as a permanent pg_hba.conf option, and every later release must document and support it. If few production sites set it, the project keeps a security-relevant knob, chosen per build, that serves mostly its own test suite. A committer would find this out only after removal is no longer possible.

The commit message says the option "was mostly implemented for the sake of testing". Its only production case is DBAs without root, given in one sentence with no field report. Such a DBA still needs a sysadmin to install PAM modules and grant access to credential stores, and a one-time /etc/pam.d/postgresql file covers that today. Message #3 asks for pg_hba_file_rules support, which grows the surface, but no one in the thread asks for the option itself.

**Question:** Which production sites need a non-default PAM directory per pg_hba line, and why can their sysadmin not install a file in /etc/pam.d once?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAKK5BkHsWCMLS4B8N%3D9juny1kCzkoHFXHkesEA%3D-ng4pRAHdUw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAGKCzNCOOFk1%3DpkUfDA0c6k%2B9_Q7C0NwAs%3DnsQ5vkJ_O17RRRA%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Why is a server-side pamconfdir option better than running the tests under pam_wrapper, which also covers the default pam_start path?
2. Which production sites need a non-default PAM directory per pg_hba.conf line, and why can their sysadmin not install a file in /etc/pam.d once?
3. What does a server built without pam_start_confdir do when pg_hba.conf contains pamconfdir: error, warning, or silent ignore?
4. Will the next version show pamconfdir in pg_hba_file_rules.options, as message #3 asks?
5. Does the server check that the pamconfdir directory exists when it loads pg_hba.conf, or does a wrong path show up only at login?

## Touch points

- **catalog changes:** The new option should appear in the pg_hba_file_rules.options column, and message #3 reports that it does not.
- **ABI / extension API:** The patch touches src/include/libpq/hba.h to carry the new pamconfdir value.
- **security / privileges:** pamconfdir changes which directory supplies the PAM policy that decides authentication.
- **maintenance burden ("who maintains this in 10 years"):** A permanent pg_hba.conf option, two build probes and a Linux-only opt-in test suite depend on a linux-pam function.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** pam_wrapper may give rootless PAM tests without a server option, and the thread does not discuss it.

## Who would commit it, and when

- Owner: silent. Someone who knows the client authentication code (auth.c, hba.c) and the configure and meson probes would have to own it. The visible thread has the author and one reviewer.
- Priority: fix design first; audience niche; review cost low. The patch is about 20 KiB and touches auth.c, hba.c and the build files, so review is cheap. The value is test coverage for PAM users, but the choice between a permanent pg_hba.conf option and a test-only redirect should be settled first.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
