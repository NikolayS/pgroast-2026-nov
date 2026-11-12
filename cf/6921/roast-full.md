# Design pre-review: Continuous re-validation of session credentials

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6921](https://commitfest.postgresql.org/patch/6921/) (Needs review); authors: ajitpostgres awekar (ajit.awekar)
- Thread: [Continuous re-validation of session credentials](https://www.postgresql.org/message-id/flat/CAER375PeMJkejmRueUoWjpMwg%2BmsX4HahP6_%3DUFxJyge_boD4Q%40mail.gmail.com), started 2026-06-16
- Messages read: 8 of 8
- Patch prose read: v3-0001-Add-continuous-credential-validation-framework.patch, v3-0002-Add-TLS-client-certificate-expiry-and-revocation-che.patch, 0002-Add-TLS-client-certificate-expiry-to-credential-vali.patch, 0003-Add-OAuth-token-expiry-to-credential-validation.patch, 0001-Add-continuous-credential-validation-framework.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem statement is clear and no objection is fatal, but several level 2 points decide what the feature guarantees. standards-compat-1 shows the baseline check gives rolvaliduntil a new meaning for sessions that never used a password. security-1 shows PGC_SU_BACKEND keeps an opt-out that the v3 docs say must not exist. security-3 and security-2 show that NOLOGIN, long statements and unreachable sources are not covered. The list should agree on these semantics before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so timer handling at the idle-wait point, the catalog lookup inside an open transaction and the per-backend CRL re-read are unjudged. The v3 state of 0003 to 0005 is not given.

## The proposal, restated

**Problem.** Postgres checks credentials only at connection time, so a session keeps running after its role is dropped or passes VALID UNTIL, or after its OAuth token or client certificate expires. Revoking access has no effect on sessions that are already open. The author targets sites that use short-lived credentials such as OAuth tokens, short-lived certificates and time-boxed accounts.

**Proposed design.** A per-backend timer fires every credential_validation_interval seconds; the check runs at command boundaries and at the idle-wait point, and the backend exits with FATAL when credentials are no longer valid. A baseline check covers role existence and rolvaliduntil for every session, and validators registered through RegisterCredentialValidator() add checks per method: certificate expiry and CRL, OAuth expiry, GSS context lifetime, and an LDAP re-search.

**User-visible surface.**
- GUC credential_validation_enabled (bool, default off), PGC_SU_BACKEND in v3, so it cannot be changed within a session.
- GUC credential_validation_interval (5 to 3600 s, default 60), PGC_SU_BACKEND in v3.
- FATAL "session credentials have expired" with a reconnect HINT, and errdetail_log() naming the validator and the user.
- OAuth validator ABI: optional expire_cb in OAuthValidatorCallbacks, with versioned magic PG_OAUTH_VALIDATOR_MAGIC_V1 and V2.
- C API RegisterCredentialValidator() and the CredentialValidationType enum for new auth methods.

**What the author claims.**
- Method validators for certificates and GSS check locally, with no round trip to the network or the KDC.
- New auth methods can be added in three small steps, without changes to core.
- The OAuth change stays ABI-compatible because the server accepts both V1 and V2 validator modules.
- Test runtime fell from about 57 s and 23 s to about 22 to 30 s, with no shorter timeouts.
- v3 fixes all issues from the first review: error resets of the timer, superuser SET, logical walsenders, idle sessions and clientcert=verify-full.

**Thread state.** Jakub Wartak reviewed 0001 and 0002; v3 posted only those two patches, which fail to build when applied one at a time and fail the Windows cfbot. Daniel Gustafsson agreed that the TLS test should use the fixtures in src/test/ssl/conf and sslfiles.mk; nobody has reviewed 0003 to 0005.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-06-16; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Baseline check applies rolvaliduntil to sessions that never used a password

fixable · level 2, rubric item 12 · design, catalog changes · `standards-compat-1`

A site moves roles to cert or OAuth login and lets their old passwords expire through VALID UNTIL, which is the documented way to block password login. After the DBA turns on credential_validation_enabled, every cert or OAuth session of those roles gets FATAL at its first check. The DBA finds out from users who keep being disconnected.

Postgres documents VALID UNTIL as the password expiry time, and only password authentication uses rolvaliduntil. The proposal applies it 'to every authenticated session' (message #1, v3-0001 doc hunk), whatever the method. So the feature changes what an existing catalog column means once it is enabled. A committer would likely ask for the baseline check to follow the existing contract, or for the docs to state the new meaning.

Thread: [#1 (Ajit Awekar, 2026-06-16)](https://www.postgresql.org/message-id/CAER375PeMJkejmRueUoWjpMwg%2BmsX4HahP6_%3DUFxJyge_boD4Q%40mail.gmail.com), [#6 (Ajit Awekar, 2026-09-29)](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com)

**Question:** Why should a past rolvaliduntil end a cert, GSS or OAuth session when the same role can still open a new session with that method today?

Counter-check: held. Postgres documents VALID UNTIL as password expiry and checks it only in password authentication. The v3 doc hunk applies it to every session, and the thread does not discuss this change of meaning.

### 2. PGC_SU_BACKEND lets roles and connections opt out of validation

fixable · level 2, rubric item 7 · security / privileges, new GUC · `security-1`

A superuser runs `alter role all set credential_validation_enabled = off`, or connects with `options='-c credential_validation_enabled=off'`. Validation stops for those sessions even though postgresql.conf says on, and nothing records it. The admin who revoked a role finds out after the old session keeps running.

Message #5 offered PGC_SIGHUP so that only the server config controls the check. Message #6 chose PGC_SU_BACKEND and gave no reason for it. The v3 docs say a session must not be able to disable the check on itself, but PGC_SU_BACKEND still accepts the setting from the startup packet and from per-role or per-database settings. Any role granted SET on the parameter also keeps that opt-out.

Thread: [#5 (Jakub Wartak, 2026-09-24)](https://www.postgresql.org/message-id/CAKZiRmy9240VWFZV-nth048S86fbk%3DyOvgKJX%2BnLfqNtCGijgQ%40mail.gmail.com), [#6 (Ajit Awekar, 2026-09-29)](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com)

**Question:** Why should a session-start or per-role override exist for a revocation control, rather than PGC_SIGHUP set only in the server configuration?

Counter-check: held. PGC_SU_BACKEND accepts values from the startup packet and from per-role or per-database settings, so the opt-out exists. Message #6 does not say why PGC_SIGHUP, which the reviewer offered, was not chosen.

### 3. Common revocation steps and long statements are not covered

fixable · level 2, rubric item 7 · security / privileges · `security-3`

An admin runs `alter role x nologin` to cut off a user, enables the feature and expects the session to end. The baseline check looks only at role existence and rolvaliduntil, so the session stays open. A user whose role expired can also keep a single long `copy ... to stdout` running for hours.

The baseline in the v3-0001 commit message checks role existence and rolvaliduntil only. NOLOGIN is how many sites revoke access, and the proposal does not mention it. The docs promise that a running statement is never interrupted, so the time to cut off access is limited by statement length, not by credential_validation_interval. The feature is sold as revocation, so admins will rely on guarantees it does not give.

Thread: [#1 (Ajit Awekar, 2026-06-16)](https://www.postgresql.org/message-id/CAER375PeMJkejmRueUoWjpMwg%2BmsX4HahP6_%3DUFxJyge_boD4Q%40mail.gmail.com), [#6 (Ajit Awekar, 2026-09-29)](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com)

**Question:** Will the baseline check rolcanlogin, and how does the proposal bound how long a revoked session can keep reading data inside one statement?

Counter-check: held. The baseline in v3-0001 checks only role existence and rolvaliduntil, yet a NOLOGIN role is rejected on a fresh connection. The statement-length limit is documented, but the proposal gives no bound for it.

### 4. Behaviour on unreachable LDAP or CRL is undefined or fails open

fixable · level 2, rubric item 7 · security / privileges, maintenance burden ("who maintains this in 10 years") · `security-2`

The directory server is down for a few minutes. Every LDAP session on the cluster then either gets a FATAL error or keeps running unchecked, and the proposal does not say which. With hundreds of backends each binding once per interval, the directory may also throttle or lock out the bind account.

Message #3 has each backend re-bind with ldapbindpasswd and re-run the search on every cycle. The v3-0002 commit message says any CRL lookup failure fails open. So a revocation that depends on an unreadable file or an unreachable server has no effect, and the log may not show it. Message #3 also counts zero search results as "deleted or disabled", but many directories keep disabled accounts visible to the same filter.

Thread: [#3 (Ajit Awekar, 2026-07-01)](https://www.postgresql.org/message-id/CAER375O4pGq8xmhsQG9SoVeh6htAiJHGW-MA_2u5C8tN1Z24%3Dg%40mail.gmail.com), [#6 (Ajit Awekar, 2026-09-29)](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com)

**Question:** For each validator, what happens when its source cannot be reached, how is that logged, and what directory load does one bind per backend per interval produce?

Counter-check: held. The v3-0002 commit message confirms the CRL lookup fails open. Message #3 gives no rule for an unreachable directory and no cost for one bind per backend per interval.

### 5. Coverage gaps per auth method stay silent, nothing fails loudly

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), security / privileges · `maintenance-fit-2`

An administrator enables credential_validation_enabled and believes a revoked certificate or deleted account ends the session. A later auth option or new HBA mode lacks a validator, so only the baseline role check runs. Nobody learns this until a revoked user's session is still open.

Each per-method path depends on a registered validator, and a missing one gives no error. The review in message #5 already found three silent gaps: clientcert=verify-full, logical walsenders and the error path that reset the timer. Message #6 leaves PAM and ssl_crl_dir out, and v3-0002 documents revoked intermediate CAs as not covered. Each later auth change must remember this registry, and a reviewer can easily miss it.

Thread: [#5 (Jakub Wartak, 2026-09-24)](https://www.postgresql.org/message-id/CAKZiRmy9240VWFZV-nth048S86fbk%3DyOvgKJX%2BnLfqNtCGijgQ%40mail.gmail.com), [#6 (Ajit Awekar, 2026-09-29)](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com)

**Question:** How does a session, an administrator or a regression test find out that a session's auth method has no method-specific validator?

Counter-check: held. The review in message #5 found several silent gaps, and message #6 leaves PAM and ssl_crl_dir uncovered. The proposal has no way to tell an administrator that a session's method has no validator.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | v3-0001 touches walsender.c so that logical walsenders are validated, and the check also runs in standby sessions. |
| wire protocol | no |  |
| catalog changes | **yes** | No catalog change, but the baseline check applies pg_authid.rolvaliduntil to sessions of every auth method. |
| ABI / extension API | **yes** | 0003 adds expire_cb to OAuthValidatorCallbacks with V1 and V2 magic; 0001 adds RegisterCredentialValidator(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Adds credential_validation_enabled and credential_validation_interval, both PGC_SU_BACKEND in v3. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The feature ends sessions with FATAL when role, certificate, token, GSS context or LDAP account is no longer valid. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Five per-method validators and a registry that every later auth change must keep in step. |
| performance on the hot path | **yes** | A per-backend timer and a check at command boundaries in postgres.c; the LDAP validator binds and searches once per backend per interval. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why should a past rolvaliduntil end a cert, GSS or OAuth session when the same role can still open a new session with that method?
2. Why are the two settings PGC_SU_BACKEND, which allows startup-packet and per-role overrides, instead of PGC_SIGHUP as the reviewer offered in message #5?
3. Will the baseline check also cover rolcanlogin, and what bounds the time a revoked session can keep running inside one long statement?
4. For each validator, what happens when the CRL file or the LDAP server cannot be reached, and how does the server log it?
5. Which deployments or compliance rules need open sessions ended when credentials lapse, and what do those sites do today?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the authentication code, including TLS, OAuth validators, GSS and LDAP. Daniel Gustafsson commented on the TLS test fixtures only; nobody has taken on the design in the visible thread.
- **Liability:** The owner signs for a security guarantee across five auth methods, an OAuth validator ABI change and a registry that later auth work must keep updated. That lasts as long as the feature exists.
- **Author follow-up:** The author posted v2 and v3 within weeks of feedback and moved the entry to the next CommitFest. v3 carries only 0001 and 0002.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** unclear
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Message #1 names short-lived-credential sites but shows no field report, so the audience is not established. The series spans postgres.c, walsender, TLS, OAuth, GSS and LDAP, and its guarantees are not yet settled.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-3` (fixable, level 2) Versioned OAuth validator magic commits the ABI for one callback. Not listed: not among the strongest objections.
- `demand-1` (fixable, level 1b) No users outside the thread have asked for mid-session revalidation. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-2` PGC_SU_BACKEND still lets a superuser turn off the check. Refuted (not specific): Duplicate of security-1, which makes the same PGC_SU_BACKEND point with the same startup-packet and per-role examples.
- `replication-ondisk-1` Revocation on the primary reaches standby sessions only after WAL replay. Refuted (no failure scenario): A delayed or paused standby also accepts a fresh login by the dropped role until replay applies the change. The session therefore holds credentials the standby still accepts, which matches the proposal's stated rule; what remains is a documentation note.
- `maintenance-fit-1` LDAP and CRL checks add external I/O per backend per interval. Refuted (not specific): Duplicate of security-2, which already asks about directory load and unreachable sources. The load falls only on sites that enable the feature with LDAP, so rubric item 8 (cost for non-users) does not apply.
- `already-exists-1` Baseline role check repeats what a scheduled pg_terminate_backend() query does. Refuted (already handled): The timer, GUCs and registry exist for the certificate, OAuth and GSS checks, which the objection agrees an outside job cannot do. The baseline check is a small addition to that framework, and 0001 is the infrastructure-first step of the series.
- `demand-2` The baseline role check has a working external workaround. Refuted (already handled): Same point as already-exists-1, and it fails for the same reason: the cron query cannot see certificate, token or GSS state. Its remaining question about demand is covered by demand-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only listed row, [P22], checks whether the client socket is still alive during a query, which is a different idea, and it has no recorded reason. No message in the thread points to an earlier discussion of session credential re-validation, so this angle gives no grounds to object.
- standards-compat: 2 objection(s). Two objections. The OAuth magic versioning in 0003 follows the existing practice of versioned module magic, and FATAL at the idle-wait point matches idle_session_timeout, so I raise neither.
- replication-ondisk: 1 objection(s). The series adds no on-disk format, WAL record or catalog change. The OAuth V1/V2 magic keeps existing validator modules loadable. Physical walsender exemption was accepted in message #5.
- security: 3 objection(s). Three fixable objections. The feature is security-positive. The weak points are the opt-out path, fail-open behaviour, and gaps in what counts as revocation.
- maintenance-fit: 3 objection(s). The baseline check in 0001 has a small per-backend cost and is useful on its own. These objections are about the method-specific layers.
- already-exists: 1 objection(s). The certificate, OAuth and GSS checks need state that only the backend has, and I know of no existing feature or extension that provides them. So the angle raises an objection only against 0001 standing alone.
- demand: 2 objection(s). The problem is real and the statement is clear. The feature is off by default, so its cost falls mostly on its users. The objections ask for evidence of demand rather than rejection.

### Precedents retrieved

- P1: [Session Start/End hooks](https://commitfest.postgresql.org/patch/1318/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P2: [TRUNCATE behavior when session_replication_role = replica](https://commitfest.postgresql.org/patch/1447/) (cf, Rejected, 2018-04-07) - judged not relevant; reason summarised from its thread
- P3: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant; reason summarised from its thread
- P4: [background sessions](https://commitfest.postgresql.org/patch/873/) (cf, Returned with feedback, 2017-04-06) - judged not relevant
- P5: [Support multiple password-based authentication tokens concurrently](https://commitfest.postgresql.org/patch/3748/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P6: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1148/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P7: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1568/) (cf, Returned with feedback, 2018-03-01) - judged not relevant; reason summarised from its thread
- P8: [Implement the SQL-standard mechanism whereby REVOKE ROLE revokes only the privilege granted by the invoking role, and not those granted by other roles](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Administration) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [Log query parameters for terminated execute](https://commitfest.postgresql.org/patch/1693/) (cf, Rejected, 2018-07-30) - judged not relevant; reason summarised from its thread
- P10: [Revert "Add support for Kerberos credential delegation"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3d03b24c350ab060bb223623bdff38835bd7afd0) (revert, Reverted, 2023-04-08) - judged not relevant
- P11: [OAuth Followup Patches](https://commitfest.postgresql.org/patch/5635/) (cf, Withdrawn, 2025-06-26) - judged not relevant; reason summarised from its thread
- P12: [Allow cluster owner to bypass authentication](https://commitfest.postgresql.org/patch/2244/) (cf, Returned with feedback, 2020-04-06) - judged not relevant
- P13: [Multiple passwords, interval expirations](https://commitfest.postgresql.org/patch/4432/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P14: [ldapbindpasswdfile](https://commitfest.postgresql.org/patch/2117/) (cf, Withdrawn, 2019-07-08) - judged not relevant
- P15: [Support pg_ident mapping for LDAP](https://commitfest.postgresql.org/patch/3314/) (cf, Withdrawn, 2022-08-17) - judged not relevant
- P16: [Client Connection redirection support for PostgreSQL](https://commitfest.postgresql.org/patch/1673/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P17: [Improve OAuth discovery logging \[accidental duplicate\]](https://commitfest.postgresql.org/patch/6536/) (cf, Withdrawn, 2026-02-26) - judged not relevant
- P18: [CREATE ROLE IF NOT EXISTS](https://commitfest.postgresql.org/patch/3370/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P19: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - judged not relevant
- P20: [Revert b2a459edf "Fix GRANTED BY support in REVOKE ROLE statements"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e68570e388f08c2e36ce7d2a9564941b89db6549) (revert, Reverted, 2021-12-30) - judged not relevant
- P21: [Addition of authenticated ID to pg_stat_activity](https://commitfest.postgresql.org/patch/3095/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P22: [Add client connection check during the execution of the query](https://commitfest.postgresql.org/patch/1882/) (cf, Returned with feedback, 2019-08-01) - used in the review
- P23: [Support custom authentication methods using hooks](https://commitfest.postgresql.org/patch/3546/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P24: [Revert "oauth: Add unit tests for multiplexer handling"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebaaf386adb133010c2024256521b993c5e53e98) (revert, Reverted, 2025-08-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 193 s; cost reported by the CLI: $2.71
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 110 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:09:18.433Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
