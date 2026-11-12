# Design pre-review: [BUG] Fix EXCLUDED virtual generated columns using tableoid

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7295](https://commitfest.postgresql.org/patch/7295/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [\[BUG\] ON CONFLICT DO UPDATE SET x = EXCLUDED.<virtual-generated-column> errors or silently writes NULL](https://www.postgresql.org/message-id/flat/CAHg%2BQDf7wTLz_vqb1wi1EJ_4Uh%2BVxm75%2Bb4c-Ky%3D6P%2ByOAHjbQ%40mail.gmail.com), started 2026-04-16
- Messages read: 11 of 11
- Patch prose read: vgc-excluded-fix.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the choice to raise an error: v2 follows Dean Rasheed's argument that EXCLUDED has no system columns. Two fixable design points remain. standards-compat-1 asks whether a back-patch to v18 may make statements fail that run today. standards-compat-2 asks whether stored and virtual columns over tableoid now behave differently in EXCLUDED. Both are release and consistency decisions for a committer who knows this area.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v2 patch and its commit message were not shown, so the placement of the check and its coverage of all expansion paths could not be judged. The behaviour of the unpatched WHERE cases rests on the objection's account of message #11.

## The proposal, restated

**Problem.** Commit 783425175 (back-patched to v18) stopped expanding virtual generated columns in the EXCLUDED target list. Since then, a virtual column whose expression uses tableoid, referenced as excluded.c in ON CONFLICT DO UPDATE, fails with the internal error "variable not found in subplan target lists". In some WHERE cases a reviewer saw it run without error instead.

**Proposed design.** v1 added a tableoid entry to the EXCLUDED target list so the reference resolves. After Dean Rasheed objected that EXCLUDED is a proposed row with no system columns, v2 rejects the expanded tableoid reference with a user-facing error. The check sits in expand_virtual_generated_columns() in prepjointree.c, with a regression test.

**User-visible surface.**
- New error: cannot use system column "tableoid" in EXCLUDED, raised when excluded.<virtual column> expands to an expression using tableoid.
- Statements that ran before, such as WHERE excluded.c IS NOT NULL with a tableoid-based column, now fail with this error.

**What the author claims.**
- v2 turns the internal "variable not found in subplan target lists" error into a proper error for the reported case.
- The author is unsure whether to back-patch to PG 18.

**Thread state.** Dean Rasheed argued for an error and Ayush posted v2 to match. João Rito tested v2 on master and confirmed it works, and suggested naming the column in the message; placement, message wording and back-patching are still open.

## 2. Prior attempts

### COPY WHERE clause generated/system column reference (related)

- **Link:** [https://commitfest.postgresql.org/patch/6179/](https://commitfest.postgresql.org/patch/6179/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CACJufxHb8YPQ095R_pYDr77W9XKNaXg5Rzy-WP525mkq+hRM3g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CACJufxHb8YPQ095R_pYDr77W9XKNaXg5Rzy-WP525mkq+hRM3g@mail.gmail.com)
- **Date:** decided 2026-01-08; started 2025-10-27
- **Outcome:** Withdrawn (cf)
- **Who objected:** Masahiko Sawada, Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Peter Eisentraut committed a smaller fix that rejects generated columns in COPY FROM WHERE with a clear error, and left tableoid for a separate thread. He then argued that support should not be added, because generated columns belong to the target table while the WHERE clause filters input data. Masahiko Sawada had shown that computing stored generated columns before the WHERE clause runs them before BEFORE INSERT triggers, so the WHERE clause changes the row the trigger sees.
- **Reason (quote):** "This seems fundamentally wrong, because generated columns are a property of the target table, but the COPY WHERE clause is meant to filter the input data." ([message](https://www.postgresql.org/message-id/70f3402f-30a2-48e1-9fb3-a4da2e77296f%40eisentraut.org))
- **What is different this time:** P1 tried to make generated columns work in COPY FROM WHERE, which filters input rows. v2 here does what that thread chose: it rejects the reference with a clear error instead of giving the proposed row a value.


## 3. Objections a reviewer is likely to raise

### 1. Back-patching a new error breaks statements that run today on v18

fixable · level 2, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

An application on 18.x runs INSERT ... ON CONFLICT DO UPDATE ... WHERE excluded.c IS NOT NULL, where c is a virtual column over tableoid. Message #11 shows this succeeds and updates rows on the unpatched build. After a minor update with v2, the same statement fails with the new error, and the application stops writing at upgrade time.

Message #11 shows WHERE cases that run without error before v2 and fail after it, on plain and partitioned tables. The CF entry targets stable, and message #10 leaves the back-patch open. Turning an internal error into a user-facing error is safe to back-patch, but these cases do not raise an internal error today. The proposal does not say why they work or what tableoid value they see, so nobody can judge whether their results are wrong enough to justify failing them in a minor release.

Thread: [#5 (Alexander Lakhin, 2026-09-11)](https://www.postgresql.org/message-id/18fbeb33-9631-4f6e-9280-2c787aef254e%40gmail.com), [#10 (Ayush Tiwari, 2026-09-13)](https://www.postgresql.org/message-id/CAJTYsWXH1WxRbD3KBrxMm%2BApHKC_7zwhhY2BhK9Gk1fOZHKo9A%40mail.gmail.com), [#11 (João Rito, 2026-11-12)](https://www.postgresql.org/message-id/CADMs5OPKxkM53-R_Us5MG0NHoAkQrjckcPw3dRLj4sPaTa7_%2BA%40mail.gmail.com)

**Question:** Do the WHERE cases that succeed today give wrong results, and if not, why should a minor release make them fail?

Counter-check: held. Message #11 shows WHERE excluded.c IS NOT NULL runs on the unpatched build and fails with v2. Message #10 leaves the back-patch open, and the thread does not explain why those cases run today.

### 2. Stored and virtual columns over tableoid may differ in EXCLUDED

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax · `standards-compat-2`

A user defines c as GENERATED ALWAYS AS (tableoid) STORED and uses excluded.c in ON CONFLICT DO UPDATE. They then change c to VIRTUAL. If the stored form likely yields the target partition's oid, the same statement now fails with the new error, and the user finds out only when it runs.

Dean's case in message #9 compares the rule with a direct reference to excluded.tableoid, which is a reasonable comparison. It does not cover a stored column over tableoid, whose value is computed for the proposed row before the conflict check. Message #11 tested a stored column over b * 10 only. If the stored form works, v2 adds a new difference between virtual and stored columns, and the documentation would need to describe it.

Thread: [#8 (Ayush Tiwari, 2026-09-12)](https://www.postgresql.org/message-id/CAJTYsWW2uGkcL%3DE7GuOWTtzFSxh3wEm%2BxGz63XquhOJ0kywr5g%40mail.gmail.com), [#9 (Dean Rasheed, 2026-09-13)](https://www.postgresql.org/message-id/CAEZATCUQaAuBiKyHrUevTWsnM9yxSNtgAKp2twyv_eQ2mHkYpw%40mail.gmail.com), [#11 (João Rito, 2026-11-12)](https://www.postgresql.org/message-id/CADMs5OPKxkM53-R_Us5MG0NHoAkQrjckcPw3dRLj4sPaTa7_%2BA%40mail.gmail.com)

**Question:** What does excluded.c return for a stored generated column over tableoid, and is the virtual-only error documented as an intended difference?

Counter-check: held. Postgres allows tableoid in a stored generation expression, and the thread does not compare that case with the new virtual-only error. The stored-to-virtual scenario is weak because it needs a drop and re-add of the column, but the consistency question is unanswered.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | v2 makes ON CONFLICT DO UPDATE reject excluded.<virtual column> when the column's expression uses tableoid. |
| new GUC | no |  |
| planner behaviour change | **yes** | The check sits in expand_virtual_generated_columns() in prepjointree.c and replaces a planner internal error. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CF entry targets stable, and v2 adds an error for WHERE cases that run today on v18. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new user-facing error message and a virtual-only rule for tableoid in EXCLUDED need to stay consistent with stored columns. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do the WHERE cases that run without error on unpatched v18 return wrong results, and which tableoid value do they see?
2. If those WHERE cases return correct results, why should a minor release of v18 make them fail?
3. What does excluded.c return for a stored generated column whose expression uses tableoid?
4. If the stored form works, is the virtual-only error an intended difference, and where will the documentation describe it?
5. Will the patch go to v18 as well as master, and does the v2 commit message say so?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Eisentraut is the assigned committer on the CF entry, and Dean Rasheed set the direction in the thread. The owner needs to know virtual generated column expansion and ON CONFLICT planning.
- **Liability:** The owner signs for a new user-facing error in ON CONFLICT DO UPDATE and, if back-patched, for statements that stop working in a v18 minor release. The check is small and adds no persistent state.
- **Author follow-up:** The author posted v2 after Dean Rasheed's objection, which shows response to review. The author has left the back-patch decision open.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Few users define a virtual generated column over tableoid and reference it through EXCLUDED, but the current result is an internal error in v18. The check is small, a committer is assigned, and the open points are the back-patch decision and the stored-column comparison.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The only related precedent, P1, rejected giving input-side rows target-table values. v2 follows that reasoning, as Dean Rasheed argued in messages #7 and #9, so no earlier rejection reason applies against it. The earlier fix in this thread (message #4, commit 783425175) caused this regression and was not rejected.
- standards-compat: 2 objection(s). Other system columns are not a gap, because generation expressions already reject system columns other than tableoid. The message wording point in message #11 is minor and left to review.
- replication-ondisk: 0 objection(s). v2 adds a parse-time check in expand_virtual_generated_columns() and nothing persistent or cross-node: no on-disk, WAL, catalog, protocol or ABI change. Virtual columns are not stored, so old or upgraded data is unaffected; whether to back-patch to v18 is a compatibility question for another angle.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. v2 replaces an internal error with a user-facing error when excluded.<virtual column> expands to tableoid; it grants no new access, reveals no data, and adds no denial-of-service path. The error names only a column of a table the user already has privileges to insert into.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The change is a small check that runs only while virtual generated columns are being expanded, and it adds no GUC, catalog or API. Dean Rasheed knows the area and set the direction in message #7 and message #9, and Peter Eisentraut is the assigned committer, so ownership is clear. Generation expressions already reject system columns other than tableoid, so a tableoid-only check covers every case. [P1] concerns a different feature, COPY FROM WHERE, and does not apply here. The open questions are about release handling, not long-run maintenance: whether a new error belongs in a back-patch to 18, and why some unpatched WHERE cases ran without error.
- already-exists: 0 objection(s). The proposal fixes a planner internal error in core ON CONFLICT handling. No existing feature, setting or extension can avoid the error or reject the case properly, so the need is not already met. The one existing behaviour, the parser error for direct excluded.tableoid, is what v2 follows; message wording is out of scope here.
- demand: 0 objection(s). This is a bug fix: since commit 783425175 the reported statement fails with an internal error, so demand evidence is not needed. Back-patching and the WHERE cases that ran before the patch and now fail are back-patch and design questions, outside the demand angle.

### Precedents retrieved

- P1: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - used in the review; reason summarised from its thread
- P2: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant; reason summarised from its thread
- P3: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P4: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P5: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - judged not relevant; reason summarised from its thread
- P6: [add option to pg_dumpall to exclude tables from the dump](https://commitfest.postgresql.org/patch/671/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P7: [pg_dumpall --exclude-database option](https://commitfest.postgresql.org/patch/1743/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P8: [Reduce the log spam by excluding certain SQLSTATEs](https://commitfest.postgresql.org/patch/4964/) (cf, Withdrawn, 2025-04-03) - judged not relevant; reason summarised from its thread
- P9: [Exclude partition constaint checking in query execution plan for partitioned table queries](https://commitfest.postgresql.org/patch/1264/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P10: [Revert "Fix bug in checking of IDENTIFY_SYSTEM result."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c4762886539b06da682045e19ba73dcee3789638) (revert, Reverted, 2014-08-19) - judged not relevant
- P11: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P12: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant; reason summarised from its thread
- P13: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant
- P14: [Revert "Allow ON CONFLICT .. DO NOTHING on a partitioned table."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f05230752d53c4aa74cffa9b699983bbb6bcb118) (revert, Reverted, 2017-03-31) - judged not relevant
- P15: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1148/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P16: [JIT: Optimize generated functions earlier to lower memory usage](https://commitfest.postgresql.org/patch/2298/) (cf, Withdrawn, 2019-11-12) - judged not relevant
- P17: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1568/) (cf, Returned with feedback, 2018-03-01) - judged not relevant
- P18: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant
- P19: [Allow excluding pg_temp or pg_catalog from search_path.](https://commitfest.postgresql.org/patch/4508/) (cf, Withdrawn, 2023-11-17) - judged not relevant
- P20: [Add and report the new "session_read_only" GUC pseudo-variable.](https://commitfest.postgresql.org/patch/1090/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P21: [Subplan result cache](https://commitfest.postgresql.org/patch/1646/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P22: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant
- P23: [Remove dead macro exec_subplan_get_plan](https://commitfest.postgresql.org/patch/3894/) (cf, Rejected, 2023-04-04) - judged not relevant
- P24: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 136 s; cost reported by the CLI: $1.21
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 54 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:37:45.812Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
