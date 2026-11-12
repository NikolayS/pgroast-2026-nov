# Design pre-review: Fix read/write races with pg_control file in case when postgres was compiled with DEXEC_BACKEND option

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6040](https://commitfest.postgresql.org/patch/6040/) (Needs review); authors: Maksim Melnikov (manaldush)
- Thread: [Incorrect checksum in control file with pg_rewind test](https://www.postgresql.org/message-id/flat/f59335a4-83ff-438a-a30e-7cf2200276b6%40postgrespro.ru), started 2025-09-04
- Messages read: 6 of 6
- Patch prose read: v1-0001-Try-to-handle-torn-reads-of-pg_control-in-sub-pos.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the bug or either fix. The race is reproduced in message #3, and both designs reuse existing mechanisms. The one open point is prior-attempts-1: message #5 names another thread with its own fix and CommitFest entry, and nobody has said which patch is the candidate. That is a coordination question to settle before commit, not a flaw in the idea.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The BackendParameters patch from message #4 and messages #4 to #6 were not shown, so its handling of a stale postmaster copy could not be checked. The fix in the thread linked in message #5 was not seen either.

## The proposal, restated

**Problem.** In builds with EXEC_BACKEND, child processes read pg_control from disk without ControlFileLock. A child can read a partly written file while the startup process updates it during WAL replay, and then fails with "incorrect checksum in control file". The failure is rare and was seen in TAP tests on the author's local CI.

**Proposed design.** The v1 patch retries the pg_control read on a CRC failure, as commit 5725e4ebe7a9 did for frontend programs. Korotkov proposed instead that the postmaster pass its copy of the control file to the child through BackendParameters, as fork builds already inherit it. The author accepted that alternative.

**What the author claims.**
- The race affects only EXEC_BACKEND builds, because LocalProcessControlFile cannot take ControlFileLock.
- The failure is not specific to pg_rewind and can happen at any time.
- A dirty test hack plus a pg_update_control_file() loop and pgbench reproduces it on master.
- Korotkov claims passing the control file through BackendParameters stops torn reads and removes a difference between fork and EXEC_BACKEND.
- v1 copies the frontend fix with little change.

**Thread state.** Kovmir confirmed the repro, Korotkov posted an alternative patch, and the author approved it. Lakhin pointed to another thread with its own fix and CommitFest entry, and nobody said how the two relate; CFbot reports the patch needs a rebase.

## 2. Prior attempts

### Fixes for non-atomic read of read of control file on ext4 + ntfs (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4025/](https://commitfest.postgresql.org/patch/4025/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20221123014224.xisi44byq3cf5psi@awork3.anarazel.de](https://www.postgresql.org/message-id/flat/20221123014224.xisi44byq3cf5psi@awork3.anarazel.de)
- **Date:** decided 2024-02-01; started 2022-11-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Thomas Munro committed the frontend retry loop and the missing locks in the SQL functions, so only the backup patches were left in the entry. Robert Haas asked for comments and a clearer commit message on 0002 and a cleaner refactoring of update_controlfile() in 0001. A proposal to put the control file in backup_label was also still under debate. The author did not post a new version, so the entry was returned with feedback.
- **Reason (quote):** "I have changed the status of this to "Waiting on Author" as Robert's comments have not yet been handled." ([message](https://www.postgresql.org/message-id/CALDaNm01LGGbQ0DJATqwtmg4xx62dQTbrRjg%3DfFhObGCJ2EzqA%40mail.gmail.com))
- **What is different this time:** P1 fixed torn pg_control reads in frontend programs with a retry loop, and that part was committed. This proposal targets EXEC_BACKEND child processes, and the thread now prefers passing the postmaster's copy through BackendParameters. P1 was returned only for unhandled review comments on its backup patches, which does not apply here.


## 3. Objections a reviewer is likely to raise

### 1. Overlap with the fix in the thread linked in message #5 is unresolved

fixable · level 1a, rubric item 1 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

Two CommitFest entries carry different fixes for the same EXEC_BACKEND torn read. A committer reviews this entry while the other fix is being reviewed or committed elsewhere. Reviewer time is split, and the second fix to land either conflicts with the first or duplicates its work.

Message #5 reports that the same issue was discussed in another thread, which has its own fix and its own CommitFest entry. No later message says how the two fixes compare or which one should go forward. Message #6 approves Korotkov's BackendParameters patch, but the entry still names only the v1 author and needs a rebase. A committer would likely ask which patch, in which entry, is the candidate.

Thread: [#4 (Alexander Korotkov, 2026-04-21)](https://www.postgresql.org/message-id/CAPpHfdsXkEWUeLUG4zh9q%3DhjpsOCMgsbN_XZh-6JL0z1NaNMqQ%40mail.gmail.com), [#5 (Alexander Lakhin, 2026-04-21)](https://www.postgresql.org/message-id/033028ce-8ef7-421e-92dc-bcd8a421c4c3%40gmail.com), [#6 ("Maksim(dot)Melnikov", 2026-05-12)](https://www.postgresql.org/message-id/4ff7bf45-3a95-471a-8dc2-84a8a3b3924a%40postgrespro.ru)

**Question:** Which of the two fixes, this entry's BackendParameters patch or the one in the thread from message #5, should be committed, and should the other entry be closed?

Counter-check: held. Message #5 names another thread with its own fix and CommitFest entry, and message #6 does not answer it. The entry also does not say whether it tracks v1 or the BackendParameters patch, so a committer cannot tell which patch is the candidate.

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
| backpatchability | **yes** | The entry is tagged Bugfix, and the thread does not say which branches should get the fix. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Two threads carry different fixes for the same race, and the change touches control file handling in xlog.c. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which fix should be committed, the BackendParameters patch here or the one in the thread linked in message #5, and should the other CommitFest entry be closed?
2. Does CommitFest entry 6040 now track v1 or the BackendParameters patch, and can a rebased version be posted?
3. Can a child process need a newer pg_control than the postmaster's copy at the time it starts?
4. Should the fix be back-patched, and to which branches?
5. Do Windows production builds hit this race, or only test builds compiled with -DEXEC_BACKEND?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows postmaster child startup under EXEC_BACKEND and the control file code in xlog.c. Korotkov posted the alternative patch, but the material does not show that he intends to commit it.
- **Liability:** A small, internal change to how EXEC_BACKEND children get pg_control. The owner signs for children starting with the postmaster's copy, and for any back-patched branches.
- **Author follow-up:** The author posted a reproduction two months after the first message and reviewed and accepted the alternative patch. The entry now needs a rebase.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug is reproduced and affects builds that start children with exec, which the demand notes say includes Windows. v1 is 2.3 KB in one file. First decide which of the two entries goes forward, then post a rebased patch.

## Appendix

### Refuted in the counter-check

- `already-exists-1` Second fix for a bug already tracked in another thread. Refuted (not specific): Duplicate of prior-attempts-1. Both rest on messages #4 to #6 and ask which fix and which CommitFest entry should go forward.

### Review angles

- prior-attempts: 1 objection(s). No precedent rejected this idea. P1's committed frontend retry is the model for v1, and P6 is about backup_label and does not apply. The only concern from this angle is the duplicate fix reported in message #5.
- standards-compat: 0 objection(s). The proposal has no user-facing contract: no SQL syntax, GUC, catalog object, pg_dump output or psql change, and no client-visible behaviour beyond removing a spurious FATAL. Both designs (CRC retry, or passing pg_control through BackendParameters) change only internal EXEC_BACKEND startup, and BackendParameters is private to the server binary. Back-patching therefore raises no ABI concern from this angle. The unresolved overlap with the other thread and CommitFest entry is a process question for other angles.
- replication-ondisk: 0 objection(s). Neither patch changes the on-disk format of pg_control, the WAL, the catalogs, the wire protocol or any extension-facing struct. BackendParameters passes only between a postmaster and children of the same binary, and children get the same snapshot fork builds already inherit, so I see no replication or pg_upgrade failure scenario.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes how EXEC_BACKEND child processes get their copy of pg_control, either by retrying the read or by inheriting the postmaster's copy through BackendParameters. It adds no SQL surface, privilege, role or setting, and exposes no new data to any user. A child already reads the whole control file today, so it learns nothing new. The BackendParameters file is created by the postmaster for the server's own children with the server's existing file permissions. No security objection applies.
- maintenance-fit: 0 objection(s). No maintenance or fit objection holds. The BackendParameters patch from message #4 makes EXEC_BACKEND children inherit the same pg_control snapshot that fork children already get, so it removes a divergence and adds no per-query cost, no new API and no new dependency. What remains open is not a design flaw: which patch the entry tracks, how it relates to the fix Lakhin linked in message #5, and the back-patch scope.
- already-exists: 1 objection(s). This is a bugfix. Both designs reuse existing mechanisms: the frontend retry loop and BackendParameters. No extension or existing setting can prevent the torn read, so the only concern from this angle is the duplicate fix in another thread.
- demand: 0 objection(s). This entry fixes a real bug: child processes in EXEC_BACKEND builds fail with "incorrect checksum in control file". Kovmir reproduced it independently (message #3), Lakhin points to a second thread about the same issue (message #5), and the frontend version of the race was already fixed ([P1]), so demand is not in doubt. Windows builds always start children with exec, so real users are exposed, and the cost falls only on the EXEC_BACKEND startup path.

### Precedents retrieved

- P1: [Fixes for non-atomic read of read of control file on ext4 + ntfs](https://commitfest.postgresql.org/patch/4025/) (cf, Returned with feedback, 2024-02-01) - used in the review; reason summarised from its thread
- P2: [Configurable location for extension .control files](https://commitfest.postgresql.org/patch/170/) (cf, Returned with feedback, 2015-07-30) - judged not relevant; reason summarised from its thread
- P3: [Fix PGXS so built control files work with VPATH builds](https://commitfest.postgresql.org/patch/2461/) (cf, Returned with feedback, 2020-07-31) - judged not relevant; reason summarised from its thread
- P4: [add the last checkpoint kind to the control file](https://commitfest.postgresql.org/patch/3493/) (cf, Withdrawn, 2022-02-03) - judged not relevant; reason summarised from its thread
- P5: [Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file](https://commitfest.postgresql.org/patch/3442/) (cf, Withdrawn, 2022-02-27) - judged not relevant; reason summarised from its thread
- P6: [Add recovery to pg_control and remove backup_label](https://commitfest.postgresql.org/patch/4629/) (cf, Withdrawn, 2024-01-28) - used in the review; reason summarised from its thread
- P7: [logical replication access control](https://commitfest.postgresql.org/patch/1033/) (cf, Returned with feedback, 2017-04-06) - judged not relevant; reason summarised from its thread
- P8: [Add a GUC variable that control logical replication](https://commitfest.postgresql.org/patch/2296/) (cf, Returned with feedback, 2019-11-28) - judged not relevant; reason summarised from its thread
- P9: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P10: [tracking last allocated XLOG segment in control data](https://commitfest.postgresql.org/patch/444/) (cf, Rejected, 2016-01-31) - judged not relevant
- P11: [Checksums for slru files](https://commitfest.postgresql.org/patch/1462/) (cf, Returned with feedback, 2018-03-02) - judged not relevant
- P12: [Close backend file handles on exec](https://commitfest.postgresql.org/patch/886/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P13: [Add control-C support for some psql meta-commands](https://commitfest.postgresql.org/patch/3412/) (cf, Withdrawn, 2021-11-15) - judged not relevant
- P14: [Atomic rename feature for Windows](https://commitfest.postgresql.org/patch/3347/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P15: [Atomic pgrename on Windows](https://commitfest.postgresql.org/patch/1386/) (cf, Returned with feedback, 2018-03-06) - judged not relevant
- P16: [Strip Windows newlines from extension script files manually.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cfb3a33746990602c790199adc2debb2c4bbb87) (revert, Partially reverted, 2024-10-28) - judged not relevant
- P17: [Revert addition of pg_terminate_backend() because of race conditions.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=76365960d229dca0ad8454040233e3eaaeb413bb) (revert, Reverted, 2008-04-15) - judged not relevant
- P18: [fix spinlock contention in LogwrtResult](https://commitfest.postgresql.org/patch/2716/) (cf, Withdrawn, 2022-09-23) - judged not relevant
- P19: [Revert "For all ppc compilers, implement pg_atomic_fetch_add_ with inline asm."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e9fae0696d9e3ff70a1438775ad9f786b854a5) (revert, Reverted, 2019-09-15) - judged not relevant
- P20: [Adding linux cachestat in file Readv and Prefetch](https://commitfest.postgresql.org/patch/4779/) (cf, Rejected, 2024-03-14) - judged not relevant
- P21: [Fix lock contention for HASHHDR.mutex](https://commitfest.postgresql.org/patch/442/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P22: [Move PinBuffer and UnpinBuffer to atomics](https://commitfest.postgresql.org/patch/370/) (cf, Returned with feedback, 2015-12-16) - judged not relevant
- P23: [audo-detect and use -moutline-atomics compilation flag for aarch64](https://commitfest.postgresql.org/patch/2637/) (cf, Rejected, 2020-09-29) - judged not relevant
- P24: [A separate table level option to control compression](https://commitfest.postgresql.org/patch/2001/) (cf, Returned with feedback, 2019-04-08) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1a objection (prior-attempts-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 118 s; cost reported by the CLI: $1.49
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 55 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:39:27.682Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
