# Design pre-review: Delete GIN posting-tree pages without excessive locking

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7199](https://commitfest.postgresql.org/patch/7199/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Delete GIN posting tree pages without excessive locking](https://www.postgresql.org/message-id/flat/DDD3E964-5047-499C-8208-594F182D140D%40yandex-team.ru), started 2026-07-21
- Messages read: 1 of 1
- Patch prose read: 0001-Delete-GIN-posting-tree-pages-without-locking-out-th.patch, 0002-Add-isolation-test-for-GIN-posting-tree-page-deletio.patch, 0003-Add-deadlock-provocation-tests-for-GIN-posting-tree-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

Level 1b fails on demand-1. Message #1 names no current workload that stalls behind the root cleanup lock, and it says the 2016 case was solved by what remained after fd83c83d094. The author asks whether anyone is interested, and nobody has replied. This is a "not now, show demand" result; the design objections at level 2 were not evaluated.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code and the truncated part of the README were not read, so lock ordering, inserter recovery and WAL replay are unchecked. Three level 2 objections survived (prior-attempts-1, replication-ondisk-1, maintenance-fit-2) and remain open.

## The proposal, restated

**Problem.** Vacuum deletes empty posting tree pages in a second pass that holds a cleanup lock on the tree root for the whole pass. This blocks all searches and insertions into that posting tree, and the pass cannot be interrupted. The author does not name current users who suffer from this; the original case was Yandex.Mail folder search in 2016.

**Proposed design.** Vacuum deletes empty leaf pages during its normal left-to-right sweep of the leaf level, using lock coupling in the same order as stepping right. It finds the parent by walking right one level above the leaves and deletes a page only if no other backend holds a pin on it. Internal pages, the leftmost and rightmost leaves, and leaves under a parent's last downlink are never deleted.

**User-visible surface.**
- No SQL, GUC, catalog or C API change; the patch touches only ginbtree.c, ginvacuum.c and the GIN README.
- New injection points in the GIN vacuum sweep, used by new isolation tests in src/test/modules/gin.

**What the author claims.**
- Searches and insertions into a posting tree proceed while vacuum is paused mid-sweep.
- Vacuum and insertions both lock leaf-to-parent and left-to-right, so they cannot form a lock cycle.
- Inserters that follow a stale downlink to a deleted page recover by moving right; existing XID stamping prevents early page reuse.
- The 0003 deadlock provocation tests would hang if the locking protocol regressed.
- amcheck for GIN and injection points make getting this right easier than in 2016-2018.

**Thread state.** Only the first message is visible, and nobody has replied. The author asks whether anyone is interested, and CFbot reports the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-21; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No current user is shown to suffer from the root cleanup lock

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer takes on a new posting-tree deletion protocol in GIN vacuum. No workload is known to be blocked by the second pass today, so the change has no measured benefit, and users only meet its risk.

Message #1 says the only known case, Yandex.Mail in 2016, was solved by what remained after fd83c83d094. It offers to continue only "if anyone actually suffers from this". The thread has no replies, no field reports and no numbers on how long the root cleanup lock blocks searches. A reference to another thread that "reminded" the author is not a report of a user problem.

Thread: [#1 (Andrey Borodin, 2026-07-21)](https://www.postgresql.org/message-id/DDD3E964-5047-499C-8208-594F182D140D%40yandex-team.ru)

**Question:** Which current workloads see searches or inserts stalled behind the posting-tree deletion pass, and for how long, on which index sizes?

Counter-check: held. Message #1 asks whether anyone suffers from this and says the 2016 case was solved by what remained after the revert. The thread has no replies, field reports or stall measurements, so demand is not shown.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | Page deletion now runs without the root cleanup lock, and replay on a hot standby cannot see the primary's pin check. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new posting tree deletion protocol in ginvacuum.c and ginbtree.c that later GIN locking changes must keep intact. |
| performance on the hot path | **yes** | The patch changes ginbtree.c, which insertions and searches use, and leaves some empty pages for later scans to walk. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #1 shows no current user who needs the change. |

## 5. Questions the author must answer before anyone reviews code

1. Which current workloads see searches or insertions stall behind the posting tree deletion pass, for how long, and on which index sizes?
2. What was incorrect in the locking reverted by fd83c83d094, and which part of the new protocol rules out that failure?
3. How does replay of a posting tree page deletion stay safe for a hot standby scan that holds a pin on the deleted leaf?
4. After a mass delete, how much posting tree space stays unreclaimed compared with the current two-pass deletion, and what does that cost later scans?
5. What do before and after measurements show for search and insertion latency during vacuum of a large posting tree?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows GIN posting tree locking, page deletion and its WAL replay. Nobody has replied to message #1.
- **Liability:** The owner signs for a new concurrency protocol on every GIN index, with no end date. Message #1 says earlier locking work in this area was incorrect and was reverted.
- **Author follow-up:** Message #1 offers further work only "if anyone actually suffers from this". The author moved the entry to the next CommitFest, but the rebase that CFbot asked for is still open.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A report from at least one current workload where searches or insertions stall behind the root cleanup lock, with stall times and index sizes. A reproducible benchmark that shows the stall before the patch and its absence after would also count.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** park
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The patch adds no user-visible surface, but the locking protocol needs care for ever and expert review of GIN concurrency. No current user is shown and nobody has replied, so it can wait until someone reports the stall.

## Appendix

### Held up in the counter-check, not listed above

- `prior-attempts-1` (fixable, level 2) No account of how this avoids the bug behind the earlier revert. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Standby replay gets no equivalent of the primary-side pin check. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Empty internal and last-downlink pages are never reclaimed. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-1` Subtle new locking protocol with no shown users and no engaged owner. Refuted (not specific): Duplicate of demand-1 for the demand part. The missing owner is a committability matter, which the rubric says must not raise the verdict, and the failure scenario names no concrete broken assumption.
- `demand-2` Every GIN user carries the locking risk for a narrow gain. Refuted (not specific): Duplicate of demand-1 for the size of the group that gains. The risk part names no concrete flaw in the protocol and would fit any concurrency change; maintenance-fit-2 covers the concrete cost to non-users.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent matches. [P4] (GiST page deletion) and [P20] (btree vacuum replay) have no recorded reason, so neither supports an objection. The only prior attempt is the reverted 2016 work that message #1 names.
- standards-compat: 0 objection(s). The patch changes no SQL syntax, GUC, catalog, psql command or client-visible behaviour, so pg_dump round-trip and driver compatibility are not affected. WAL record changes, standby safety and on-disk effects of the new deletion rules belong to other angles and need the code to check.
- replication-ondisk: 1 objection(s). The patch changes no on-disk format, catalog, wire protocol or C API, and the touched files suggest the WAL record format is unchanged, so pg_upgrade and mixed-version replication are not affected.
- security: 0 objection(s). The proposal changes only internal GIN vacuum locking. It adds no SQL, GUC, catalog, privilege or monitoring surface, and no new information path, so it gives no user access they lacked before. An open scan's pin can delay deleting a page, which is a bloat or correctness question for the design angle. I found no privilege concern.
- maintenance-fit: 2 objection(s). Two objections raised. Hot-path cost for insertions and searches could not be judged without code or measurements, and the proposal includes neither.
- already-exists: 0 objection(s). No existing feature or extension removes the root cleanup lock during posting tree page deletion, because this is internal to GIN vacuum and no hook can reach it. Skipping index cleanup with VACUUM (INDEX_CLEANUP off) avoids the lock but stops removing dead TIDs, so it does not do the same job.
- demand: 2 objection(s). Two demand objections are raised. Both are "not now, show demand" points, and neither is a rejection of the design.

### Precedents retrieved

- P1: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P2: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant; reason summarised from its thread
- P3: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P4: [Delete pages during GiST VACUUM](https://commitfest.postgresql.org/patch/1416/) (cf, Returned with feedback, 2018-02-07) - used in the review
- P5: [Revert ill-fated patch to release exclusive lock early after vacuum truncates a table. Introduces race condition, as shown by buildfarm failures.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=eb5f4d6c5cece3e3fa80a506153e5202b093b7ca) (revert, Reverted, 2007-09-20) - judged not relevant
- P6: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P7: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P8: [Fixes a trivial bug in dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5996/) (cf, Withdrawn, 2025-09-02) - judged not relevant; reason summarised from its thread
- P9: [Add OID descriptions to dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5987/) (cf, Withdrawn, 2025-08-25) - judged not relevant; reason summarised from its thread
- P10: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P11: [KNN for B-tree](https://commitfest.postgresql.org/patch/1804/) (cf, Returned with feedback, 2020-03-16) - judged not relevant; reason summarised from its thread
- P12: [btree_gin and ranges](https://commitfest.postgresql.org/patch/24/) (cf, Rejected, 2015-02-03) - judged not relevant; reason summarised from its thread
- P13: [Revert gincostestimate changes.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=61f621b506a7cd29eb1b042a8510b709791034d0) (revert, Reverted, 2007-01-31) - judged not relevant
- P14: [DELETE CASCADE](https://commitfest.postgresql.org/patch/3195/) (cf, Returned with feedback, 2022-01-31) - judged not relevant; reason summarised from its thread
- P15: [use AV work items infrastructure for GIN pending list's cleanup](https://commitfest.postgresql.org/patch/3060/) (cf, Returned with feedback, 2021-09-08) - judged not relevant; reason summarised from its thread
- P16: [Reduce lock level for ALTER TABLE ... ADD CHECK](https://commitfest.postgresql.org/patch/3088/) (cf, Withdrawn, 2021-11-21) - judged not relevant
- P17: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P18: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant
- P19: [Allow reads to proceed during FK/trigger drops by reducing relation-level lock from AccessExclusive to ShareRowExclusive](https://commitfest.postgresql.org/patch/6124/) (cf, Withdrawn, 2025-11-25) - judged not relevant
- P20: [Improving replay of XLOG_BTREE_VACUUM records](https://commitfest.postgresql.org/patch/364/) (cf, Rejected, 2016-04-03) - used in the review
- P21: [Add MAIN_RELATION_CLEANUP and TOAST_TABLE_CLEANUP options to VACUUM](https://commitfest.postgresql.org/patch/2426/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P22: [num_sa_scans in genericcostestimate](https://commitfest.postgresql.org/patch/3871/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P23: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P24: [Retail IndexTuple Deletion](https://commitfest.postgresql.org/patch/1802/) (cf, Returned with feedback, 2019-01-31) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 123 s; cost reported by the CLI: $1.39
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 78 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:27:37.906Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
