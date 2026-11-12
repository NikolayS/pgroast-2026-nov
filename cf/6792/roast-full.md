# Design pre-review: to_date()/to_timestamp() silently accept month=0 and day=0

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6792](https://commitfest.postgresql.org/patch/6792/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [to_date()/to_timestamp() silently accept month=0 and day=0](https://www.postgresql.org/message-id/flat/CAJTYsWWNfF%2BhMpbsTDF8NMr%2BAuqoDfNxR%3DoHUZm7xqGP%2BdJ9rA%40mail.gmail.com), started 2026-04-22
- Messages read: 18 of 18
- Patch prose read: 0001-Add-regression-tests-for-to_date-to_timestamp-zero-m.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The attached tests-only patch draws no objection. The behavior change that the CommitFest entry keeps open is disputed at the desirability level: prior-attempts-1 and standards-compat-1 show that an error for month and day 00 breaks input that has loaded for years, and that [P5] was rejected for the same kind of narrow change. No message proposes one rule for invalid to_date() input, and the objection in message #5 is unanswered. The list needs to agree on that before a behavior patch is worth writing.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: No patch that changes behavior exists, so the effect of presence flags on other fields that read zero as missing could not be judged. Only 3 of 18 messages were read directly; later messages are known through the objections.

## The proposal, restated

**Problem.** to_date() and to_timestamp() accept month 00 and day 00 and turn them into January and day 1 without an error. The reporter expected an out-of-range error, as for an invalid date literal. The thread sees this as a surprise for users who pass bad input, but also as behavior that existing applications may rely on.

**Proposed design.** The reporter first suggested presence flags in TmFromChar, so that an explicit 00 is told apart from an omitted field and ValidateDate() rejects it. The attached patch changes no behavior. It adds regression tests in horology that record the current mapping of 00 to 1, and Daniel Gustafsson's v2 trims and moves these tests.

**User-visible surface.**
- No visible change in the attached patch; it only adds regression tests.
- If the behavior change goes ahead, to_date() and to_timestamp() would raise an error for month 00 or day 00 instead of returning month 1 or day 1.

**What the author claims.**
- to_date('2024-00-15', 'YYYY-MM-DD') returns 2024-01-15, and day 00 returns day 1, on master.
- The cause is that do_to_timestamp() copies tmfc.mm and tmfc.dd only when they are nonzero, so ZERO_tm() defaults remain.
- The commit message says zero is intentionally treated as missing data, matching ZERO_tm() initialization.
- Oracle rejects these inputs, according to Peter Eisentraut.
- The same behavior exists in v14, according to Daniel Gustafsson.

**Thread state.** All agree on no backpatch. Tom Lane opposes a behavior change and Peter Eisentraut supports rejection, while Michael Paquier would not object on HEAD for v20. The tests-only v2 has support from Daniel and Michael, but no one has posted a patch that changes behavior.

## 2. Prior attempts

### Correct negative/zero year in to_date/to_timestamp (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/533/](https://commitfest.postgresql.org/patch/533/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com)
- **Date:** decided 2016-03-11; started 2016-02-22
- **Outcome:** Rejected (cf)
- **Who objected:** Robert Haas, Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Robert Haas and Tom Lane held that input such as '-2-06-01' does not match the format 'YYYY-MM-DD', so its current result is an implementation artifact that should not be redefined. Tom Lane added that the patch adds an error for year zero while still accepting other bad input, and that changes at the margins create another backward-compatibility hazard for a later rewrite of to_date. Both voted to reject, and the author accepted that.
- **Reason (quote):** "But I'm not seeing the argument that hacking at the margins like this moves us forward on either point; what it does do is create another backward-compatibility hazard for any such rewrite." ([message](https://www.postgresql.org/message-id/32072.1457724660%40sss.pgh.pa.us))
- **What is different this time:** P5 changed year handling and added a year-0 error. This proposal targets month and day 00, which do match the MM and DD format, and its current patch only adds tests.

### to_date_valid() (related)

- **Link:** [https://commitfest.postgresql.org/patch/662/](https://commitfest.postgresql.org/patch/662/)
- **Date:** decided 2016-09-29; started 2016-07-05
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P2 proposed a separate function, to_date_valid(). This proposal would change existing to_date() and to_timestamp(), and P2's recorded reason is missing.


## 3. Objections a reviewer is likely to raise

### 1. A narrow 00 check repeats the margin-fix pattern rejected for year 0

fixable · level 1b, rubric item 6 · prior rejection, backpatchability, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

An ETL job that loads legacy text dates with 00 placeholders through to_date() starts failing after the upgrade to v20. Other input that does not match the format still parses. A later strict-mode rewrite of to_date() then needs a second compatibility break.

[P5] was rejected because a year-0 error added at the margins kept other bad input accepted and created a backward-compatibility hazard. The same reason fits a month and day 00 error here. In message #5 the zero-as-missing logic is described as intentional, with DDD backfill depending on it, and messages #13 and #14 leave the dispute open. The tests-only v2 avoids this, but the CommitFest entry keeps the behavior change pending.

**Precedents.**
- [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (Rejected, 2016-03-11); thread: [link](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com)

Thread: [#5 (Tom Lane, 2026-04-23)](https://www.postgresql.org/message-id/1444609.1776958814%40sss.pgh.pa.us), [#13 (Peter Eisentraut, 2026-04-24)](https://www.postgresql.org/message-id/d77aae98-f563-4561-b30e-699924d4d9c6%40eisentraut.org), [#14 (Michael Paquier, 2026-04-26)](https://www.postgresql.org/message-id/ae6VJ-FllaLg92xw%40paquier.xyz), [#17 (Ayush Tiwari, 2026-04-30)](https://www.postgresql.org/message-id/CAJTYsWU0_CxphN%3DdfY%3DDCMO%3Dn2yg4h2wvt8W%3DwP30zAhxoiK6A%40mail.gmail.com)

**Question:** How does rejecting only month and day 00 fit a coherent rule for invalid to_date() input, and not add one more special case a later rewrite must keep?

Counter-check: held. In [P5], '-2' did not match YYYY, while 00 does match MM and DD, so that part of [P5] does not carry over. Tom Lane's point about changes at the margins does apply, and no message proposes a rule for other invalid input.

### 2. Rejecting 00 in to_date() breaks existing loads after a major upgrade

fixable · level 1b, rubric item 6 · SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-1`

An ETL job reads partial dates such as '2024-00-00' (year known, month and day unknown) through to_date(col, 'YYYY-MM-DD'). Today these rows load as 2024-01-01. After upgrading to v20, the whole load fails on the first such row, and the team finds out only at that point.

Message #5 states that zero is intentionally read as missing data, and message #3 shows the behavior goes back at least to v14, so applications have had years to depend on it. Message #13 answers with Oracle compatibility and Postgres date literal rules, and message #14 calls the risk hypothetical. No one shows field reports in either direction. [P5] was rejected because changing to_date at the margins creates a new compatibility hazard, and that reason applies here.

**Precedents.**
- [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (Rejected, 2016-03-11); thread: [link](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com)

Thread: [#3 (Daniel Gustafsson, 2026-04-23)](https://www.postgresql.org/message-id/FE1D12FF-7E5D-4ED0-A59F-BEA6DEF7EE2C%40yesql.se), [#5 (Tom Lane, 2026-04-23)](https://www.postgresql.org/message-id/1444609.1776958814%40sss.pgh.pa.us), [#13 (Peter Eisentraut, 2026-04-24)](https://www.postgresql.org/message-id/d77aae98-f563-4561-b30e-699924d4d9c6%40eisentraut.org), [#14 (Michael Paquier, 2026-04-26)](https://www.postgresql.org/message-id/ae6VJ-FllaLg92xw%40paquier.xyz)

**Question:** What evidence shows that users want 00 rejected, and what migration path would a v20 compatibility note give to users who rely on 00 meaning 'unknown'?

Counter-check: held. The thread limits the change to a major release, which is where Postgres makes such breaks, and message #14 weighs the risk. But Tom Lane's objection in message #5 is unanswered, and no compatibility note or usage evidence is offered.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | A v20 change would alter what to_date() and to_timestamp() return for month 00 and day 00; the thread cites Oracle's behavior. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry is tagged Bugfix with target "stable", but the thread rules out a backpatch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | An error for MM and DD only adds one more special case to the to_date() parser in formatting.c. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the CommitFest entry now ask only for the tests-only v2, or does it still ask for a v20 change that rejects month 00 and day 00?
2. How does rejecting only month and day 00 fit one rule for invalid to_date() input, so that a later rewrite does not have to keep one more special case?
3. Which to_date() fields would reject an explicit zero under the new rule, and how would year 0000 and the DDD backfill behave?
4. What evidence shows that users want 00 rejected, and what migration path would a v20 compatibility note give users who rely on 00 meaning "unknown"?
5. Should the entry's Bugfix tag and "stable" target change, given that the thread rules out a backpatch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson reviews the entry and posted the tests-only v2, and Michael Paquier supports it. A behavior change needs someone who knows the to_date() parsing code in formatting.c.
- **Liability:** For the tests-only v2, close to nothing. For a v20 behavior change, the owner signs for upgrade breakage reports from users whose 00 input used to load, and for defending the special case in any later to_date() rewrite.
- **Author follow-up:** The author moved the entry to two later CommitFests. No behavior-change patch has been posted.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The tests-only v2 touches two horology files and can go ahead as it is. The behavior change affects only users who pass 00 to to_date(), and it is stuck because no message gives a rule that covers YYYY 0000 and the DDD backfill.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Erroring on month and day 00 but not year 0000 is inconsistent. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-1` Rejecting only 00 month/day is a narrow change to a lenient parser. Refuted (not specific): Duplicate of standards-compat-2. Its request for one rule covering YYYY and the DDD backfill is the same, and its point about a later rewrite is covered by prior-attempts-1.
- `demand-1` No user report shows that the 00 mapping causes harm. Refuted (already handled): The thread has the demand evidence the rubric names: a pgsql-bugs report, Oracle rejecting these inputs, and Peter Eisentraut's support in message #13. The upgrade-break scenario is the one in standards-compat-1, which survives.

### Review angles

- prior-attempts: 1 objection(s). The tests-only v2 matches no precedent reason. The objection applies only to the proposed v20 behavior change.
- standards-compat: 2 objection(s). The attached patch only adds tests, and that raises no compatibility concern. Both objections are about the v20 behavior change that the thread discusses but that no patch implements yet.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state; the attached patch only adds horology regression tests. A later v20 change would affect only the parsing of input text, and stored date and timestamp values are already normalized. to_date() and to_timestamp() are not immutable, so expression indexes and generated columns cannot depend on them. Nothing here touches the on-disk format, WAL, catalogs, the wire protocol or ABI.
- security: 0 objection(s). The patch only adds regression tests that record how to_date() and to_timestamp() map 00 to 1. A later change to raise an error would affect only input validation, so neither version changes who can call these functions, what a caller can see, or what resources a caller can use. I see no security or privilege objection from this angle.
- maintenance-fit: 1 objection(s). The tests-only v2 adds little maintenance cost, and Daniel Gustafsson and Michael Paquier support it. The objection applies only to the behavior change that the CommitFest entry keeps open for v20.
- already-exists: 0 objection(s). The attached patch only adds regression tests, so it duplicates nothing. A cast such as '2024-00-15'::date rejects month 00, but only for formats the date input parser accepts, so it does not cover arbitrary to_date() format strings. No contrib module or extension does this job, and a to_char() round-trip check is a user workaround that cannot change the default.
- demand: 1 objection(s). The tests-only v2 has no demand problem because it is cheap and changes no behavior. The objection covers only the proposed v20 behavior change.

### Precedents retrieved

- P1: [Recognize D part for to_date() and to_timestamp() in 'WW-D' pattern](https://commitfest.postgresql.org/patch/2389/) (cf, Returned with feedback, 2020-07-10) - judged not relevant
- P2: [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (cf, Returned with feedback, 2016-09-29) - used in the review
- P3: [Add generate_series(date,date,\[step\])](https://commitfest.postgresql.org/patch/492/) (cf, Returned with feedback, 2016-03-17) - judged not relevant; reason summarised from its thread
- P4: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P5: [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (cf, Rejected, 2016-03-11) - used in the review; reason summarised from its thread
- P6: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P7: [explain: show JIT details in non-text format, even if zero](https://commitfest.postgresql.org/patch/2766/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P8: [invalid bit/varbit input causes the log file to be unreadable](https://commitfest.postgresql.org/patch/2618/) (cf, Withdrawn, 2020-06-30) - judged not relevant; reason summarised from its thread
- P9: [allow 'semester' in functions EXTRACT, DATE_PART, TO_CHAR and so on.](https://commitfest.postgresql.org/patch/326/) (cf, Rejected, 2015-08-15) - judged not relevant; reason summarised from its thread
- P10: [Revert "Tweak "line" test to avoid negative zeros on some platforms"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f7a47912af2206698d14650f2149fa951b8ce07) (revert, Reverted, 2013-10-25) - judged not relevant
- P11: [to_char: revert cc0d90b73b2e6dd2f301d46818a7265742c41a14](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=33a2c5ecd60a2cf68cfcdd37916f3989bb956581) (revert, Partially reverted, 2015-03-23) - judged not relevant
- P12: [Revert "Get rid of the "new" and "old" entries in a view's rangetable."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0e6d6d3c909e14eedd370dd77d7111c2ae24c02) (revert, Reverted, 2023-01-12) - judged not relevant
- P13: [TimestampTz->Text->TimestampTz casting fails with DateStyle 'Postgres'](https://commitfest.postgresql.org/patch/5371/) (cf, Returned with feedback, 2025-01-07) - judged not relevant; reason summarised from its thread
- P14: [Useless field ispartitioned in CreateStmtContext](https://commitfest.postgresql.org/patch/5340/) (cf, Rejected, 2024-11-28) - judged not relevant; reason summarised from its thread
- P15: [Upper limit arguments of pg_logical_slot_xxx_changes functions accept invalid values](https://commitfest.postgresql.org/patch/1725/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P16: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P17: [pg_basebackup has stricter tablespace rules than the server](https://commitfest.postgresql.org/patch/1125/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P18: [Free list same_input_transnos in preprocess_aggref](https://commitfest.postgresql.org/patch/3920/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P19: [Parse dates that include CE and BCE](https://commitfest.postgresql.org/patch/3682/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P20: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant
- P21: [Add function - pg_accept_connections_start_time()](https://commitfest.postgresql.org/patch/5576/) (cf, Returned with feedback, 2025-07-25) - judged not relevant
- P22: [Revert commit 66c0185a3 and follow-on patches.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7204f35919b7e021e8d1bc9f2d76fd6bfcdd2070) (revert, Reverted, 2024-05-20) - judged not relevant
- P23: [Table AM modifications to accept column projection lists](https://commitfest.postgresql.org/patch/2922/) (cf, Withdrawn, 2021-06-30) - judged not relevant
- P24: [Fix to_char() behaviour with 'WW-D' pattern](https://commitfest.postgresql.org/patch/2388/) (cf, Returned with feedback, 2020-07-10) - judged not relevant

### Warnings

- review angle "maintenance-fit" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 131 s; cost reported by the CLI: $1.67
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 74 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:03:56.449Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
