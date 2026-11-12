# Design pre-review: pg_upgrade: Prevent premature startup of the new cluster

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7443](https://commitfest.postgresql.org/patch/7443/); [thread](https://www.postgresql.org/message-id/flat/CAM527d_MAVFE8RVM_Qr8-P4opUSxemY2r%3DMeEiV%2BodTu6dw4cw%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection challenges the problem or the goal: message #3 reproduces the unprotected window and message #2 agrees in principle. The one surviving objection, replication-ondisk-2, is a fixable design point about the rsync standby case, which v1 defers and does not make worse. What remains is a decision for a committer who knows pg_upgrade and server start, because the patch adds a new refusal at startup. Confidence is medium because the thread is short and nobody has reviewed v1.

Not checked: Without the code, the review could not confirm that the marker is created before the first change to the new cluster, or that every server start by pg_upgrade passes -b. It also could not check the fsync behaviour under --no-sync.

## Prior attempts

- [Set default transactions to read-only at servers start in pg_upgrade](https://www.postgresql.org/message-id/flat/20210121152357.s6eflhqyh4g5e6dv@dalibo.com) (similar idea; Withdrawn, 2021-12-01). The visible messages do not show why. The last messages, from August 2021, still debate where to block background workers: in bgworker_should_start_now(), in RegisterBackgroundWorker(), or not in the backend at all. They also ask whether extensions could fail when their workers do not run. No message explains the withdrawal in December 2021. (AI summary of the thread) Different now: P7 limited writes by the servers that pg_upgrade itself starts. This proposal refuses servers that someone else starts while pg_upgrade works, so the P7 debate about where to block background workers does not apply.
- [Do not start bgworker processes during binary upgrade](https://www.postgresql.org/message-id/flat/20210121152357.s6eflhqyh4g5e6dv@dalibo.com) (similar idea; Returned with feedback, 2022-03-14). reason not recorded Different now: P14 kept background workers from starting in binary-upgrade mode. This proposal leaves binary-upgrade servers alone and refuses normal starts while the marker exists.
- [pg_upgrade allows itself to be run twice (initdb)](https://www.postgresql.org/message-id/flat/20220625160437.GI22452@telsasoft.com) (similar idea; Returned with feedback, 2022-12-01). reason not recorded Different now: P24 dealt with pg_upgrade running twice against the same new cluster. This proposal blocks server start after a failed run but does not say whether pg_upgrade refuses a new cluster that already has the marker.
- [Improve documentation for pg_upgrade, standbys and rsync](https://www.postgresql.org/message-id/flat/22f129004bb66cd91e1dfd3345a9787f5039f3ae.camel@cybertec.at) (related; Withdrawn, 2022-04-08). "To be more explicit though- we should write a tool to do this. We shouldn't try to document a way to do it because it's hard to get right." ([message](https://www.postgresql.org/message-id/20220405171038.GU10577%40tamriel.snowman.net)) Different now: P6 rewrote the documented rsync procedure for standbys. v1 leaves the rsync case out, so P6's objection would apply only to a follow-up that adds a manual marker removal step.

## Objections a reviewer is likely to raise

### 1. Marker does not cover the documented rsync standby upgrade

An operator upgrades a primary and then runs the documented rsync to its standbys. Automation starts the new primary after pg_upgrade exits but before rsync finishes, because the marker is already gone. The standby ends up with files that do not match the primary, and the mismatch surfaces only after failover.

Message #1 named this case as one where the marker must outlive pg_upgrade. Message #4 defers it, so v1 protects only the single-node path. A later keep-marker option would need a supported way to remove the marker on the primary and on each rsynced standby. Today the only way is a manual file delete, which adds one more step to a procedure that [P6] records as hard to get right by hand.

Precedent: [Improve documentation for pg_upgrade, standbys and rsync](https://www.postgresql.org/message-id/flat/22f129004bb66cd91e1dfd3345a9787f5039f3ae.camel@cybertec.at) (Withdrawn, 2022-04-08)

**Question:** How would the marker design extend to the rsync standby procedure, and by what supported command would an operator remove it on the primary and the standbys?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAM527d_MAVFE8RVM_Qr8-P4opUSxemY2r%3DMeEiV%2BodTu6dw4cw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAOh5eDVOtc4m0v19Rqcog%2B2gH75LbakScD3PC33aiw%2BSxppiEA%40mail.gmail.com); `replication-ondisk-2`</sub>

## Questions for the author

1. How would the marker extend to the rsync standby procedure, and which supported command would remove it on the primary and on each standby?
2. After a failed upgrade, is deleting pg_upgrade_in_progress by hand a supported way to start the new cluster, or is initdb the only supported path?
3. Is the rule to re-run initdb after any failure stricter than today's documented recovery steps for each transfer mode?
4. Does pg_upgrade refuse a new cluster that already has the marker, and do pg_resetwal, pg_rewind and pg_checksums respect it?
5. Under --no-sync, does pg_upgrade still fsync the creation and the removal of the marker?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade creates and removes a new file, pg_upgrade_in_progress, in the new cluster's data directory.
- **WAL / replication / standby:** Message #1 says the rsync standby procedure may need the marker to stay after pg_upgrade; v1 defers this.
- **maintenance burden ("who maintains this in 10 years"):** checkDataDir() gains a permanent startup check, and pg_upgrade must keep marker creation and removal correctly placed.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson agreed in principle in message #2, before any patch existed; nobody has reviewed v1. The owner needs to know pg_upgrade's step order and the server start path.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is about 11 KiB over five files and guards pg_upgrade users against a reproduced path to silent corruption (message #3). It needs a rebase and an answer on the rsync standby case before review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
