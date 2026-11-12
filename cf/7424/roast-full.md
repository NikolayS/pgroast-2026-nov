# Design pre-review: Fix out-of-bounds access in pg_bsd_indent's parser stack

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7424](https://commitfest.postgresql.org/patch/7424/) (Needs review); authors: Vadim Shakirov (vadimshakirov)
- Thread: [\[PATCH v1\] Fix out-of-bounds access in pg_bsd_indent's parser stack](https://www.postgresql.org/message-id/flat/20260930105134.568062-1-vadimsakirov5%40gmail.com), started 2026-09-30
- Messages read: 4 of 4
- Patch prose read: none (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated in message #1, and message #3 reproduces a segfault that filled a disk and confirms the patch turns it into a "Parser stack overflow" error. The fix changes two array sizes and one bound check in a developer tool, so it adds no cost for server users. Confidence is medium because no one has reviewed the code yet.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only the diff in message #1 was read, not the rest of parse.c. Whether every push in parse() and every use of il and cstk stays within the new bounds could not be checked.

## The proposal, restated

**Problem.** pg_bsd_indent indexes three arrays in parser_state (p_stack, il, cstk) with one shared index, tos, but checks overflow only against p_stack, the largest. Deeply nested input can therefore write past the end of il and cstk. This affects developers who run pgindent on source code.

**Proposed design.** The patch sizes il and cstk at 256 entries, the same as p_stack. It also makes the overflow check one element stricter, because the lbrace case pushes two entries (lbrace, then stmt) in one call to parse().

**User-visible surface.**
- pg_bsd_indent now stops with "Parser stack overflow" on deeply nested input instead of crashing.

**What the author claims.**
- The overflow check used only p_stack, so cstk and il could be accessed out of bounds.
- Giving all three arrays the same size closes the gap.
- The lbrace case needs room for two more entries when parse() starts.

**Thread state.** Daniel Gustafsson asked for a reproducer. Álvaro Herrera built one with 256 nested if statements and a switch, which segfaulted and filled his disk; with the patch, pg_bsd_indent stops with "Parser stack overflow". No one has reviewed the code yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-30; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | **yes** | The CommitFest entry targets version 16, but the thread does not say which branches should get the fix. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch changes src/tools/pg_bsd_indent, code derived from FreeBSD indent, so it adds a small divergence from upstream. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which branches should get the fix, given that the CommitFest entry targets version 16?
2. Can any other path in parse() push more than two entries per call, so that the "- 2" bound is still too loose?
3. Should the reproducer from message #3 become a regression test for pg_bsd_indent?
4. Does upstream FreeBSD indent have the same defect, and if it is fixed there, does that fix match this one?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson and Álvaro Herrera take part in the thread, and Álvaro tested the patch in message #3; neither has reviewed the code. The owner needs to know the pg_bsd_indent parser and pgindent.
- **Liability:** The owner signs for a bounds fix in a developer tool: that the new check covers every push in parse(), and that pgindent output does not change. A back-patch extends this to each branch that carries the tool.
- **Author follow-up:** No sign either way; the author does not post again in the visible messages after message #1.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch changes three lines in two files of a developer tool and has a confirmed reproducer in message #3. It helps only developers who run pgindent, but it adds no user-visible surface and little to maintain.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at this fix, and no thread message points to an earlier discussion. Nothing from the prior-attempts angle stands against this small bug fix, which message #3 confirms with a reproducer.
- standards-compat: 0 objection(s). The patch fixes a crash in a developer-only tool. For input within the old limits it adds no syntax, GUC, catalog object, dump format or client-visible behaviour, and it does not change pgindent output. The stricter check only rejects nesting one level shallower than before, about 254 levels, which no real source file in the tree reaches. Branch targeting is a backpatch policy question, not a compatibility objection.
- replication-ondisk: 0 objection(s). The patch changes only the in-memory parser_state struct of pg_bsd_indent, a developer tool that is not linked into the server or libpq. No on-disk format, WAL, catalog, wire protocol or extension ABI is touched, so nothing here crosses a process or machine boundary.
- security: 0 objection(s). No objection from the security angle. pg_bsd_indent is a developer tool that runs locally on source files the developer chooses, so it has no privilege model, server exposure or user data. The patch makes deeply nested input stop with "Parser stack overflow" instead of writing out of bounds, which reduces risk, including the disk-filling crash shown in message #3.
- maintenance-fit: 0 objection(s). This is a three-line bounds fix in a developer tool that is already in the tree, with a working reproducer in message #3. It adds no runtime cost to the server, no new API, and no external dependency. The larger arrays cost about 1 KiB in one struct, and any divergence from upstream indent is small and fixes a crash. Without the code I could not check whether other pushes in parse() add more than two entries per call, or which branches carry this tool.
- already-exists: 0 objection(s). This is a memory-safety bugfix in pg_bsd_indent, a tool that ships in the source tree. No other feature, option or extension already gives this fix, so no existing alternative applies. Whether to use upstream FreeBSD indent instead is a separate question that the material does not raise.
- demand: 0 objection(s). This is a memory-safety bugfix in a project tool, and the demand angle does not apply in the usual way. Message #3 reproduces a segfault that filled a disk, and the fix changes only the sizes of two arrays and one bound check, so it adds no user surface or maintenance cost to weigh against the size of the user group.

### Precedents retrieved

- P1: [Add code indentation check to cirrus-ci](https://commitfest.postgresql.org/patch/4691/) (cf, Withdrawn, 2024-01-24) - judged not relevant; reason summarised from its thread
- P2: [Revert pgindent length back to 79 because we are going to fix the BSD indent bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19cb4571465024d7068eddf3eda8c9fe6851cde1) (revert, Reverted, 2005-11-13) - judged not relevant
- P3: [Revert pgindent changes to ecpg include files that are part of ecpg regession test output, and update pgindent script to avoid them in the future.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2cc6ff45f8813593e7c8c0757cd13a7b63a41768) (revert, Reverted, 2010-02-26) - judged not relevant
- P4: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant; reason summarised from its thread
- P5: [Revert 9.5 pgindent changes to atomics directory files](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=befa3e648ce018d84cd2a0df701927c56fe3da4e) (revert, Reverted, 2015-05-25) - judged not relevant
- P6: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant; reason summarised from its thread
- P7: [Improve overflow checks potentially optimized away by compiler](https://commitfest.postgresql.org/patch/400/) (cf, Returned with feedback, 2015-12-01) - judged not relevant; reason summarised from its thread
- P8: [Reverted changes made by pgindent Made show.pgc set all variables before displaying them.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fcc2fbac22148ed640dff24caef7c57d20bdad06) (revert, Reverted, 2006-10-04) - judged not relevant
- P9: [prefetching for nestloop / indexscan](https://commitfest.postgresql.org/patch/5208/) (cf, Withdrawn, 2024-11-22) - judged not relevant; reason summarised from its thread
- P10: [Optimize Numeric using fixed buffer reducing pallocs for small numbers](https://commitfest.postgresql.org/patch/4196/) (cf, Returned with feedback, 2023-09-13) - judged not relevant; reason summarised from its thread
- P11: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P12: [Fix out-of-bounds with function PQescapeInternal](https://commitfest.postgresql.org/patch/4910/) (cf, Rejected, 2024-04-04) - judged not relevant; reason summarised from its thread
- P13: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P14: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant
- P15: [postgres.exe doesn't output crash dump when it crashes before main()](https://commitfest.postgresql.org/patch/1525/) (cf, Rejected, 2019-11-12) - judged not relevant
- P16: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant
- P17: [Allow EXPLAIN to Output Page Fault Information](https://commitfest.postgresql.org/patch/5477/) (cf, Withdrawn, 2025-02-08) - judged not relevant
- P18: [Detect escape of ErrorContextCallback stack pointers (and from PG_TRY() )](https://commitfest.postgresql.org/patch/2722/) (cf, Withdrawn, 2020-09-10) - judged not relevant
- P19: [LockHasWaiters() crashes on fast-path locks](https://commitfest.postgresql.org/patch/6732/) (cf, Withdrawn, 2026-09-03) - judged not relevant
- P20: [Revert "Add some temporary code to record stack usage at server process exit."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96112ee7c60557bb192a9aa07b514db2400fd45e) (revert, Reverted, 2016-07-10) - judged not relevant
- P21: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant
- P22: [Fix standby may crash when switching-over in certain special cases](https://commitfest.postgresql.org/patch/5199/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P23: [BUG Segfault triggered by reentrancy in ri_triggers.c](https://commitfest.postgresql.org/patch/6805/) (cf, Withdrawn, 2026-05-29) - judged not relevant
- P24: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 47 s; cost reported by the CLI: $0.46
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 41 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:53:43.209Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
