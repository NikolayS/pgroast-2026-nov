# Design pre-review: Require SSL connection to postgres for oauth

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6897](https://commitfest.postgresql.org/patch/6897/); [thread](https://www.postgresql.org/message-id/flat/CAN4CZFOa0iA3PV%3DoE0%2BO%3DjdZyu_KtS9vGLA%2B4a7hjAhfzVLAJg%40mail.gmail.com), 9 of 9 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread, and the doc patch 0001 raises no objection. The client-side check in 0002 fails at level 2: already-exists-1 and standards-compat-1 show an OAuth-only transport rule whose only opt-out is a debug flag, against setups message #6 calls common. security-1 and standards-compat-2 are open design points on 0002 and 0003. The list has to agree on the policy scope and the opt-out before code review of 0002 makes sense.

Not checked: The code was not read, so the behavior under sslmode=prefer fallback, the exact warning text, and how pg_hba_file_rules() reports the warning could not be checked. Only 3 of 9 messages were visible; later messages are known through the objections and the brief.

## Prior attempts

None found before 2026-06-16.

## Objections a reviewer is likely to raise

### 1. 0002 breaks working plaintext OAuth setups, and the only opt-out is a debug flag

A team reaches Postgres through kubectl port-forward or a service-mesh tunnel without TLS. After a distribution upgrades libpq, every OAuth login fails, even though the server did not change. The only fix is setting PGOAUTHDEBUG=UNSAFE:plaintext-server in production, which the docs describe as a debug option for local testing.

Message #6 reports that plaintext OAuth over Kubernetes and cloud port-forwards is common and sometimes required, because TLS through those tunnels gets reset. Message #4 objects that a transport default for one authentication type makes users track per-method rules, and asks for one policy for all methods. Message #7 defers the change to v20, but the opt-out design is still the debug flag. The author's other ideas, an explicit sslmode=disable or the file-based configuration from message #2, are not part of the patch.

**Question:** What supported, non-debug setting will a production user in a non-TLS tunnel use, and how does it fit a policy that later covers SCRAM and password too?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/CAOYmi%2BkR89L5LHQ61xH1qORKfmFtqkL6mjk6n%2Bv9N-UShZJRYA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAOYmi%2BmyssXQASD5maXjedo-76wBC_EC1Ntfypto6UjOYdBODA%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/87mrwnyfeh.fsf%40gmail.com), [#7](https://www.postgresql.org/message-id/01CFD582-9679-4010-A4F5-033D2ED9F6BD%40yesql.se); `standards-compat-1`</sub>

### 2. OAuth-only transport check duplicates sslmode and require_auth

A DBA hardens a cluster on the assumption that libpq now refuses plaintext for credentials. SCRAM and password connections from the same client still go over plaintext TCP, because only OAUTHBEARER has the new check. A user with a tunnel setup has to move between sslmode, require_auth and a PGOAUTHDEBUG flag to get either the old behavior or a strict one.

Clients can already refuse plaintext with sslmode=require or verify-full, combined with require_auth=oauth, and servers can already enforce it with hostssl entries. 0002 adds a second, OAuth-only way to get the same result, and its opt-out is a debug flag. In message #4 Jacob argues that per-mechanism transport defaults are "probably not a net win". In message #2 he names file-based libpq configuration as the general place for a stricter default. The author's answer, that the RFC requires TLS for OAUTHBEARER, explains why the default should change but not why the change should be OAuth-only.

**Question:** Why should OAuth get its own transport check instead of a stricter default in sslmode or require_auth that covers every authentication method?

<sub>fixable, level 2, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#2](https://www.postgresql.org/message-id/CAOYmi%2BkR89L5LHQ61xH1qORKfmFtqkL6mjk6n%2Bv9N-UShZJRYA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAOYmi%2BmyssXQASD5maXjedo-76wBC_EC1Ntfypto6UjOYdBODA%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/01CFD582-9679-4010-A4F5-033D2ED9F6BD%40yesql.se); `already-exists-1`</sub>

### 3. Client check accepts TLS without server certificate verification

A user connects with sslmode=require, as the proposed error message would suggest. An active attacker on the path presents its own certificate, libpq accepts it because the connection is encrypted, and the bearer token is sent to the attacker. The attacker can replay the token against the real server, and nobody notices.

The 0002 commit message cites RFC 6750 Sec. 5 but accepts any SSL/TLS connection, and the RFC protection assumes the client verifies the server certificate. OAUTHBEARER has no channel binding (protocol.sgml), so an unauthenticated TLS peer can capture a token and replay it. Message #3 proposes an error hint of "consider sslmode=require", which points users to a mode that does not authenticate the server. Message #4 notes that sslmode defaults are the real gap. A check that blocks passive sniffing but lets an active MITM through may give users more confidence than the design supports.

**Question:** Will the check and its error message require or recommend sslmode=verify-full or sslrootcert=system, and if not, why is unauthenticated TLS enough for a replayable token?

<sub>fixable, level 2, security / privileges; messages [#3](https://www.postgresql.org/message-id/CAN4CZFNFR8HkPS0gZtacX9HTBBxTVMk3xFhmS%3DY9Bnys76jakw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAOYmi%2BmyssXQASD5maXjedo-76wBC_EC1Ntfypto6UjOYdBODA%40mail.gmail.com); `security-1`</sub>

### 4. 0003 warns about oauth HBA lines only, with no way to silence it

A DBA whose clients all set sslmode=require keeps a plain host oauth line. The DBA gets a WARNING at every start and reload, and monitoring that queries pg_hba_file_rules() gets one too. A host password line on the same server produces no warning, which suggests that password over plaintext is safer than OAuth.

The patch adds warnings for oauth only, and the author's question in message #5 about extending them to password and ident got no answer. pg_hba_file_rules() reports problems in its error column, so a WARNING raised during a SELECT works differently from that. Users whose plaintext setup is intended, as in message #6, have no setting to turn the warning off. The claim in message #8 that 0003 breaks no use case does not cover log noise or what tools see.

**Question:** Why should only oauth entries warn, and how does a DBA with an intended plaintext setup stop the warning on every reload?

<sub>fixable, level 2, psql UX conventions; messages [#5](https://www.postgresql.org/message-id/CAN4CZFNy23wWKF%2B78QG3cfunjJpQWFc7eNP7qcYPB6Sg-NhTQQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/87mrwnyfeh.fsf%40gmail.com), [#8](https://www.postgresql.org/message-id/CAN4CZFNnPhmd3-oLKqU87ktCsBNM9u3j6RDxdfbrnqYonThjow%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. What supported, non-debug setting will a production user on a non-TLS tunnel use to keep OAuth working after 0002?
2. Why should libpq enforce encryption for OAUTHBEARER alone, instead of a stricter default that covers SCRAM and password too?
3. Does the client check accept sslmode=require without server certificate verification, and if so, why is that enough for a replayable token?
4. How does 0002 treat sslmode=prefer or allow when the connection falls back to plaintext?
5. Why does 0003 warn only for oauth entries, and how does a DBA with an intended plaintext setup silence it?

## Touch points

- **wire protocol:** libpq refuses the OAUTHBEARER SASL exchange on unencrypted TCP; no message format changes.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: 0002 breaks working plaintext OAuth setups, and the only opt-out is a debug flag
- **psql UX conventions:** Every libpq client, psql included, gets a new connection failure for OAuth over plaintext TCP.
- **security / privileges:** 0002 changes when libpq sends a bearer token; 0003 warns about oauth HBA entries that allow plaintext.
- **backpatchability:** The author wanted the fix in earlier branches; 0002 changes default behavior, so only the doc patch is planned for 18.
- **maintenance burden ("who maintains this in 10 years"):** A new PGOAUTHDEBUG option and an OAuth-only transport rule may overlap with later file-based libpq configuration.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** sslmode, require_auth and hostssl entries already let clients and servers refuse plaintext OAuth.

## Who would commit it, and when

- Owner: qualified committer engaged. Jacob Champion and Daniel Gustafsson take part in the thread; the brief says Daniel Gustafsson plans to commit a reworked doc patch. 0002 and 0003 need someone who knows libpq authentication and HBA parsing.
- Priority: fix design first; audience a sizeable minority; review cost low. The patches are small and touch only libpq OAuth code, hba.c and docs, but 0002 changes a default for OAuth users and its opt-out is a debug flag. The doc patch can proceed alone; 0002 and 0003 need agreement on policy scope and opt-out first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
