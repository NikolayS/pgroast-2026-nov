# Design pre-review: BUG #19723: CREATE INDEX racing with ALTER INDEX ATTACH PARTITION triggers unexpected internal error

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7353](https://commitfest.postgresql.org/patch/7353/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [BUG #19723: CREATE INDEX racing with ALTER INDEX ATTACH PARTITION triggers unexpected internal error](https://www.postgresql.org/message-id/flat/19723-8a47b625c2c64f75%40postgresql.org), started 2026-09-26
- Messages read: 4 of 4
- Patch prose read: v1-0001-Lock-child-index-before-checking-its-parent.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducer and a bisect in message #1, and the fix is local to DefineIndex(). The one design concern, new blocking or deadlock in v1, was raised in message #4 and answered there with a fast-path check plus a recheck after locking. Which version to commit and the back-patch range are still open.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The commit message, touched files and code of the recheck patch in message #4 were not shown, so its locking behaviour and its isolation test could not be checked. The back-patch range also depends on where 8b08f7d48 landed.

## The proposal, restated

**Problem.** CREATE INDEX on a partitioned table can fail with "bogus pg_inherit row" when it races with ALTER INDEX ... ATTACH PARTITION on a partition's index. The reporter reproduced it on 19beta4 and bisected it to commit 8b08f7d48. Users who run both commands at the same time get an internal error (XX000).

**Proposed design.** DefineIndex() calls has_superclass() on a partition's existing index before it locks that index, so a concurrent ATTACH can commit while CREATE INDEX waits for the lock. v1 takes the lock first, then checks. Rafia Sabih's version keeps the unlocked check as a fast path and checks again after taking the lock, and adds an isolation test.

**What the author claims.**
- has_superclass() requires the caller to hold a lock on the child index, and DefineIndex() calls it without one.
- Locking the child index before the check fixes the reported race in the author's testing.
- Rafia Sabih says v1 makes CREATE INDEX on the parent wait on, or deadlock with, sessions holding locks on an already attached index, where unpatched it completes.
- Rafia Sabih says the fast-path check plus a recheck after locking avoids that new blocking.

**Thread state.** Rafia Sabih reviewed v1, found a deadlock it introduces, and posted an alternative patch with an isolation test. The author has not replied to it, and no committer has commented.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The fix stops DefineIndex() from attempting a conflicting pg_inherits row; it adds no catalog change. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry targets stable branches, but nobody says which branches carry commit 8b08f7d48. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch changes lock order in DefineIndex() in src/backend/commands/indexcmds.c, which future changes to that loop must preserve. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which branches carry commit 8b08f7d48, and so which branches need this fix?
2. Does the author agree to replace v1 with the fast-path check plus recheck from message #4, given the deadlock that message reports for v1?
3. Can the recheck version still block or deadlock when ALTER INDEX ... ATTACH PARTITION commits during the lock wait on the child index?
4. Do other callers of has_superclass(), or other partition attach paths, make the same check without holding a lock on the child?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows partitioned index DDL and the lock rules in DefineIndex() would have to own it; none has commented in the visible thread.
- **Liability:** The owner signs for the lock order in DefineIndex() on every branch that gets the fix, including any new blocking or deadlock that shows up in the field after a minor release.
- **Author follow-up:** The author posted a diff and then a patch file within two days, but has not replied to the review and alternative patch in message #4.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug needs CREATE INDEX on a partitioned table to race with ALTER INDEX ... ATTACH PARTITION, so few users hit it, but it is a reproducible internal error bisected to a recent commit. The fix touches one file and adds no user-visible surface.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: [P15] is a sequence-sync race fix reverted for blocking a command, a different area and mechanism. The thread cites no earlier attempt, and the new blocking that v1 adds was raised and answered in message #4 by the fast-path check plus recheck.
- standards-compat: 0 objection(s). No standards or compatibility objection. The fix adds no syntax, GUC, catalog object, dump output or psql change. The one user-visible regression, v1 making CREATE INDEX on the parent block or deadlock on already attached indexes, was raised in message #4. The recheck version there takes no lock beyond what unpatched code takes, so the concern looks settled. [P15] is a different area and does not apply. Without the code, one point cannot be checked: whether 8b08f7d48 is only in 19, which sets the back-patch range.
- replication-ondisk: 0 objection(s). Both versions change only lock order and add a recheck in DefineIndex(), with no new on-disk format, WAL record, catalog column, protocol message or exported struct, so nothing blocks a back-patch on ABI grounds. The fix stops a race from attempting a conflicting pg_inherits row, which protects catalog consistency.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It moves a lock acquisition and adds a recheck inside DefineIndex() for an index the caller already has the right to build on. It adds no new privilege, view, error text or default, and it lets no user see or do anything new. The one new blocking case, from v1, was raised in message #4 and is handled by the fast-path recheck. Any session that can hold conflicting locks on a child index can already block CREATE INDEX on the parent through the partition table lock, so no new denial-of-service path follows. [P15] is about a different subsystem and does not apply.
- maintenance-fit: 0 objection(s). This is a small, local bug fix in DefineIndex(). It adds no new GUC, catalog state, API or dependency, and costs nothing on any common path. The one design concern, new blocking in v1, was raised in message #4 and answered there by the fast-path check plus recheck. [P15] matches only on topic: it was reverted because the fix blocked users, and the recheck version avoids exactly that. A committer may still ask whether other has_superclass() callers make the same unlocked check, but the material gives no failure scenario for them.
- already-exists: 0 objection(s). This is a fix for an internal error ("bogus pg_inherit row") in DefineIndex() when it races with ALTER INDEX ... ATTACH PARTITION. No existing feature, option, extension or external tool can stand in for a race fix inside core DDL code, and neither version adds a second mechanism. Precedent [P15] is about blocking sequence refreshes, so its reason does not apply here.
- demand: 0 objection(s). This is a bug fix for an internal error (XX000, "bogus pg_inherit row") that a reporter reproduced with a script in message #1 and bisected to a recent commit, so the problem is real and stated. A demand objection does not apply to a fix for a race in ordinary DDL; the open design question, the new blocking found in message #4, belongs to other angles.

### Precedents retrieved

- P1: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant; reason summarised from its thread
- P2: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant; reason summarised from its thread
- P3: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant; reason summarised from its thread
- P4: [Adding a { LOCK ROWS | LOCK KEY INDEX } clause to FK definitions](https://commitfest.postgresql.org/patch/6923/) (cf, Withdrawn, 2026-09-09) - judged not relevant
- P5: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant; reason summarised from its thread
- P6: [Tab completion for ATTACH PARTITION](https://commitfest.postgresql.org/patch/4561/) (cf, Returned with feedback, 2024-01-29) - judged not relevant; reason summarised from its thread
- P7: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant; reason summarised from its thread
- P8: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P9: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - judged not relevant
- P10: [Revert "Make pg_dump emit ATTACH PARTITION instead of PARTITION OF"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a1ec7402e9a0392f95bc79542af0efcd5c6e7929) (revert, Reverted, 2019-05-06) - judged not relevant
- P11: [Autovacuum (analyze) on partitioned tables for ATTACH/DETACH/DROP commands](https://commitfest.postgresql.org/patch/3188/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P12: [Automatic HASH and LIST partition creation](https://commitfest.postgresql.org/patch/2694/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P13: [Revert lowering of lock level for ATTACH PARTITION](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8ca984b2c739e096c08f260f477792495f4dfe4) (revert, Reverted, 2018-04-12) - judged not relevant
- P14: [LockHasWaiters() crashes on fast-path locks](https://commitfest.postgresql.org/patch/6732/) (cf, Withdrawn, 2026-09-03) - judged not relevant
- P15: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - used in the review
- P16: [pg_get_domain_ddl: DDL reconstruction function for CREATE DOMAIN statement](https://commitfest.postgresql.org/patch/6983/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P17: [Surround CheckRelation\[Oid\]LockedByMe() with USE_ASSERT_CHECKING](https://commitfest.postgresql.org/patch/5096/) (cf, Withdrawn, 2024-07-01) - judged not relevant
- P18: [Avoiding deadlock errors in CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/2141/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P19: [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (cf, Withdrawn, 2024-03-28) - judged not relevant
- P20: [Adjust pg_stat_get_lock() prorows to match lock types](https://commitfest.postgresql.org/patch/6740/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P21: [Fix NULL dereference in pg_get_database_ddl()](https://commitfest.postgresql.org/patch/6669/) (cf, Withdrawn, 2026-04-13) - judged not relevant
- P22: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P23: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1133/) (cf, Returned with feedback, 2018-03-30) - judged not relevant
- P24: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1666/) (cf, Returned with feedback, 2018-12-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 67 s; cost reported by the CLI: $0.60
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 41 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:43:47.804Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
