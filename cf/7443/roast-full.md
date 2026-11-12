# Design pre-review: pg_upgrade: Prevent premature startup of the new cluster

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7443](https://commitfest.postgresql.org/patch/7443/) (Needs review); authors: Shubhra -Ksolves (shubhrajain)
- Thread: [Prevent premature startup of pg_upgrade targets](https://www.postgresql.org/message-id/flat/CAM527d_MAVFE8RVM_Qr8-P4opUSxemY2r%3DMeEiV%2BodTu6dw4cw%40mail.gmail.com), started 2026-08-14
- Messages read: 4 of 4
- Patch prose read: v1-0001-pg_upgrade-Prevent-premature-startup-of-the-new-c.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection challenges the problem or the goal: message #3 reproduces the unprotected window and message #2 agrees in principle. The one surviving objection, replication-ondisk-2, is a fixable design point about the rsync standby case, which v1 defers and does not make worse. What remains is a decision for a committer who knows pg_upgrade and server start, because the patch adds a new refusal at startup. Confidence is medium because the thread is short and nobody has reviewed v1.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not confirm that the marker is created before the first change to the new cluster, or that every server start by pg_upgrade passes -b. It also could not check the fsync behaviour under --no-sync.

## The proposal, restated

**Problem.** pg_upgrade checks only at the start that the new cluster is not running, so an external start during file transfer, pg_resetwal or the final sync goes undetected. Such a server can write WAL and consume xids and MultiXacts while pg_upgrade still reports success. This affects operators whose automation or tooling may start the target cluster too early.

**Proposed design.** pg_upgrade creates an fsynced marker file in the new data directory before it first modifies the cluster and removes it, again fsynced, after the last step that starts the new cluster. checkDataDir() refuses to start the server while the file exists unless binary-upgrade mode (-b) is set, which pg_upgrade uses for its own starts. A failed or interrupted upgrade leaves the file in place.

**User-visible surface.**
- New file pg_upgrade_in_progress in the new cluster's data directory during and after a failed upgrade.
- New FATAL error "data directory ... is being upgraded by pg_upgrade" with DETAIL and HINT at server start, also in single-user mode.
- New macro for the file name in access/xlog_internal.h, next to XLOG_CONTROL_FILE, usable from frontend code.
- New paragraph in the pg_upgrade documentation describing the marker and the need to re-run initdb after a failure.

**What the author claims.**
- A production copy-mode upgrade completed successfully, then showed corruption weeks later, with evidence the target was started during the upgrade; causation is not proven.
- Shubhra reproduced a 16 to 17 copy-mode upgrade where an externally created table survived into the upgraded cluster.
- With the patch, that reproduction fails with the new error and the upgrade then completes normally.
- check-world passes, including all transfer modes in 006_transfer_modes.pl, and --check never creates the marker.
- Daniel Gustafsson called the idea reasonable as an extra layer of protection.

**Thread state.** Nik proposed the idea, Daniel Gustafsson agreed in principle, and Shubhra posted v1, which CFbot reports needs a rebase. Nobody has replied to v1 or to its open points on the file name, fsync under --no-sync and the rsync/standby case.

## 2. Prior attempts

### Set default transactions to read-only at servers start in pg_upgrade (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2970/](https://commitfest.postgresql.org/patch/2970/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20210121152357.s6eflhqyh4g5e6dv@dalibo.com](https://www.postgresql.org/message-id/flat/20210121152357.s6eflhqyh4g5e6dv@dalibo.com)
- **Date:** decided 2021-12-01; started 2021-01-21
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. The last messages, from August 2021, still debate where to block background workers: in bgworker_should_start_now(), in RegisterBackgroundWorker(), or not in the backend at all. They also ask whether extensions could fail when their workers do not run. No message explains the withdrawal in December 2021.
- **Reason (quote):** quote unavailable
- **What is different this time:** P7 limited writes by the servers that pg_upgrade itself starts. This proposal refuses servers that someone else starts while pg_upgrade works, so the P7 debate about where to block background workers does not apply.

### Do not start bgworker processes during binary upgrade (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3299/](https://commitfest.postgresql.org/patch/3299/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20210121152357.s6eflhqyh4g5e6dv@dalibo.com](https://www.postgresql.org/message-id/flat/20210121152357.s6eflhqyh4g5e6dv@dalibo.com)
- **Date:** decided 2022-03-14; started 2021-01-21
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P14 kept background workers from starting in binary-upgrade mode. This proposal leaves binary-upgrade servers alone and refuses normal starts while the marker exists.

### pg_upgrade allows itself to be run twice (initdb) (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3872/](https://commitfest.postgresql.org/patch/3872/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20220625160437.GI22452@telsasoft.com](https://www.postgresql.org/message-id/flat/20220625160437.GI22452@telsasoft.com)
- **Date:** decided 2022-12-01; started 2022-06-25
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P24 dealt with pg_upgrade running twice against the same new cluster. This proposal blocks server start after a failed run but does not say whether pg_upgrade refuses a new cluster that already has the marker.

### Improve documentation for pg_upgrade, standbys and rsync (related)

- **Link:** [https://commitfest.postgresql.org/patch/3125/](https://commitfest.postgresql.org/patch/3125/)
- **Thread:** [https://www.postgresql.org/message-id/flat/22f129004bb66cd91e1dfd3345a9787f5039f3ae.camel@cybertec.at](https://www.postgresql.org/message-id/flat/22f129004bb66cd91e1dfd3345a9787f5039f3ae.camel@cybertec.at)
- **Date:** decided 2022-04-08; started 2021-05-18
- **Outcome:** Withdrawn (cf)
- **Who objected:** Stephen Frost, Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Stephen Frost argued that this procedure is too hard to get right in an rsync command or a documented procedure, and that a tool should replace it. He proposed moving the text out of the documentation to a wiki. Robert Haas found the new order of the preparation steps confusing. The author saw no agreement and withdrew the patch.
- **Reason (quote):** "To be more explicit though- we should write a tool to do this. We shouldn't try to document a way to do it because it's hard to get right." ([message](https://www.postgresql.org/message-id/20220405171038.GU10577%40tamriel.snowman.net))
- **What is different this time:** P6 rewrote the documented rsync procedure for standbys. v1 leaves the rsync case out, so P6's objection would apply only to a follow-up that adds a manual marker removal step.


## 3. Objections a reviewer is likely to raise

### 1. Marker does not cover the documented rsync standby upgrade

fixable · level 2, rubric item 14 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-2`

An operator upgrades a primary and then runs the documented rsync to its standbys. Automation starts the new primary after pg_upgrade exits but before rsync finishes, because the marker is already gone. The standby ends up with files that do not match the primary, and the mismatch surfaces only after failover.

Message #1 named this case as one where the marker must outlive pg_upgrade. Message #4 defers it, so v1 protects only the single-node path. A later keep-marker option would need a supported way to remove the marker on the primary and on each rsynced standby. Today the only way is a manual file delete, which adds one more step to a procedure that [P6] records as hard to get right by hand.

**Precedents.**
- [Improve documentation for pg_upgrade, standbys and rsync](https://commitfest.postgresql.org/patch/3125/) (Withdrawn, 2022-04-08); thread: [link](https://www.postgresql.org/message-id/flat/22f129004bb66cd91e1dfd3345a9787f5039f3ae.camel@cybertec.at)

Thread: [#1 (Nikolay Samokhvalov, 2026-08-14)](https://www.postgresql.org/message-id/CAM527d_MAVFE8RVM_Qr8-P4opUSxemY2r%3DMeEiV%2BodTu6dw4cw%40mail.gmail.com), [#4 (Shubhra Jain, 2026-10-13)](https://www.postgresql.org/message-id/CAOh5eDVOtc4m0v19Rqcog%2B2gH75LbakScD3PC33aiw%2BSxppiEA%40mail.gmail.com)

**Question:** How would the marker design extend to the rsync standby procedure, and by what supported command would an operator remove it on the primary and the standbys?

Counter-check: held. Message #1 names the rsync case as needing the marker to outlive pg_upgrade, and message #4 defers it without saying how an operator would remove the marker. The scenario exists today and v1 does not worsen it, so it is a design question for the author and stays fixable.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade creates and removes a new file, pg_upgrade_in_progress, in the new cluster's data directory. |
| WAL / replication / standby | **yes** | Message #1 says the rsync standby procedure may need the marker to stay after pg_upgrade; v1 defers this. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | checkDataDir() gains a permanent startup check, and pg_upgrade must keep marker creation and removal correctly placed. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How would the marker extend to the rsync standby procedure, and which supported command would remove it on the primary and on each standby?
2. After a failed upgrade, is deleting pg_upgrade_in_progress by hand a supported way to start the new cluster, or is initdb the only supported path?
3. Is the rule to re-run initdb after any failure stricter than today's documented recovery steps for each transfer mode?
4. Does pg_upgrade refuse a new cluster that already has the marker, and do pg_resetwal, pg_rewind and pg_checksums respect it?
5. Under --no-sync, does pg_upgrade still fsync the creation and the removal of the marker?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson agreed in principle in message #2, before any patch existed; nobody has reviewed v1. The owner needs to know pg_upgrade's step order and the server start path.
- **Liability:** The owner signs for a permanent refusal in checkDataDir(): any case where the marker wrongly stays, or wrongly goes away, becomes a cluster that will not start or an unprotected upgrade.
- **Author follow-up:** Shubhra took the work as a first patch and posted v1 with open points listed; CFbot has reported a needed rebase since 2026-10-16, with no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 11 KiB over five files and guards pg_upgrade users against a reproduced path to silent corruption (message #3). It needs a rebase and an answer on the rsync standby case before review.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Marker removal point may fall before the final data directory sync. Refuted (wrong facts): In pg_upgrade's main(), the data directory sync (initdb --sync-only) runs before issue_warnings_and_set_wal_level(), which is the last step that starts the new cluster. Removal "after the last step that starts the new cluster" is therefore already after the final sync.
- `maintenance-fit-1` Protected window depends on call placement that nothing enforces. Refuted (no failure scenario): The scenario needs a hypothetical future patch that adds a modifying step after the last server start, where only the completion banner runs today. The claim that the final sync is uncovered is wrong, because that sync runs before the last server start.

### Review angles

- prior-attempts: 0 objection(s). No recorded rejection reason in the precedents applies to v1. P7 and P14 blocked writes from pg_upgrade's own servers, and P6's objection to hand-run rsync steps would apply only to the deferred standby follow-up, which v1 does not include.
- standards-compat: 0 objection(s). From the standards and compatibility angle the proposal adds no SQL syntax, no GUC, no catalog object and nothing for pg_dump or psql. A successful upgrade leaves no file behind, so existing scripts and the documented standby rsync procedure behave as today. The one new behaviour is that a failed upgrade's new cluster refuses to start, and the documentation already tells users to re-run initdb in that case. The commit message and message #4 differ on whether the marker is removed before or after the final sync; this is a safety question for another angle. The xlog_internal.h placement and the file name are internal and open, as message #4 says.
- replication-ondisk: 2 objection(s). The proposal adds no WAL records, catalog entries, wire protocol messages or ABI changes. The marker is a transient file that exists only while pg_upgrade runs, so the persistent-state concerns are limited to the two objections above.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. Only the OS user that owns the data directory can create or remove the marker, or start the server with -b. A SQL role with pg_write_server_files could create the file to block a restart, but that role can already overwrite postgresql.auto.conf, so no new path opens.
- maintenance-fit: 1 objection(s). From the maintenance angle the design is small and general, and it cannot live in an extension. The startup cost is one file check at server start. Only the placement concern above holds. Committability: Daniel Gustafsson agreed only in principle, and no committer has reviewed v1.
- already-exists: 0 objection(s). No existing feature covers this. Renaming pg_control, as disable_old_cluster() does, would break pg_upgrade's own -b starts and pg_resetwal on the new cluster, and postmaster.pid only guards a running server. An extension cannot act before checkDataDir(), and operator measures such as masking service units do not protect against the automation errors the proposal targets.
- demand: 0 objection(s). The demand angle finds no objection. The patch guards every pg_upgrade user against a reproduced corruption path (message #3), backed by one production case and agreement from a committer (message #2). Its cost is a file check at startup, and only users of a failed upgrade see the stricter initdb rule.

### Precedents retrieved

- P1: [pg_upgrade: Support for upgrading to checksums enabled](https://commitfest.postgresql.org/patch/5206/) (cf, Returned with feedback, 2025-04-03) - judged not relevant; reason summarised from its thread
- P2: [add timing information to pg_upgrade](https://commitfest.postgresql.org/patch/4464/) (cf, Withdrawn, 2023-12-02) - judged not relevant; reason summarised from its thread
- P3: [pg_upgrade documentation improvement](https://commitfest.postgresql.org/patch/583/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P4: [pg_upgrade version and path checking](https://commitfest.postgresql.org/patch/2095/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P5: [Backport perl tests for pg_upgrade](https://commitfest.postgresql.org/patch/3997/) (cf, Withdrawn, 2022-12-19) - judged not relevant; reason summarised from its thread
- P6: [Improve documentation for pg_upgrade, standbys and rsync](https://commitfest.postgresql.org/patch/3125/) (cf, Withdrawn, 2022-04-08) - used in the review; reason summarised from its thread
- P7: [Set default transactions to read-only at servers start in pg_upgrade](https://commitfest.postgresql.org/patch/2970/) (cf, Withdrawn, 2021-12-01) - used in the review; reason summarised from its thread
- P8: [pg_upgrade: Pass -j option down to vacuumdb](https://commitfest.postgresql.org/patch/1943/) (cf, Returned with feedback, 2019-04-09) - judged not relevant; reason summarised from its thread
- P9: [Refuse setting toast.* reloptions when TOAST relation does not exist](https://commitfest.postgresql.org/patch/1486/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P10: [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (cf, Returned with feedback, 2021-11-26) - judged not relevant
- P11: [pg_upgrade: remove C comment](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=84a6f63e32dbefe3dc76cbe628fab6cbfc26141e) (revert, Partially reverted, 2018-01-05) - judged not relevant
- P12: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P13: [Add recovery to pg_control and remove backup_label](https://commitfest.postgresql.org/patch/4629/) (cf, Withdrawn, 2024-01-28) - judged not relevant
- P14: [Do not start bgworker processes during binary upgrade](https://commitfest.postgresql.org/patch/3299/) (cf, Returned with feedback, 2022-03-14) - used in the review
- P15: [Revert "postmaster: Start syslogger earlier".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=833451552925d0175e1e15128e411ddef9a36996) (revert, Reverted, 2019-05-19) - judged not relevant
- P16: [pg_upgrade check for Unicode update](https://commitfest.postgresql.org/patch/5563/) (cf, Withdrawn, 2025-03-18) - judged not relevant
- P17: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P18: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P19: [Revert "Add basic TAP test setup for pg_upgrade"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=58ffe141eb37c3f027acd25c1fc6b36513bf9380) (revert, Reverted, 2017-09-22) - judged not relevant
- P20: [Startup Centralization](https://commitfest.postgresql.org/patch/2259/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P21: [Binary I/O for isn extension](https://commitfest.postgresql.org/patch/639/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P22: [Use SIGIO to detect postmaster death](https://commitfest.postgresql.org/patch/3066/) (cf, Withdrawn, 2021-07-13) - judged not relevant
- P23: [Startup Centralization](https://commitfest.postgresql.org/patch/2498/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P24: [pg_upgrade allows itself to be run twice (initdb)](https://commitfest.postgresql.org/patch/3872/) (cf, Returned with feedback, 2022-12-01) - used in the review

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 126 s; cost reported by the CLI: $1.24
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 68 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:56:37.209Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
