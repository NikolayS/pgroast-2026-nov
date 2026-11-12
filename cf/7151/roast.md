# Design pre-review: Let OpenSSL auto-select DH parameters by default

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7151](https://commitfest.postgresql.org/patch/7151/); [thread](https://www.postgresql.org/message-id/flat/953bceb7-340d-4717-9682-0ebe65c4b57e%40eisentraut.org), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). One objection survived, security-1, and it is fixable. A server with a key below 2048 bits and an empty ssl_dh_params_file loses the 2048-bit DH floor on upgrade, with no warning or release note. The goal and the design are otherwise sound, and no level fails. Someone who knows the OpenSSL code must confirm the library behavior and the version coverage.

Not checked: Without the code, nobody can confirm that a failed DH file load on reload keeps the old SSL context, or how the patch behaves on LibreSSL. The sizes OpenSSL picks for common key types were also not checked.

## Prior attempts

None found before 2026-08-11.

## Objections a reviewer is likely to raise

### 1. Silent drop below 2048-bit DH for servers with small keys

A site runs an old 1024-bit RSA certificate and upgrades Postgres. With the empty default, SSL_CTX_set_dh_auto() picks DH parameters matched to that key, so TLS 1.2 DHE sessions use a smaller group than the old 2048-bit built-in one. Nothing in the server log or the docs tells the administrator.

Message #1 accepts that small keys can now give DH parameters below the old default, and calls that the user's own fault. Today the compiled-in parameters set a 2048-bit floor for DHE sessions, whatever the certificate size. The patch removes that floor on upgrade, with no setting change, no warning and no planned release note. OpenSSL security levels may reject such keys on some builds, but the proposal does not rely on that or say which builds do.

**Question:** Why not keep a 2048-bit minimum, for example by logging a warning or refusing auto-selected groups below it, and will the release notes mention the change?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/953bceb7-340d-4717-9682-0ebe65c4b57e%40eisentraut.org); `security-1`</sub>

## Questions for the author

1. Why not keep a 2048-bit minimum for auto-selected DH parameters, or log a warning when the selected group is smaller?
2. What DH parameter sizes does SSL_CTX_set_dh_auto() pick for RSA 2048, RSA 3072 and ECDSA P-256 server keys?
3. Which OpenSSL and LibreSSL versions first provide SSL_CTX_set_dh_auto(), and are they at or below the minimum versions Postgres supports?
4. Is the reload behavior change for an unreadable ssl_dh_params_file meant for back-patching, and will it get a release note?
5. Does any known extension use the FILE_DH2048 macro from libpq-be.h?

## Touch points

- **ABI / extension API:** The patch removes the FILE_DH2048 macro from src/include/libpq/libpq-be.h, a backend header that extensions can include.
- **new GUC:** No new GUC, but the empty default of ssl_dh_params_file changes meaning and the docs mark the setting obsolescent.
- **security / privileges:** The patch changes which DH parameters TLS 1.2 DHE sessions use, and small keys can get groups below 2048 bits.
- **backpatchability:** Message #1 calls the old reload fallback a bug but does not say whether that fix is meant for back branches.
- **maintenance burden ("who maintains this in 10 years"):** The patch removes compiled-in parameters that had no update process and makes the server depend on SSL_CTX_set_dh_auto() in OpenSSL and LibreSSL.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Eisentraut, a committer per the review notes, wrote the patch. A second reviewer who knows the OpenSSL backend code and LibreSSL differences has not spoken in the thread.
- Priority: proceed; audience most users; review cost low. The change affects every server that leaves ssl_dh_params_file empty, which is the default, and it removes compiled-in parameters that nobody knew how to update. The patch is about 9 KiB in one backend file plus docs, with one fixable point (security-1) to settle.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
