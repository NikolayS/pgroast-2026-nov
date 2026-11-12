# Design pre-review: BUG #19702: decode() accepts Base64 payload after terminal padding

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7535](https://commitfest.postgresql.org/patch/7535/); [thread](https://www.postgresql.org/message-id/flat/19702-9ed4a131fcfadb9d%40postgresql.org), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The bug report in message #1 states the problem with a reproducible case, and the fix reuses the rule base32hex already applies. The plan in message #3 keeps the stricter rule out of released stable branches, which limits who is affected. Confidence is medium because the code and the callers of pg_b64_decode() were not read.

Not checked: Without the code I could not check the SQLSTATE and message text, the whitespace handling after padding, or whether SCRAM and other pg_b64_decode() callers can receive data after padding. The cost of the added check in the decode loop was not measured in the thread.

## Prior attempts

None found before 2026-09-19.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which callers of pg_b64_decode() exist, such as SCRAM, and can any of them receive data after padding from a client or stored secret that works today?
2. Which parts of the squashed patch go into v19 besides base64url decoding?
3. Which SQLSTATE and message text does the new error use, and do they match the existing base32hex error?
4. Will the release notes list the stricter decode() and dearmor() behaviour as an incompatibility, including the restore failure for expression indexes and check constraints over malformed rows?

## Touch points

- **on-disk format / pg_upgrade:** A SCRAM secret in pg_authid with data after the salt padding could stop parsing after pg_upgrade; Postgres does not generate such secrets.
- **ABI / extension API:** pg_b64_decode() in src/common/base64.c rejects input it used to accept, which changes behaviour for its callers.
- **pg_dump / restore round-trip:** Expression indexes, check constraints or generated columns that call decode(..., 'base64') on malformed rows would fail on restore into the new version.
- **security / privileges:** The patch narrows what decode(), dearmor() and pg_b64_decode() accept, and pg_b64_decode() may serve SCRAM authentication.
- **backpatchability:** The change rejects input that used to pass; the thread plans master only, plus the relevant parts for v19.
- **performance on the hot path:** The patch adds a check to the base64 decode loop in encode.c, used for bytea decoding.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson reviewed the pgcrypto part against RFC 9580, squashed the patches and said in message #3 that he will apply them to master.
- Priority: proceed; audience a sizeable minority; review cost low. The three patches are under 8 KiB in total, touch three decoders plus tests, and add no new user-visible surface. A committer who reviewed the code is ready to apply it, and the open points are release-note and v19 scope details.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
