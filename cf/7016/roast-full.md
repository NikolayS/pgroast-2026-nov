# Design pre-review: Fix LSN of logical decoding message

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7016](https://commitfest.postgresql.org/patch/7016/) (Needs review); authors: Atsushi Torikoshi (atsushitorikoshi)
- Thread: [Why is the LSN reported for pg_logical_emit_message() different from other decoded operations?](https://www.postgresql.org/message-id/flat/d99c688994ab3a998afe26e61fe4f69f%40oss.nttdata.com), started 2026-06-08
- Messages read: 13 of 13
- Patch prose read: v3-0001-Correct-documentation-of-logical-decoding-message-v15-19.txt, v3-0001-Use-start-LSN-for-transactional-logical-decoding-.patch, v3-0001-Correct-documentation-of-logical-decoding-message-v14.txt, v1-0001-Align-logical-message-LSN-with-other-operations.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible case and a field report of a lost INSERT in Debezium (message #1). The review notes say reviewers accept the master-only change with documentation fixes in the back branches (messages #10 to #13). Confidence is medium because only 3 of 13 messages and no code were read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so it could not be checked that pgoutput and test_decoding pass the new value through the same way. Messages #4 to #13 are known here only through the brief and the review notes.

## The proposal, restated

**Problem.** Logical decoding reports a logical message at the end LSN of its WAL record, but INSERT, UPDATE and DELETE use the start LSN. A transactional message can then share the LSN of the next change. Debezium resumed after a crash at such an LSN, treated the following INSERT as processed and lost it.

**Proposed design.** On master, decode.c queues transactional messages with origptr (start LSN) and keeps endptr for non-transactional messages, which receivers handle as separate transactions. The back branches keep the current behavior, and their documentation now says the end LSN is reported. The behavior change is to be described in the commit message for the release notes.

**User-visible surface.**
- LSN reported for transactional messages in pg_logical_slot_get_changes() and the logical replication protocol Message changes from end to start of the WAL record.
- The message_lsn argument of the LogicalDecodeMessageCB output plugin callback changes the same way.
- The protocol.sgml text for the Message LSN field now distinguishes transactional and non-transactional messages.
- Back branches 14 to 19 get a documentation change that says the Message LSN is the end LSN.

**What the author claims.**
- The current behavior contradicts the documentation and can cause data loss in consumers that resume from persisted change LSNs.
- Postgres itself does not use this LSN, so there is no correctness problem in core.
- Debezium can accommodate a fix in either a minor or a major release.
- Fujitsu confirmed its proprietary Userlog Operation is not affected.
- Few consumers likely have special handling for message LSNs today.

**Thread state.** Sawada, Horiguchi, Nagata, Kuroda, solai v and João Rito replied, and all accept the master-only fix with doc changes in back branches. Open requests are more code comments on the non-transactional case, a matching update to message_cb text in logicaldecoding.sgml, and a regression test that tells old and new behavior apart.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-06-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | decode.c changes which LSN logical decoding reports for transactional messages; the WAL record itself does not change. |
| wire protocol | **yes** | The Message LSN field in the logical replication protocol changes meaning for transactional messages, per the protocol.sgml hunk. |
| catalog changes | no |  |
| ABI / extension API | **yes** | The message_lsn argument of the LogicalDecodeMessageCB output plugin callback changes from end LSN to start LSN for transactional messages. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The code change is master-only; branches 14 to 19 get a protocol.sgml correction that says the end LSN is reported. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The Message LSN now has two meanings, start for transactional and end for non-transactional, which documentation and plugins must keep straight. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the next version update the message_cb text in logicaldecoding.sgml, which still says the lsn is the WAL location of the message?
2. Which regression test tells the old and new LSN apart for a transactional message, and which one covers the non-transactional case?
3. What happens to a consumer that stored a transactional message's end LSN on 19 and resumes from it after upgrade to 20?
4. Does v3 keep or drop the v1 change to 020_messages.pl?
5. Were the back-branch documentation patches for 14 and for 15 to 19 checked against each branch's actual behavior?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The review notes say Sawada shaped the transactional/non-transactional split in message #10 and agreed to own it in message #12; those messages were not shown here.
- **Liability:** The owner signs for a behavior change in the logical replication protocol and the output plugin callback in 20, and for any external consumer that depended on the end LSN.
- **Author follow-up:** The author posted three patch versions and moved the entry to the current CommitFest on 2026-11-03.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches decode.c and protocol.sgml only and fixes a data-loss case reported for Debezium. It affects users of logical decoding who emit transactional messages, and the remaining requests are documentation, comments and a test.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the index search returned no earlier attempt at this idea. In message #2 the author's reply notes that the original thread adding pg_logical_emit_message() decoding has no discussion of the LSN choice, so no earlier rejection reason applies.
- standards-compat: 0 objection(s). No objection from the standards and compatibility angle. The change is a semantic fix with no new syntax, GUC, catalog object or dump state, and the thread settled the main risk by limiting it to master, documenting back branches and flagging it for the release notes (messages #10-#12). The one remaining contract gap, the message_cb text in logicaldecoding.sgml, is already an open request in message #12 and is a documentation fix. The Message LSN field will mean the start LSN for transactional messages and the end LSN for non-transactional ones. Sawada proposed this split and defended it in message #10 on the same ground as commit_lsn and end_lsn, so it follows existing practice. Without the code I could not check that pgoutput and test_decoding emit the new value the same way.
- replication-ondisk: 0 objection(s). The patch changes the meaning of the Message LSN field, which is part of the replication protocol, but adds no on-disk, WAL, or catalog state. Making it master-only, as settled in messages #10 and #12, keeps old positions valid. A consumer that keeps an old workaround against 20 at worst decodes a change twice. The open items (message_cb docs, a test, the non-transactional case) belong to other angles.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes which LSN, start or end of the same WAL record, is reported to consumers that can already read the slot, so nobody can do or see anything new.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The change is a one-line choice in decode.c that core does not use and that adds no hot-path cost, API or GUC. The mixed start/end rule follows the existing commit_lsn/end_lsn practice, and Sawada, a committer who knows the area, shaped the design in message #10 and agreed to own it in message #12. The missing regression test (message #13) and the message_cb text in logicaldecoding.sgml are implementation items. Neither can be checked without the next patch version.
- already-exists: 0 objection(s). No existing feature, option or extension gives transactional messages a start LSN, so consumers can only compensate in their own code. The patch fixes core behavior to match its documentation, and that cannot move to an extension or hook.
- demand: 0 objection(s). The material shows real demand: Debezium, a widely used CDC connector, lost an INSERT because a transactional message shared the next change's LSN (messages #1, #11), and its maintainers acknowledged the issue (message #7). The fix costs nothing on any hot path, matches the documented behaviour, and reviewers in messages #10 to #13 agree on the master-only scope.

### Precedents retrieved

- P1: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant
- P2: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P3: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P4: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant; reason summarised from its thread
- P5: [Filter irrelevant change before reassemble transactions during logical decoding](https://commitfest.postgresql.org/patch/5585/) (cf, Withdrawn, 2025-08-22) - judged not relevant; reason summarised from its thread
- P6: [logical decoding and replication of sequences, take 2](https://commitfest.postgresql.org/patch/3823/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P7: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2283/) (cf, Withdrawn, 2020-06-11) - judged not relevant; reason summarised from its thread
- P8: [Filter irrelevant change before reassemble transactions during logical decoding](https://commitfest.postgresql.org/patch/5592/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P9: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2683/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P10: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P11: [Logical decoding of two-phase transactions](https://commitfest.postgresql.org/patch/944/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P12: [Minimal logical decoding on standbys (take 6)](https://commitfest.postgresql.org/patch/2968/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P13: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - judged not relevant
- P14: [Improve logicalrep_worker_launch() logic](https://commitfest.postgresql.org/patch/5734/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P15: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P16: [Fix documentation and logging for START_REPLICATION](https://commitfest.postgresql.org/patch/3174/) (cf, Withdrawn, 2021-06-16) - judged not relevant
- P17: [Skip collecting decoded changes of already-aborted transactions](https://commitfest.postgresql.org/patch/4436/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P18: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P19: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - judged not relevant
- P20: [Revert "Make pg_dump emit ATTACH PARTITION instead of PARTITION OF"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a1ec7402e9a0392f95bc79542af0efcd5c6e7929) (revert, Reverted, 2019-05-06) - judged not relevant
- P21: [Inconsistent error handling in START_REPLICATION command](https://commitfest.postgresql.org/patch/474/) (cf, Rejected, 2016-03-11) - judged not relevant
- P22: [SQL function to report log message](https://commitfest.postgresql.org/patch/314/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P23: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4739/) (cf, Rejected, 2024-04-08) - judged not relevant
- P24: [LSN Map](https://commitfest.postgresql.org/patch/89/) (cf, Returned with feedback, 2015-07-06) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 157 s; cost reported by the CLI: $2.05
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 47 text field(s) checked, 3 finding(s), 3 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:00:12.416Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
