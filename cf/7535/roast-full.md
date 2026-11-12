# Design pre-review: BUG #19702: decode() accepts Base64 payload after terminal padding

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7535](https://commitfest.postgresql.org/patch/7535/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [BUG #19702: decode() accepts Base64 payload after terminal padding](https://www.postgresql.org/message-id/flat/19702-9ed4a131fcfadb9d%40postgresql.org), started 2026-09-19
- Messages read: 5 of 5
- Patch prose read: v1-0002-Add-tests-for-data-after-base64-padding.patch, v1-0001-Reject-data-after-padding-in-base64-decoding.patch, v1-0003-Reject-data-after-base64-padding-in-other-decoder.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The bug report in message #1 states the problem with a reproducible case, and the fix reuses the rule base32hex already applies. The plan in message #3 keeps the stricter rule out of released stable branches, which limits who is affected. Confidence is medium because the code and the callers of pg_b64_decode() were not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code I could not check the SQLSTATE and message text, the whitespace handling after padding, or whether SCRAM and other pg_b64_decode() callers can receive data after padding. The cost of the added check in the decode loop was not measured in the thread.

## The proposal, restated

**Problem.** decode(..., 'base64') keeps reading after the terminal "=" padding and adds later characters to the result, so 'YQ==AAAA' returns \x6100 with no error. Applications that use decode() to validate base64 input accept malformed input. pg_b64_decode() in src/common and dearmor() in pgcrypto share the same logic.

**Proposed design.** The decoders raise an error for anything except whitespace after the padding, the rule base32hex already uses. The fix covers decode() in encode.c, pg_b64_decode() in src/common and the armor decoder in pgcrypto, with regression tests. Daniel Gustafsson squashed the three patches and plans master only, plus the relevant parts for v19.

**User-visible surface.**
- decode(text, 'base64') now errors on characters other than whitespace after terminal padding.
- base64url decoding (new in v19) follows the same rule, per the v19 backpatch plan.
- pgcrypto dearmor() rejects armor with data after base64 padding, even when the CRC matches.
- pg_b64_decode() in src/common rejects the same input, which affects its callers.

**What the author claims.**
- The decoder sets its end marker at the first "=" and never checks it again, so later data and later "=" pass.
- The new rule matches what base32hex already does.
- The change rejects input that used to pass, so back branches are left to the committer.
- dearmor() shows the same bug when the CRC matches.

**Thread state.** Daniel Gustafsson reviewed, squashed the patches and will commit to master only; Masahiko Sawada agreed and asked for v19 too, since base64url is new there. Gustafsson agreed to prepare a v19 backpatch of the relevant portions.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-19; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | A SCRAM secret in pg_authid with data after the salt padding could stop parsing after pg_upgrade; Postgres does not generate such secrets. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | pg_b64_decode() in src/common/base64.c rejects input it used to accept, which changes behaviour for its callers. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Expression indexes, check constraints or generated columns that call decode(..., 'base64') on malformed rows would fail on restore into the new version. |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch narrows what decode(), dearmor() and pg_b64_decode() accept, and pg_b64_decode() may serve SCRAM authentication. |
| backpatchability | **yes** | The change rejects input that used to pass; the thread plans master only, plus the relevant parts for v19. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | The patch adds a check to the base64 decode loop in encode.c, used for bytea decoding. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which callers of pg_b64_decode() exist, such as SCRAM, and can any of them receive data after padding from a client or stored secret that works today?
2. Which parts of the squashed patch go into v19 besides base64url decoding?
3. Which SQLSTATE and message text does the new error use, and do they match the existing base32hex error?
4. Will the release notes list the stricter decode() and dearmor() behaviour as an incompatibility, including the restore failure for expression indexes and check constraints over malformed rows?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson reviewed the pgcrypto part against RFC 9580, squashed the patches and said in message #3 that he will apply them to master.
- **Liability:** The owner signs for any report of input that used to decode and now fails, in decode(), dearmor() and callers of pg_b64_decode(), from the first release that carries the change.
- **Author follow-up:** The author posted the three patches on the day of the bug report; no sign either way about later fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The three patches are under 8 KiB in total, touch three decoders plus tests, and add no new user-visible surface. A committer who reviewed the code is ready to apply it, and the open points are release-note and v19 scope details.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] reverted hex-code overflow checks because they slowed bytea decoding, and [P7] added encoding checks to pgcrypto decryption. Neither concerns trailing data after base64 padding. Message #4 cites an earlier base64url truncation fix backpatched to v19 as a model, not as a rejected attempt.
- standards-compat: 0 objection(s). No standards or compatibility objection holds up. The patch makes decode(..., 'base64') follow RFC 4648, uses the base32hex rule already in the tree, and goes to master only, so no stable branch changes behaviour; v19 is still in RC and base64url is new there. The one break is input such as 'YQ==Yg==', which used to decode and will now fail; that needs a release-note incompatibility entry, a routine step at release time. Without the code, I could not check the SQLSTATE and message text, or whether SCRAM and other pg_b64_decode() callers can receive data after padding from clients that worked before.
- replication-ondisk: 0 objection(s). The fix changes only how base64 input is checked when it is parsed, so it touches no WAL, page format, catalog, wire message or exported struct. Two edge cases are worth a release-note line but are not objections. Expression indexes, check constraints or stored generated columns that call decode(..., 'base64') on malformed rows would fail when pg_dump output is restored into the new version. A SCRAM secret in pg_authid with junk after the salt padding could stop parsing after pg_upgrade, but Postgres never generates such secrets.
- security: 0 objection(s). From the security angle the change only rejects malformed base64 that used to pass, which narrows what decode(), pgcrypto dearmor() and pg_b64_decode() accept; it adds no privilege, exposure or resource path. Without the code I could not check whether SCRAM callers of pg_b64_decode() and the new error text leak nothing beyond what current errors already show.
- maintenance-fit: 0 objection(s). From the maintenance angle this is a small correctness fix that uses the base32hex rule already in core, adds no GUC, catalog, API or dependency, and adds no measurable cost on common paths. Two committers who know the code reviewed it. A remaining risk is pg_b64_decode() callers such as SCRAM verifiers stored in pg_authid by external tools, but that is a compatibility question for another angle.
- already-exists: 0 objection(s). No existing Postgres feature or extension gives strict base64 checking in decode(), so nothing makes this fix redundant. The patch extends the existing decoders and copies the existing base32hex rule, so it adds no second mechanism. Neither precedent applies: [P1] is a performance revert of hex refactoring, and [P7] is about pgcrypto encoding checks.
- demand: 0 objection(s). This is a correctness fix for a filed bug report (message #1), not a new feature, so the demand test does not apply. Two committers reviewed it and agreed (messages #3 to #5), and keeping it to master and v19 limits who pays for the stricter rule.

### Precedents retrieved

- P1: [Revert refactoring of hex code to src/common/](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2576dcfb76aa71e4222bac5a3a43f71875bfa9e8) (revert, Reverted, 2021-08-19) - used in the review
- P2: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P3: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P4: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant; reason summarised from its thread
- P5: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant; reason summarised from its thread
- P6: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P7: [Encoding protection for pgcrypto](https://commitfest.postgresql.org/patch/4809/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P8: [pg_basebackup has stricter tablespace rules than the server](https://commitfest.postgresql.org/patch/1125/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P9: [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P10: [Implement TODOs for index operator strategy number validation](https://commitfest.postgresql.org/patch/7152/) (cf, Withdrawn, 2026-09-09) - judged not relevant; reason summarised from its thread
- P11: [COPY: validate option presence rather than option values](https://commitfest.postgresql.org/patch/6744/) (cf, Withdrawn, 2026-06-29) - judged not relevant; reason summarised from its thread
- P12: [logical decoding and replication of sequences, take 2](https://commitfest.postgresql.org/patch/3823/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P13: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2283/) (cf, Withdrawn, 2020-06-11) - judged not relevant; reason summarised from its thread
- P14: [Logical decoding of two-phase transactions](https://commitfest.postgresql.org/patch/944/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P15: [always use runtime checks for CRC-32C instructions](https://commitfest.postgresql.org/patch/4634/) (cf, Withdrawn, 2023-10-31) - judged not relevant
- P16: [Revert "Reject degenerate SPLIT PARTITION with DEFAULT partition"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0392fb900eb89f52988cccd33046443c39c70d1c) (revert, Reverted, 2026-05-20) - judged not relevant
- P17: [Removing unused trailing linepointers from heap pages](https://commitfest.postgresql.org/patch/3035/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P18: [CRC32C Parallel Computation Optimization on ARM](https://commitfest.postgresql.org/patch/4620/) (cf, Returned with feedback, 2025-04-02) - judged not relevant
- P19: [CREATE DATABASE ... STRICT_UNICODE](https://commitfest.postgresql.org/patch/4876/) (cf, Withdrawn, 2024-12-11) - judged not relevant
- P20: [bytea PL/Perl transform](https://commitfest.postgresql.org/patch/4252/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P21: [invalid bit/varbit input causes the log file to be unreadable](https://commitfest.postgresql.org/patch/2618/) (cf, Withdrawn, 2020-06-30) - judged not relevant
- P22: [pgbench - implement strict TPC-B benchmark](https://commitfest.postgresql.org/patch/2090/) (cf, Rejected, 2019-11-06) - judged not relevant
- P23: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant
- P24: [Revert "Add missing padding from MemoryChunk struct"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5495796ad12aae971137d184ff25add8f6b03968) (revert, Reverted, 2022-08-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 63 s; cost reported by the CLI: $0.66
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 45 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:06:30.832Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
