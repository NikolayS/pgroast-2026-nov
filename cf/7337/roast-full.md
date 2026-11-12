# Design pre-review: pg_walinspect: fix LSN validation messages and empty range handling

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7337](https://commitfest.postgresql.org/patch/7337/) (Needs review); authors: Chao Li (chaol)
- Thread: [pg_walinspect: fix LSN validation messages and empty range handling](https://www.postgresql.org/message-id/flat/7B8F5F12-98A5-4618-867A-904EA1334FD4%40gmail.com), started 2026-09-20
- Messages read: 16 of 16
- Patch prose read: v5-0001-Fix-input-LSN-validation-error-messages-in-pg_wal.patch, v5-0002-Fix-inaccurate-inclusive-bound-error-messages.patch, v1-0001-Fix-inaccurate-LSN-validation-messages-in-pg_wali.patch, v1-0002-Handle-empty-LSN-ranges-in-pg_walinspect.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The patches reword error messages so they match checks that already accept equality, and change no check, SQLSTATE, catalog or format. The withdrawn v1-0002 was the only behaviour change. Confidence is medium because the code was not read and no committer has replied in the visible thread.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so it is unverified that each reworded message matches its comparison and that no other site has the same mismatch. Stability of the new boundary tests in subscription.sql and 001_basic.pl was also not checked.

## The proposal, restated

**Problem.** Commit 5c1b6628075a relaxed pg_walinspect LSN checks to accept equality but left messages saying "must be less than". ALTER SUBSCRIPTION ... SKIP and psql's ParseVariableDouble() have the same mismatch between an inclusive check and a strict-sounding message. Users get error text that describes the wrong condition.

**Proposed design.** Reword the messages to "less than or equal to" or "greater than or equal to" so they match the existing checks; no check changes. The second patch covers the two core sites and adds tests for the accepted boundary values. The original v1-0002, which returned zero rows for empty LSN ranges without starting a WAL reader, was withdrawn.

**User-visible surface.**
- pg_walinspect error messages for input LSN versus current LSN, and start LSN versus end LSN, now say "less than or equal to".
- ALTER SUBSCRIPTION ... SKIP error now says the skip LSN must be "greater than or equal to" the origin LSN.
- psql variable range messages from ParseVariableDouble() (for example WATCH_INTERVAL) now describe inclusive [min,max] bounds.

**What the author claims.**
- The message mismatch was an oversight in 5c1b6628075a.
- The new wording matches wording used elsewhere in the tree.
- Back-patching is not needed because the fix is not critical and the inaccuracy went unnoticed for years.
- Current pg_walinspect documentation already allows an empty range to error when start_lsn is not available, so the empty-range change was dropped.

**Thread state.** Kiran Kaki, Bharath Rupireddy, Surya Poondla and Xuneng Zhou reviewed and tested v5 and call it ready for committer. Bharath CC'd Michael on back-patching; no committer has replied.

## 2. Prior attempts

### Fix for REFRESH MATERIALIZED VIEW ownership error message (related)

- **Link:** [https://commitfest.postgresql.org/patch/1766/](https://commitfest.postgresql.org/patch/1766/)
- **Thread:** [https://www.postgresql.org/message-id/flat/B611CA13-0AF0-4FD8-9A33-0CC44F3B861B@postgresql.org](https://www.postgresql.org/message-id/flat/B611CA13-0AF0-4FD8-9A33-0CC44F3B861B@postgresql.org)
- **Date:** decided 2018-08-19; started 2018-08-17
- **Outcome:** Rejected (cf)
- **Who objected:** Alvaro Herrera (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers considered the change cosmetic and argued against back-patching: the existing message is not wrong, and changing behavior in released branches can affect applications that depend on the current text. While writing a regression test, the author found that HEAD/v11 already produced the correct message because of an earlier commit. With no back-patch planned, nothing was left to fix, and the author withdrew the patch.
- **Reason (quote):** "As the plan is not to backpatch, I’ll withdraw my patch." ([message](https://www.postgresql.org/message-id/50F95E32-26E4-4A47-BD66-90C8D0758692%40postgresql.org))
- **What is different this time:** P5 changed a message that was not wrong, and stopped once back-patching was ruled out. Here the messages misstate the inclusive checks on master, and the commit messages already say not back-patched.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch rewords the ALTER SUBSCRIPTION ... SKIP error in subscriptioncmds.c; the check itself does not change. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | ParseVariableDouble() range messages in psql/variables.c change, for example for WATCH_INTERVAL. |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets "stable" while the commit message says not back-patched. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Changed message strings need new translations and affect anyone who matches on the error text. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should the CommitFest target be changed from "stable" to match the commit message, which says the fix is not back-patched?
2. Should 0002, which touches subscriptioncmds.c and psql, go in a separate commit and thread subject from the pg_walinspect fix?
3. Beyond Surya's survey of "must be less than" messages, were "must be greater than" and similar strict wordings also checked against their comparisons?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Any committer can own the message changes; one who knows pg_walinspect or logical replication would confirm the checks are meant to be inclusive. Michael was CC'd on back-patching but has not replied.
- **Liability:** Small: a few changed message strings on master and two new boundary tests. The owner signs for the wording matching the checks and for the new tests being stable.
- **Author follow-up:** The author answered reviews promptly, withdrew 0002 on his own finding, and posted five versions in about a month.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Two patches of about 6 KiB each change only message text and tests, and four reviewers call v5 ready for committer. Few users hit these boundary errors, but the cost of committing is also small.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The only precedent, [P5], died over back-patching a cosmetic message change; this proposal fixes wrong text on master only, so that reason does not apply. The withdrawn empty-range change (message #3) was dropped by the author, and the thread cites no earlier attempt at either change.
- standards-compat: 0 objection(s). The patches change only error message text to match existing checks, keep the SQLSTATE, and go to master only, so no query, dump or client sees different behaviour. The 'stable' target in the CommitFest entry conflicts with the commit message, but no back-patch is planned, and [P5] advises against one anyway.
- replication-ondisk: 0 objection(s). The proposal changes only error message text in pg_walinspect, ALTER SUBSCRIPTION ... SKIP and psql, and adds tests. It touches no on-disk format, WAL record, catalog, wire protocol message or extension ABI, and the withdrawn v1-0002 was the only behaviour change. Changed message strings matter only for translators and for anyone matching error text, which is outside this angle.
- security: 0 objection(s). The patch only rewords error messages so they match existing inclusive checks. It changes no check, privilege, default or exposed data. The messages already reported the same LSN values before, so no new information leaks. The new subscription test runs as the existing superuser test role and grants nothing new.
- maintenance-fit: 0 objection(s). From the maintenance angle there is nothing to object to. The patch rewords error messages to match checks already in the tree, adds no GUC, catalog change, API or hot-path work, and uses wording that already exists in core. Translators get the usual string churn of a major release. [P5] objected to back-patching a cosmetic message change, and this proposal is not back-patched, so [P5] does not apply. The CommitFest target of "stable" disagrees with the commit message, which is a tracking mismatch, not a design problem. No committer has engaged yet, but a fix this small needs little expertise to own.
- already-exists: 0 objection(s). The proposal rewords error messages so they match checks already in core and in pg_walinspect. No existing feature, extension or tool gives users an accurate message, so there is nothing for this angle to point to. [P5] was about cosmetic back-patching of a message that was not wrong; here the messages are wrong, and the patch is not back-patched.
- demand: 0 objection(s). The demand test does not fit here. The patch rewords error messages that describe the wrong condition, adds no feature, and changes no check. The only cost is translator churn on a few strings, and correct error text helps every user who hits these errors. [P5] objected to a cosmetic change and to back-patching it; this proposal fixes inaccurate text and is not back-patched.

### Precedents retrieved

- P1: [Mention column name in error messages](https://commitfest.postgresql.org/patch/324/) (cf, Returned with feedback, 2015-12-22) - judged not relevant; reason summarised from its thread
- P2: [Re: MarkBufferDirtyHint() and LSN update](https://commitfest.postgresql.org/patch/2352/) (cf, Withdrawn, 2019-11-25) - judged not relevant; reason summarised from its thread
- P3: [Mention column name in error messages](https://commitfest.postgresql.org/patch/818/) (cf, Returned with feedback, 2016-11-29) - judged not relevant; reason summarised from its thread
- P4: [LSN Map](https://commitfest.postgresql.org/patch/89/) (cf, Returned with feedback, 2015-07-06) - judged not relevant; reason summarised from its thread
- P5: [Fix for REFRESH MATERIALIZED VIEW ownership error message](https://commitfest.postgresql.org/patch/1766/) (cf, Rejected, 2018-08-19) - used in the review; reason summarised from its thread
- P6: [NLS: use gettext() to translate system error messages](https://commitfest.postgresql.org/patch/6172/) (cf, Rejected, 2026-03-24) - judged not relevant; reason summarised from its thread
- P7: [Align verify_heapam.c error message offset with test expectations](https://commitfest.postgresql.org/patch/6423/) (cf, Withdrawn, 2026-05-12) - judged not relevant; reason summarised from its thread
- P8: [Add LSN <-> time conversion facility](https://commitfest.postgresql.org/patch/4754/) (cf, Withdrawn, 2024-10-22) - judged not relevant; reason summarised from its thread
- P9: [Make async slave to wait for lsn to be replayed](https://commitfest.postgresql.org/patch/772/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P10: [Add last failed connection error message to pg_stat_wal_receiver](https://commitfest.postgresql.org/patch/3666/) (cf, Withdrawn, 2022-10-11) - judged not relevant
- P11: [ANALYZE: hash-accelerate MCV tracking for equality-only types](https://commitfest.postgresql.org/patch/6422/) (cf, Withdrawn, 2026-05-03) - judged not relevant
- P12: [Fix alter subscription concurrency errors](https://commitfest.postgresql.org/patch/3842/) (cf, Withdrawn, 2023-03-28) - judged not relevant
- P13: [Revert error message on GLOBAL/LOCAL pending further discussion](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=28ac7972873bd314d4837807396fe44571b5cb50) (revert, Reverted, 2012-06-10) - judged not relevant
- P14: [Bug in ALTER SUBSCRIPTION ... SERVER / ... CONNECTION with broken old server](https://commitfest.postgresql.org/patch/6707/) (cf, Withdrawn, 2026-06-22) - judged not relevant
- P15: [using an end-of-recovery record in all cases](https://commitfest.postgresql.org/patch/3365/) (cf, Withdrawn, 2022-01-17) - judged not relevant
- P16: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant
- P17: [psql BACKEND_PID variable](https://commitfest.postgresql.org/patch/4165/) (cf, Rejected, 2023-03-18) - judged not relevant
- P18: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - judged not relevant
- P19: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P20: [psql \setfileref - new psql variable type - reference to file](https://commitfest.postgresql.org/patch/787/) (cf, Returned with feedback, 2016-12-15) - judged not relevant
- P21: [Add "\gi file" to psql to allow COPY with variable expansion](https://commitfest.postgresql.org/patch/5674/) (cf, Returned with feedback, 2025-06-17) - judged not relevant
- P22: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - judged not relevant
- P23: [Add regexp_positions(string text, pattern text, flags text) → setof record (starts integer\[\], lengths integer\[\])](https://commitfest.postgresql.org/patch/3027/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P24: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 80 s; cost reported by the CLI: $1.22
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 45 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:41:43.388Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
