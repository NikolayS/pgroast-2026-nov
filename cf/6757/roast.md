# Design pre-review: Avoid unnecessary StringInfo allocation in tablesync COPY buffer

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6757](https://commitfest.postgresql.org/patch/6757/); [thread](https://www.postgresql.org/message-id/flat/5B2C9B4C-EAE6-4F21-AF99-613A561D26DC%40gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The change is a refactoring of one file, tablesync.c, with no user-visible surface. P20 does not carry over, because v3 uses initReadOnlyStringInfo(), which is meant for buffers that are not NUL-terminated. The open point is a choice of form between v2 (CopyBuf struct) and v3 (read-only StringInfoData), which Álvaro Herrera has not yet answered.

Not checked: The code was not read, and v3 has no commit message or file list in the material. Whether v3 resets the static copybuf before each COPY and keeps leftover data valid between calls could not be checked.

## Prior attempts

- [Revert "Optimize various aggregate deserialization functions"](https://www.postgresql.org/message-id/flat/CAApHDvorfO3iBZ=xpiZvp3uHtJVLyFaPBSvcAhAq2HPLnaNSwQ@mail.gmail.com) (similar idea; Reverted, 2023-10-10). "On 2nd thoughts, the StringInfo API requires that strings are NUL terminated and pointing directly to the data in a bytea Datum isn't NUL terminated." Different now: P20 pointed a regular StringInfo at bytea data that is not NUL-terminated. v3 wraps the received buffer with initReadOnlyStringInfo(), the read-only form meant for such externally owned buffers.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which version is now proposed for commit: v2 with the file-local CopyBuf struct, or v3 with a read-only StringInfoData?
2. Does wrapping each received buffer with initReadOnlyStringInfo() answer the "misuse of the StringInfo abstraction" concern from message #2?
3. Is the static copybuf in v3 reset before each COPY, including when one worker syncs a second table?
4. Will v3 get a commit message that explains why a static variable is used and how leftover data survives between calls?

## Touch points

None of the 15 is hit.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera objected to v1 in message #2 but has not replied to v3. A committer who knows the logical replication tablesync code would have to pick between v2 and v3.
- Priority: proceed; audience niche; review cost low. The patch is under 3 KiB, touches only tablesync.c and benefits the people who maintain that file. It has waited through three CommitFests for agreement on v2 or v3, which one reply from the objecting committer could settle.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
