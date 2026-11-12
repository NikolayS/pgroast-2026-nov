# Design pre-review: Validate GIN posting lists before decoding them

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7352](https://commitfest.postgresql.org/patch/7352/); [thread](https://www.postgresql.org/message-id/flat/CAH7T-aoyWXh1GDZRJg5AzG_sAmUnvPdWoV9ybW%2BEA5VP%3DhMwmw%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal: the patches remove out-of-bounds reads on corrupt GIN pages. Two fixable design points remain at level 2, both tied to the backpatch. replication-ondisk-1 asks what redo does with a corrupt posting list in back branches and how a stopped node recovers. maintenance-fit-1 asks for a measurement against unpatched code. Only message #1 is visible, so the judgement rests on thin material.

Not checked: The code was not read. The error level in redo, the errcode, which callers reach the new checks during replay, and the real cost of the loop form of decode_varbyte() could not be checked.

## Prior attempts

- [Revert refactoring of hex code to src/common/](https://www.postgresql.org/message-id/flat/1629039545467.80333@nidsa.net) (related; Reverted, 2021-08-19). "As reported by Hans Buschmann, those overflow checks are expensive, and it is possible to see a performance drop in the decoding/encoding of bytea or LOs the longer they are." Different now: P13 added overflow checks to hex encoding and decoding in src/common/. This proposal adds bounds checks to GIN posting list decoding, so only the recorded cost of checks in a decode loop carries over.

## Objections a reviewer is likely to raise

### 1. Backpatched replay error can stop standbys and crash recovery after a minor update

A site updates its standby to the new minor release first, as the release notes advise, while the primary on the old minor keeps writing GIN insert records against a corrupt leaf page. The standby's startup process hits the new error during redo, exits, and loops on the same record at each restart. The site finds out from replication lag alerts and must rebuild the standby.

An error in the startup process ends recovery, so on a standby, or on a primary in crash recovery, it takes the whole node down. Before the patch the same record replayed and the node kept running with one bad index. Message #1 calls the change preferable but leaves the backpatch question open. It does not say what level the redo path raises, or how a user gets past the record without a new base backup or pg_resetwal. Today a REINDEX after promotion or startup fixes a bad GIN index, and that path is lost once the node cannot start.

**Question:** In back branches, what does redo do with a corrupt posting list, and how does an operator bring a standby or crashed primary back up without restoring from backup?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAH7T-aoyWXh1GDZRJg5AzG_sAmUnvPdWoV9ybW%2BEA5VP%3DhMwmw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Per-byte bounds checks on the GIN decode hot path are not measured against unpatched code

A user on a minor release runs a full-text or array GIN query that decodes many large posting lists, so decode_varbyte() runs once per item. The new end-pointer test and the loop form add work per byte. The user sees a slower query after the update with no error, and finds out only by comparing before and after.

Message #1 reports no measurable difference between the loop form and the nested-if-with-goto form, but both variants have the checks, so the cost of the checks against unpatched code is not shown. [P13] was reverted because overflow checks added to hex decoding slowed bytea decoding in a visible way, which is the same kind of change: per-byte validation in a decode routine. 0001 to 0005 are meant for 14 and later, where a slowdown found after release is hard to take back.

Precedent: [Revert refactoring of hex code to src/common/](https://www.postgresql.org/message-id/flat/1629039545467.80333@nidsa.net) (Reverted, 2021-08-19)

**Question:** What is the decode cost of patched versus unpatched master on a scan over large, dense GIN posting lists, such as a common-word full-text search?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/CAH7T-aoyWXh1GDZRJg5AzG_sAmUnvPdWoV9ybW%2BEA5VP%3DhMwmw%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. In back branches, what error level does redo raise on a corrupt posting list: ERROR, PANIC, or a WARNING that keeps the old behaviour?
2. How does an operator bring back a standby or a crashed primary stopped by this error without restoring from a base backup?
3. What is the decode cost of patched versus unpatched master on a scan over large, dense GIN posting lists, such as a common-word full-text search?
4. What are the error message and errcode, and does the error use ERRCODE_INDEX_CORRUPTED?
5. Is 0006's Assert() meant for back branches, given that 0005 is listed for backpatching and 0006 is not?

## Touch points

- **WAL / replication / standby:** 0001 to 0003 make WAL replay raise an error and stop recovery on a corrupt GIN posting list.
- **ABI / extension API:** 0001 changes decode_varbyte() to take a segment end pointer; whether any exported signature changes cannot be seen without the code.
- **security / privileges:** The patches remove out-of-bounds reads reachable with crafted bytes through pageinspect's gin_leafpage_items(); the author states it is not a security issue.
- **backpatchability:** The author proposes 0001 to 0005 for branches back to 14, including the replay behaviour change.
- **maintenance burden ("who maintains this in 10 years"):** New error paths and a rewritten decode_varbyte() in ginpostinglist.c, plus new pageinspect tests with crafted page bytes.
- **performance on the hot path:** decode_varbyte() gains an end-pointer check per byte and item checks, and it runs in GIN index scans and VACUUM.

## Who would commit it, and when

- Owner: silent. Someone who knows the GIN access method and its redo routines, and who is willing to judge a behaviour change in stable branches.
- Priority: needs champion; audience niche; review cost medium. The fix helps only GIN users with a corrupt page, but it removes out-of-bounds reads and silent invalid TIDs, and adds no new user-visible surface. The patches are small and in one file, but the replay change and the backpatch need a committer who knows GIN, and none has spoken.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
