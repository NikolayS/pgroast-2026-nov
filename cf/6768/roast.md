# Design pre-review: pg_rewind does not rewind diverging timelines

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6768](https://commitfest.postgresql.org/patch/6768/); [thread](https://www.postgresql.org/message-id/flat/CAN305gBeJr8m7ZRW9mH0zakEFR4hDUPDo8fJRKJOHWMORG5_Bg%40mail.gmail.com), 36 of 36 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem is stated and several independent people confirm it, so levels 1a and 1b pass. The design is still open at level 2. The UUID is only in the history file, so pg_rewind's own archive reads can still pick the other branch's segment (replication-ondisk-1), and the proposal does not say which timeline comparisons must use the UUID (maintenance-fit-1). The back-branch intent conflicts with the format and API change (standards-compat-1).

Not checked: Only 3 of 36 messages and no code were read. How the patch parses the fourth field next to the free-text reason, and how pg_rewind picks WAL segments after the ancestor search, could not be checked.

## Prior attempts

- [Add timeline to partial WAL segments](https://www.postgresql.org/message-id/flat/c6433bac-65bc-555e-e318-2077dd153eab@pgmasters.net) (related; Returned with feedback, 2020-09-30). "As nothing has happened here, I have marked the patch as RwF." ([message](https://www.postgresql.org/message-id/20200930071750.GM1996%40paquier.xyz)) Different now: P5 renamed archived .partial segments to avoid archive conflicts after several promotions. This proposal adds a per-promotion UUID to history file entries and changes only pg_rewind matching, not WAL file names.

## Objections a reviewer is likely to raise

### 1. Archived WAL segments with the same TLI stay ambiguous to pg_rewind

S1 and S2 both promote to TLI 2 and share an archive, which then holds 000000020000000000000004 from S2. pg_rewind --restore-target-wal on S1 correctly backs up to TLI 1, then restores that segment as S1's own TLI 2 WAL. It misses S1's changed blocks, and S1 keeps diverged pages with no error.

The UUID lives only in the history file, so nothing ties a WAL segment to the promotion that wrote it. Message #23 raises this and proposes the UUID in XLogLongPageHeaderData. Message #27 calls WAL format changes too intrusive but does not show that pg_rewind's own archive reads are safe. The fix covers ancestor selection and leaves block collection in the same collision.

**Question:** When pg_rewind restores target WAL from a shared archive after two same-TLI promotions, what stops it from reading the other branch's segment of the same name?

<sub>fixable, level 2, WAL / replication / standby; messages [#23](https://www.postgresql.org/message-id/CANwKhkO0jA5-V%2BOhQtPOwscLrq1%2BO9sCZXfuo4MYecjd1r9wag%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/c9bfdf63-0c04-41c5-bd68-93e62bfa1f21%40gmail.com); `replication-ondisk-1`</sub>

### 2. Second timeline identity checked only by pg_rewind

Two nodes promote to TLI 2 and both archive WAL. Recovery or restore_command later replays the wrong node's TLI 2 segments, because only pg_rewind compares the UUID. Operators find out after the restore, when the data has diverged.

The patch makes the UUID part of timeline identity but checks it in one frontend tool. Recovery, the archiver and WAL page headers ignore it. Ants Aasma (message #23) gives the archive case, and the author's reply in message #27 does not close it. Any future code that compares timelines would also need to check the UUID, and nothing fails if it does not.

**Question:** Which other code paths that compare timelines must honour the UUID, and how would a future change that misses one be caught?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#22](https://www.postgresql.org/message-id/amaZ6bN0pJM75OEs%40paquier.xyz), [#23](https://www.postgresql.org/message-id/CANwKhkO0jA5-V%2BOhQtPOwscLrq1%2BO9sCZXfuo4MYecjd1r9wag%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/c9bfdf63-0c04-41c5-bd68-93e62bfa1f21%40gmail.com); `maintenance-fit-1`</sub>

### 3. Bugfix tag and back-branch intent conflict with format and API changes

A packager ships a minor release with this fix. An HA tool or extension built against the old writeTimeLineHistory() signature breaks on the minor update. In a rolling minor upgrade, patched and unpatched nodes then write history files in two formats in the same cluster and archive.

The CF entry is tagged Bugfix and targets version 18. The patch changes the timeline history file format and the signature of an exported backend function. Message #17 asked whether the signature change is an ABI break if the fix is back-patched, and no later message answers it. A committer would likely ask for a master-only design, or a separate back-branch variant that keeps the old signature and file format.

**Question:** Is this meant for master only, and if not, how does a back-branch version keep the writeTimeLineHistory() ABI and the history file format stable within a major release?

<sub>fixable, level 2, backpatchability; messages [#17](https://www.postgresql.org/message-id/CAN4CZFO%2BHVWRWELsu4CuQ%2BFr%3DJ4e%2BjdHM5siRQ3G3vyJ5-Y6Sg%40mail.gmail.com); `standards-compat-1`</sub>

### 4. History file format change has no stated contract for external parsers

A backup or HA tool reads a new history file, from the archive or through the TIMELINE_HISTORY replication command, and splits each line on tabs. It shows the UUID as the switch reason, or rejects a line with an unexpected field count. The operator finds out during a failover or a restore.

History files are read by more than the server and pg_rewind: they travel through archives and replication to tools the project does not control. The proposal adds an optional fourth field, but the patch contains no documentation hunk. It does not say where the field sits relative to the free-text reason, or how an older server or older tool should treat it. [P22] shows that a change in field count broke replication utilities across versions, and that commit was reverted.

Precedent: [Revert "Fix bug in checking of IDENTIFY_SYSTEM result."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c4762886539b06da682045e19ba73dcee3789638) (Reverted, 2014-08-19)

**Question:** What is the exact new line layout, where is it documented, and how do older servers and existing tools that parse history files read a line with the UUID field?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#17](https://www.postgresql.org/message-id/CAN4CZFO%2BHVWRWELsu4CuQ%2BFr%3DJ4e%2BjdHM5siRQ3G3vyJ5-Y6Sg%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CAHza6qczQXOwEyqNX2rq5UNo2XQb_bWY33F%3D9sNXzVPr6DztMA%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. When pg_rewind restores target WAL from a shared archive after two same-TLI promotions, what stops it from reading the other branch's segment of the same name?
2. Which code paths that compare timelines must use the UUID, which may ignore it, and how would a future change that misses one be caught?
3. Is the patch meant for master only, and if not, how does a back-branch version keep the writeTimeLineHistory() signature and the history file format unchanged?
4. What is the exact new history file line layout, where is it documented, and how do older servers and external tools read a line with the UUID field?
5. Can a zero UUID compared with a non-zero UUID force a rewind that is not needed when one node was promoted before the upgrade and another after it?

## Touch points

- **on-disk format / pg_upgrade:** Each timeline history file entry gains an optional fourth field with a UUID.
- **WAL / replication / standby:** Promotion writes the UUID, and pg_rewind uses it to match timelines between source and target.
- **ABI / extension API:** writeTimeLineHistory() gains a UUID argument, and utils/uuid.h exports generate_uuidv7_r() and generate_uuidv7().
- **backpatchability:** The CF entry is tagged Bugfix with target 18, but the patch changes a file format and an exported function signature.
- **maintenance burden ("who maintains this in 10 years"):** The UUID becomes a second timeline identity that only pg_rewind checks; later timeline code must know about it.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #22 argues that the archive design should guarantee unique TLIs, so a second identifier is unneeded.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier takes part in the thread but opposes a second timeline identifier (message #22). The owner must know timeline history handling, promotion in StartupXLOG() and pg_rewind.
- Priority: fix design first; audience a sizeable minority; review cost medium. HA users who run pg_rewind after failover are affected, and messages #19, #23 and #24 report the problem in practice. The patch is about 44 KiB across xlog.c, timeline.c and pg_rewind, and the thread has not agreed on whether a UUID in the history file alone is the right scope.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
