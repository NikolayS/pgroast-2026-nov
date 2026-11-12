# Design pre-review: Warn when creating or enabling a subscription with max_logical_replication_workers = 0

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6642](https://commitfest.postgresql.org/patch/6642/) (Needs review); authors: Yugo Nagata (yugo.nagata)
- Thread: [Warn when creating or enabling a subscription with max_logical_replication_workers = 0](https://www.postgresql.org/message-id/flat/20260204140731.0e4b511906ac748abad1f3d9%40sraoss.co.jp), started 2026-02-04
- Messages read: 13 of 13
- Patch prose read: 0001-Warn-when-creating-or-enabling-a-subscription-with-l.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the problem is stated, the cost falls only on misconfigured servers, and reviewers found the warning useful. The one surviving objection, maintenance-fit-1, is a fixable design point about where the check lives. Message #13 already shows a gap between CREATE and ALTER ... SET (retain_dead_tuples), so a committer would likely ask for one shared test of whether the launcher runs.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the exact warning conditions, the behaviour during a pg_dump restore, and how the check relates to ApplyLauncherRegister() could not be confirmed. Only 3 of 13 messages were shown in full.

## The proposal, restated

**Problem.** With max_logical_replication_workers = 0 the logical replication launcher never starts, yet CREATE SUBSCRIPTION and ALTER SUBSCRIPTION ... ENABLE succeed silently. Users then wonder why replication does not start. The author reports user reports of this confusion.

**Proposed design.** CREATE SUBSCRIPTION and ALTER SUBSCRIPTION ... ENABLE emit a WARNING with a HINT when max_logical_replication_workers is 0. The check sits in subscriptioncmds.c, next to existing messages such as "subscription was created, but is not connected". Running out of worker slots when the setting is above 0 is out of scope.

**User-visible surface.**
- New WARNING on CREATE SUBSCRIPTION: "subscription was created, but logical replication is disabled".
- New WARNING on ALTER SUBSCRIPTION ... ENABLE: "subscription was enabled, but logical replication is disabled".
- New HINT: "To initiate replication, set "max_logical_replication_workers" to a non zero value."
- The CREATE SUBSCRIPTION warning also fires with enabled = false and retain_dead_tuples = true.

**What the author claims.**
- Users today get no sign from these commands that replication cannot start.
- Users have reported confusion over this behaviour.
- Warning only for the 0 case is enough, because with a value above 0 the launcher logs "out of logical replication worker slots".
- A backend check of free worker slots needs new infrastructure and an LWLock, and its result can change before the worker starts.

**Thread state.** Peter Smith, Hou zj and Dilip Kumar asked for a warning based on available workers; the author declined, and two reviewers (Shubhra Jain, João Rito) tested the February patch and accept the narrow scope. Open items: a rebase, a test, hint wording for retain_dead_tuples, ALTER ... SET (retain_dead_tuples) coverage, and Rafia Sabih's separate pg_stat_subscription_stats counter proposal.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-02-04; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Warning condition copied into command sites, so it drifts from launcher.c

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a subscription option that needs the launcher, or changes when the launcher registers. The patch author updates launcher.c and calls ApplyLauncherWakeupAtCommit(), but does not add the new check in subscriptioncmds.c. Users on a server with the launcher absent then get silence again, and no test fails.

The check tests max_logical_replication_workers directly in subscriptioncmds.c, while ApplyLauncherRegister() in launcher.c decides on its own conditions (message #5). The gaps already exist. Message #13 shows that ALTER ... SET (retain_dead_tuples) does not warn and that max_worker_processes = 0 is silent. Peter Smith raised the other callers in message #9, and the author kept the per-site approach in message #10. No test exercises the path, so drift will not show up.

Thread: [#5 (Peter Smith, 2026-02-05)](https://www.postgresql.org/message-id/CAHut%2BPto3FdeAaQJgcXZHk8qOnGOTpWW7K98mxzVckgQjWWXHw%40mail.gmail.com), [#9 (Peter Smith, 2026-02-06)](https://www.postgresql.org/message-id/CAHut%2BPtmxmkP4acNT0b8b49vySXSEAARvDFYuqbyBWBho4iH%2Bg%40mail.gmail.com), [#10 (Yugo Nagata, 2026-04-13)](https://www.postgresql.org/message-id/20260413160405.a22d5a0111c612176ac9eb3f%40sraoss.co.jp), [#13 (João Rito, 2026-11-10)](https://www.postgresql.org/message-id/CADMs5OO%2B1h1LhASBY0zMoTyUeWfpp-_wCD6azMtRT_%3D0ym4A1A%40mail.gmail.com)

**Question:** Can the decision whether the launcher will run come from one function in launcher.c, so that new callers and new registration conditions cannot miss the warning?

Counter-check: held. Message #13 shows the drift already exists: CREATE warns for retain_dead_tuples, but ALTER ... SET (retain_dead_tuples) does not. Peter Smith's question in message #9 about other ApplyLauncherWakeup() callers was answered by keeping the per-site check, and no test covers the path.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The warning reports that the logical replication launcher will not start on the subscriber; no WAL or protocol behaviour changes. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | A restore that runs CREATE SUBSCRIPTION with retain_dead_tuples = true on a server with the setting at 0 may emit the new warning. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The check in subscriptioncmds.c repeats a condition that launcher.c decides, and no test covers it. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can one function in launcher.c report whether the launcher will run, so that every command site uses the same test?
2. Should ALTER SUBSCRIPTION ... SET (retain_dead_tuples) warn, given that CREATE SUBSCRIPTION warns for the same option?
3. Should the warning also cover max_worker_processes = 0, where the launcher is also absent and the commands succeed silently?
4. Will the next version add a regression or TAP test that fails when a command site misses the warning?
5. Does a pg_dump restore of a subscription with retain_dead_tuples = true emit the warning on a server with the setting at 0, and is that intended?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the logical replication launcher and subscription commands would have to own it; the visible thread and CommitFest entry show no committer engaged.
- **Liability:** Two new message strings and a check that must stay in step with the launcher registration conditions in launcher.c for as long as those conditions change.
- **Author follow-up:** The author answered review in February and moved the entry in August, but posted no new version after February and the patch needs a rebase since October.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches one file and only helps users who set max_logical_replication_workers to 0, which message #6 calls rare. It needs a rebase, a test and an answer on where the check lives before a committer can take it.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only listed row, [P24], is about an ALTER TABLE notice on partition recursion and has no recorded reason, so it shares only the general idea of adding a notice. The thread does not point to any earlier discussion of a warning for max_logical_replication_workers = 0.
- standards-compat: 0 objection(s). From the standards and compatibility angle, the proposal adds no syntax, GUC, catalog or dump change. It follows the existing "subscription was created, but is not connected" message pattern, so existing scripts and dumps keep working. Hint wording and the missing warning on ALTER ... SET (retain_dead_tuples) are open items already raised in messages #12 and #13. Without the code, I could not check whether a pg_dump restore with retain_dead_tuples = true into a server with the setting at 0 triggers the new warning.
- replication-ondisk: 0 objection(s). The patch adds only a WARNING and HINT in subscriptioncmds.c. It changes no catalog, WAL record, on-disk format, wire message or extension API, so nothing affects pg_upgrade, standbys or subscribers. The publisher slot that CREATE SUBSCRIPTION leaves inactive with the setting at 0 exists today, and the warning makes it easier to notice. Rafia Sabih's pg_stat_subscription_stats counter would change persisted stats, but it is a separate proposal.
- security: 0 objection(s). The proposal has no real privilege or exposure aspect. The new warning reports the value of max_logical_replication_workers, which any user can already read with SHOW, and only users allowed to run CREATE or ALTER SUBSCRIPTION see it. The proposal adds no new privilege, no new code path that runs user-controlled code, and no new resource use.
- maintenance-fit: 1 objection(s). Apart from this point, the maintenance cost is small. It is one file, adds no cost to any hot path, and adds no GUC, catalog or API. Ten such warnings per release would be acceptable.
- already-exists: 0 objection(s). Message #1 names the existing way, checking pg_stat_subscription (empty pid) or pg_stat_replication, but users only check once they suspect a problem, and a command-time WARNING costs little and falls only on this case. No extension or general hook covers this, and the patch reuses the existing "subscription was created, but is not connected" message pattern instead of adding a second mechanism.
- demand: 0 objection(s). Demand evidence is thin: message #1 cites unquantified user reports, and message #6 calls the 0 setting rare. Still, the cost is one check on the CREATE and ENABLE paths, seen only by misconfigured users, and three reviewers (messages #7, #12, #13) found it useful, so a show-demand objection would not hold.

### Precedents retrieved

- P1: [Improving the latch handling between logical replication launcher and worker processes.](https://commitfest.postgresql.org/patch/5095/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P2: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P3: [Improve logicalrep_worker_launch() logic](https://commitfest.postgresql.org/patch/5734/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P4: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant; reason summarised from its thread
- P5: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant
- P6: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant; reason summarised from its thread
- P7: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - judged not relevant; reason summarised from its thread
- P8: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant; reason summarised from its thread
- P9: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P10: [Changing the autovacuum launcher scheduling; oldest table first algorithm](https://commitfest.postgresql.org/patch/1573/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P11: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P12: [Subscription code improvements](https://commitfest.postgresql.org/patch/1248/) (cf, Returned with feedback, 2018-04-07) - judged not relevant
- P13: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1710/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P14: [Fix alter subscription concurrency errors](https://commitfest.postgresql.org/patch/3842/) (cf, Withdrawn, 2023-03-28) - judged not relevant
- P15: [Allow CI to only run the compiler warnings task](https://commitfest.postgresql.org/patch/5249/) (cf, Withdrawn, 2025-11-24) - judged not relevant
- P16: [Introduce the ability to enable/disable indexes using ALTER INDEX](https://commitfest.postgresql.org/patch/5274/) (cf, Withdrawn, 2025-09-08) - judged not relevant
- P17: [create and use subscription for nonsuperuser](https://commitfest.postgresql.org/patch/1893/) (cf, Returned with feedback, 2019-03-26) - judged not relevant
- P18: [Fix documentation and logging for START_REPLICATION](https://commitfest.postgresql.org/patch/3174/) (cf, Withdrawn, 2021-06-16) - judged not relevant
- P19: [Enable jitlink as an alternative jit linker of legacy Rtdyld and add riscv jitting support](https://commitfest.postgresql.org/patch/3857/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P20: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant
- P21: [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (cf, Returned with feedback, 2021-10-06) - judged not relevant
- P22: [Introduce the ability to enable/disable indexes](https://commitfest.postgresql.org/patch/5267/) (cf, Withdrawn, 2024-09-26) - judged not relevant
- P23: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - judged not relevant
- P24: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - used in the review

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 85 s; cost reported by the CLI: $1.61
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 51 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:57:25.587Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
