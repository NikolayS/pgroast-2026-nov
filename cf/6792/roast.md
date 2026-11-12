# Design pre-review: to_date()/to_timestamp() silently accept month=0 and day=0

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6792](https://commitfest.postgresql.org/patch/6792/); [thread](https://www.postgresql.org/message-id/flat/CAJTYsWWNfF%2BhMpbsTDF8NMr%2BAuqoDfNxR%3DoHUZm7xqGP%2BdJ9rA%40mail.gmail.com), 18 of 18 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The attached tests-only patch draws no objection. The behavior change that the CommitFest entry keeps open is disputed at the desirability level: prior-attempts-1 and standards-compat-1 show that an error for month and day 00 breaks input that has loaded for years, and that [P5] was rejected for the same kind of narrow change. No message proposes one rule for invalid to_date() input, and the objection in message #5 is unanswered. The list needs to agree on that before a behavior patch is worth writing.

Not checked: No patch that changes behavior exists, so the effect of presence flags on other fields that read zero as missing could not be judged. Only 3 of 18 messages were read directly; later messages are known through the objections.

## Prior attempts

- [Correct negative/zero year in to_date/to_timestamp](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com) (similar idea; Rejected, 2016-03-11). "But I'm not seeing the argument that hacking at the margins like this moves us forward on either point; what it does do is create another backward-compatibility hazard for any such rewrite." ([message](https://www.postgresql.org/message-id/32072.1457724660%40sss.pgh.pa.us)) Different now: P5 changed year handling and added a year-0 error. This proposal targets month and day 00, which do match the MM and DD format, and its current patch only adds tests.
- [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (related; Returned with feedback, 2016-09-29). reason not recorded Different now: P2 proposed a separate function, to_date_valid(). This proposal would change existing to_date() and to_timestamp(), and P2's recorded reason is missing.

## Objections a reviewer is likely to raise

### 1. A narrow 00 check repeats the margin-fix pattern rejected for year 0

An ETL job that loads legacy text dates with 00 placeholders through to_date() starts failing after the upgrade to v20. Other input that does not match the format still parses. A later strict-mode rewrite of to_date() then needs a second compatibility break.

[P5] was rejected because a year-0 error added at the margins kept other bad input accepted and created a backward-compatibility hazard. The same reason fits a month and day 00 error here. In message #5 the zero-as-missing logic is described as intentional, with DDD backfill depending on it, and messages #13 and #14 leave the dispute open. The tests-only v2 avoids this, but the CommitFest entry keeps the behavior change pending.

Precedent: [Correct negative/zero year in to_date/to_timestamp](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com) (Rejected, 2016-03-11)

**Question:** How does rejecting only month and day 00 fit a coherent rule for invalid to_date() input, and not add one more special case a later rewrite must keep?

<sub>fixable, level 1b, prior rejection; messages [#5](https://www.postgresql.org/message-id/1444609.1776958814%40sss.pgh.pa.us), [#13](https://www.postgresql.org/message-id/d77aae98-f563-4561-b30e-699924d4d9c6%40eisentraut.org), [#14](https://www.postgresql.org/message-id/ae6VJ-FllaLg92xw%40paquier.xyz), [#17](https://www.postgresql.org/message-id/CAJTYsWU0_CxphN%3DdfY%3DDCMO%3Dn2yg4h2wvt8W%3DwP30zAhxoiK6A%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Rejecting 00 in to_date() breaks existing loads after a major upgrade

An ETL job reads partial dates such as '2024-00-00' (year known, month and day unknown) through to_date(col, 'YYYY-MM-DD'). Today these rows load as 2024-01-01. After upgrading to v20, the whole load fails on the first such row, and the team finds out only at that point.

Message #5 states that zero is intentionally read as missing data, and message #3 shows the behavior goes back at least to v14, so applications have had years to depend on it. Message #13 answers with Oracle compatibility and Postgres date literal rules, and message #14 calls the risk hypothetical. No one shows field reports in either direction. [P5] was rejected because changing to_date at the margins creates a new compatibility hazard, and that reason applies here.

Precedent: [Correct negative/zero year in to_date/to_timestamp](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com) (Rejected, 2016-03-11)

**Question:** What evidence shows that users want 00 rejected, and what migration path would a v20 compatibility note give to users who rely on 00 meaning 'unknown'?

<sub>fixable, level 1b, SQL-standard conflict or non-standard syntax; messages [#3](https://www.postgresql.org/message-id/FE1D12FF-7E5D-4ED0-A59F-BEA6DEF7EE2C%40yesql.se), [#5](https://www.postgresql.org/message-id/1444609.1776958814%40sss.pgh.pa.us), [#13](https://www.postgresql.org/message-id/d77aae98-f563-4561-b30e-699924d4d9c6%40eisentraut.org), [#14](https://www.postgresql.org/message-id/ae6VJ-FllaLg92xw%40paquier.xyz); `standards-compat-1`</sub>

## Questions for the author

1. Does the CommitFest entry now ask only for the tests-only v2, or does it still ask for a v20 change that rejects month 00 and day 00?
2. How does rejecting only month and day 00 fit one rule for invalid to_date() input, so that a later rewrite does not have to keep one more special case?
3. Which to_date() fields would reject an explicit zero under the new rule, and how would year 0000 and the DDD backfill behave?
4. What evidence shows that users want 00 rejected, and what migration path would a v20 compatibility note give users who rely on 00 meaning "unknown"?
5. Should the entry's Bugfix tag and "stable" target change, given that the thread rules out a backpatch?

## Touch points

- **SQL-standard conflict or non-standard syntax:** A v20 change would alter what to_date() and to_timestamp() return for month 00 and day 00; the thread cites Oracle's behavior.
- **backpatchability:** The CommitFest entry is tagged Bugfix with target "stable", but the thread rules out a backpatch.
- **maintenance burden ("who maintains this in 10 years"):** An error for MM and DD only adds one more special case to the to_date() parser in formatting.c.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson reviews the entry and posted the tests-only v2, and Michael Paquier supports it. A behavior change needs someone who knows the to_date() parsing code in formatting.c.
- Priority: needs design rethink; audience niche; review cost low. The tests-only v2 touches two horology files and can go ahead as it is. The behavior change affects only users who pass 00 to to_date(), and it is stuck because no message gives a rule that covers YYYY 0000 and the DDD backfill.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
