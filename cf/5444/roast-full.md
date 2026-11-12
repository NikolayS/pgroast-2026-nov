# Design pre-review: using index to speedup add not null constraints to a table

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5444](https://commitfest.postgresql.org/patch/5444/) (Needs review); authors: Jian He (stayreal)
- Thread: [using index to speedup add not null constraints to a table](https://www.postgresql.org/message-id/flat/CACJufxFiW%3D4k1is%3DF1J%3Dr-Cx1RuByXQPUrWB331U47rSnGz%2Bhw%40mail.gmail.com), started 2024-12-16
- Messages read: 22 of 22
- Patch prose read: bench.txt, v16-0001-using-indexscan-to-speedup-add-not-null-constraints.patch, v1-0001-using-index-to-speedup-add-not-null-constraint-to.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and a committer called the idea great in message #11. The open points are fixable design questions. maintenance-fit-1 asks which check guarantees the index is not rebuilt in the same alter table. maintenance-fit-2 asks for a cost bound when the index holds dead null entries. already-exists-1 stands only as a request to compare with `not valid` plus `validate constraint`.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot confirm which indexes the patch excludes or that the phase 3 conditions cover every subcommand that rebuilds an index. The visibility argument in the v16 commit message was also not checked against the code.

## The proposal, restated

**Problem.** Adding a not-null constraint to an existing table scans the whole heap to look for nulls, which is slow on large tables. The patch targets users who run alter table ... set not null or add constraint ... not null on big tables that already have a suitable index.

**Proposed design.** In phase 3 of alter table, the patch collects all new not-null constraints and checks that each column is the leading column of a valid btree index. If no table scan or rewrite is needed, the table holds AccessExclusiveLock and no virtual generated column is involved, an index scan looks for one null under a fresh MVCC snapshot. Otherwise the normal table scan runs.

**User-visible surface.**
- A DEBUG1 message: "all new not-null constraints on relation "%s" have been validated by using index scan".
- Faster alter table ... alter column ... set not null and add constraint ... not null when a suitable index exists.

**What the author claims.**
- On a table with 1 million deleted rows, adding not null took 1.873 ms with the patch against 648.312 ms on master.
- In a later benchmark on 2 million rows, repeated runs took about 1.6 ms with an index against about 213 ms without one.
- The v1 commit message says the patch can be slower than master when the index is bloated with dead null entries (1080 ms against 971 ms).
- Under AccessExclusiveLock, a fresh MVCC snapshot sees the same rows a table scan would check, so the result is the same.
- Using one multicolumn index for several non-leading columns was slower than a table scan, so only leading columns qualify.

**Thread state.** Álvaro Herrera reviewed it, called it a great idea and asked for restructuring, then removed himself as committer in April 2026. Jian He posted v14 to v16 since then, and no reply to them is visible.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-12-16; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Correctness rests on an implicit list of phase 3 conditions

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds or changes an alter table subcommand so that an index is rebuilt or changed in the same command, and the fast-path checks are not updated. The index scan then runs on an index that does not match the final table. A not-null constraint is accepted while nulls exist, and users find out only when queries or pg_dump restores fail.

Message #1 and the v1 commit message show that subcommands which trigger index rebuilds, such as SET EXPRESSION, already broke this approach. v4 (message #4) listed "no new indexes" as a condition, but the v16 commit message lists only scan or rewrite, the lock level and virtual generated columns. The only tests are a TAP test that greps a DEBUG1 message (message #18), and none of them fails when a new subcommand changes these conditions.

Thread: [#1 (jian he, 2024-12-16)](https://www.postgresql.org/message-id/CACJufxFiW%3D4k1is%3DF1J%3Dr-Cx1RuByXQPUrWB331U47rSnGz%2Bhw%40mail.gmail.com), [#4 (jian he, 2025-04-18)](https://www.postgresql.org/message-id/CACJufxEqdgwUTo3xmUR3Bs49dHvD9VZW%2BT1D8sVdBV4-UdDcrg%40mail.gmail.com), [#18 (jian he, 2026-04-07)](https://www.postgresql.org/message-id/CACJufxE-%2BVSkxMvBRbpM45iex%3D7ws4-km4MvmOBPXtTJ7jjdQA%40mail.gmail.com), [#22 (jian he, 2026-09-30)](https://www.postgresql.org/message-id/CACJufxEkSyhOotK0b3hj_D3c9HO%3DdxGoGxL3Cx4j6enG_t_P6Q%40mail.gmail.com)

**Question:** Which single check guarantees the chosen index is not rebuilt or changed in the same alter table, and what test fails if a future subcommand breaks that guarantee?

Counter-check: held. Message #1 shows that index rebuilds in the same command already broke the approach once. The v16 commit message no longer lists the v4 "no new indexes" condition, and nothing shown names the check that replaces it.

### 2. No bound or opt-out when the index holds many dead nulls

fixable · level 2, rubric item 8 · performance on the hot path, planner behaviour change · `maintenance-fit-2`

An operator deletes or updates many rows that had nulls, and vacuum has not run. They then run set not null on a large table whose cache is cold. Each dead index entry needs a random heap fetch under AccessExclusiveLock, so the command can run longer than the sequential scan it replaced, and no setting turns the fast path off.

The v1 commit message reports that the patch is slower than master when null entries are bloated (1080 ms against 971 ms), and that test used a warm table with only a million rows. In bench.txt the first run took about 480 ms with and without the index, so the gains shown come from repeated runs on a warm cache. The patch picks the index path whenever an index exists and does not look at cost or bloat first.

Thread: [#1 (jian he, 2024-12-16)](https://www.postgresql.org/message-id/CACJufxFiW%3D4k1is%3DF1J%3Dr-Cx1RuByXQPUrWB331U47rSnGz%2Bhw%40mail.gmail.com), [#22 (jian he, 2026-09-30)](https://www.postgresql.org/message-id/CACJufxEkSyhOotK0b3hj_D3c9HO%3DdxGoGxL3Cx4j6enG_t_P6Q%40mail.gmail.com)

**Question:** How slow is the index path on a cold cache when the index holds many dead null entries, and why is choosing it with no cost check safe enough?

Counter-check: held. The v1 commit message reports the slowdown with dead null entries, and no later version gives a bound or a cost check. In bench.txt the indexed table has dead null entries, and its first run is no faster than the table scan.

### 3. Overlap with the existing not valid plus validate constraint path

fixable · level 1b, rubric item 12 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A DBA with a large table runs `alter table ... add constraint ... not null` and expects the index fast path. If the index is bloated, the cache is cold, or a second clause forces a scan, the full scan runs under AccessExclusiveLock. The existing path would have avoided that lock: add a not valid constraint, then run validate constraint, which takes a weaker lock.

Postgres already supports adding a not-null constraint as `not valid` and validating it later with `validate constraint` under ShareUpdateExclusiveLock, so writes continue during the scan. Older releases had the same option through a validated `check (col is not null)` constraint, which lets `set not null` skip the scan. The proposal makes the one-step command faster but does not compare itself with these paths. In bench.txt the first run is about the same with and without the index (about 480 ms), so the gain may depend on a warm cache.

Thread: [#1 (jian he, 2024-12-16)](https://www.postgresql.org/message-id/CACJufxFiW%3D4k1is%3DF1J%3Dr-Cx1RuByXQPUrWB331U47rSnGz%2Bhw%40mail.gmail.com), [#22 (jian he, 2026-09-30)](https://www.postgresql.org/message-id/CACJufxEkSyhOotK0b3hj_D3c9HO%3DdxGoGxL3Cx4j6enG_t_P6Q%40mail.gmail.com)

**Question:** For which users is `not valid` plus `validate constraint` not enough, and does the index path still help on a cold cache?

Counter-check: held. `not valid` plus `validate constraint` avoids the long AccessExclusiveLock, and the thread does not compare the patch with it. The failure scenario describes current behaviour, so the objection stands only as a request for that comparison.

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
| planner behaviour change | **yes** | see objection maintenance-fit-2: No bound or opt-out when the index holds many dead nulls |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The fast path in tablecmds.c depends on a list of phase 3 conditions that later alter table subcommands must keep true. |
| performance on the hot path | **yes** | The index scan runs under AccessExclusiveLock and was slower than the table scan when the index held dead null entries. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | `not valid` plus `validate constraint` already avoids the long scan under AccessExclusiveLock. |

## 5. Questions the author must answer before anyone reviews code

1. Which single check guarantees that the chosen index is not rebuilt or changed in the same alter table, and which test fails if a future subcommand breaks it?
2. How slow is the index scan on a cold cache when the index holds many dead null entries, compared with the table scan?
3. Why is it safe to choose the index scan with no cost or bloat check, given the 1080 ms against 971 ms result in the v1 commit message?
4. For which users is `not valid` plus `validate constraint` not enough, so that the one-step fast path is needed?
5. Does v16 use the index scan for partitioned tables, and does it exclude partial, invalid and expression indexes?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows alter table phase 3 and not-null constraints. Álvaro Herrera reviewed it in the thread but removed himself as committer in April 2026.
- **Liability:** The owner signs for the claim that the index scan never accepts a not-null constraint while a null exists, for every future alter table subcommand combination.
- **Author follow-up:** The author posted sixteen versions over almost two years and kept rebasing after the committer left, so follow-up looks likely.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch helps only when the column leads a valid btree index and no scan or rewrite is pending. v16 touches only tablecmds.c and adds no user-visible surface, but it has had no committer since April 2026 and no visible reply to v14 to v16.

## Appendix

### Refuted in the counter-check

- `demand-1` No field evidence that the indexed case is common or unsolved. Refuted (not specific): Duplicate of already-exists-1, which asks the same question about the existing workaround. The rest is weak: the scenario describes current behaviour, a committer called the idea great in message #11, and the patch adds no cost for other users.

### Review angles

- prior-attempts: 0 objection(s). The only listed precedent, [P6], is about making an existing index unique with NOT VALID. It stalled because its author ran out of time, a reason that does not apply here. Message #1 says this design follows an earlier list suggestion to pull one null from an index, which warned it 'needs careful thought because of visibility issues'. The v16 commit message answers that warning with the AccessExclusiveLock and fresh-snapshot argument, and message #21 adds the indcheckxmin check, so the prior-attempts angle finds no rejection reason left open.
- standards-compat: 0 objection(s). The patch adds no syntax, keyword, GUC, catalog column or psql change, and pg_dump output is unchanged. The fast path only replaces the heap scan inside existing alter table commands under the same lock, so existing scripts and dumps get the same result. Safety and the bloated-index cost belong to other angles.
- replication-ondisk: 0 objection(s). The proposal adds no persistent or cross-node state: v16 touches only tablecmds.c and changes no on-disk format, WAL record, catalog, wire message or exported API. The standby replays the same catalog change, and the commit message explains why the MVCC invariant holds under AccessExclusiveLock. Without the code, this angle cannot check that the indcheckxmin and invalid-index exclusions match the commit message.
- security: 0 objection(s). The proposal adds no privilege, role or setting, and the fast path runs only inside alter table, which already requires table ownership and takes AccessExclusiveLock. A btree null search needs no user-defined comparison code, and the only new output is a DEBUG1 message naming the relation. Whether expression and partial indexes are excluded is a correctness question for other angles, and it cannot be checked without the code.
- maintenance-fit: 2 objection(s). The cost falls only on alter table with not-null validation, and nothing changes for other paths. The patch has had no committer since April 2026, which matters for committability but not for the verdict.
- already-exists: 1 objection(s). One objection only. The existing alternative works with modest extra steps, so this is a question of convenience and not of capability.
- demand: 1 objection(s). The problem is real, the change adds no syntax or GUC, and the cost falls mostly on users who hit the fast path. A committer called it a great idea in message #11, so this is a request for demand evidence, not a rejection.

### Precedents retrieved

- P1: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P2: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P3: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P4: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P5: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P6: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P7: [Revert "Catalog NOT NULL constraints" and fallout](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9ce04b50e120275afbc03ca0b80839dde3da8308) (revert, Reverted, 2023-04-12) - judged not relevant
- P8: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant; reason summarised from its thread
- P9: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P10: [ALTER TABLE SET LOGGED/UNLOGGED on a partitioned table does nothing silently](https://commitfest.postgresql.org/patch/2857/) (cf, Withdrawn, 2021-04-07) - judged not relevant; reason summarised from its thread
- P11: [Detect the null type correctly in setop case by peeking the later exprs.](https://commitfest.postgresql.org/patch/2653/) (cf, Withdrawn, 2020-07-23) - judged not relevant; reason summarised from its thread
- P12: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant
- P13: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/3556/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P14: [Skip vacuum log report code in lazy_scan_heap() if possible](https://commitfest.postgresql.org/patch/3404/) (cf, Rejected, 2021-12-04) - judged not relevant
- P15: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/4636/) (cf, Withdrawn, 2023-11-01) - judged not relevant
- P16: [Restructure ALTER TABLE notes to clarify table rewrites and verification scans](https://commitfest.postgresql.org/patch/3604/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P17: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant
- P18: [Add support for ON UPDATE/DELETE actions on ALTER CONSTRAINT](https://commitfest.postgresql.org/patch/1533/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P19: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant
- P20: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P21: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P22: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P23: [Speed up LockReleaseAll](https://commitfest.postgresql.org/patch/3220/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P24: [Speed up releasing of locks](https://commitfest.postgresql.org/patch/3501/) (cf, Withdrawn, 2024-05-08) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 121 s; cost reported by the CLI: $1.74
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 65 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:31:58.846Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
