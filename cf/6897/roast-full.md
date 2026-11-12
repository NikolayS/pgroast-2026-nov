# Design pre-review: Require SSL connection to postgres for oauth

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6897](https://commitfest.postgresql.org/patch/6897/) (Needs review); authors: Zsolt Parragi (zsolt.parragi)
- Thread: [Require SSL connection to postgres for oauth](https://www.postgresql.org/message-id/flat/CAN4CZFOa0iA3PV%3DoE0%2BO%3DjdZyu_KtS9vGLA%2B4a7hjAhfzVLAJg%40mail.gmail.com), started 2026-06-16
- Messages read: 9 of 9
- Patch prose read: v3-0001-doc-warn-that-OAuth-over-plaintext-connections-is.patch, v3-0002-libpq-require-encrypted-connections-for-OAUTHBEAR.patch, v3-0003-Warn-when-OAuth-is-configured-on-plaintext-capabl.patch, 0001-libpq-require-encrypted-connections-for-OAUTHBEARER.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread, and the doc patch 0001 raises no objection. The client-side check in 0002 fails at level 2: already-exists-1 and standards-compat-1 show an OAuth-only transport rule whose only opt-out is a debug flag, against setups message #6 calls common. security-1 and standards-compat-2 are open design points on 0002 and 0003. The list has to agree on the policy scope and the opt-out before code review of 0002 makes sense.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the behavior under sslmode=prefer fallback, the exact warning text, and how pg_hba_file_rules() reports the warning could not be checked. Only 3 of 9 messages were visible; later messages are known through the objections and the brief.

## The proposal, restated

**Problem.** libpq sends the OAuth bearer token to the server over plaintext TCP without warning, though RFC 7628 says TLS MUST protect it. Clients can leak a token by accident, and the docs warn about clear-text passwords but say nothing about OAuth.

**Proposed design.** Three patches. 0001 adds doc warnings that OAuth over unencrypted connections exposes the token. 0002 makes libpq refuse the OAUTHBEARER exchange unless the connection uses SSL/TLS, GSSAPI encryption or a Unix-domain socket, with an unsafe debug flag to restore the old behavior. 0003 makes the server warn when an oauth HBA entry allows plaintext connections.

**User-visible surface.**
- libpq refuses OAUTHBEARER over unencrypted TCP connections by default (0002).
- New PGOAUTHDEBUG option UNSAFE:plaintext-server; the legacy PGOAUTHDEBUG=UNSAFE enables it implicitly.
- Server WARNING from HBA parsing for oauth entries of type host, hostnossl or hostnogss, at start, reload and in pg_hba_file_rules() (0003).
- New warnings about plaintext OAuth in client-auth.sgml and protocol.sgml (0001).

**What the author claims.**
- RFC 7628 and RFC 6750 require TLS to protect bearer tokens, unlike SCRAM, which only recommends it.
- The libpq default is inconsistent: HTTPS to the identity provider needs an unsafe flag to bypass, the server connection does not.
- Unix-domain sockets are local and so a safe exception.
- 0003 improves safety without breaking any use case.
- The doc patch alone, or with the server warning, helps all supported versions.

**Thread state.** Jacob Champion and Jonathan Gonzalez raised compatibility concerns about non-TLS setups in Kubernetes and tunnels, and Daniel Gustafsson plans to commit a reworked doc patch for master through 18. The client-side change is deferred to v20 at the earliest, and the server warning has no review yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-06-16; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. 0002 breaks working plaintext OAuth setups, and the only opt-out is a debug flag

fixable · level 2, rubric item 12 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A team reaches Postgres through kubectl port-forward or a service-mesh tunnel without TLS. After a distribution upgrades libpq, every OAuth login fails, even though the server did not change. The only fix is setting PGOAUTHDEBUG=UNSAFE:plaintext-server in production, which the docs describe as a debug option for local testing.

Message #6 reports that plaintext OAuth over Kubernetes and cloud port-forwards is common and sometimes required, because TLS through those tunnels gets reset. Message #4 objects that a transport default for one authentication type makes users track per-method rules, and asks for one policy for all methods. Message #7 defers the change to v20, but the opt-out design is still the debug flag. The author's other ideas, an explicit sslmode=disable or the file-based configuration from message #2, are not part of the patch.

Thread: [#2 (Jacob Champion, 2026-06-16)](https://www.postgresql.org/message-id/CAOYmi%2BkR89L5LHQ61xH1qORKfmFtqkL6mjk6n%2Bv9N-UShZJRYA%40mail.gmail.com), [#4 (Jacob Champion, 2026-06-17)](https://www.postgresql.org/message-id/CAOYmi%2BmyssXQASD5maXjedo-76wBC_EC1Ntfypto6UjOYdBODA%40mail.gmail.com), [#6 ("Jonathan Gonzalez V(dot)", 2026-06-21)](https://www.postgresql.org/message-id/87mrwnyfeh.fsf%40gmail.com), [#7 (Daniel Gustafsson, 2026-11-06)](https://www.postgresql.org/message-id/01CFD582-9679-4010-A4F5-033D2ED9F6BD%40yesql.se)

**Question:** What supported, non-debug setting will a production user in a non-TLS tunnel use, and how does it fit a policy that later covers SCRAM and password too?

Counter-check: held. Message #6 describes non-TLS tunnels as common, and the 0002 commit message limits the opt-out to "local development and testing". Message #2 doubts these users should need a debug flag, and no later patch version changes that.

### 2. OAuth-only transport check duplicates sslmode and require_auth

fixable · level 2, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A DBA hardens a cluster on the assumption that libpq now refuses plaintext for credentials. SCRAM and password connections from the same client still go over plaintext TCP, because only OAUTHBEARER has the new check. A user with a tunnel setup has to move between sslmode, require_auth and a PGOAUTHDEBUG flag to get either the old behavior or a strict one.

Clients can already refuse plaintext with sslmode=require or verify-full, combined with require_auth=oauth, and servers can already enforce it with hostssl entries. 0002 adds a second, OAuth-only way to get the same result, and its opt-out is a debug flag. In message #4 Jacob argues that per-mechanism transport defaults are "probably not a net win". In message #2 he names file-based libpq configuration as the general place for a stricter default. The author's answer, that the RFC requires TLS for OAUTHBEARER, explains why the default should change but not why the change should be OAuth-only.

Thread: [#2 (Jacob Champion, 2026-06-16)](https://www.postgresql.org/message-id/CAOYmi%2BkR89L5LHQ61xH1qORKfmFtqkL6mjk6n%2Bv9N-UShZJRYA%40mail.gmail.com), [#4 (Jacob Champion, 2026-06-17)](https://www.postgresql.org/message-id/CAOYmi%2BmyssXQASD5maXjedo-76wBC_EC1Ntfypto6UjOYdBODA%40mail.gmail.com), [#7 (Daniel Gustafsson, 2026-11-06)](https://www.postgresql.org/message-id/01CFD582-9679-4010-A4F5-033D2ED9F6BD%40yesql.se)

**Question:** Why should OAuth get its own transport check instead of a stricter default in sslmode or require_auth that covers every authentication method?

Counter-check: held. Message #4 argues that per-method transport defaults are probably not a net win, and the thread does not answer that. The RFC argument explains a stricter default but does not explain why only OAUTHBEARER gets it.

### 3. Client check accepts TLS without server certificate verification

fixable · level 2, rubric item 7 · security / privileges · `security-1`

A user connects with sslmode=require, as the proposed error message would suggest. An active attacker on the path presents its own certificate, libpq accepts it because the connection is encrypted, and the bearer token is sent to the attacker. The attacker can replay the token against the real server, and nobody notices.

The 0002 commit message cites RFC 6750 Sec. 5 but accepts any SSL/TLS connection, and the RFC protection assumes the client verifies the server certificate. OAUTHBEARER has no channel binding (protocol.sgml), so an unauthenticated TLS peer can capture a token and replay it. Message #3 proposes an error hint of "consider sslmode=require", which points users to a mode that does not authenticate the server. Message #4 notes that sslmode defaults are the real gap. A check that blocks passive sniffing but lets an active MITM through may give users more confidence than the design supports.

Thread: [#3 (Zsolt Parragi, 2026-06-17)](https://www.postgresql.org/message-id/CAN4CZFNFR8HkPS0gZtacX9HTBBxTVMk3xFhmS%3DY9Bnys76jakw%40mail.gmail.com), [#4 (Jacob Champion, 2026-06-17)](https://www.postgresql.org/message-id/CAOYmi%2BmyssXQASD5maXjedo-76wBC_EC1Ntfypto6UjOYdBODA%40mail.gmail.com)

**Question:** Will the check and its error message require or recommend sslmode=verify-full or sslrootcert=system, and if not, why is unauthenticated TLS enough for a replayable token?

Counter-check: held. 0002 accepts any SSL/TLS connection, and sslmode=require does not verify the server certificate. The patch does not make this worse than today, but it cites the RFCs as its basis, so the level of protection it requires is a fair design question.

### 4. 0003 warns about oauth HBA lines only, with no way to silence it

fixable · level 2, rubric item 12 · psql UX conventions, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A DBA whose clients all set sslmode=require keeps a plain host oauth line. The DBA gets a WARNING at every start and reload, and monitoring that queries pg_hba_file_rules() gets one too. A host password line on the same server produces no warning, which suggests that password over plaintext is safer than OAuth.

The patch adds warnings for oauth only, and the author's question in message #5 about extending them to password and ident got no answer. pg_hba_file_rules() reports problems in its error column, so a WARNING raised during a SELECT works differently from that. Users whose plaintext setup is intended, as in message #6, have no setting to turn the warning off. The claim in message #8 that 0003 breaks no use case does not cover log noise or what tools see.

Thread: [#5 (Zsolt Parragi, 2026-06-18)](https://www.postgresql.org/message-id/CAN4CZFNy23wWKF%2B78QG3cfunjJpQWFc7eNP7qcYPB6Sg-NhTQQ%40mail.gmail.com), [#6 ("Jonathan Gonzalez V(dot)", 2026-06-21)](https://www.postgresql.org/message-id/87mrwnyfeh.fsf%40gmail.com), [#8 (Zsolt Parragi, 2026-11-06)](https://www.postgresql.org/message-id/CAN4CZFNnPhmd3-oLKqU87ktCsBNM9u3j6RDxdfbrnqYonThjow%40mail.gmail.com)

**Question:** Why should only oauth entries warn, and how does a DBA with an intended plaintext setup stop the warning on every reload?

Counter-check: held. A DBA whose clients all use TLS can switch to hostssl, but an intended plaintext setup has no way to silence the warning. The oauth-only scope and the WARNING raised from pg_hba_file_rules() are open; message #5's question got no answer.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | libpq refuses the OAUTHBEARER SASL exchange on unencrypted TCP; no message format changes. |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: 0002 breaks working plaintext OAuth setups, and the only opt-out is a debug flag |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | Every libpq client, psql included, gets a new connection failure for OAuth over plaintext TCP. |
| security / privileges | **yes** | 0002 changes when libpq sends a bearer token; 0003 warns about oauth HBA entries that allow plaintext. |
| backpatchability | **yes** | The author wanted the fix in earlier branches; 0002 changes default behavior, so only the doc patch is planned for 18. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new PGOAUTHDEBUG option and an OAuth-only transport rule may overlap with later file-based libpq configuration. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | sslmode, require_auth and hostssl entries already let clients and servers refuse plaintext OAuth. |

## 5. Questions the author must answer before anyone reviews code

1. What supported, non-debug setting will a production user on a non-TLS tunnel use to keep OAuth working after 0002?
2. Why should libpq enforce encryption for OAUTHBEARER alone, instead of a stricter default that covers SCRAM and password too?
3. Does the client check accept sslmode=require without server certificate verification, and if so, why is that enough for a replayable token?
4. How does 0002 treat sslmode=prefer or allow when the connection falls back to plaintext?
5. Why does 0003 warn only for oauth entries, and how does a DBA with an intended plaintext setup silence it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Jacob Champion and Daniel Gustafsson take part in the thread; the brief says Daniel Gustafsson plans to commit a reworked doc patch. 0002 and 0003 need someone who knows libpq authentication and HBA parsing.
- **Liability:** For 0002, a default behavior change in libpq that breaks existing non-TLS OAuth setups, plus a PGOAUTHDEBUG option that must stay supported once production users depend on it. For 0003, a recurring server WARNING that users will ask to silence.
- **Author follow-up:** The author posted v2 and v3 in response to review and moved the entry to a later CommitFest; the entry has needed a rebase since 2026-11-07.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patches are small and touch only libpq OAuth code, hba.c and docs, but 0002 changes a default for OAuth users and its opt-out is a debug flag. The doc patch can proceed alone; 0002 and 0003 need agreement on policy scope and opt-out first.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Client-side OAuth-only TLS rule repeats a design choice made on purpose. Refuted (not specific): Duplicate of already-exists-1. The prior-rejection framing is also weak: message #2 says "I wouldn't mind enforcing this if there's consensus", and message #7 hopes to raise the bar in v20.
- `maintenance-fit-1` OAuth-only transport rule pushes production users onto a debug flag. Refuted (not specific): Duplicate of standards-compat-1. The point about legacy PGOAUTHDEBUG=UNSAFE enabling the other unsafe options is a useful detail, but it supports the same objection about the debug-flag opt-out.
- `maintenance-fit-2` HBA warning fires on ordinary host entries and cannot be silenced. Refuted (not specific): Duplicate of standards-compat-2. The added scenario depends on an extension to password and ident that message #5 only asks about and no patch contains.
- `demand-1` Client-side refusal rests on one in-house report, with a workaround available. Refuted (already handled): Two committers who know the area say they want stricter transport defaults (messages #4 and #7), so interest goes beyond one report. hostssl and sslmode=require do not protect a client user who is unaware of the risk, and the breakage scenario duplicates standards-compat-1.
- `demand-2` OAuth-only HBA warning lands on intentional plaintext setups. Refuted (not specific): Duplicate of standards-compat-2. Message #2 also supports a server-side warning regardless of the client change, which weakens the desirability framing.

### Review angles

- prior-attempts: 1 objection(s). [P23] proposed a warning about unencrypted passwords and was withdrawn with no recorded reason, so it is not listed as a prior attempt. The doc patch 0001 is moving toward commit, and the server warning 0003 has no recorded earlier objection.
- standards-compat: 2 objection(s). The 0001 doc warnings raise no standards or compatibility concern, and a committer plans to apply them.
- replication-ondisk: 0 objection(s). The proposal changes no persistent or replicated state: no on-disk format, WAL record, catalog, wire message or extension ABI. The client check and the HBA warning act at authentication time only, and messages #2, #6 and #7 already raised the tunnel and pooler compatibility concern that led to deferring 0002.
- security: 1 objection(s). The proposal hardens security, and the server warning in 0003 exposes nothing new to unprivileged users. The one gap is that encrypted is treated as safe even when the server is not authenticated.
- maintenance-fit: 2 objection(s). No objection to the doc patch 0001, which a committer has reworked and plans to commit.
- already-exists: 1 objection(s). 0001 (docs) and 0003 (HBA warning) do not duplicate an existing mechanism, so this angle raises nothing against them.
- demand: 2 objection(s). The doc patch (0001) has committer support and real value, so there is no demand objection to it. [P23] proposed a similar password warning and was withdrawn, but its reason is not recorded, so it is not cited.

### Precedents retrieved

- P1: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant; reason summarised from its thread
- P2: [Add sslmode "no-clientcert" to avoid auth failure in md5/scram connections](https://commitfest.postgresql.org/patch/4064/) (cf, Withdrawn, 2023-02-21) - judged not relevant; reason summarised from its thread
- P3: [OAuth Followup Patches](https://commitfest.postgresql.org/patch/5635/) (cf, Withdrawn, 2025-06-26) - judged not relevant; reason summarised from its thread
- P4: [Revert "Only provide new libpq sslpasskey hook for openssl-enabled builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6741cfa5964768279a13e01e46931664fe0903a4) (revert, Reverted, 2020-04-17) - judged not relevant
- P5: [Improve OAuth discovery logging \[accidental duplicate\]](https://commitfest.postgresql.org/patch/6536/) (cf, Withdrawn, 2026-02-26) - judged not relevant; reason summarised from its thread
- P6: [pg_hba_lookup function to get all matching pg_hba.conf entries](https://commitfest.postgresql.org/patch/356/) (cf, Returned with feedback, 2016-04-08) - judged not relevant; reason summarised from its thread
- P7: [Add inline comments to the pg_hba_file_rules view](https://commitfest.postgresql.org/patch/4546/) (cf, Withdrawn, 2023-10-10) - judged not relevant; reason summarised from its thread
- P8: [Revert "oauth: Add unit tests for multiplexer handling"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebaaf386adb133010c2024256521b993c5e53e98) (revert, Reverted, 2025-08-08) - judged not relevant
- P9: [pg_hba_file_settings](https://commitfest.postgresql.org/patch/723/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P10: [Fix warnings under ENFORCE_REGRESSION_TEST_NAME_RESTRICTIONS](https://commitfest.postgresql.org/patch/4451/) (cf, Rejected, 2023-07-19) - judged not relevant; reason summarised from its thread
- P11: [Support multiple password-based authentication tokens concurrently](https://commitfest.postgresql.org/patch/3748/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [Relax requirement for INTO with SELECT in pl/pgsql](https://commitfest.postgresql.org/patch/586/) (cf, Returned with feedback, 2016-09-22) - judged not relevant
- P13: [Add radiustimeout parameter for RADIUS HBA](https://commitfest.postgresql.org/patch/846/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P14: [create index failed. PARALLEL UNSAFE function does not prevent parallel index build](https://commitfest.postgresql.org/patch/4860/) (cf, Rejected, 2024-03-13) - judged not relevant
- P15: [Revert "Add support for Kerberos credential delegation"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3d03b24c350ab060bb223623bdff38835bd7afd0) (revert, Reverted, 2023-04-08) - judged not relevant
- P16: [Support tls-exporter as channel binding for TLSv1.3](https://commitfest.postgresql.org/patch/3850/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P17: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P18: [Use role name "system_user" instead of "user" for unsafe_tests](https://commitfest.postgresql.org/patch/4271/) (cf, Withdrawn, 2023-04-13) - judged not relevant
- P19: [Support for NSS as TLS backend](https://commitfest.postgresql.org/patch/2592/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P20: [Revert changes for SSL compression in libpq](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0ba71107efeeccde9158f47118f95043afdca0bb) (revert, Reverted, 2021-03-10) - judged not relevant
- P21: [Data at rest encryption](https://commitfest.postgresql.org/patch/2104/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P22: [Support for NSS as a libpq TLS backend](https://commitfest.postgresql.org/patch/3138/) (cf, Rejected, 2022-02-23) - judged not relevant
- P23: [Add a warning message when using unencrypted passwords](https://commitfest.postgresql.org/patch/5426/) (cf, Withdrawn, 2025-02-27) - used in the review
- P24: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 155 s; cost reported by the CLI: $2.01
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 92 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:06:57.908Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
