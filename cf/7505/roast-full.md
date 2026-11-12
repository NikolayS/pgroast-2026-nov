# Design pre-review: Fix the language for new CREATE SUBSCRIPTION features

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7505](https://commitfest.postgresql.org/patch/7505/) (Needs review); authors: Thom Brown (thom)
- Thread: [Fixes for create subscription page](https://www.postgresql.org/message-id/flat/CAA-aLv4tAU6yY%3DOUa7Fisy%2BKk_4bS-mJreQUJjU-HhJOA9TB-w%40mail.gmail.com), started 2026-10-07
- Messages read: 10 of 10
- Patch prose read: create_subscription_changes.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at any level. The patch changes documentation wording only, mostly in text new in v19, and adds no syntax, GUC, catalog or behaviour change. Peter Smith and Matemática A3K approved v2 in the thread. One accuracy check remains for the committer: the reworded retention-limit note for retain_dead_tuples.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The material shows only the v1 hunks, so the final v2 wording could not be checked. The accuracy of the retention-limit note against server code was not verified.

## The proposal, restated

**Problem.** Thom Brown found grammar and wording errors in the create subscription reference page, mostly in text for options new in v19. The target readers are users of logical replication who read that page.

**Proposed design.** The patch changes wording only, in create_subscription.sgml. It edits text for the connect option, slot_name, binary, retain_dead_tuples and its retention limit, and two notes. The v2 patch from Laurenz Albe also drops "the" before "conflict detection" in the pg_conflict_detection notes of func-admin.sgml and protocol.sgml.

**What the author claims.**
- The page has grammar errors, for example "if it does index scan" in the read-only subscribers scenario.
- Masahiko Sawada says the fixes can be backpatched to v19, where most of the touched text was added.

**Thread state.** Laurenz Albe posted a v2 patch with the agreed wording; Peter Smith and Matemática A3K approved it. Open questions are whether it goes into v19 before release and who commits it.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-07; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

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
| backpatchability | **yes** | Masahiko Sawada says the fixes can be backpatched to v19, where most of the touched text was added. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the reworded retention-limit note for retain_dead_tuples match what the server does when another subscription has the limit set to 0 or not yet exceeded?
2. Does v2 change protocol.sgml as well as func-admin.sgml, given that message #7 mentions only the function documentation?
3. Which v2 hunks apply to v19 and which only to master, if any touched text does not exist in v19?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the logical replication documentation, in particular the retain_dead_tuples text; the visible thread shows no committer taking the patch.
- **Liability:** Small: wording in three documentation files. The one item to sign for is that the reworded retention-limit note still describes server behaviour.
- **Author follow-up:** Thom Brown answered the review the same day in message #3 and accepted most of the suggested changes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 5 KiB of wording fixes in documentation read by users of logical replication, and reviewers approved v2. It needs a committer to pick it up for v19.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and the thread points to no earlier discussion of these edits. The patch is a docs-only wording fix for text new in v19, and three reviewers approved v2, so no earlier rejection reason applies.
- standards-compat: 0 objection(s). The patch changes documentation wording only. It adds no syntax, keyword, GUC, catalog object, psql behaviour or dump content, so no existing query, script, dump or client behaves differently after upgrade. The reworded retention-limit note for retain_dead_tuples should be checked against server behaviour, but that is a documentation accuracy question outside this angle.
- replication-ondisk: 0 objection(s). The patch changes documentation wording only, so it touches no on-disk format, WAL, replication protocol, catalog, wire protocol or extension ABI. The retention-limit note for retain_dead_tuples describes replication behaviour, but whether the new wording is accurate is a docs question for another angle.
- security: 0 objection(s). The patch changes wording only in documentation for create subscription, func-admin and protocol pages. It adds no privilege, default, exposure path or denial-of-service surface, and the one wording change about access control between subscriber and publisher was reverted to the original in message #7.
- maintenance-fit: 0 objection(s). The patch changes documentation wording only. It adds no hot-path cost, no API, no catalog entry and no external dependency, and it leaves no new place that later features must update. From the maintenance angle there is nothing to object to. One check remains open for whoever commits it: the reworded retention-limit note for retain_dead_tuples should still match server behaviour.
- already-exists: 0 objection(s). The patch fixes grammar and wording in existing reference pages. No existing feature, extension or external tool can fix wording in the core documentation, so nothing from this angle applies.
- demand: 0 objection(s). The patch fixes grammar and wording in user-facing reference docs, mostly in text new in v19. Every reader of the create subscription page gains, and it adds no maintenance cost, GUC, catalog change or user interface, so the demand question does not apply. Four people reviewed or approved it in the thread.

### Precedents retrieved

- P1: [Make selectivity example match wording](https://commitfest.postgresql.org/patch/3695/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P2: [Revert "docs: change "default role" wording to "predefined role""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c185a57753e64fe3ffc996ea18da8672b302a5a4) (revert, Reverted, 2020-02-09) - judged not relevant
- P3: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P4: [Tab-completion refactoring and word-removal feature (was IF-NOT_EXISTS for tab-completion)](https://commitfest.postgresql.org/patch/855/) (cf, Returned with feedback, 2017-03-17) - judged not relevant; reason summarised from its thread
- P5: [Detection and resolution of logical replication conflict](https://commitfest.postgresql.org/patch/5021/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P6: [Revert dubious message wording change.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1c8f03d4de5bca70437aaff5548a18a2d4e16a11) (revert, Reverted, 2009-06-22) - judged not relevant
- P7: [Select Reference Page - Make Join Syntax More Prominent](https://commitfest.postgresql.org/patch/3726/) (cf, Returned with feedback, 2022-09-10) - judged not relevant; reason summarised from its thread
- P8: [Fix documentation and logging for START_REPLICATION](https://commitfest.postgresql.org/patch/3174/) (cf, Withdrawn, 2021-06-16) - judged not relevant; reason summarised from its thread
- P9: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P10: [Build the docs if there are changes in docs and don't run other tasks if the changes are only in docs](https://commitfest.postgresql.org/patch/4555/) (cf, Withdrawn, 2024-05-17) - judged not relevant; reason summarised from its thread
- P11: [Remove size limitations of vacuums dead_tuples array](https://commitfest.postgresql.org/patch/2302/) (cf, Returned with feedback, 2019-11-30) - judged not relevant
- P12: [create and use subscription for nonsuperuser](https://commitfest.postgresql.org/patch/1893/) (cf, Returned with feedback, 2019-03-26) - judged not relevant
- P13: [Document creating an extension in replication setup](https://commitfest.postgresql.org/patch/3494/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P14: [Doc: Fix definition of asynchronous replication](https://commitfest.postgresql.org/patch/6005/) (cf, Withdrawn, 2025-10-03) - judged not relevant
- P15: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P16: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P17: [Revert "Add GROUP BY ALL".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a32733d8f10f0e35f7e4ee772ddbef552296577f) (revert, Reverted, 2026-07-17) - judged not relevant
- P18: [Revert "Avoid spurious deadlocks when upgrading a tuple lock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9d20b0ec8f2af43041b1a65e5fcd91acc47e9ace) (revert, Reverted, 2019-06-17) - judged not relevant
- P19: [Fix typo 586/686 in atomics/arch-x86.h](https://commitfest.postgresql.org/patch/6295/) (cf, Returned with feedback, 2026-03-12) - judged not relevant
- P20: [Making WAL receiver startup rely on GUC context for primary_conninfo and primary_slot_name](https://commitfest.postgresql.org/patch/1916/) (cf, Rejected, 2019-02-16) - judged not relevant
- P21: [Fix typo about WalSndPrepareWrite](https://commitfest.postgresql.org/patch/2945/) (cf, Withdrawn, 2021-02-18) - judged not relevant
- P22: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P23: [\[PATCH\] Fix typos in pqsignal.c comment](https://commitfest.postgresql.org/patch/6871/) (cf, Withdrawn, 2026-06-10) - judged not relevant
- P24: [Doc: update or remove dead external links.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e91cd951b1114a009a0256f211a41d0ec873b2fc) (revert, Partially reverted, 2020-07-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 53 s; cost reported by the CLI: $0.82
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 38 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:02:35.319Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
