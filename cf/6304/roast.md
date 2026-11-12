# Design pre-review: Incorrect historic snapshot may be serialized to disk during fast-forwarding

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6304](https://commitfest.postgresql.org/patch/6304/); [thread](https://www.postgresql.org/message-id/flat/tencent_3A071B760AA1A38540B57F297332B7781C08%40qq.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the problem is stated with a reproducer in message #1, and message #10 reports an output plugin failure that v4 fixes. The one surviving objection, replication-ondisk-1, is a fixable level 2 point about snapshot files written before the fix and about back branches. A committer who knows the snapshot builder has to decide those two points.

Not checked: The code was not read, so the snapbuild.c change in v4 and its effect on the serialized snapshot format could not be judged. Only 3 of 10 messages were shown in full; the rest is known from the brief.

## Prior attempts

None found before 2025-11-22.

## Objections a reviewer is likely to raise

### 1. Wrong snapshot files already on disk keep being reused after the fix

A user installs a minor release with the fix on a server where an earlier fast-forward run, such as pg_replication_slot_advance, wrote a snapshot file to pg_logical/snapshots without the NEW_CID transaction. A slot created or restarted at that LSN restores the file instead of rebuilding the snapshot. An output plugin that scans a user_catalog_table then reads the wrong rows, with no error to show it.

The patch changes how snapshots are built from now on, but message #1 shows the defect persists as a serialized file that a later run at the same LSN can read. CheckPointSnapBuild removes such files only once no slot needs them. The proposal does not say whether it targets back branches, or whether files written before the fix stay valid, need removal, or need a SNAPBUILD_VERSION bump. The commit message also does not say what the snapbuild.c change does, so a reviewer cannot tell whether the on-disk snapshot format changes.

**Question:** After a minor upgrade, can a snapshot file written by an unpatched fast-forward run still be restored, and if so, what removes or rejects it?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#1](https://www.postgresql.org/message-id/tencent_3A071B760AA1A38540B57F297332B7781C08%40qq.com), [#5](https://www.postgresql.org/message-id/tencent_4B8339BBC550D1C29B5AA1142A1918BCC207%40qq.com), [#10](https://www.postgresql.org/message-id/CAPgNnN0s7B6jdWT8QBjVA5pkdiPa_zS%2BDO4DgXdXuAp9tWV7fA%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. After a minor upgrade, can a snapshot file written by an unpatched fast-forward run still be restored, and if so, what removes or rejects it?
2. Is the fix meant for back branches, and which ones?
3. What does the snapbuild.c change in v4 do, and does it change the serialized snapshot file format?
4. Does any core path or supported extension write NEW_CID without invalidation messages, other than a write to a user_catalog_table?
5. What error or wrong result did the output plugin show in the reproduction in message #10?

## Touch points

- **on-disk format / pg_upgrade:** Serialized snapshot files in pg_logical/snapshots written before the fix may be restored after a minor upgrade; v4 also touches snapbuild.c.
- **WAL / replication / standby:** The patch changes how fast-forward logical decoding in decode.c treats XLOG_HEAP2_NEW_CID records.
- **backpatchability:** The entry is tagged Bugfix, but the thread does not say which back branches the fix targets.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows the snapshot builder and logical decoding. Masahiko Sawada asked for a real-world problem earlier in the thread, and no committer has replied since the reproduction in message #10.
- Priority: needs champion; audience niche; review cost low. The fix is under 3 KiB in decode.c and snapbuild.c, adds no user-visible surface, and affects only output plugins and extensions that scan a user_catalog_table. It has been in 5 CommitFests, and no committer has answered the reproduction in message #10.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
