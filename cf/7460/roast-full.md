# Design pre-review: Fix double-counting of pending relation stats by VACUUM and ANALYZE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7460](https://commitfest.postgresql.org/patch/7460/) (Needs review); authors: Manuel Reyes Bravo (manu15), Ilya Staroverov (ilyastar)
- Thread: [VACUUM/ANALYZE double-count the backend's own pending table stats (n_live_tup = 2*N after INSERT + ANALYZE)](https://www.postgresql.org/message-id/flat/1DC9D815-8754-4A76-B2B1-99C1A614A55A%40localhost.localdomain), started 2026-10-14
- Messages read: 12 of 12
- Patch prose read: v2-0001-Fix-double-counting-of-pending-relation-stats-by-.patch, v2-0002-Don-t-count-pending-inserts-in-ins_since_vacuum-a.patch, v1-0001-Fix-double-counting-of-pending-relation-stats-by-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and no level fails. The thread shows a deterministic single-session reproduction on 14 through master and measured extra autovacuum runs. Two fixable points remain. prior-attempts-1 asks why an earlier thread judged a similar case not a bug. standards-compat-1 asks whether 0002 may change a pgstat.h struct in stable branches. A committer who knows the cumulative stats code has to settle both.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can check that dropping pending deltas is correct with subtransactions, 2PC and concurrent sessions. The same holds for whether the 0002 struct change is ABI-safe on 14 through 18.

## The proposal, restated

**Problem.** VACUUM and ANALYZE overwrite live_tuples and dead_tuples with their own counts, but the same backend's earlier committed deltas may still be pending and get added again at the next flush. After INSERT of N rows and ANALYZE in one session, n_live_tup shows 2*N, and mod_since_analyze or dead_tuples stay inflated, so autovacuum runs extra passes. A separate bug loses a pending TRUNCATE reset when a later transaction commits.

**Proposed design.** When VACUUM or ANALYZE report new totals, the backend drops its own pending live/dead deltas, and pending changed_tuples for ANALYZE with resetcounter. Commit and COMMIT PREPARED only set truncdropped and never clear it. Patch 0002 adds a pending ins_before_vacuum field so the flush skips inserts that VACUUM already covered for ins_since_vacuum, while n_tup_ins stays cumulative.

**User-visible surface.**
- Changed values in pg_stat_user_tables: n_live_tup, n_dead_tup, n_mod_since_analyze, n_ins_since_vacuum after same-session VACUUM or ANALYZE.
- New field ins_before_vacuum appended to the pending table stats struct in pgstat.h, visible to C extensions.

**What the author claims.**
- The bug is deterministic in one session, unlike the known inaccuracy from concurrent sessions.
- Every tested release reproduces it, 14 through 18 and master, so it predates shared-memory stats.
- Sebastian measured 20 extra autoanalyze or autovacuum runs per 20 tables without the fix and 0 with v1 plus 0002.
- Commit 81e82a2bd48 fixed the same class of problem in 2016 and called it an oversight.
- The struct field goes at the end so inline pgstat_count_* offsets stay the same, which keeps a back-patch possible.

**Thread state.** Manu, Sebastian and João tested v2 and report correct counts, but Bertrand Drouvot says a similar case was earlier judged not a bug and suggests fixing only TRUNCATE. Ilya disagrees and offers to split out the TRUNCATE fix, and no committer has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-14; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Same cause as an earlier case judged not a bug

fixable · level 1b, rubric item 2 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A committer who took part in the earlier thread sees the same root cause raised again and declines 0001 for the same reason. Only the TRUNCATE fix goes in. The VACUUM and ANALYZE double count stays, and users running ETL scripts keep seeing extra autovacuum and autoanalyze runs.

Message #10 points to an earlier thread where a similar case was not treated as a bug. Message #11 agrees the VACUUM example there has the same cause. The thread does not report why that thread decided as it did. Message #11 reads the earlier reason as a wish to keep stats batched and says the patch keeps batching. If the earlier reason was that these counters are approximate by design, the fix is still partial: other backends' pending deltas, including those seen by autovacuum's own VACUUM, still inflate the counters, as message #11 concedes.

Thread: [#10 (Bertrand Drouvot, 2026-11-05)](https://www.postgresql.org/message-id/auwYtm6qM3fPltNF%40bdtpg), [#11 (Staroverov Ilja, 2026-11-05)](https://www.postgresql.org/message-id/2C6AA9AE-4348-4952-B971-E0D63B2B1A53%40localhost.localdomain)

**Question:** What reason did the earlier thread give for calling this not a bug, and why does that reason not cover the same-backend case fixed here?

Counter-check: held. Message #11 argues the same-backend case differs and keeps batching, and ANALYZE already subtracts pending delta_dead_tuples. But the thread does not state the earlier thread's reason, and Bertrand has not replied, so the question stays open.

### 2. Back-patching 0002 changes a public pgstat.h struct in stable branches

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `standards-compat-1`

A packager ships a 14-18 minor release with 0002, and an extension built against the old pgstat.h keeps running unrebuilt. If that extension embeds, copies or sizes the pending table stats struct, it now reads or allocates the wrong layout. The breakage would show up as wrong counters or memory errors after the minor update.

0002 adds ins_before_vacuum to a struct in pgstat.h that C extensions can see. Message #2 and message #5 argue that appending the field keeps the inline pgstat_count_* offsets stable, so a back-patch stays possible. That covers offsets inside the struct. It does not cover the struct's size, or any containing struct that holds it ahead of other members, and nobody in the thread ran an ABI check on 14-18. 0001 touches only pgstat_relation.c, so it can be back-patched alone, with 0002 kept for master only.

Thread: [#2 (Manu, 2026-10-14)](https://www.postgresql.org/message-id/179200362437.252424.7000490135467740876%40gmail.com), [#5 (Staroverov Ilja, 2026-10-15)](https://www.postgresql.org/message-id/1CBF745F-CB0D-4D35-9E25-D6B5208335E4%40localhost.localdomain), [#12 (João Rito, 2026-11-12)](https://www.postgresql.org/message-id/CADMs5ONUqqmPcNWnqobGy8wWujFbYHusZjwdJsehShGH3SsS1Q%40mail.gmail.com)

**Question:** Which stable branches would receive 0002, and does an ABI comparison of 14-18 show no size or layout change in any struct visible to extensions?

Counter-check: held. Appending the field keeps offsets inside the counts struct but grows it, which moves later members of any struct that embeds it. The thread shows no ABI check on 14-18 and does not say which branches get 0002.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0002 appends ins_before_vacuum to the pending table stats struct in pgstat.h, which C extensions can see. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bug reproduces on 14 through 18, and the thread offers back-branch versions, including the struct change in 0002. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The reset of pending deltas conflicts with the in-transaction stats flush patch (v17), and the VACUUM test is best-effort. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What reason did the earlier thread cited in message #10 give for calling the similar case not a bug, and why does it not cover the same-backend case?
2. Which stable branches should receive 0002, and does an ABI comparison on 14 through 18 show no size or layout change visible to extensions?
3. Should this patch or the in-transaction stats flush patch (v17) go first, and how does the reset look with flushed_xact baselines?
4. Do 0001 alone and 0001 plus 0002 each pass check-world, and what do the counters show when another session modifies the table concurrently?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the cumulative stats code in pgstat_relation.c and how autovacuum reads these counters; no committer has replied in the thread.
- **Liability:** The owner signs for the counters that drive autovacuum and autoanalyze on every branch that gets the fix. With 0002, the owner also signs for a pgstat.h struct change.
- **Author follow-up:** Ilya posted v2, offers to rebase in either order against the v17 flush patch, and offers to split out the TRUNCATE fix. Manu wrote 0002 and offers more repros.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patches are small and touch one source file plus one header field. They fix a deterministic miscount that causes extra autovacuum runs after a bulk load or delete followed by ANALYZE or VACUUM in the same session. A committer must answer the "not a bug" question and the back-patch question.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Back-patching a new field in the pgstat.h pending struct. Refuted (not specific): Duplicate of standards-compat-1. Its extra evidence from message #12 is a stale incremental build of core, which João attributes to his build, so it does not show an extension failure.
- `maintenance-fit-2` Pending-delta reset is tied to the current flush model. Refuted (already handled): Message #1 names the conflict with the uncommitted v17 patch and states what the reset must become there. The scenario needs a later rebase error in that other patch, and 0001's ANALYZE and TRUNCATE tests are pinned and would fail.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent matches. [P18] is the in-transaction flush patch this one conflicts with, and it has no recorded reason. [P2] is about analyze of partitioned tables. The only earlier discussion is the one cited in message #10.
- standards-compat: 1 objection(s). The SQL surface does not change: there is no new syntax, GUC, catalog or psql change, and nothing for pg_dump. The changed pg_stat_user_tables values are corrections, and they move toward count(*) and reltuples. Only the back-branch ABI question falls within this angle.
- replication-ondisk: 0 objection(s). The proposal touches no persistent or cross-node state. Pending table stats live only in backend memory and are freed after a flush, so the stats file format, WAL, replication, catalogs and pg_upgrade are unaffected. Standbys keep their own cumulative stats. The only ABI point is 0002's new field in the pending struct in pgstat.h. It is appended at the end, so inline pgstat_count_* offsets do not move, and core allocates pending entries. I could not build a concrete extension failure for master. For a back-patch to 14 through 18, a committer would likely ask for an ABI check of the struct size change, or for back-patching 0001 alone. That cannot be checked without the code. João's "write past chunk end" report came from stale objects in his own incremental build, not from the patch.
- security: 0 objection(s). The proposal changes only how one backend accounts for its own pending table stats before a flush. It adds no new privilege, function, view or setting, and it does not expose any data that pg_stat_user_tables does not already show. Autovacuum scheduling changes only by removing extra runs, so no unprivileged user gains a new way to use resources or learn about other sessions.
- maintenance-fit: 2 objection(s). On fit and hot-path cost there is no objection. The change runs only in VACUUM, ANALYZE and commit bookkeeping, adds no GUC or catalog, and fixes core accounting that cannot live in an extension.
- already-exists: 0 objection(s). No existing feature or extension gives correct same-session counts. The closest is calling pg_stat_force_next_flush() before ANALYZE or VACUUM, a workaround users would have to know about, not a way to fix wrong stats. The in-transaction flush patch [P18] does not change this behavior, per message #1.
- demand: 0 objection(s). This is a deterministic bug fix with a common trigger (bulk load or delete, then ANALYZE or VACUUM in one session), reproduced on 14 through master, and measured at 20 extra autovacuum or autoanalyze runs per 20 tables. Whether it is a bug at all (message #10) is a desirability and design question for other angles; demand itself is shown.

### Precedents retrieved

- P1: [Revert "Fix issues with Windows' stat() for files pending on deletion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c9b46c090e76c62f24563b9be2c34e6b92e9329) (revert, Reverted, 2021-07-12) - judged not relevant
- P2: [Revert analyze support for partitioned tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8127b7390119c21479f5ce495b7d2168930e82) (revert, Reverted, 2021-08-16) - used in the review
- P3: [Revert "Show index search count in EXPLAIN ANALYZE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d00107cd63e780753aa25563fa37603369997d0c) (revert, Reverted, 2025-03-05) - judged not relevant
- P4: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P5: [Revert "vacuumdb: Add option for analyzing only relations missing stats."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19e57f4f78e4354d9a21c284868373d28bb0d368) (revert, Reverted, 2025-03-07) - judged not relevant
- P6: [Flush pgstats file during checkpoints](https://commitfest.postgresql.org/patch/5047/) (cf, Returned with feedback, 2024-10-02) - judged not relevant; reason summarised from its thread
- P7: [explain to report bucket and memory stats for TupleHashTable nodes](https://commitfest.postgresql.org/patch/2428/) (cf, Returned with feedback, 2020-07-23) - judged not relevant; reason summarised from its thread
- P8: [Change tuple table slot for Unique node to "virtual"](https://commitfest.postgresql.org/patch/4531/) (cf, Withdrawn, 2024-03-28) - judged not relevant; reason summarised from its thread
- P9: [Fix pg_stat_statements display of normalized FETCH counts](https://commitfest.postgresql.org/patch/6756/) (cf, Withdrawn, 2026-05-12) - judged not relevant; reason summarised from its thread
- P10: [Allow to specify a index name as ANALYZE parameter](https://commitfest.postgresql.org/patch/1721/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P11: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P12: [Enhance file_fdw to report processed and skipped tuples in COPY progress](https://commitfest.postgresql.org/patch/5282/) (cf, Withdrawn, 2025-03-09) - judged not relevant; reason summarised from its thread
- P13: [System username in pg_stat_activity](https://commitfest.postgresql.org/patch/4769/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P14: [Making autovacuum logs indicate if insert-based threshold was the triggering condition](https://commitfest.postgresql.org/patch/3802/) (cf, Withdrawn, 2022-11-27) - judged not relevant
- P15: [Track skipped vacuum and analyze in pg_stat_all_tables](https://commitfest.postgresql.org/patch/6640/) (cf, Withdrawn, 2026-04-03) - judged not relevant; reason summarised from its thread
- P16: [Decoupling our alignment assumptions about int64 and double](https://commitfest.postgresql.org/patch/6441/) (cf, Withdrawn, 2026-02-03) - judged not relevant
- P17: [Revert ill-considered change in pg_resetwal output.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=918de37652138da7f7e54c79351178367d60f501) (revert, Reverted, 2022-09-15) - judged not relevant
- P18: [Flush some statistics within running transactions](https://commitfest.postgresql.org/patch/6389/) (cf, Returned with feedback, 2026-03-25) - used in the review
- P19: [Resume vacuum and autovacuum from interruption and cancellation](https://commitfest.postgresql.org/patch/2211/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P20: [Add autovacuum_warning to surface concurrent vacuum collisions](https://commitfest.postgresql.org/patch/6830/) (cf, Withdrawn, 2026-08-22) - judged not relevant
- P21: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant
- P22: [use AV work items infrastructure for GIN pending list's cleanup](https://commitfest.postgresql.org/patch/3060/) (cf, Returned with feedback, 2021-09-08) - judged not relevant
- P23: [Revert "Rename routines for write/read of pgstats file"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=381b3cfe2be02aa114ef7a28c4ec24feaf377c63) (revert, Reverted, 2026-07-14) - judged not relevant
- P24: [Compress prune/freeze records with Delta Frame of Reference algorithm](https://commitfest.postgresql.org/patch/6574/) (cf, Returned with feedback, 2026-07-07) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (prior-attempts-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 116 s; cost reported by the CLI: $1.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 64 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:57:08.446Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
