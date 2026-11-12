# Design pre-review: Reject unassigned bytes in single-byte encodings

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7542](https://commitfest.postgresql.org/patch/7542/); [thread](https://www.postgresql.org/message-id/flat/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is reasonable, and the design questions rest on facts the author states in message #1. Existing data stops restoring and pg_upgrade keeps it with no check (replication-ondisk-1). Input that works today starts to error (standards-compat-2). The mapping alternative is left open (standards-compat-3), and COPY slows by about 20% (maintenance-fit-1). The list needs to agree on reject versus map, and on the upgrade path, before code review.

Not checked: Without the code, nobody can judge how the reject lists are built, whether the apply worker validates streamed changes, or which paths besides verifystr, chr() and the LATIN2 to WIN1250 table still let these bytes in.

## Prior attempts

None found before 2026-10-26.

## Objections a reviewer is likely to raise

### 1. pg_upgrade carries forward bytes that dump and restore will then reject

A WIN1252 cluster holds 0x81 in a text column, perhaps from earlier LATIN2 to WIN1250 conversions. The admin runs pg_upgrade to the new release, which succeeds. Months later a pg_dump restore or a migration fails at COPY, or at a function body or column default that holds the byte, long after the old cluster is gone.

Message #1 says pg_upgrade keeps the bytes and the restore fails, and it leaves a pg_upgrade check as a maybe. Without a check or a scan tool, the upgraded cluster holds data the server itself now calls invalid, and nothing tells the user until a restore. The same bytes in catalogs, such as function bodies, defaults or comments, break the schema part of the restore. That is a larger failure than the per-table COPY failure the message describes.

**Question:** What will pg_upgrade --check, or a shipped query, report for unassigned bytes in user tables and catalogs before the upgrade commits the user to the new rules?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#1](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Mapping holes to C1 code points is dismissed too fast

After the patch, a WIN1252 user with 0x81 in the data has no conversion path left, so the data cannot be read through a UTF8 client and cannot be moved to a UTF8 database. With a hole-to-C1 mapping for the five 0x80-0x9F encodings, the same data would convert with no runtime cost and no input break.

Message #1 rejects the mapping because the maps would no longer match the Unicode.org files, and because it covers only 5 of 13 encodings. Common Windows converters map these holes to U+0081 and similar code points, so a mixed design may fit what clients send better. Using the mapping for 5 encodings and rejection for the other 8 would shrink the compatibility break.

**Question:** Why is matching Unicode.org files worth more than readable old data, and why not map the 0x80-0x9F holes and reject only the rest?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com); `standards-compat-3`</sub>

### 3. About 20% slower COPY for every user of 13 single-byte encodings

A site with a WIN1252 database loads data with COPY and never converts it to another encoding. After upgrade its bulk loads of mostly non-ASCII text run about 20% slower. Its data was never at risk from the bug being fixed.

Message #1 measures +20% for WIN1252 at 50% non-ASCII and +23% for ISO 8859-6. The failure in bug #14197 only appears when bytes are converted to another encoding, yet every input in these encodings pays the check. A committer would likely ask for a check that costs no more than the existing per-byte loop, such as a 256-entry lookup per encoding. The proposal does not show that the cost was weighed against that option.

**Question:** Can the check run as a per-encoding 256-byte table lookup with no measurable COPY cost, and what do the numbers look like then?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. Logical replication from an older publisher stops on unassigned bytes

An admin upgrades a WIN1252 database to the new release by logical replication from the old server. The old publisher sends a row that holds 0x81. The new subscriber's apply worker rejects the text, errors, restarts and fails on the same change again, so replication stalls until someone finds and fixes the row on the publisher.

Logical replication is a common way to upgrade across major versions, and the subscriber validates incoming text in its own encoding. The old publisher accepts the bytes, so the change is a rule mismatch between the two nodes. Message #1 does not cover replication or mixed-version setups. This is the same pre-upgrade detection gap as the pg_upgrade case, but here it shows up as a stuck slot and growing WAL on the publisher.

**Question:** How should a user upgrading a WIN1252 database by logical replication find and clean unassigned bytes on the old publisher before the subscriber stops on them?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com); `replication-ondisk-2`</sub>

### 5. Client input that works today errors after upgrade

An application with client_encoding=WIN1252, or one that calls chr(129) in a WIN1252 database, stores these bytes and reads them back unchanged. After upgrade, its INSERT, COPY or chr() call raises an error. Nothing in the application changed, and the failure comes only when such a byte appears in production data.

Bug #14197 shows that conversion fails later. That harms users who convert, but users who store and read back in the same encoding see no failure today. The patch moves the error from conversion to input for every user, including users who never convert. Message #1 gives no evidence that this second group is small, and it does not say whether the change targets master only.

**Question:** What evidence shows that users who store these bytes and never convert them are few enough to justify breaking their input path?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAGRkXqTdQ5Zin7Ayw-QY2pjnWvVOrm8x6%2B0ruSrKsT2FguRBaQ%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. What will pg_upgrade --check, or a shipped query, report for unassigned bytes in user tables and catalog text before the upgrade?
2. Why not map the 0x80-0x9F holes to code points for the five encodings where that works, and reject only in the other eight?
3. What evidence shows that few users store these bytes and read them back in the same encoding without conversion?
4. Can the check be a 256-entry lookup per encoding inside the existing per-byte loop, and what are the COPY numbers then?
5. How does a user who upgrades by logical replication find and fix unassigned bytes on the old publisher before the subscriber stops?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade keeps existing unassigned bytes that the new server rejects on input.
- **WAL / replication / standby:** A new subscriber may reject rows with unassigned bytes sent by an older publisher.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-3: Mapping holes to C1 code points is dismissed too fast
- **pg_dump / restore round-trip:** Message #1 says a dump of a database with such bytes fails on restore at COPY.
- **backpatchability:** The entry is tagged Bugfix, but the change breaks input, chr() and one conversion; the proposal targets master only.
- **maintenance burden ("who maintains this in 10 years"):** Reject lists for 13 encodings in wchar.c must stay equal to the *_to_utf8.map files.
- **performance on the hot path:** The author measures COPY about 20% to 23% slower for non-ASCII input in these encodings.

## Who would commit it, and when

- Owner: silent. Someone who knows the encoding verifiers and conversion maps, and who is willing to own a compatibility break in pg_upgrade and restore behaviour. Nobody has replied to message #1.
- Priority: fix design first; audience niche; review cost medium. The patch is small and affects only databases in 13 single-byte encodings. The open choice between reject and map, and the missing pg_upgrade check, need an answer before line-by-line review is useful.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
