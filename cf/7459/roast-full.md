# Design pre-review: plpgsql: detoast a returned record before keeping it in a non-atomic context

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7459](https://commitfest.postgresql.org/patch/7459/) (Needs review); authors: Muzzammil Sarwar (muzzammil242)
- Thread: [\[PATCH\] plpgsql: detoast a returned record before keeping it in a non-atomic context](https://www.postgresql.org/message-id/flat/CABUcsRRuMHXWqgiZ5wy%3D4OPbET0niFQEVY_d8VofeumTxjBJjw%40mail.gmail.com), started 2026-10-15
- Messages read: 1 of 1
- Patch prose read: v1-0001-Detoast-a-returned-record-before-keeping-it-in-a-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived at levels 1a, 1b or 2. Message #1 states a reproducible failure on 14 through master and a fix that applies the detoast rule of 2efc924180f to one more assignment path. Confidence is low because only one message is visible and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody checked that ER_get_flat_size() inlines fields in the right memory context and leaves the record valid for takeover. Whether the CALL p(f()) path reaches the new step is also unchecked.

## The proposal, restated

**Problem.** In a DO block or procedure that can COMMIT or ROLLBACK, r := f() keeps the TOAST pointers of the record returned by a PL/pgSQL function. After a ROLLBACK, or after a COMMIT once the row is deleted and vacuumed, reading r fails with "missing chunk number 0 for toast value". CALL p(f()) fails the same way after the procedure's own ROLLBACK.

**Proposed design.** exec_move_row_from_datum() still takes over a read/write expanded record of the right type, but in a non-atomic context it first inlines the record's out-of-line fields in place. It does this when ER_FLAG_HAVE_EXTERNAL is set, by computing the flat size, which detoasts each external field and clears the flag. Atomic contexts and records with no out-of-line fields are unchanged.

**What the author claims.**
- The failure reproduces on master and on 14 through 19, in DO blocks and procedures.
- The fix closes an oversight in 2efc924180f, which detoasts other assignment paths in a non-atomic context.
- A loop of 100,000 such assignments leaves the backend's memory contexts unchanged, and inlining in place is cheaper than copying the record.
- check-world (without TAP) passes on master, and PL/pgSQL tests pass under Valgrind; the patch cherry-picks cleanly to 14 through 19.
- It should be back-patched to 14, as the similar fix f21fadafaf0 was back-patched to 11.

**Thread state.** Only the first message is visible. The CommitFest entry is in Needs review, with no reviewer named.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-15; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | **yes** | The author asks for a back-patch to 14 through 19 and tags the entry Backport. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds one more detoast step in pl_exec.c that future takeover paths must also follow. |
| performance on the hot path | **yes** | Non-atomic r := f() now detoasts each out-of-line field of the returned record on assignment. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the new step in exec_move_row_from_datum() also fix CALL p(f()), and does a test cover that case?
2. Do other paths that take over an expanded object in a non-atomic context, such as expanded arrays or OUT parameters, have the same gap?
3. What is the time cost when a non-atomic block assigns a record with large out-of-line fields many times, compared with unpatched master?
4. Do the TAP tests pass on master and on each back branch with the patch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows PL/pgSQL's expanded-record handling and the non-atomic detoast rules of 2efc924180f would have to own it. None has spoken in the visible thread.
- **Liability:** The owner signs for a behaviour change in six branches, 14 through master: correctness of in-place detoast of a taken-over record, and any memory or timing change in minor releases.
- **Author follow-up:** No sign either way; only message #1 is visible. The author tested each back branch and ran Valgrind, which suggests care.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 5 KiB, touches pl_exec.c and one isolation test, and fixes a reproducible "missing chunk number 0" error in non-atomic blocks. The entry has no reviewer, and a back-patch needs a committer who knows PL/pgSQL.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found in the provided list, and the thread names no rejected attempt at this idea. Message #1 cites earlier fixes in the same area (2efc924180f, f21fadafaf0) and bug #17669 as related accepted work. The patch presents itself as closing an oversight in that work, so no earlier rejection reason applies.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch adds no syntax, keyword, GUC, catalog object, dump content or psql change. The only behaviour change is that r := f() and CALL p(f()) in a non-atomic context no longer fail with "missing chunk number 0", which is the same detoast-on-assignment rule that 2efc924180f already applies to every other assignment path. Existing scripts that work today keep working after a minor upgrade, so the back-patch does not alter a user-facing contract. I could not check, without the code, whether the extra detoast changes memory or timing for large records. That belongs to the performance angle.
- replication-ondisk: 0 objection(s). The patch touches only in-memory PL/pgSQL handling in pl_exec.c, plus an isolation test. It changes no on-disk format, WAL record, catalog, wire protocol or exported struct, so pg_upgrade, standbys and back-patch ABI are unaffected.
- security: 0 objection(s). The patch only changes when Postgres detoasts a record that a PL/pgSQL function has already returned to the caller, so no one can read data they could not read before. It adds no privilege, role, GUC or SQL surface. The extra memory use matches what the other non-atomic assignment paths have done since 2efc924180f.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to reject. The patch is a small fix in pl_exec.c that follows the rule commit 2efc924180f already set for other assignment paths. Its cost applies only to non-atomic blocks that assign a record with out-of-line fields, and those blocks already pay the same detoast cost on every other assignment path. It adds no API, GUC, catalog or format change, so a back-patch commits the project to nothing new. Two points cannot be checked from the material: whether CALL p(f()) goes through the same code path, and whether other takeover paths have the same gap. A committer who knows PL/pgSQL should confirm both before back-patching. No such committer has engaged in the visible thread yet.
- already-exists: 0 objection(s). This is a bug fix that brings exec_move_row_from_datum() into line with the existing detoast rule for non-atomic contexts. Postgres has no existing feature, extension or hook that gives the same guarantee. Users can copy the field into a local variable, but that is a workaround for a crash, so no already-exists objection applies.
- demand: 0 objection(s). This is a bug fix, so demand is not in question. Message #1 shows a reproducible read failure ("missing chunk number 0") on 14 through master, reached through ordinary DO blocks and procedures, and links it to bug #17669, where a non-atomic procedure was the suggested workaround and now fails the same way. The cost falls only on non-atomic assignments of records that hold out-of-line fields.

### Precedents retrieved

- P1: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P2: [iteration over record in plpgsql](https://commitfest.postgresql.org/patch/124/) (cf, Returned with feedback, 2015-07-07) - judged not relevant; reason summarised from its thread
- P3: [Expand applicability of aggregate's sortop optimization](https://commitfest.postgresql.org/patch/4980/) (cf, Returned with feedback, 2025-07-08) - judged not relevant; reason summarised from its thread
- P4: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P5: [Return ERROR on COMMIT instead of silently ROLLBACK](https://commitfest.postgresql.org/patch/2522/) (cf, Returned with feedback, 2021-03-25) - judged not relevant; reason summarised from its thread
- P6: [Statement-level rollback](https://commitfest.postgresql.org/patch/1050/) (cf, Returned with feedback, 2017-10-01) - judged not relevant; reason summarised from its thread
- P7: [Expand the use of check_canonical_path() for more GUCs](https://commitfest.postgresql.org/patch/2564/) (cf, Returned with feedback, 2020-07-10) - judged not relevant; reason summarised from its thread
- P8: [Statement-level rollback](https://commitfest.postgresql.org/patch/1358/) (cf, Returned with feedback, 2018-03-02) - judged not relevant; reason summarised from its thread
- P9: [Dynamic result sets from procedures](https://commitfest.postgresql.org/patch/1361/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P10: [Revert "Close stdin where it's not needed in TestLib.pm procedures"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca266a069a20c32a8f0a1df982a5a67d9483bcb3) (revert, Reverted, 2019-11-27) - judged not relevant
- P11: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P12: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5974/) (cf, Withdrawn, 2025-08-19) - judged not relevant; reason summarised from its thread
- P13: [Fixes for non-atomic read of read of control file on ext4 + ntfs](https://commitfest.postgresql.org/patch/4025/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P14: [Revert the logic for expanding AND/OR conditions in pred_test() to what it was in 7.4, and add some comments explaining why it has to be this way.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4e89bae704f79af8b2b6ac3d43ea963c22df1947) (revert, Reverted, 2005-03-01) - judged not relevant
- P15: [Support for 8-byte TOAST values (aka the TOAST infinite loop problem)](https://commitfest.postgresql.org/patch/5830/) (cf, Returned with feedback, 2026-02-10) - judged not relevant
- P16: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5975/) (cf, Withdrawn, 2025-12-31) - judged not relevant
- P17: [Optimize Numeric using fixed buffer reducing pallocs for small numbers](https://commitfest.postgresql.org/patch/4196/) (cf, Returned with feedback, 2023-09-13) - judged not relevant
- P18: [64-bit TOAST value ID](https://commitfest.postgresql.org/patch/4296/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P19: [new plpgsql.extra_errors check - strict_expr_check](https://commitfest.postgresql.org/patch/5044/) (cf, Withdrawn, 2026-03-24) - judged not relevant
- P20: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P21: [Revert function to get memory context stats for processes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb844b9f06568aa29303122ac0bb7b4c3cce2542) (revert, Reverted, 2025-05-23) - judged not relevant
- P22: [Statement-level rollback](https://commitfest.postgresql.org/patch/1910/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P23: [Various small unrelated doc patches: plpgsql, schemas, permissions, oidvector](https://commitfest.postgresql.org/patch/4581/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P24: [Revert doc change so we mention moving past the last row of a cursor.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf4bf092e581df1cfa830fd6bc02e036b05dbb5d) (revert, Reverted, 2003-02-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 59 s; cost reported by the CLI: $0.72
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:57:05.191Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
