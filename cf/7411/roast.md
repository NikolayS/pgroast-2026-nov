# Design pre-review: to_date()/to_timestamp() silently truncate 4+-digit day-of-year input to the first three digits

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7411](https://commitfest.postgresql.org/patch/7411/); [thread](https://www.postgresql.org/message-id/flat/19650-d7ea430084cdc44a%40postgresql.org), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal: the patch enforces ranges that the documentation already states, and a second tester confirmed the boundaries in message #3. Two fixable points remain. maintenance-fit-1 asks which branches the fix targets, because input accepted today would start to fail. prior-attempts-1 asks for the rule that checks ID but leaves D unchecked. A committer who knows formatting.c has to decide the back-patch question.

Not checked: The code was not read. The SSSSS minus-sign recovery and the RM leftover-digit check could change results for in-range input with unusual separators, and only the code can show that.

## Prior attempts

- [Correct negative/zero year in to_date/to_timestamp](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com) (related; Rejected, 2016-03-11). "But I'm not seeing the argument that hacking at the margins like this moves us forward on either point; what it does do is create another backward-compatibility hazard for any such rewrite." ([message](https://www.postgresql.org/message-id/32072.1457724660%40sss.pgh.pa.us)) Different now: P6 redefined what negative years mean for input that did not match the format. This patch enforces ranges that the documentation already states, but it shares the concern about partial tightening of to_date.
- [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (similar idea; Returned with feedback, 2016-09-29). reason not recorded Different now: P2 proposed a separate validating function. This patch tightens to_date and to_timestamp themselves. No reason for P2's outcome is recorded.

## Objections a reviewer is likely to raise

### 1. Behavior change for accepted input needs a stated master-only scope

An ETL job passes values such as SSSSS '-1', RM 'XIII' or ID '0' to to_timestamp() and gets a timestamp today. If the fix is back-patched, the job starts failing with an out-of-range error after a routine minor upgrade, with no release-note warning it would expect for a major version.

The patch turns input that returns a value today into an error, and Clemenza Zhang's tests in message #3 show master's current result for negative SSSSS. The thread was filed on pgsql-bugs, the CommitFest entry is tagged Bugfix, and neither message #2 nor the commit message says which branches it targets. [P6] records that changes at the margins of to_date create a backward-compatibility hazard. That case redefined results instead of adding errors for documented ranges, so it applies only to the compatibility point.

Precedent: [Correct negative/zero year in to_date/to_timestamp](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com) (Rejected, 2016-03-11)

**Question:** Is this fix meant for master only, with a release-note entry listing the inputs that now raise errors, or for back-branches too?

<sub>fixable, level 1b, backpatchability; messages [#2](https://www.postgresql.org/message-id/CAB8bMiu9%2B9H_fNCD_i3D-Q4rz6oCJJGNvtZDgx-8FKiToSO2%3Dg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAL9_%2BFGsoq65JmDePQ2E5m9XUuJtnsV9%2BA_NJvvmvpub9dr5Rg%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Partial range checks leave the reported DDDD case and D unchecked

A user runs to_date('2024 1000', 'YYYY DDDD') after the patch and still gets 2024-04-09. The patch does not list D among the checked fields, so the trailing 0 is accepted and ignored. The reporter's own example is unchanged, while other bad input now raises errors.

[P6] was rejected partly because it added an error for one bad input while other bad input was still accepted. Tom Lane saw such margin changes as a backward-compatibility hazard for a later rewrite of to_date. This patch adds errors for six fields. Message #2 explains that DDDD parses as DDD plus D, and message #3 confirms the bug's headline cases still return dates. The patch therefore changes behavior for some invalid input but not for the input that started the report.

Precedent: [Correct negative/zero year in to_date/to_timestamp](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com) (Rejected, 2016-03-11)

**Question:** Why is D left without a 1-7 range check while ID gets one, and what rule decides which invalid inputs this patch rejects?

<sub>fixable, level 2, prior rejection; messages [#1](https://www.postgresql.org/message-id/19650-d7ea430084cdc44a%40postgresql.org), [#2](https://www.postgresql.org/message-id/CAB8bMiu9%2B9H_fNCD_i3D-Q4rz6oCJJGNvtZDgx-8FKiToSO2%3Dg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAL9_%2BFGsoq65JmDePQ2E5m9XUuJtnsV9%2BA_NJvvmvpub9dr5Rg%40mail.gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. Is this fix meant for master only, or for back-branches too, where queries that return a date today would start to fail?
2. Why does D get no 1-7 range check while ID gets one, and what rule decides which invalid inputs the patch rejects?
3. What does bug #19651, cited in the commit message, report, and which part of the patch fixes it?
4. Should the documentation say that DDDD parses as DDD followed by D, since the reporter's headline cases still return dates?

## Touch points

- **backpatchability:** Input that returns a date today, such as SSSSS -1 or RM XIII, raises an error after the patch, and no target branch is named.
- **maintenance burden ("who maintains this in 10 years"):** The patch tightens six fields in formatting.c and leaves D unchecked, so the set of rejected input stays uneven.

## Who would commit it, and when

- Owner: silent. Someone who knows the to_char/to_date parser in formatting.c and the project's back-patch policy for stricter input checks.
- Priority: proceed; audience niche; review cost low. The patch is 9.1 KB in one source file plus regression tests, and it fixes silent wrong dates for documented ranges. Few users pass out-of-range values in these six fields, but those who do get wrong dates today.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
