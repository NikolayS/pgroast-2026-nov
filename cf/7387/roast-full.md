# Design pre-review: Refactor pgbench - pure mechanical modularization into 10 module pairs (stats, variable, poller, random, expr, script, commands, init, engine, report)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7387](https://commitfest.postgresql.org/patch/7387/) (Needs review); authors: Hannu Krosing (hannukrosing)
- Thread: [\[PATCH\] Refactor pgbench to make future improvements easier](https://www.postgresql.org/message-id/flat/CAMT0RQT1BMSKqgO31QJw4razSDruW_HX6G-R_Xr7yXsfMM_H2g%40mail.gmail.com), started 2026-10-03
- Messages read: 14 of 14
- Patch prose read: v7-0001-Refactor-pgbench-extract-statistics-structures-an.patch, v7-0002-Refactor-pgbench-extract-variable-management-and-.patch, v7-0003-Refactor-pgbench-extract-socket-polling-abstracti.patch, v7-0004-Refactor-pgbench-extract-PRNG-distributions-hashi.patch, v7-0005-Refactor-pgbench-extract-expression-and-built-in-.patch, v7-0006-Refactor-pgbench-extract-script-parsing-and-built.patch, v7-0007-Refactor-pgbench-extract-meta-command-execution-a.patch, v7-0008-Refactor-pgbench-extract-database-initialization-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The surviving objections demand-1 and maintenance-fit-1 both fail level 1b. The split has no user-visible effect, and the only named consumer is the author's own later work, which message #12 still lists as a follow-up. Messages #13 and #14 ask to start from one feature with only the refactoring it needs, and the thread has no answer to that yet. The goal may be fine, but the list has to agree on the approach before code review makes sense.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: No one has confirmed that v7 is move-only; that claim and the amount of coupling through exported globals need the code. The commit messages for 0009 and 0010 were not in the material.

## The proposal, restated

**Problem.** pgbench.c is a single file of about 8,000 lines that mixes statistics, variables, polling, script parsing, meta-commands, initialization and the client state machine. The author says this makes later pgbench work harder, in particular his planned contrib/pgbench extension and features for TPC-like workloads. Reviewers found the motivation weak without a concrete feature that needs the split.

**Proposed design.** v7 moves code out of pgbench.c into ten module pairs: stats, variable, poller, random, expr, script, commands, init, engine and report. The author states that function bodies and signatures stay identical; the only changes are dropping static on cross-file functions and globals, adding extern declarations, and moving two static inline timing helpers to stats.h. After the series pgbench.c keeps about 930 lines for option globals, usage() and main().

**What the author claims.**
- All 136 function definitions keep identical bodies, parameter lists and return types at every commit 0001 to 0010.
- Remaining code in pgbench.c is not reordered; each diff is only include updates, static removals and block deletions.
- Earlier versions built with Make and Meson and passed all 686 pgbench TAP tests.
- The split prepares moving random distributions, hashing and permute into src/common and exposing them in SQL through contrib/pgbench.

**Thread state.** Zsolt Parragi and Koshi Shibagaki asked for move-only patches; Parragi found v6 0004 still reordered pgbench.c, and v7 claims to fix this. Dmitry Dolgov and Shibagaki then asked to drop the split-only approach and start from one feature, such as the distribution generators, with only the refactoring it needs.

## 2. Prior attempts

### split tablecmds.c (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/6269/](https://commitfest.postgresql.org/patch/6269/)
- **Thread:** [https://www.postgresql.org/message-id/flat/aS3PeXHLe7ir_gPc@nathan](https://www.postgresql.org/message-id/flat/aS3PeXHLe7ir_gPc@nathan)
- **Date:** decided 2026-08-24; started 2025-12-01
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P14 split a backend file, tablecmds.c; this splits a client program, pgbench.c. The reason for P14's withdrawal is not recorded, so it gives no lesson here.

### Split xlog.c (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3173/](https://commitfest.postgresql.org/patch/3173/)
- **Thread:** [https://www.postgresql.org/message-id/flat/b3b71061-4919-e882-4857-27e370ab134a@iki.fi](https://www.postgresql.org/message-id/flat/b3b71061-4919-e882-4857-27e370ab134a@iki.fi)
- **Date:** decided 2021-11-15; started 2021-06-16
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P15 split xlog.c in the backend; this splits pgbench.c into ten modules. The reason P15 was returned is not recorded.


## 3. Objections a reviewer is likely to raise

### 1. No concrete feature or user shown to need this module layout

fixable · level 1b, rubric item 2 · demand, maintenance burden ("who maintains this in 10 years"), backpatchability · `demand-1`

After the split is committed, every open pgbench patch must be rebased onto ten new files, and each back-patched pgbench fix must be adapted by hand. If the feature that motivated the split never lands, or needs different boundaries, the project pays that cost and may move the code a second time.

The only stated beneficiary is the author's own planned work: contrib/pgbench and TPC-like features (messages #3, #12). No other user or developer asks for the split. Dolgov calls the motivation weak and asks for one feature with only the refactoring it needs (message #13); Shibagaki withdraws support for split-only review for the same reason (message #14). Message #12 says the split helps "anyone wanting to improve pgbench" but names no such person or patch, and that argument would fit any large source file.

**Precedents.**
- [Docs: make source file path references consistent](https://commitfest.postgresql.org/patch/6380/) (Withdrawn, 2026-01-08); thread: [link](https://www.postgresql.org/message-id/flat/CAEoWx2n=X2vvkJqsNDENgqkyw_adG=7wSHG-og-dLbyM4G_7SA@mail.gmail.com)

Thread: [#3 (Hannu Krosing, 2026-10-04)](https://www.postgresql.org/message-id/CAMT0RQSQiLuOWw6R750Dn3hPC3qy_mJrBEst-Xrw68tqSkYWHQ%40mail.gmail.com), [#12 (Hannu Krosing, 2026-11-01)](https://www.postgresql.org/message-id/CAMT0RQTKqefLQJropNPKsSo1i%3DahxjbGRFHJUT6A%2BUn37q4wwQ%40mail.gmail.com), [#13 (Dmitry Dolgov, 2026-11-04)](https://www.postgresql.org/message-id/autS5tAQoZWpZQCL%40ddolgov-thinkpadt14sgen1.rmtde.csb), [#14 ("Koshi Shibagaki (Fujitsu)", 2026-11-05)](https://www.postgresql.org/message-id/TY5PR01MB1844582E13D20621BE09CA1DCFA772%40TY5PR01MB18445.jpnprd01.prod.outlook.com)

**Question:** Which concrete pgbench feature, wanted by someone besides the author, cannot reasonably be built on today's pgbench.c, and which of the ten modules does it need?

Counter-check: held. Messages #13 and #14 ask for one feature with only the refactoring it needs, and message #12 names that feature only as a follow-up. Shibagaki's earlier interest in the refactoring is the one sign of wider demand, and message #14 makes it conditional on a concrete use case.

### 2. Module boundaries chosen before any feature that would use them

fixable · level 1b, rubric item 11 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

The split goes in, then the planned distribution or contrib/pgbench work needs random.c in src/common and a different expr/commands interface. A second large reshuffle follows. Back-branch fixes and git blame in pgbench pay for both rounds.

v7 makes about 66 functions and many globals external, including throttle_delay, timer_exceeded, use_quiet and scale, without any encapsulation. The new files are therefore coupled through shared globals, so the split adds files but not real interfaces. Messages #13 and #14 ask for one feature first, so that the boundaries are judged against a real consumer. Message #12 still names that feature only as a later follow-up.

Thread: [#12 (Hannu Krosing, 2026-11-01)](https://www.postgresql.org/message-id/CAMT0RQTKqefLQJropNPKsSo1i%3DahxjbGRFHJUT6A%2BUn37q4wwQ%40mail.gmail.com), [#13 (Dmitry Dolgov, 2026-11-04)](https://www.postgresql.org/message-id/autS5tAQoZWpZQCL%40ddolgov-thinkpadt14sgen1.rmtde.csb), [#14 ("Koshi Shibagaki (Fujitsu)", 2026-11-05)](https://www.postgresql.org/message-id/TY5PR01MB1844582E13D20621BE09CA1DCFA772%40TY5PR01MB18445.jpnprd01.prod.outlook.com)

**Question:** Which part of the distribution-generator or contrib/pgbench work needs exactly this ten-module layout, and would random.c move again for it?

Counter-check: held. The v7 commit messages confirm that the split exports globals such as throttle_delay, timer_exceeded, use_quiet and scale unchanged. Messages #1 and #6 plan to move the random code to src/common, so a second move of random.c is the author's own stated next step.

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
| backpatchability | **yes** | Fixes to code moved out of pgbench.c need manual ports to older branches; message #2 counts about four such commits per year. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The series adds ten module pairs and makes about 66 functions and many globals external inside src/bin/pgbench. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which concrete pgbench feature needs these module boundaries, and which of the ten modules does that feature use?
2. Does anyone besides the author have a pgbench patch or plan that today's single pgbench.c blocks or makes harder?
3. Will random.c move again to src/common for the contrib/pgbench work, and if so, why not move it there in one step?
4. Can the series be reduced to the distribution-generator feature plus only the code moves it needs, as messages #13 and #14 ask?
5. Where is the promised table of which function goes to which module, and what do patches 0009 and 0010 move?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows pgbench and its back-branch history would have to own it. The material shows reviewers in the thread but does not identify any of them as a committer.
- **Liability:** The owner signs for a move-only claim across about 7,000 moved lines, and for manual ports of pgbench fixes to older branches for about five years.
- **Author follow-up:** The author answered reviews and posted reworked versions up to v7 within a month, and plans follow-up work on top of the split.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A posted feature patch, such as SQL-callable distribution generators, that uses specific module boundaries from this series, or pgbench patches from other developers that the single file demonstrably hinders.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The split changes nothing for users and its only named consumer is unposted follow-up work, while verifying about 7,000 moved lines as move-only is costly. Reviewers in messages #13 and #14 ask for a feature-first series with only the refactoring that feature needs.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Split has no feature that needs it; earlier refactor-only steps stalled for this. Refuted (not specific): Duplicate of demand-1, which makes the same point from messages #13 and #14 without leaning on precedents. [P2] was returned for not going far enough and [P4] is a documentation cleanup, so neither shows that an earlier split-only step stalled for this reason.
- `maintenance-fit-2` Every back-patched pgbench fix must be hand-ported across the split. Refuted (already handled): Message #2 measured the cost at about four back-patched pgbench commits per year and judged it acceptable, and no one in the thread disputed that. The remaining point, that no gain is shown to justify the cost, is the same as demand-1.

### Review angles

- prior-attempts: 1 objection(s). Back-patching cost is not raised separately, because message #2 estimates about four back-patched pgbench commits per year and calls that manageable. The thread also points to an earlier proposal, CF 7225 for contrib/pgbench (messages #3 and #14). Its outcome is not in the material.
- standards-compat: 0 objection(s). From the standards and compatibility angle the series changes nothing users see: no SQL syntax, GUCs, pgbench options, script language, output formats or dump behaviour. Back-patching was raised in message #2, where the reviewer judged about four back-patched pgbench commits a year to be manageable. Whether v7 is really move-only needs the code and is outside this angle.
- replication-ondisk: 0 objection(s). The series moves code inside the pgbench client binary and adds no persistent or cross-node state: no on-disk format, WAL, catalog, wire protocol or server ABI change. The cost of back-patching pgbench fixes was raised and judged acceptable in message #2, at about four back-patched commits per year.
- security: 0 objection(s). v7 is a move-only split of the pgbench client source. It adds no option, no SQL object, no server-side code and no default. It does not touch privileges, authentication or what a user can see. Removing static on symbols inside one client binary does not expose anything to other users. The later contrib/pgbench extension would need its own security review, but it is not part of this proposal.
- maintenance-fit: 2 objection(s). No hot-path, catalog or external-dependency cost: the change is limited to the pgbench client binary.
- already-exists: 0 objection(s). v7 is a move-only split of pgbench.c with no user-visible behaviour, so no existing feature, contrib module or extension does the same job. The later goal of exposing pgbench distributions in SQL is not part of v7, and I cannot name an existing extension that reliably provides them, so nothing is raised from this angle.
- demand: 1 objection(s). One demand objection only. The split has no direct user-visible effect, so demand has to come from a feature built on top of it, and none is shown yet. The suggestion in message #14 to start from the distribution generators would supply that.

### Precedents retrieved

- P1: [pgbench - add random_zipfian function](https://commitfest.postgresql.org/patch/1206/) (cf, Returned with feedback, 2018-06-26) - judged not relevant; reason summarised from its thread
- P2: [Split StdRdOptions into HeapOptions and ToastOptions](https://commitfest.postgresql.org/patch/2370/) (cf, Returned with feedback, 2021-10-04) - used in the review; reason summarised from its thread
- P3: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant; reason summarised from its thread
- P4: [Docs: make source file path references consistent](https://commitfest.postgresql.org/patch/6380/) (cf, Withdrawn, 2026-01-08) - used in the review; reason summarised from its thread
- P5: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/4792/) (cf, Returned with feedback, 2024-10-02) - judged not relevant; reason summarised from its thread
- P6: [array_random](https://commitfest.postgresql.org/patch/5881/) (cf, Withdrawn, 2026-01-06) - judged not relevant; reason summarised from its thread
- P7: [Introduce some randomness to autovacuum](https://commitfest.postgresql.org/patch/5732/) (cf, Rejected, 2026-06-07) - judged not relevant; reason summarised from its thread
- P8: [Add WAL recovery info such as current WAL file and WAL source to server logs](https://commitfest.postgresql.org/patch/3443/) (cf, Withdrawn, 2022-10-26) - judged not relevant; reason summarised from its thread
- P9: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant
- P10: [Enforce creation of destination folders for source files in pg_regress](https://commitfest.postgresql.org/patch/100/) (cf, Rejected, 2015-03-01) - judged not relevant
- P11: [pgbench - more operators and functions](https://commitfest.postgresql.org/patch/596/) (cf, Returned with feedback, 2017-01-25) - judged not relevant
- P12: [Review of arc4random for strong randomness](https://commitfest.postgresql.org/patch/1382/) (cf, Rejected, 2018-01-12) - judged not relevant
- P13: [Reorganize CacheMemoryContext](https://commitfest.postgresql.org/patch/5337/) (cf, Withdrawn, 2024-10-29) - judged not relevant
- P14: [split tablecmds.c](https://commitfest.postgresql.org/patch/6269/) (cf, Withdrawn, 2026-08-24) - used in the review
- P15: [Split xlog.c](https://commitfest.postgresql.org/patch/3173/) (cf, Returned with feedback, 2021-11-15) - used in the review
- P16: [pgbench: allow to cancel queries during benchmark](https://commitfest.postgresql.org/patch/4399/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P17: [Improving the heapgetpage function improves performance in common scenarios](https://commitfest.postgresql.org/patch/4524/) (cf, Withdrawn, 2023-09-06) - judged not relevant
- P18: [pgbench - implement strict TPC-B benchmark](https://commitfest.postgresql.org/patch/2090/) (cf, Rejected, 2019-11-06) - judged not relevant
- P19: [psql - refactor echo code](https://commitfest.postgresql.org/patch/3140/) (cf, Returned with feedback, 2023-04-03) - judged not relevant
- P20: [Revert "Reorganise jsonpath operators and methods"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d) (revert, Reverted, 2024-01-03) - judged not relevant
- P21: [Refactoring of heapam code](https://commitfest.postgresql.org/patch/700/) (cf, Returned with feedback, 2016-09-26) - judged not relevant
- P22: [pgbench --continue-on-error: clarify TPS and failure reporting](https://commitfest.postgresql.org/patch/6942/) (cf, Withdrawn, 2026-06-28) - judged not relevant
- P23: [Collation code refactoring](https://commitfest.postgresql.org/patch/4058/) (cf, Withdrawn, 2023-01-11) - judged not relevant
- P24: [Reorganize collation lookup time and place](https://commitfest.postgresql.org/patch/1956/) (cf, Returned with feedback, 2019-02-01) - judged not relevant

### Warnings

- attachment limit (8) reached; v7-0009-Refactor-pgbench-extract-benchmark-execution-engi.patch not read
- attachment limit (8) reached; v7-0010-Refactor-pgbench-extract-transaction-logging-prog.patch not read
- attachment limit (8) reached; v2-0001-Refactor-pgbench-extract-statistics-and-latency-t.patch not read
- attachment limit (8) reached; v2-0004-Refactor-pgbench-extract-script-parsing-and-expre.patch not read
- attachment limit (8) reached; v2-0002-Refactor-pgbench-extract-scoped-variable-store-an.patch not read
- attachment limit (8) reached; v2-0007-Refactor-pgbench-extract-execution-engine-state-m.patch not read
- attachment limit (8) reached; v2-0006-Refactor-pgbench-extract-initialization-schema-an.patch not read
- attachment limit (8) reached; v2-0003-Refactor-pgbench-extract-socket-multiplexing-and-.patch not read
- attachment limit (8) reached; v2-0005-Refactor-pgbench-extract-meta-command-execution-a.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 161 s; cost reported by the CLI: $3.30
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 64 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:49:12.812Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
