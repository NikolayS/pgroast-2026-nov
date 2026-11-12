# Design pre-review: Fix LSN of logical decoding message

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7016](https://commitfest.postgresql.org/patch/7016/); [thread](https://www.postgresql.org/message-id/flat/d99c688994ab3a998afe26e61fe4f69f%40oss.nttdata.com), 13 of 13 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible case and a field report of a lost INSERT in Debezium (message #1). The review notes say reviewers accept the master-only change with documentation fixes in the back branches (messages #10 to #13). Confidence is medium because only 3 of 13 messages and no code were read.

Not checked: The code was not read, so it could not be checked that pgoutput and test_decoding pass the new value through the same way. Messages #4 to #13 are known here only through the brief and the review notes.

## Prior attempts

None found before 2026-06-08.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Will the next version update the message_cb text in logicaldecoding.sgml, which still says the lsn is the WAL location of the message?
2. Which regression test tells the old and new LSN apart for a transactional message, and which one covers the non-transactional case?
3. What happens to a consumer that stored a transactional message's end LSN on 19 and resumes from it after upgrade to 20?
4. Does v3 keep or drop the v1 change to 020_messages.pl?
5. Were the back-branch documentation patches for 14 and for 15 to 19 checked against each branch's actual behavior?

## Touch points

- **WAL / replication / standby:** decode.c changes which LSN logical decoding reports for transactional messages; the WAL record itself does not change.
- **wire protocol:** The Message LSN field in the logical replication protocol changes meaning for transactional messages, per the protocol.sgml hunk.
- **ABI / extension API:** The message_lsn argument of the LogicalDecodeMessageCB output plugin callback changes from end LSN to start LSN for transactional messages.
- **backpatchability:** The code change is master-only; branches 14 to 19 get a protocol.sgml correction that says the end LSN is reported.
- **maintenance burden ("who maintains this in 10 years"):** The Message LSN now has two meanings, start for transactional and end for non-transactional, which documentation and plugins must keep straight.

## Who would commit it, and when

- Owner: qualified committer engaged. The review notes say Sawada shaped the transactional/non-transactional split in message #10 and agreed to own it in message #12; those messages were not shown here.
- Priority: proceed; audience a sizeable minority; review cost low. The patch touches decode.c and protocol.sgml only and fixes a data-loss case reported for Debezium. It affects users of logical decoding who emit transactional messages, and the remaining requests are documentation, comments and a test.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
