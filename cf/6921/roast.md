# Design pre-review: Continuous re-validation of session credentials

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6921](https://commitfest.postgresql.org/patch/6921/); [thread](https://www.postgresql.org/message-id/flat/CAER375PeMJkejmRueUoWjpMwg%2BmsX4HahP6_%3DUFxJyge_boD4Q%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem statement is clear and no objection is fatal, but several level 2 points decide what the feature guarantees. standards-compat-1 shows the baseline check gives rolvaliduntil a new meaning for sessions that never used a password. security-1 shows PGC_SU_BACKEND keeps an opt-out that the v3 docs say must not exist. security-3 and security-2 show that NOLOGIN, long statements and unreachable sources are not covered. The list should agree on these semantics before code review.

Not checked: The code was not read, so timer handling at the idle-wait point, the catalog lookup inside an open transaction and the per-backend CRL re-read are unjudged. The v3 state of 0003 to 0005 is not given.

## Prior attempts

None found before 2026-06-16.

## Objections a reviewer is likely to raise

### 1. Baseline check applies rolvaliduntil to sessions that never used a password

A site moves roles to cert or OAuth login and lets their old passwords expire through VALID UNTIL, which is the documented way to block password login. After the DBA turns on credential_validation_enabled, every cert or OAuth session of those roles gets FATAL at its first check. The DBA finds out from users who keep being disconnected.

Postgres documents VALID UNTIL as the password expiry time, and only password authentication uses rolvaliduntil. The proposal applies it 'to every authenticated session' (message #1, v3-0001 doc hunk), whatever the method. So the feature changes what an existing catalog column means once it is enabled. A committer would likely ask for the baseline check to follow the existing contract, or for the docs to state the new meaning.

**Question:** Why should a past rolvaliduntil end a cert, GSS or OAuth session when the same role can still open a new session with that method today?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAER375PeMJkejmRueUoWjpMwg%2BmsX4HahP6_%3DUFxJyge_boD4Q%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com); `standards-compat-1`</sub>

### 2. PGC_SU_BACKEND lets roles and connections opt out of validation

A superuser runs `alter role all set credential_validation_enabled = off`, or connects with `options='-c credential_validation_enabled=off'`. Validation stops for those sessions even though postgresql.conf says on, and nothing records it. The admin who revoked a role finds out after the old session keeps running.

Message #5 offered PGC_SIGHUP so that only the server config controls the check. Message #6 chose PGC_SU_BACKEND and gave no reason for it. The v3 docs say a session must not be able to disable the check on itself, but PGC_SU_BACKEND still accepts the setting from the startup packet and from per-role or per-database settings. Any role granted SET on the parameter also keeps that opt-out.

**Question:** Why should a session-start or per-role override exist for a revocation control, rather than PGC_SIGHUP set only in the server configuration?

<sub>fixable, level 2, security / privileges; messages [#5](https://www.postgresql.org/message-id/CAKZiRmy9240VWFZV-nth048S86fbk%3DyOvgKJX%2BnLfqNtCGijgQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com); `security-1`</sub>

### 3. Common revocation steps and long statements are not covered

An admin runs `alter role x nologin` to cut off a user, enables the feature and expects the session to end. The baseline check looks only at role existence and rolvaliduntil, so the session stays open. A user whose role expired can also keep a single long `copy ... to stdout` running for hours.

The baseline in the v3-0001 commit message checks role existence and rolvaliduntil only. NOLOGIN is how many sites revoke access, and the proposal does not mention it. The docs promise that a running statement is never interrupted, so the time to cut off access is limited by statement length, not by credential_validation_interval. The feature is sold as revocation, so admins will rely on guarantees it does not give.

**Question:** Will the baseline check rolcanlogin, and how does the proposal bound how long a revoked session can keep reading data inside one statement?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/CAER375PeMJkejmRueUoWjpMwg%2BmsX4HahP6_%3DUFxJyge_boD4Q%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com); `security-3`</sub>

### 4. Behaviour on unreachable LDAP or CRL is undefined or fails open

The directory server is down for a few minutes. Every LDAP session on the cluster then either gets a FATAL error or keeps running unchecked, and the proposal does not say which. With hundreds of backends each binding once per interval, the directory may also throttle or lock out the bind account.

Message #3 has each backend re-bind with ldapbindpasswd and re-run the search on every cycle. The v3-0002 commit message says any CRL lookup failure fails open. So a revocation that depends on an unreadable file or an unreachable server has no effect, and the log may not show it. Message #3 also counts zero search results as "deleted or disabled", but many directories keep disabled accounts visible to the same filter.

**Question:** For each validator, what happens when its source cannot be reached, how is that logged, and what directory load does one bind per backend per interval produce?

<sub>fixable, level 2, security / privileges; messages [#3](https://www.postgresql.org/message-id/CAER375O4pGq8xmhsQG9SoVeh6htAiJHGW-MA_2u5C8tN1Z24%3Dg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com); `security-2`</sub>

### 5. Coverage gaps per auth method stay silent, nothing fails loudly

An administrator enables credential_validation_enabled and believes a revoked certificate or deleted account ends the session. A later auth option or new HBA mode lacks a validator, so only the baseline role check runs. Nobody learns this until a revoked user's session is still open.

Each per-method path depends on a registered validator, and a missing one gives no error. The review in message #5 already found three silent gaps: clientcert=verify-full, logical walsenders and the error path that reset the timer. Message #6 leaves PAM and ssl_crl_dir out, and v3-0002 documents revoked intermediate CAs as not covered. Each later auth change must remember this registry, and a reviewer can easily miss it.

**Question:** How does a session, an administrator or a regression test find out that a session's auth method has no method-specific validator?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#5](https://www.postgresql.org/message-id/CAKZiRmy9240VWFZV-nth048S86fbk%3DyOvgKJX%2BnLfqNtCGijgQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAER375Nw65h70WW8YyGQemxnOwCDMxJUWZaimZ%3Dzyk54U%3DGQ6g%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Why should a past rolvaliduntil end a cert, GSS or OAuth session when the same role can still open a new session with that method?
2. Why are the two settings PGC_SU_BACKEND, which allows startup-packet and per-role overrides, instead of PGC_SIGHUP as the reviewer offered in message #5?
3. Will the baseline check also cover rolcanlogin, and what bounds the time a revoked session can keep running inside one long statement?
4. For each validator, what happens when the CRL file or the LDAP server cannot be reached, and how does the server log it?
5. Which deployments or compliance rules need open sessions ended when credentials lapse, and what do those sites do today?

## Touch points

- **WAL / replication / standby:** v3-0001 touches walsender.c so that logical walsenders are validated, and the check also runs in standby sessions.
- **catalog changes:** No catalog change, but the baseline check applies pg_authid.rolvaliduntil to sessions of every auth method.
- **ABI / extension API:** 0003 adds expire_cb to OAuthValidatorCallbacks with V1 and V2 magic; 0001 adds RegisterCredentialValidator().
- **new GUC:** Adds credential_validation_enabled and credential_validation_interval, both PGC_SU_BACKEND in v3.
- **security / privileges:** The feature ends sessions with FATAL when role, certificate, token, GSS context or LDAP account is no longer valid.
- **maintenance burden ("who maintains this in 10 years"):** Five per-method validators and a registry that every later auth change must keep in step.
- **performance on the hot path:** A per-backend timer and a check at command boundaries in postgres.c; the LDAP validator binds and searches once per backend per interval.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the authentication code, including TLS, OAuth validators, GSS and LDAP. Daniel Gustafsson commented on the TLS test fixtures only; nobody has taken on the design in the visible thread.
- Priority: fix design first; audience unclear; review cost high. Message #1 names short-lived-credential sites but shows no field report, so the audience is not established. The series spans postgres.c, walsender, TLS, OAuth, GSS and LDAP, and its guarantees are not yet settled.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
