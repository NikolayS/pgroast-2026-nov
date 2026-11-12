# Design pre-review: Reject unassigned bytes in single-byte encodings

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7542](https://commitfest.postgresql.org/patch/7542/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [Reject unassigned bytes in single-byte encodings](https://www.postgresql.org/message-id/flat/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com), started 2026-10-26
- Messages read: 1 of 1
- Patch prose read: v1-0002-Add-tests-for-unassigned-bytes-in-single-byte-enc.patch, v1-0001-Reject-unassigned-bytes-in-single-byte-encodings.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is reasonable, and the design questions rest on facts the author states in message #1. Existing data stops restoring and pg_upgrade keeps it with no check (replication-ondisk-1). Input that works today starts to error (standards-compat-2). The mapping alternative is left open (standards-compat-3), and COPY slows by about 20% (maintenance-fit-1). The list needs to agree on reject versus map, and on the upgrade path, before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can judge how the reject lists are built, whether the apply worker validates streamed changes, or which paths besides verifystr, chr() and the LATIN2 to WIN1250 table still let these bytes in.

## The proposal, restated

**Problem.** Thirteen single-byte encodings, WIN1252 among them, accept bytes that have no character, and these bytes then fail to convert to any other encoding. Bug #14197 is one report. The author cites Tom saying the real fix for an earlier psql problem was the WIN1252 validator.

**Proposed design.** The verifystr functions reject the unassigned bytes, taken from each *_to_utf8.map file. chr() now validates its result, since it skipped the validator. The LATIN2 to WIN1250 table no longer maps five bytes into WIN1250 holes, so those conversions now fail.

**User-visible surface.**
- Input in 13 encodings (ISO_8859_6, ISO_8859_7, ISO_8859_8, LATIN3, WIN874, WIN1250-1255, WIN1257, WIN1258) now errors on unassigned bytes, for example 0x81 in WIN1252.
- chr() errors when the result is an unassigned byte in the database encoding, for example chr(129) in WIN1252.
- LATIN2 to WIN1250 conversion fails for five bytes it used to map into WIN1250 holes.
- A dump of a database that holds such bytes fails on restore at COPY for that table.

**What the author claims.**
- The unassigned bytes are accepted today and then fail to convert, as in bug #14197.
- COPY of 1M 60-byte rows: no change for ASCII only, about 20% slower for WIN1252 with 50% non-ASCII, about 23% slower for ISO 8859-6 all non-ASCII.
- UTF8 databases do not pay for the extra check.
- pg_upgrade keeps existing bytes as they are, and the author suggests it may need a check for them.
- Mapping holes to characters (0x81 to U+0081) costs nothing but breaks match with Unicode.org files and fits only 5 of 13 encodings.

**Thread state.** Only the first message is visible, and nobody has replied. The author leaves open a pg_upgrade check and the alternative of mapping holes to code points.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. pg_upgrade carries forward bytes that dump and restore will then reject

fixable · level 2, rubric item 14 · on-disk format / pg_upgrade, pg_dump / restore round-trip · `replication-ondisk-1`

A WIN1252 cluster holds 0x81 in a text column, perhaps from earlier LATIN2 to WIN1250 conversions. The admin runs pg_upgrade to the new release, which succeeds. Months later a pg_dump restore or a migration fails at COPY, or at a function body or column default that holds the byte, long after the old cluster is gone.

Message #1 says pg_upgrade keeps the bytes and the restore fails, and it leaves a pg_upgrade check as a maybe. Without a check or a scan tool, the upgraded cluster holds data the server itself now calls invalid, and nothing tells the user until a restore. The same bytes in catalogs, such as function bodies, defaults or comments, break the schema part of the restore. That is a larger failure than the per-table COPY failure the message describes.

Thread: [#1 (shihao zhong, 2026-10-26)](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com)

**Question:** What will pg_upgrade --check, or a shipped query, report for unassigned bytes in user tables and catalogs before the upgrade commits the user to the new rules?

Counter-check: held. Message #1 itself says pg_upgrade keeps the bytes and restore fails, and it leaves the pg_upgrade check as a maybe. The proposal names no way to find affected rows or catalog text before upgrade.

### 2. Mapping holes to C1 code points is dismissed too fast

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax · `standards-compat-3`

After the patch, a WIN1252 user with 0x81 in the data has no conversion path left, so the data cannot be read through a UTF8 client and cannot be moved to a UTF8 database. With a hole-to-C1 mapping for the five 0x80-0x9F encodings, the same data would convert with no runtime cost and no input break.

Message #1 rejects the mapping because the maps would no longer match the Unicode.org files, and because it covers only 5 of 13 encodings. Common Windows converters map these holes to U+0081 and similar code points, so a mixed design may fit what clients send better. Using the mapping for 5 encodings and rejection for the other 8 would shrink the compatibility break.

Thread: [#1 (shihao zhong, 2026-10-26)](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com)

**Question:** Why is matching Unicode.org files worth more than readable old data, and why not map the 0x80-0x9F holes and reject only the rest?

Counter-check: held. Message #1 gives two short reasons against mapping but does not weigh a mixed design, and it leaves the question open. The scenario's first sentence describes today's behaviour, so only the comparison with the alternative stands.

### 3. About 20% slower COPY for every user of 13 single-byte encodings

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-1`

A site with a WIN1252 database loads data with COPY and never converts it to another encoding. After upgrade its bulk loads of mostly non-ASCII text run about 20% slower. Its data was never at risk from the bug being fixed.

Message #1 measures +20% for WIN1252 at 50% non-ASCII and +23% for ISO 8859-6. The failure in bug #14197 only appears when bytes are converted to another encoding, yet every input in these encodings pays the check. A committer would likely ask for a check that costs no more than the existing per-byte loop, such as a 256-entry lookup per encoding. The proposal does not show that the cost was weighed against that option.

Thread: [#1 (shihao zhong, 2026-10-26)](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com)

**Question:** Can the check run as a per-encoding 256-byte table lookup with no measurable COPY cost, and what do the numbers look like then?

Counter-check: held. The 20% and 23% figures are the author's own, and the cost falls on all non-ASCII input in these encodings. Message #1 does not say whether a cheaper form of the check was tried.

### 4. Logical replication from an older publisher stops on unassigned bytes

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-2`

An admin upgrades a WIN1252 database to the new release by logical replication from the old server. The old publisher sends a row that holds 0x81. The new subscriber's apply worker rejects the text, errors, restarts and fails on the same change again, so replication stalls until someone finds and fixes the row on the publisher.

Logical replication is a common way to upgrade across major versions, and the subscriber validates incoming text in its own encoding. The old publisher accepts the bytes, so the change is a rule mismatch between the two nodes. Message #1 does not cover replication or mixed-version setups. This is the same pre-upgrade detection gap as the pg_upgrade case, but here it shows up as a stuck slot and growing WAL on the publisher.

Thread: [#1 (shihao zhong, 2026-10-26)](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com)

**Question:** How should a user upgrading a WIN1252 database by logical replication find and clean unassigned bytes on the old publisher before the subscriber stops on them?

Counter-check: held. The initial table sync uses COPY, which validates input, so a row with 0x81 would stop the sync. Whether the apply worker also validates streamed changes cannot be confirmed without the code, but the gap in message #1 is real.

### 5. Client input that works today errors after upgrade

fixable · level 1b, rubric item 6 · desirability, SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-2`

An application with client_encoding=WIN1252, or one that calls chr(129) in a WIN1252 database, stores these bytes and reads them back unchanged. After upgrade, its INSERT, COPY or chr() call raises an error. Nothing in the application changed, and the failure comes only when such a byte appears in production data.

Bug #14197 shows that conversion fails later. That harms users who convert, but users who store and read back in the same encoding see no failure today. The patch moves the error from conversion to input for every user, including users who never convert. Message #1 gives no evidence that this second group is small, and it does not say whether the change targets master only.

Thread: [#1 (shihao zhong, 2026-10-26)](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com)

**Question:** What evidence shows that users who store these bytes and never convert them are few enough to justify breaking their input path?

Counter-check: held. Message #1 confirms that input and chr() now raise errors, and it shows no evidence about users who store these bytes and never convert them. It also does not say how such users learn of the change.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade keeps existing unassigned bytes that the new server rejects on input. |
| WAL / replication / standby | **yes** | A new subscriber may reject rows with unassigned bytes sent by an older publisher. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-3: Mapping holes to C1 code points is dismissed too fast |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Message #1 says a dump of a database with such bytes fails on restore at COPY. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, but the change breaks input, chr() and one conversion; the proposal targets master only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Reject lists for 13 encodings in wchar.c must stay equal to the *_to_utf8.map files. |
| performance on the hot path | **yes** | The author measures COPY about 20% to 23% slower for non-ASCII input in these encodings. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What will pg_upgrade --check, or a shipped query, report for unassigned bytes in user tables and catalog text before the upgrade?
2. Why not map the 0x80-0x9F holes to code points for the five encodings where that works, and reject only in the other eight?
3. What evidence shows that few users store these bytes and read them back in the same encoding without conversion?
4. Can the check be a 256-entry lookup per encoding inside the existing per-byte loop, and what are the COPY numbers then?
5. How does a user who upgrades by logical replication find and fix unassigned bytes on the old publisher before the subscriber stops?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the encoding verifiers and conversion maps, and who is willing to own a compatibility break in pg_upgrade and restore behaviour. Nobody has replied to message #1.
- **Liability:** The owner signs for restore failures, input errors and replication stalls on existing data in 13 encodings, for as long as old clusters with such bytes are upgraded.
- **Author follow-up:** No sign either way; the thread has only the first message.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patch is small and affects only databases in 13 single-byte encodings. The open choice between reject and map, and the missing pg_upgrade check, need an answer before line-by-line review is useful.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Existing data stops round-tripping through dump, restore and logical replication. Refuted (not specific): Duplicate of replication-ondisk-1 for the pg_upgrade and restore part, and of replication-ondisk-2 for the logical replication part. Those two state each case more exactly.
- `maintenance-fit-2` Existing data becomes a delayed dump/restore failure. Refuted (not specific): Duplicate of replication-ondisk-1. [P1] is about out-of-bounds reads in text functions and was rejected as unnecessary, so it does not support this point.
- `maintenance-fit-3` Hand-copied reject lists can drift from the map files. Refuted (no failure scenario): The *_to_utf8.map files for these encodings come from Unicode.org tables that are frozen, so the drift scenario needs a change nobody expects. How the reject lists are built cannot be judged without the code.
- `demand-1` One bug report set against a break for every database in 13 encodings. Refuted (not specific): Duplicate of standards-compat-2, which asks the same question about users who store these bytes on purpose. The entry is tagged as a bug fix, so a count of users asking for it fits poorly as a separate test.
- `demand-2` Every user of these encodings pays about 20% on COPY. Refuted (not specific): Duplicate of maintenance-fit-1, which uses the same numbers and adds a concrete request for a cheaper check.

### Review angles

- prior-attempts: 0 objection(s). No precedent shows this idea rejected before. Message #1 cites an earlier thread where Tom called the WIN1252 validator the real fix, but that thread is not in the material. [P1] repaired one text function after bad bytes got in; this patch rejects them at input, so P1's reason does not apply. [P9]'s rejection reason is unknown.
- standards-compat: 3 objection(s). Objections found; see list.
- replication-ondisk: 2 objection(s). The patch does not change any on-disk format, WAL record, catalog or wire message. Both objections are about existing data that the new validation rules no longer accept.
- security: 0 objection(s). No security objection. The patch narrows what the server accepts: verifystr, chr() and one conversion table reject bytes that no other encoding can represent, and nothing here adds a privilege, a new capability, or new information exposure. The restore failure for old data and the COPY slowdown are compatibility and performance questions for other angles. The extra check is linear per byte, so it gives an unprivileged user no new way to cause a denial of service.
- maintenance-fit: 3 objection(s). Not applicable.
- already-exists: 0 objection(s). No existing feature, extension or hook does what the patch does. The input verifier and chr() are inside core, and an extension cannot tighten them. Per-column workarounds such as a check constraint that calls convert_to() do not cover client input or chr(), and choosing a UTF8 database does not help existing databases. The patch also extends the existing verifystr functions and adds no second mechanism.
- demand: 2 objection(s). The problem is real but shown by one bug report. Both objections ask for evidence of demand, so the fitting outcome is "not now, show demand", not a rejection.

### Precedents retrieved

- P1: [Bug: Reading from single byte character column type may cause out of bounds memory reads](https://commitfest.postgresql.org/patch/3767/) (cf, Returned with feedback, 2022-09-08) - used in the review; reason summarised from its thread
- P2: [addFkRecurseReferencing use unassigned fkconstraint->fk_with_period value](https://commitfest.postgresql.org/patch/7359/) (cf, Withdrawn, 2026-10-02) - judged not relevant; reason summarised from its thread
- P3: [Revert COPY OUT to follow the pre-8.3 handling of ASCII control characters, namely that \r, \n, \t, \b, \f, \v are dumped as those two-character...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a63b63ff96293e153d25e8e054a830d70f69938a) (revert, Reverted, 2007-12-03) - judged not relevant
- P4: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant
- P5: [Add test module for verifying backtrace functionality](https://commitfest.postgresql.org/patch/4823/) (cf, Withdrawn, 2024-08-24) - judged not relevant; reason summarised from its thread
- P6: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P7: [Allow regex operations in PL/Perl using UTF8 characters in non-UTF8 encoded databases](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Server-Side_Languages) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant; reason summarised from its thread
- P9: [Encoding protection for pgcrypto](https://commitfest.postgresql.org/patch/4809/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P10: [Allow pg_verify_checksums to specify a database to scan](https://commitfest.postgresql.org/patch/1782/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P11: [Clarify the comments about varlena header encoding](https://commitfest.postgresql.org/patch/3820/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P12: [Add LSN <-> time conversion facility](https://commitfest.postgresql.org/patch/4754/) (cf, Withdrawn, 2024-10-22) - judged not relevant
- P13: [Revert "Reject degenerate SPLIT PARTITION with DEFAULT partition"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0392fb900eb89f52988cccd33046443c39c70d1c) (revert, Reverted, 2026-05-20) - judged not relevant
- P14: [Speed up ICU case conversions](https://commitfest.postgresql.org/patch/5474/) (cf, Returned with feedback, 2025-03-30) - judged not relevant
- P15: [Fix runtime errors from -fsanitize=undefined](https://commitfest.postgresql.org/patch/2144/) (cf, Rejected, 2019-08-13) - judged not relevant
- P16: [\[PATCH\] Equivalence Class Filters](https://commitfest.postgresql.org/patch/3524/) (cf, Returned with feedback, 2022-12-17) - judged not relevant
- P17: [Invalid permission check in pg_stats for functional indexes](https://commitfest.postgresql.org/patch/2274/) (cf, Returned with feedback, 2020-07-05) - judged not relevant
- P18: [libpq: add portaddr, the port equivalent of hostaddr](https://commitfest.postgresql.org/patch/7204/) (cf, Withdrawn, 2026-08-28) - judged not relevant
- P19: [Get rid of the pre-C11 _Alignof define in zic.c](https://commitfest.postgresql.org/patch/6453/) (cf, Withdrawn, 2026-02-13) - judged not relevant
- P20: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant
- P21: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P22: [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://commitfest.postgresql.org/patch/5403/) (cf, Withdrawn, 2024-11-25) - judged not relevant
- P23: [bytea PL/Perl transform](https://commitfest.postgresql.org/patch/4252/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P24: [COPY: validate option presence rather than option values](https://commitfest.postgresql.org/patch/6744/) (cf, Withdrawn, 2026-06-29) - judged not relevant

### Warnings

- review angle "standards-compat" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 172 s; cost reported by the CLI: $1.14
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 97 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:07:21.638Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
