# Design pre-review: Preserve statistics targets with ALTER TABLE ALTER COLUMN TYPE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7214](https://commitfest.postgresql.org/patch/7214/) (Needs review); authors: Zsolt Parragi (zsolt.parragi)
- Thread: [Preserve statistics targets with ALTER TABLE ALTER COLUMN TYPE](https://www.postgresql.org/message-id/flat/CAN4CZFNZwcCgi-igaD%3DLH1ubxMBqJJS%2Bp4ZnOKKdCi9duaMu_w%40mail.gmail.com), started 2026-08-14
- Messages read: 5 of 5
- Patch prose read: v2-0002-Preserve-statistics-targets-of-indexes-rebuilt-by.patch, v2-0001-Preserve-statistics-target-of-extended-statistics.patch, 0002-Preserve-statistics-targets-of-indexes-rebuilt-by-AL.patch, 0001-Preserve-statistics-target-of-extended-statistics-re.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the problem is stated, and the fix restores settings the user chose. The one surviving objection, standards-compat-1, is fixable. The entry is tagged Bugfix, both patches add fields to parse nodes, and the thread names no target branch. A committer has to decide the branch scope and which thread carries 0002.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the gram.y change in 0001 and the way DefineIndex() applies targets to partition indexes could not be checked. The only review in the thread tested idx_scan counters, which the patches do not claim to change.

## The proposal, restated

**Problem.** alter table ... alter column ... type (or set expression) recreates dependent indexes and extended statistics from their definition strings. Statistics targets set with alter index ... set statistics or alter statistics ... set statistics are not in those strings, so they silently revert to the default. This affects users who tune these targets.

**Proposed design.** Patch 0001 captures stxstattarget next to the owner in RememberStatisticsForRebuilding(), passes it through a new CreateStatsStmt field, and has CreateStatistics() store it. Patch 0002 captures per-column index targets where the index comment is captured, carries them in a new IndexStmt field, and has DefineIndex() apply them, including on partition indexes it creates.

**User-visible surface.**
- Behavior change: extended statistics keep their stxstattarget after alter column type or set expression.
- Behavior change: index expression columns keep their statistics targets after the rebuild, including indexes of exclusion constraints and indexes created on partitions.
- C API: CreateStatsStmt gains a stxstattarget field, with -1 meaning the default (null) target.
- C API: IndexStmt gains a field carrying per-column statistics targets.

**What the author claims.**
- The bug is the same class of problem as the statistics owner reset fixed by commit 6713a6e04cb, and 0001 follows that fix's pattern.
- Targets set directly on a partition's index are still lost when the rebuild recurses from the parent, matching existing behavior for comments.
- The patches are tagged as a bug fix.

**Thread state.** Melanie Plageman included a modified 0002 in another patch series; solai v tested v2 and reported both targets preserved. The author keeps both patches here because the other thread is still under discussion.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-14; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Bug fix tag implies a backpatch, but parse node structs change

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `standards-compat-1`

A committer backpatches both patches as a bug fix. An extension built against the earlier minor release creates an IndexStmt or CreateStatsStmt with makeNode() at the old size and passes it to DefineIndex() or CreateStatistics(). The server reads the new field past the end of the allocation, and the user sees wrong targets or a crash after a minor upgrade.

The CommitFest entry is tagged Bugfix, and the brief notes that the material does not say whether back-branches are meant. Both patches add fields to parse nodes in parsenodes.h, which back branches avoid changing. The proposal does not say whether the fix is master-only or how a back-branch version would avoid the struct change.

Thread: [#1 (Zsolt Parragi, 2026-08-14)](https://www.postgresql.org/message-id/CAN4CZFNZwcCgi-igaD%3DLH1ubxMBqJJS%2Bp4ZnOKKdCi9duaMu_w%40mail.gmail.com), [#4 (Zsolt Parragi, 2026-10-01)](https://www.postgresql.org/message-id/CAN4CZFOp2tQQ_OmjRJha%3D%2B2BuFBoCrm4QPFJFpoa0aRmL6y0Mw%40mail.gmail.com)

**Question:** Is this fix meant only for master, and if not, how would a back-branch version keep IndexStmt and CreateStatsStmt unchanged?

Counter-check: held. The entry is tagged Bugfix, both patches add fields to parse nodes, and the thread does not name a target branch. Extensions do build IndexStmt nodes and call DefineIndex(), so the back-branch scenario is concrete, and fixable is the right severity.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | IndexStmt and CreateStatsStmt each gain a field in parsenodes.h, and extensions build these nodes. |
| SQL-standard conflict or non-standard syntax | **yes** | Patch 0001 touches gram.y, and the commit message does not say whether new syntax appears. |
| new GUC | no |  |
| planner behaviour change | **yes** | Kept statistics targets change what analyze collects after the rebuild, so plans can differ from today's post-rebuild plans. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, but both patches change parse node structs that back branches normally keep stable. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The rebuild path in ATPostAlterTypeParse() gains two more properties to capture and reapply. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is this fix meant only for master, and if not, how would a back-branch version keep IndexStmt and CreateStatsStmt unchanged?
2. Why does 0001 change gram.y and parse_utilcmd.c, and does create statistics gain any new user-visible syntax?
3. Which of the two patches does the other series now carry, and should this entry keep only the patch that the other series does not include?
4. Does a target set directly on a partition's index still revert without notice when the rebuild recurses from the parent, and is that acceptable for a fix of silent loss?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows the alter table rebuild path in tablecmds.c and the extended statistics code. Melanie Plageman took a modified 0002 into another series, but nobody in the thread has reviewed the approach of 0001.
- **Liability:** The owner signs for the two new parse node fields and for the rebuild path keeping targets correct, including on partition indexes. If backpatched, the owner also signs for ABI stability in minor releases.
- **Author follow-up:** The author posted a v2, rebased after the CFbot report and moved the entry to the next CommitFest.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Only users who set statistics targets on expression indexes or extended statistics are affected, but the loss is silent. Both patches are about 12 KB each, stay in the rebuild path, and follow the pattern already used for the comment and the owner.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Bug fix tag conflicts with new fields in IndexStmt and CreateStatsStmt. Refuted (not specific): Duplicate of standards-compat-1. It raises the same back-branch ABI question with the same scenario.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no row about the same or a similar idea. The thread points only to commit 6713a6e04cb (message #1), a committed fix for the statistics owner that this proposal extends, and to a parallel series that includes a modified 0002 (message #2). Neither reports an earlier rejection.
- standards-compat: 1 objection(s). The behavior change only restores settings the user chose, pg_dump already emits these targets, and no new syntax is claimed. The gram.y change in 0001 could not be checked without the code.
- replication-ondisk: 1 objection(s). Besides the back-branch ABI point, the patches only rewrite existing catalog values (stxstattarget, index attstattarget) inside the same transaction. They add no WAL records, no catalog columns, no on-disk format change and no protocol change.
- security: 0 objection(s). No security or privilege objection. The patches restore statistics targets that an object owner already set, during an alter table that already requires table ownership, so no user gains a new power. Targets are still bounded by the existing maximum, and propagation to partition indexes matches the recursion alter index ... set statistics already does. The gram.y change in 0001 should be checked for new user-facing syntax, but even that would only let an owner set a value they can already set.
- maintenance-fit: 0 objection(s). From the maintenance angle this is a narrow bug fix that follows existing patterns. IndexStmt already carries the index comment for the rebuild, and 0001 copies the owner fix in 6713a6e04cb. No hot-path cost, new dependency or new user surface appears in the material. The duplicate 0002 in Melanie's other series affects which thread gets committed, not whether the change is wanted. No committer has yet commented on the approach, and back-branch intent is not stated.
- already-exists: 0 objection(s). Postgres has no feature, option or extension that keeps these targets across the rebuild. The only workaround is to run alter index ... set statistics or alter statistics ... set statistics again by hand, and that needs the user to know the loss happens, which the current behavior hides. The patches reuse the capture path that already keeps the comment and the owner, so they add no second mechanism. The overlap with Melanie's other series is about which thread carries the patch, and the design is not in dispute.
- demand: 0 objection(s). This is a bug fix: alter column type silently drops statistics targets that users set on purpose, which follows the same pattern as the owner reset already fixed upstream. Its cost falls only on the rebuild path, and asking for field reports before fixing silent loss of user configuration would not be a fair demand objection.

### Precedents retrieved

- P1: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant; reason summarised from its thread
- P2: [Tab completion for ALTER COLUMN SET STATISTICS](https://commitfest.postgresql.org/patch/377/) (cf, Rejected, 2015-12-21) - judged not relevant; reason summarised from its thread
- P3: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant; reason summarised from its thread
- P4: [Inconsistency in owner assignment between INDEX and STATISTICS](https://commitfest.postgresql.org/patch/6571/) (cf, Withdrawn, 2026-03-11) - judged not relevant; reason summarised from its thread
- P5: [preserve timestamps when installing headers](https://commitfest.postgresql.org/patch/3363/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P6: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P7: [Preserve versions of initdb-created collations in pg_upgrade](https://commitfest.postgresql.org/patch/2328/) (cf, Returned with feedback, 2020-01-24) - judged not relevant; reason summarised from its thread
- P8: [Extended statistics in EXPLAIN](https://commitfest.postgresql.org/patch/3050/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P9: [Statistics for array types](https://commitfest.postgresql.org/patch/323/) (cf, Returned with feedback, 2016-03-18) - judged not relevant; reason summarised from its thread
- P10: [Postgres picks suboptimal index after building extended statistics](https://commitfest.postgresql.org/patch/3245/) (cf, Withdrawn, 2022-11-24) - judged not relevant
- P11: [Implement type input functions for extended statistics types](https://commitfest.postgresql.org/patch/4114/) (cf, Withdrawn, 2023-03-01) - judged not relevant
- P12: [Enhance statistics reset functions to return reset timestamp](https://commitfest.postgresql.org/patch/5956/) (cf, Withdrawn, 2025-10-28) - judged not relevant
- P13: [Adaptive query optimization: create extented statistics using auto_explain extension](https://commitfest.postgresql.org/patch/2958/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P14: [Introduce the ability to enable/disable indexes using ALTER INDEX](https://commitfest.postgresql.org/patch/5274/) (cf, Withdrawn, 2025-09-08) - judged not relevant
- P15: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P16: [list of extended statistics on psql](https://commitfest.postgresql.org/patch/2692/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P17: [Revert recent changes to 002_pg_upgrade.pl.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e22f675fd81ba1d96b0b9a34bbf26d5ec532d5) (revert, Reverted, 2022-08-04) - judged not relevant
- P18: [estimating joins using extended statistics](https://commitfest.postgresql.org/patch/3055/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P19: [Allow AddForeignUpdateTargets to add extra target expressions other than columns](https://commitfest.postgresql.org/patch/1819/) (cf, Returned with feedback, 2019-11-27) - judged not relevant
- P20: [multivariate statistics](https://commitfest.postgresql.org/patch/120/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P21: [Showing applied extended statistics in explain Part2](https://commitfest.postgresql.org/patch/5077/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P22: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/4792/) (cf, Returned with feedback, 2024-10-02) - judged not relevant
- P23: [Revert non-text output formats for pg_dumpall](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7ca548f23a60887541493b2da47f2b2720c35ee3) (revert, Reverted, 2026-06-18) - judged not relevant
- P24: [Apply a better fix to mdunlinkfork().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=85d8b30724c0fd117a683cc72706f71b28463a05) (revert, Partially reverted, 2022-11-09) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 110 s; cost reported by the CLI: $0.95
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 54 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:29:34.351Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
