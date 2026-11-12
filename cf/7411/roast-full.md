# Design pre-review: to_date()/to_timestamp() silently truncate 4+-digit day-of-year input to the first three digits

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7411](https://commitfest.postgresql.org/patch/7411/) (Needs review); authors: Andrey Rachitskiy (a.rachitskiy)
- Thread: [BUG #19650: to_date()/to_timestamp() silently truncate 4+-digit day-of-year input to the first three digits](https://www.postgresql.org/message-id/flat/19650-d7ea430084cdc44a%40postgresql.org), started 2026-09-03
- Messages read: 3 of 3
- Patch prose read: 0001-Reject-out-of-range-to_date-to_timestamp-template.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal: the patch enforces ranges that the documentation already states, and a second tester confirmed the boundaries in message #3. Two fixable points remain. maintenance-fit-1 asks which branches the fix targets, because input accepted today would start to fail. prior-attempts-1 asks for the rule that checks ID but leaves D unchecked. A committer who knows formatting.c has to decide the back-patch question.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. The SSSSS minus-sign recovery and the RM leftover-digit check could change results for in-range input with unusual separators, and only the code can show that.

## The proposal, restated

**Problem.** to_date() and to_timestamp() accept some template field values outside their documented ranges and return a date instead of an error. The bug report shows IDDD 999 accepted, and the author lists DDD, IDDD, SSSSS, RM, IW and ID as affected. Users who rely on these functions to validate input get wrong dates silently.

**Proposed design.** The patch checks each affected field against its documented range when the field is parsed, because a later 0 in TmFromChar means the field was unset. It recovers a minus sign swallowed as a separator before SSSSS, as TZH already does. After RM, it rejects a leftover roman digit so XIII is not read as XII.

**User-visible surface.**
- to_date() and to_timestamp() raise an out-of-range error for IDDD outside 001-371 and DDD outside 001-366.
- SSSSS outside 0-86399 now errors, including negative input that unpatched master reads as a positive value.
- RM values beyond XII, such as XIII, now error instead of being read as XII.
- IW outside 01-53 and ID outside 1-7 now error, including 0.

**What the author claims.**
- The listed fields accepted values outside their documented domains.
- to_date('2024 1000', 'YYYY DDDD') is not a DDD overflow, because DDDD parses as DDD followed by D, which is ignored.
- WW 54 and above already errored, and the patch checks week fields uniformly.
- The patch includes regression tests.

**Thread state.** The author replied to the bug report with a patch. Clemenza Zhang tested it on master, confirmed the boundaries and unchanged in-range ISO normalization, and reported passing tests; no committer has replied.

## 2. Prior attempts

### Correct negative/zero year in to_date/to_timestamp (related)

- **Link:** [https://commitfest.postgresql.org/patch/533/](https://commitfest.postgresql.org/patch/533/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com)
- **Date:** decided 2016-03-11; started 2016-02-22
- **Outcome:** Rejected (cf)
- **Who objected:** Robert Haas, Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Robert Haas and Tom Lane held that input such as '-2-06-01' does not match the format 'YYYY-MM-DD', so its current result is an implementation artifact that should not be redefined. Tom Lane added that the patch adds an error for year zero while still accepting other bad input, and that changes at the margins create another backward-compatibility hazard for a later rewrite of to_date. Both voted to reject, and the author accepted that.
- **Reason (quote):** "But I'm not seeing the argument that hacking at the margins like this moves us forward on either point; what it does do is create another backward-compatibility hazard for any such rewrite." ([message](https://www.postgresql.org/message-id/32072.1457724660%40sss.pgh.pa.us))
- **What is different this time:** P6 redefined what negative years mean for input that did not match the format. This patch enforces ranges that the documentation already states, but it shares the concern about partial tightening of to_date.

### to_date_valid() (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/662/](https://commitfest.postgresql.org/patch/662/)
- **Date:** decided 2016-09-29; started 2016-07-05
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P2 proposed a separate validating function. This patch tightens to_date and to_timestamp themselves. No reason for P2's outcome is recorded.


## 3. Objections a reviewer is likely to raise

### 1. Behavior change for accepted input needs a stated master-only scope

fixable · level 1b, rubric item 6 · backpatchability · `maintenance-fit-1`

An ETL job passes values such as SSSSS '-1', RM 'XIII' or ID '0' to to_timestamp() and gets a timestamp today. If the fix is back-patched, the job starts failing with an out-of-range error after a routine minor upgrade, with no release-note warning it would expect for a major version.

The patch turns input that returns a value today into an error, and Clemenza Zhang's tests in message #3 show master's current result for negative SSSSS. The thread was filed on pgsql-bugs, the CommitFest entry is tagged Bugfix, and neither message #2 nor the commit message says which branches it targets. [P6] records that changes at the margins of to_date create a backward-compatibility hazard. That case redefined results instead of adding errors for documented ranges, so it applies only to the compatibility point.

**Precedents.**
- [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (Rejected, 2016-03-11); thread: [link](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com)

Thread: [#2 (Andrey Rachitskiy, 2026-09-03)](https://www.postgresql.org/message-id/CAB8bMiu9%2B9H_fNCD_i3D-Q4rz6oCJJGNvtZDgx-8FKiToSO2%3Dg%40mail.gmail.com), [#3 (Clemenza Zhang, 2026-09-09)](https://www.postgresql.org/message-id/CAL9_%2BFGsoq65JmDePQ2E5m9XUuJtnsV9%2BA_NJvvmvpub9dr5Rg%40mail.gmail.com)

**Question:** Is this fix meant for master only, with a release-note entry listing the inputs that now raise errors, or for back-branches too?

Counter-check: held. The commit message and messages #2 and #3 do not name a target branch, and the entry is tagged Bugfix on a pgsql-bugs thread. Message #3 confirms that input accepted today, such as SSSSS -1, would start to fail.

### 2. Partial range checks leave the reported DDDD case and D unchecked

fixable · level 2, rubric item 6 · prior rejection, backpatchability, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A user runs to_date('2024 1000', 'YYYY DDDD') after the patch and still gets 2024-04-09. The patch does not list D among the checked fields, so the trailing 0 is accepted and ignored. The reporter's own example is unchanged, while other bad input now raises errors.

[P6] was rejected partly because it added an error for one bad input while other bad input was still accepted. Tom Lane saw such margin changes as a backward-compatibility hazard for a later rewrite of to_date. This patch adds errors for six fields. Message #2 explains that DDDD parses as DDD plus D, and message #3 confirms the bug's headline cases still return dates. The patch therefore changes behavior for some invalid input but not for the input that started the report.

**Precedents.**
- [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (Rejected, 2016-03-11); thread: [link](https://www.postgresql.org/message-id/flat/CAKOSWNmwCH0wx6MApc1A8ww++EQmG07AZ3t6w_XjRrV1xeZpTA@mail.gmail.com)

Thread: [#1 (PG Bug reporting form, 2026-09-03)](https://www.postgresql.org/message-id/19650-d7ea430084cdc44a%40postgresql.org), [#2 (Andrey Rachitskiy, 2026-09-03)](https://www.postgresql.org/message-id/CAB8bMiu9%2B9H_fNCD_i3D-Q4rz6oCJJGNvtZDgx-8FKiToSO2%3Dg%40mail.gmail.com), [#3 (Clemenza Zhang, 2026-09-09)](https://www.postgresql.org/message-id/CAL9_%2BFGsoq65JmDePQ2E5m9XUuJtnsV9%2BA_NJvvmvpub9dr5Rg%40mail.gmail.com)

**Question:** Why is D left without a 1-7 range check while ID gets one, and what rule decides which invalid inputs this patch rejects?

Counter-check: held. Message #2 explains how DDDD parses but gives no rule for leaving D unchecked while ID gets a 1-7 check. [P6] is weak support, because its input did not match the format, while these fields have documented ranges.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Input that returns a date today, such as SSSSS -1 or RM XIII, raises an error after the patch, and no target branch is named. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch tightens six fields in formatting.c and leaves D unchecked, so the set of rejected input stays uneven. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is this fix meant for master only, or for back-branches too, where queries that return a date today would start to fail?
2. Why does D get no 1-7 range check while ID gets one, and what rule decides which invalid inputs the patch rejects?
3. What does bug #19651, cited in the commit message, report, and which part of the patch fixes it?
4. Should the documentation say that DDDD parses as DDD followed by D, since the reporter's headline cases still return dates?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the to_char/to_date parser in formatting.c and the project's back-patch policy for stricter input checks.
- **Liability:** The owner signs for every query that starts to fail on the six fields, including immutable wrappers used in indexes or CHECK constraints. If back-patched, that covers all supported branches.
- **Author follow-up:** The author answered the bug report with a patch the same day and opened the CommitFest entry; no later reply is visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 9.1 KB in one source file plus regression tests, and it fixes silent wrong dates for documented ranges. Few users pass out-of-range values in these six fields, but those who do get wrong dates today.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Stricter parsing breaks queries that returned dates; target branch not stated. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same branch-target question and limits its use of [P6] more carefully. The missing release-note hunk is not a gap, because committers write release notes at release time.

### Review angles

- prior-attempts: 1 objection(s). One objection raised. [P1] (WW-D parsing) is a different idea and is not listed.
- standards-compat: 1 objection(s). Apart from the back-branch question, the change matches documented ranges and adds no syntax, GUC or dump surface.
- replication-ondisk: 0 objection(s). The patch changes only how to_date() and to_timestamp() parse text input in formatting.c. It changes no on-disk format, WAL record, catalog, wire protocol message or exported struct, so pg_upgrade, standbys and subscribers are not affected. One edge case remains, and it is a compatibility question for back-branches. A user who wraps these functions in an immutable function for an index, generated column or CHECK constraint may hit new errors on reindex or restore. The same holds for any stricter input check, and the backpatch angle should weigh it.
- security: 0 objection(s). The patch only makes to_date() and to_timestamp() reject out-of-range template field values, so it has no privilege, exposure, or denial-of-service aspect. It adds no new privilege, object, setting, or code path that an unprivileged user could use to see or do more than before, and the error text echoes only the caller's own input.
- maintenance-fit: 1 objection(s). From the maintenance and fit angle the design is small and fits core. It adds range checks in formatting.c of the kind that already exist for DDD 367 and WW 54, and the SSSSS sign recovery copies TZH. It has no hot-path cost outside these functions, adds no new API or dependency, and creates no state that needs cleanup. Only the branch scope stands as an objection.
- already-exists: 0 objection(s). No objection from this angle. Postgres has no other function, option or extension that range-checks DDD, IDDD, SSSSS, RM, IW or ID in to_date() and to_timestamp() templates. The patch adds checks inside the existing parser and does not add a second mechanism. The margin-change concern in [P6] is about compatibility, which another panel member covers.
- demand: 0 objection(s). No demand objection. The patch fixes cases where to_date() and to_timestamp() return a wrong date silently instead of the documented error. A user report shows one such case (IDDD 999), and a second tester confirmed the others in message #3. Fixing documented behaviour does not need proof of a large user group. The cost is a possible break for queries that depend on the lenient parsing, and [P6] shows committers weighing that kind of risk. That question belongs to the compatibility angle, not to demand.

### Precedents retrieved

- P1: [Recognize D part for to_date() and to_timestamp() in 'WW-D' pattern](https://commitfest.postgresql.org/patch/2389/) (cf, Returned with feedback, 2020-07-10) - used in the review
- P2: [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (cf, Returned with feedback, 2016-09-29) - used in the review
- P3: [Refactor compile-time assertion checks for C/C++](https://commitfest.postgresql.org/patch/2437/) (cf, Returned with feedback, 2020-03-23) - judged not relevant; reason summarised from its thread
- P4: [invalid bit/varbit input causes the log file to be unreadable](https://commitfest.postgresql.org/patch/2618/) (cf, Withdrawn, 2020-06-30) - judged not relevant; reason summarised from its thread
- P5: [This adds a new GUC parameter that automatically rolls back prepared transactions that have remained unresolved beyond the configured timeout. The cleanup is performed by the checkpointer process.](https://commitfest.postgresql.org/patch/6611/) (cf, Rejected, 2026-07-01) - judged not relevant; reason summarised from its thread
- P6: [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (cf, Rejected, 2016-03-11) - used in the review; reason summarised from its thread
- P7: [Free list same_input_transnos in preprocess_aggref](https://commitfest.postgresql.org/patch/3920/) (cf, Withdrawn, 2022-11-07) - judged not relevant; reason summarised from its thread
- P8: [Revert "Refactor compile-time assertion checks in c.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa650a99b4bfbe95e88f44070ae0df1818944a7) (revert, Reverted, 2020-03-23) - judged not relevant
- P9: [Add generate_series(date,date,\[step\])](https://commitfest.postgresql.org/patch/492/) (cf, Returned with feedback, 2016-03-17) - judged not relevant; reason summarised from its thread
- P10: [Fix for PL/Python slow input arrays traversal issue](https://commitfest.postgresql.org/patch/687/) (cf, Rejected, 2016-09-22) - judged not relevant; reason summarised from its thread
- P11: [Revert "Reject degenerate SPLIT PARTITION with DEFAULT partition"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0392fb900eb89f52988cccd33046443c39c70d1c) (revert, Reverted, 2026-05-20) - judged not relevant
- P12: [Some refactoring for fixed-numbered stats template in injection_points](https://commitfest.postgresql.org/patch/5187/) (cf, Withdrawn, 2024-10-02) - judged not relevant; reason summarised from its thread
- P13: [Revert "Revert part of commit dddf4cdc3."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ee201520707596b058e6bbfab0a5e2006f3834c5) (revert, Reverted, 2019-10-25) - judged not relevant
- P14: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant
- P15: [Upper limit arguments of pg_logical_slot_xxx_changes functions accept invalid values](https://commitfest.postgresql.org/patch/1725/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P16: [Protect syscache from bloating with negative cache entries](https://commitfest.postgresql.org/patch/931/) (cf, Returned with feedback, 2021-10-05) - judged not relevant
- P17: [Add Roman numeral conversion to to_number](https://commitfest.postgresql.org/patch/1227/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P18: [Revert "Allow parallel workers to cope with a newly-created session user ID."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e6a9637488e2673efb87f8ead657789e9889fb17) (revert, Reverted, 2024-08-01) - judged not relevant
- P19: [Revert part of commit dddf4cdc3.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c114229ca2519620703a4be4e38181290cad8c0a) (revert, Reverted, 2019-10-25) - judged not relevant
- P20: [Replace literal 0 values with the appropriate Invalid* constants](https://commitfest.postgresql.org/patch/6480/) (cf, Withdrawn, 2026-02-13) - judged not relevant
- P21: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P22: [Add roman support for to_number function](https://commitfest.postgresql.org/patch/5233/) (cf, Withdrawn, 2024-09-03) - judged not relevant
- P23: [pg_basebackup has stricter tablespace rules than the server](https://commitfest.postgresql.org/patch/1125/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P24: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant

### Warnings

- review angle "standards-compat" returned prior attempts; ignored (only the prior-attempts angle may)
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (maintenance-fit-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 97 s; cost reported by the CLI: $1.06
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 63 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:52:58.692Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
