# Design pre-review: Resetting snapshots during the first phase of [CREATE |RE]INDEX CONCURRENTLY

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6401](https://commitfest.postgresql.org/patch/6401/) (Needs review); authors: Mihail Nikalayeu (mihailnikalayeu)
- Thread: [Resetting snapshots during the first phase of \[CREATE |RE\]INDEX CONCURRENTLY](https://www.postgresql.org/message-id/flat/CADzfLwWAiimwngq38x8Ji26AtYEmSqnFD7aa4kVGF4ObDswHMA%40mail.gmail.com), started 2026-01-15
- Messages read: 13 of 13
- Patch prose read: v8-0001-Add-stress-tests-for-concurrent-index-builds.patch, v8-0005-Allow-snapshot-resets-in-parallel-concurrent-inde.patch, v8-0002-Periodically-reset-snapshots-in-concurrent-index-.patch, v8-0004-Allow-snapshot-resets-in-concurrent-BRIN-builds.patch, v8-0003-Allow-snapshot-resets-in-concurrent-GIN-builds.patch, v8-0006-Allow-snapshot-resets-in-concurrent-unique-index-.patch, v8-0007-Reset-snapshots-after-the-heap-scan-in-concurrent.patch, v1-0001-Add-stress-tests-for-concurrent-index-builds.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and demand for the goal is established by the PG14 feature that [P1] reverted only for safety. The open points are fixable: measurements (maintenance-fit-2, demand-1), a written invariant for the unique-build path (prior-attempts-1), and enforcement of the new AM rules (maintenance-fit-1). The design changes snapshot handling where an error gives a silently incomplete index, so a committer who knows that area has to sign for it.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claims that xmin stays valid during a reset, that the parallel handshake is correct, and that the unique recheck is complete are unchecked. Only 3 of 13 messages were shown.

## The proposal, restated

**Problem.** Long CREATE INDEX CONCURRENTLY and REINDEX CONCURRENTLY runs pin xmin, so vacuum cannot remove dead tuples anywhere in the cluster. The earlier fix in PG14 (d9d0762) let vacuum ignore these snapshots, but it was reverted (e28bb88) because HOT pruning could make the build miss tuples. This affects operators of busy, large databases.

**Proposed design.** The first heap scan replaces its MVCC snapshot at page boundaries after a time interval, and later build loops do the same between tuples; the validation scan keeps one snapshot. Index AMs opt in with a flag, fetch external TOAST values they keep, and parallel workers signal when they have restored the leader's snapshot. Unique builds recheck heap status for equal keys and leave unchecked entries for validation to insert.

**User-visible surface.**
- GUC concurrent_index_reset_snapshot_interval (integer, ms, default 100ms; 0 resets at every opportunity, -1 disables).
- GUC index_build_duplicate_check_scale_factor (float, default 0.05, at least 1000 heap fetches per budget).
- IndexAmRoutine field amcanresetsnapshot, so out-of-core index AMs must opt in.
- C function index_build_rotate_snapshot() for AMs to call between tuples after the scan.
- Wait event PARALLEL_CREATE_INDEX_SNAPSHOT.

**What the author claims.**
- Resetting snapshots in the first scan is safe because validation adds any tuples the scan missed.
- No significant _bt_load slowdown from the extra per-tuple key comparison was observed; no numbers given.
- The stress test detects the 2022 bug and found other bugs that were fixed in master.
- AMs that do not set amcanresetsnapshot keep the old behavior.
- Resets are skipped at REPEATABLE READ and above, where the transaction snapshot pins xmin anyway.

**Thread state.** Matthias van de Meent reviewed v6 in depth and the author reworked it into v8, which has no reply yet; the CF entry needs a rebase. Álvaro Herrera questioned the stress test's runtime and proposed a shared PG_TEST_EXTRA stress_concurrently mechanism.

## 2. Prior attempts

### Revert changes to CONCURRENTLY that "sped up" Xmin advance (similar idea)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b)
- **Thread:** [https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org), [https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql](https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql)
- **Date:** decided 2022-05-31; started 2021-02-23
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit d9d076222f5b "VACUUM: ignore indexing operations with CONCURRENTLY". These changes caused indexes created with the CONCURRENTLY option to miss heap tuples that were HOT-updated and HOT-pruned during the index creation. Before these changes, HOT pruning would have been prevented by the Xmin of the transaction creating the index, but because this change was precisely to allow the Xmin to move forward ignoring that backend, now other backends scanning the table can prune them. This is not a problem for VACUUM (which requires a lock that conflicts with a CREATE INDEX CONCURRENTLY operation), but HOT-prune can definitely occur. In other words, Xmin advancement was sped up, but at the cost of corrupting the resulting index. Regrettably, this means that the new feature in PG14 that RIC/CIC on very large tables no longer force VACUUM to retain very old tuples goes away. We might try to implement it again in a later release, but for now the risk of indexes missing tuples is too high and there's no easy fix. Backpatch to 14, where this change appeared. Diagnosys-by: Andrey Borodin <x4mmm@yandex-team.ru> Diagnosys-by: Michael Paquier <michael@paquier.xyz> Diagnosys-by: Andres Freund <andres@anarazel.de>
- **Reason (quote):** "In other words, Xmin advancement was sped up, but at the cost of corrupting the resulting index."
- **What is different this time:** P1 let vacuum ignore the build backend's xmin while the build kept an old snapshot. This proposal replaces the snapshot itself, so the build always holds a registered snapshot and pruning removes only tuples dead to it.

### avoid bloat from CREATE INDEX CONCURRENTLY (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1047/](https://commitfest.postgresql.org/patch/1047/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20170228052302.r7nvt4sbnsgu6job@alvherre.pgsql](https://www.postgresql.org/message-id/flat/20170228052302.r7nvt4sbnsgu6job@alvherre.pgsql)
- **Date:** decided 2017-03-07; started 2017-02-28
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane objected to the new, undocumented snapmgr API and proposed instead to make index AMs responsible for popping the snapshot. Simon Riggs answered that he had chosen the snapmgr API because an index AM that pops the snapshot wrongly would cause a crash, and he offered to follow consensus. The visible messages show no further discussion before the entry was returned with feedback.
- **Reason (quote):** "However, now that I read the patch: this is a horribly ugly hack." ([message](https://www.postgresql.org/message-id/1983.1488288643%40sss.pgh.pa.us))
- **What is different this time:** P2 released the snapshot once after the first scan, for btree only, through an undocumented snapmgr API. This proposal resets during the scan and between tuples afterwards, through a documented AM call and an opt-in flag, for all built-in AMs.


## 3. Objections a reviewer is likely to raise

### 1. On by default for every concurrent build, with no measurements

fixable · level 1b, rubric item 8 · performance on the hot path, new GUC · `maintenance-fit-2`

An operator upgrades and runs REINDEX CONCURRENTLY on a large unique index. The build now resets snapshots every 100ms, compares keys per tuple in _bt_load(), and may push many entries to validation once the 5% budget runs out. It runs longer, and the material gives no way to predict by how much or how much xmin advances.

Message #1 says no significant _bt_load slowdown was seen but shows no numbers. Message #11 calls the 0.05 default "not measure". Sorting and tuplesort merges still pin one snapshot, so the share of a long build that gains is unknown. Two new user GUCs for one command will be hard to remove later if their defaults turn out wrong.

Thread: [#1 (Mihail Nikalayeu, 2026-01-15)](https://www.postgresql.org/message-id/CADzfLwWAiimwngq38x8Ji26AtYEmSqnFD7aa4kVGF4ObDswHMA%40mail.gmail.com), [#11 (Mihail Nikalayeu, 2026-10-17)](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com)

**Question:** On a large table, how far does xmin advance, and how do build time and validation work for unique builds change with the default settings?

Counter-check: held. Message #1 reports no significant _bt_load slowdown but gives no numbers, and the feature is on by default at 100ms. The build-time cost stands here; the question about xmin gain belongs to demand-1.

### 2. Gain unmeasured while the validation scan still pins xmin

fixable · level 1b, rubric item 2 · desirability, maintenance burden ("who maintains this in 10 years"), new GUC, ABI / extension API · `demand-1`

An operator runs REINDEX CONCURRENTLY on a large, busy table, expecting vacuum to keep up. The first scan now releases xmin, but the tuplesort phase and the full validation scan still hold one snapshot each. The operator sees the same horizon holdback for a large part of the run and the same bloat.

The demand for xmin advance during CIC is real: the project committed it once and reverted it only for safety [P1], and [P2] tried earlier. This set fixes only part of the run, and message #1 leaves the second phase to the separate STIR entry. The material has no numbers on how much of a long build's xmin holdback goes away. It does add two GUCs, an AM flag, a new C function and a reworked unique-build path, and a committer would likely ask whether that gain justifies that cost before the second phase lands.

**Precedents.**
- [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (Reverted, 2022-05-31); thread: [#1](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org), [#2](https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql)
- [avoid bloat from CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1047/) (Returned with feedback, 2017-03-07); thread: [link](https://www.postgresql.org/message-id/flat/20170228052302.r7nvt4sbnsgu6job@alvherre.pgsql)

Thread: [#1 (Mihail Nikalayeu, 2026-01-15)](https://www.postgresql.org/message-id/CADzfLwWAiimwngq38x8Ji26AtYEmSqnFD7aa4kVGF4ObDswHMA%40mail.gmail.com), [#11 (Mihail Nikalayeu, 2026-10-17)](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com)

**Question:** On a large table under write load, how much shorter is the xmin holdback across a whole CIC or REINDEX CONCURRENTLY run with this set, without STIR?

Counter-check: held. 0007 rotates snapshots in the post-scan build loops, so only tuplesort's internal sorts and merges and the validation scan still pin xmin. The material gives no number for how much of a run's holdback goes away, so the request for a measurement stands.

### 3. Safety argument for missed tuples is informal, as in the reverted PG14 feature

fixable · level 2, rubric item 7 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A busy table gets HOT updates and pruning while CREATE INDEX CONCURRENTLY rotates its snapshot. In a case the stress test does not cover, validation skips a live tuple, and the new index silently misses it. Users see wrong query results or amcheck errors months later, as in the 2022 case.

[P1] reverted a feature whose goal was the same: let xmin advance during CIC. The revert reason was that pruning removed tuples the build still relied on, and validation did not repair this. The proposal's safety claim is one informal sentence in message #1, a README.HOT edit, and a stress test that the author says in message #11 was mostly AI-written and not yet reviewed closely. v8 also adds a new path where unique builds leave entries out for validation to insert.

**Precedents.**
- [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (Reverted, 2022-05-31); thread: [#1](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org), [#2](https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql)

Thread: [#1 (Mihail Nikalayeu, 2026-01-15)](https://www.postgresql.org/message-id/CADzfLwWAiimwngq38x8Ji26AtYEmSqnFD7aa4kVGF4ObDswHMA%40mail.gmail.com), [#11 (Mihail Nikalayeu, 2026-10-17)](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com)

**Question:** What written invariant shows that validate_index still inserts every live tuple when the first scan and the unique-build load use rotating snapshots and pruning runs between rotations?

Counter-check: held. [P1] let vacuum ignore the backend's xmin, while this build always holds a registered snapshot, so [P1] shows the stakes only. The material still has no written invariant for the unique-build path that leaves entries for validation, and the tests are not yet reviewed.

### 4. New TOAST and snapshot rules for every index AM, enforced by nothing

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch changes the GIN accumulator, a BRIN opclass, or a GiST buffered-build path so that it keeps a datum across pages without detoasting it. Tests pass because a reset seldom lands between the copy and the use while vacuum removes that TOAST value. Months later a user's CREATE INDEX CONCURRENTLY fails or reads freed data.

0002-0007 add contracts that every build path must keep: detoast retained values, hold no extra snapshot, call index_build_rotate_snapshot() only after the scan is closed, and follow the parallel reset protocol. 0003 and 0004 show these rules are easy to miss, since GIN and BRIN both broke them. Only opt-in for out-of-core AMs answers the concern in message #10. In-core AMs still carry the rules, and a missed rule surfaces only as a rare race.

Thread: [#10 (Matthias van de Meent, 2026-08-03)](https://www.postgresql.org/message-id/CAEze2WhwwAB_DVAWfunO%3D_1QZu-J%2BThbpVbvdoOyycswQETFfw%40mail.gmail.com), [#11 (Mihail Nikalayeu, 2026-10-17)](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com)

**Question:** Which assertion or test fails right away when a build path keeps an undetoasted datum or an extra snapshot across a reset?

Counter-check: held. Interval 0 and the 0001 TOAST-with-vacuum test give some detection, so "enforced by nothing" overstates it. The material shows no assertion that fails when a build path keeps a TOAST pointer or an extra snapshot, and the author has not yet reviewed the tests.

### 5. Duplicate-check budget exposed as a GUC with an unmeasured default

fixable · level 2, rubric item 5 · new GUC, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

An operator builds a unique index concurrently on a table where the key column is updated often. The fetch budget runs out, many equal-key entries move to the validation scan, and the build takes longer. The only lever is index_build_duplicate_check_scale_factor, whose default 0.05 the author calls not measured, and whose meaning depends on build internals.

The author says in message #11 that 0.05 is "not measure". The setting describes internal mechanics, separate sort and load budgets and a floor of 1000 fetches, which a later design change such as the ReadStream batching suggested in message #11 would make obsolete. The documentation hunk places it next to post_auth_delay, so it may be meant as a developer option, but the proposal does not say so. Once a setting ships, removing or redefining it breaks configuration files.

Thread: [#11 (Mihail Nikalayeu, 2026-10-17)](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com), [#12 (Mihail Nikalayeu, 2026-10-22)](https://www.postgresql.org/message-id/CADzfLwV%2Bng11CQPju9HxJvK_rLAZ0_Zozno_%2B4Td26ZL1V_%3DrA%40mail.gmail.com)

**Question:** Is index_build_duplicate_check_scale_factor meant for users or only for developers, and what measurement supports 0.05 as the default?

Counter-check: held. The hunk sits next to post_auth_delay, which suggests a developer option and weakens the configuration-file concern. The author still calls 0.05 not measured, and the proposal does not state the intended audience.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | IndexAmRoutine gains amcanresetsnapshot, and AMs get index_build_rotate_snapshot() plus new rules for retained TOAST values and snapshots. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Adds concurrent_index_reset_snapshot_interval and index_build_duplicate_check_scale_factor. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The series changes snapmgr.c, heapam, tuplesort, nbtsort.c and every built-in index AM, and adds rules that each build path must keep. |
| performance on the hot path | **yes** | Unique builds compare keys per tuple in _bt_load(), and 0005 touches the shared parallel table scan code. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What written invariant shows that validation still inserts every live tuple when the first scan and the unique-build load use rotating snapshots and pruning runs between resets?
2. On a large table under write load, how much shorter is the xmin holdback across a whole CREATE INDEX CONCURRENTLY run with this set, without STIR?
3. How much do build time and validation work change for a large unique build with the default settings, compared with master?
4. Which assertion or test fails right away when a build path keeps a TOAST pointer or an extra snapshot across a reset?
5. What measurements support the defaults of 100ms for the reset interval and 0.05 for index_build_duplicate_check_scale_factor?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera, who committed and reverted the PG14 feature [P1], takes part in the thread, so far only on the stress test. The owner needs to know snapshot management, heap pruning and nbtree builds.
- **Liability:** The owner signs for the completeness of every index built concurrently, with no time limit, since a missed tuple shows up only as wrong results or amcheck errors. The owner also signs for two GUCs and the AM contract.
- **Author follow-up:** Strong sign: the author rebased from January to October 2026 and reworked v6 into v8 after review. The author says the tests and 0007 are not yet fully reviewed.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Operators of large, busy databases gain, and the project already accepted the goal once [P1]. The seven-patch series changes snapshot, heap, sort and index AM code, so review is costly; it needs a rebase and the measurements first.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Snapshot resets silently depend on default_transaction_isolation. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `maintenance-fit-3` Silent-corruption risk spread over many subsystems, with thin review. Refuted (already handled): The series is already staged: 0002 enables only serial non-unique builds, and later patches add GIN, BRIN, parallel and unique. The corruption risk duplicates prior-attempts-1, and ownership is a committability matter, with the committer of the original feature active in the thread.

### Review angles

- prior-attempts: 1 objection(s). Only one objection from this angle. Matthias's design concerns from message #10 got answers in v8, which has no reply yet. P9 is a different idea, and its expression-function concern is handled by keeping an active snapshot.
- standards-compat: 2 objection(s). From the standards angle the surface is small. There is no new SQL syntax, no catalog object and no pg_dump impact. The amcanresetsnapshot field follows the pattern of amcanbuildparallel and keeps old behaviour for AMs that do not set it.
- replication-ondisk: 0 objection(s). The proposal adds no new persistent or cross-node state: no on-disk format, WAL record, catalog, or protocol change, so pg_upgrade and standbys are unaffected. The one ABI change, the new amcanresetsnapshot field in IndexAmRoutine, defaults to the old behavior for out-of-core AMs in a major release. Whether the built index is complete rests on the validation scan, which belongs to the safety angle. The nbtree VACUUM bookkeeping invariant does not apply because no VACUUM-time code changes.
- security: 0 objection(s). No security objection found. Only the table owner can run the build, and the patch adds no privilege, role, or view, and does not let a user see or do more than before. Expression and opclass functions still run as the table owner. A user can set the GUCs in their own session, but this only changes the cost of that user's own build, and -1 keeps today's xmin pinning. Unique-check correctness belongs to other angles.
- maintenance-fit: 3 objection(s). The nbtree invariant about VACUUM-time bookkeeping does not apply, because this proposal changes index builds and not VACUUM.
- already-exists: 0 objection(s). Postgres has no existing feature, setting or extension that lets xmin advance during a concurrent index build; the PG14 mechanism was reverted [P1]. Workarounds such as smaller partitions or plain CREATE INDEX either do not give the same result or block writes, and snapshot handling inside the build cannot be done from an extension.
- demand: 1 objection(s). Demand for the goal is established: a PG14 feature with the same aim was committed and reverted only for safety, and a 2017 attempt had the same aim. The open point is how much of that demand this partial fix serves.

### Precedents retrieved

- P1: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - used in the review
- P2: [avoid bloat from CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1047/) (cf, Returned with feedback, 2017-03-07) - used in the review; reason summarised from its thread
- P3: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant; reason summarised from its thread
- P4: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant; reason summarised from its thread
- P5: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant; reason summarised from its thread
- P6: [Avoid duplicate XIDs at recovery when building initial snapshot - take 2](https://commitfest.postgresql.org/patch/1866/) (cf, Returned with feedback, 2018-12-03) - judged not relevant; reason summarised from its thread
- P7: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P8: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P9: [remove spurious CREATE INDEX CONCURRENTLY wait](https://commitfest.postgresql.org/patch/2720/) (cf, Withdrawn, 2020-09-01) - used in the review; reason summarised from its thread
- P10: [Avoiding deadlock errors in CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/2141/) (cf, Returned with feedback, 2019-11-25) - judged not relevant; reason summarised from its thread
- P11: [Revert "ICU: do not convert locale 'C' to 'en-US-u-va-posix'."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=455f948b0d03a556533a7e4a1a8abf45f0eb202e) (revert, Reverted, 2023-05-09) - judged not relevant
- P12: [history of vacuum / checkpoint runs](https://commitfest.postgresql.org/patch/5462/) (cf, Withdrawn, 2025-11-14) - judged not relevant; reason summarised from its thread
- P13: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P14: [Revert tests for drop index concurrently.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5ad72cee7ed3ac0016c716f0408811274d822f14) (revert, Reverted, 2012-10-18) - judged not relevant
- P15: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant
- P16: [Revert "Tidy up locale thread safety in ECPG library."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3c8e463b0d885e0d976f6a13a1fb78187b25c86f) (revert, Reverted, 2025-03-28) - judged not relevant
- P17: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P18: [Revert "Fix partition pruning setup during DETACH CONCURRENTLY"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b0ea16528cda1f3a993bbfeb2b93eed9762af4e6) (revert, Reverted, 2024-06-24) - judged not relevant
- P19: [create index failed. PARALLEL UNSAFE function does not prevent parallel index build](https://commitfest.postgresql.org/patch/4860/) (cf, Rejected, 2024-03-13) - judged not relevant
- P20: [Trigger failsafe when VACUUM is run in single user mode](https://commitfest.postgresql.org/patch/3467/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P21: [Parallel tests publication and subscription might fail due to concurrent tuple update](https://commitfest.postgresql.org/patch/5459/) (cf, Rejected, 2025-03-27) - judged not relevant
- P22: [REINDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1060/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P23: [Revert "Fix search_path to a safe value during maintenance operations."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2fcc7ee7affe2fb708d7b2727e93981216b5d2f8) (revert, Reverted, 2023-06-10) - judged not relevant
- P24: [Avoid smgrimmedsync() during index build and add unbuffered IO API](https://commitfest.postgresql.org/patch/3508/) (cf, Returned with feedback, 2022-08-02) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0004-Support-snapshot-resets-in-concurrent-builds-of-u.patch not read
- attachment limit (8) reached; v1-0002-Reset-snapshots-periodically-in-non-unique.patch not read
- attachment limit (8) reached; v1-0003-Support-snapshot-resets-in-parallel-concurrent-in.patch not read
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (maintenance-fit-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 172 s; cost reported by the CLI: $2.89
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 87 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:51:01.593Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
