# Design pre-review: Fix exclude generated columns in COPY TO with RLS

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7589](https://commitfest.postgresql.org/patch/7589/) (Needs review); authors: Ayoub Kazar (kaz)
- Thread: [Fix exclude generated columns in COPY TO with RLS](https://www.postgresql.org/message-id/flat/1915ae77-392a-4bdf-b3e2-157d63af320c%40data-bene.io), started 2026-11-04
- Messages read: 3 of 3
- Patch prose read: v2-0001-Exclude-generated-columns-in-COPY-TO-with-row-level-security.patch, 0001-Exclude-generated-columns-in-COPY-TO-with-row-level-security.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal or the design. The patch makes COPY TO under RLS follow the documented rule and the non-RLS path. The one surviving objection, standards-compat-1, asks which branches the fix is for, because a back-patch would change the output columns in a minor release. That is a fixable point for a committer to decide, and it does not stop review of the idea.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. This review could not check that policy quals on generated columns still apply, that virtual generated columns are excluded, or that partitioned and inherited tables behave the same.

## The proposal, restated

**Problem.** COPY TO without a column list exports generated columns when RLS applies to the user, which contradicts the COPY documentation. The output then cannot be loaded back with COPY FROM, which rejects generated columns. Under RLS, a user with SELECT only on the non-generated columns also gets "permission denied for table".

**Proposed design.** Under RLS, DoCopy() rewrites COPY TO into an internal SELECT that used an A_Star target list, so all non-dropped columns were included. The patch builds that target list from the attnums that CopyGetAttnums() already computes, so the RLS path copies the same columns as the non-RLS path.

**User-visible surface.**
- COPY TO without a column list on a table with RLS active no longer outputs generated columns.
- COPY TO under RLS no longer fails with "permission denied" when the user lacks SELECT only on generated columns.

**What the author claims.**
- The current output violates the documented rule that generated columns are not copied without a column list.
- The fix restores COPY TO / COPY FROM round-tripping under RLS.
- Users who want generated columns in the output can use COPY (SELECT * FROM t) TO.
- The patch makes RLS behavior match non-RLS behavior.

**Thread state.** Jim Jones tested v1, confirmed the fix, and asked for a regression test for the column-privilege case, which v2 says it adds. Jim's question whether COPY TO should ever include generated columns got only the round-trip answer from the author.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-04; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Backpatch would change COPY TO output columns in a minor release

fixable · level 1b, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

An export job runs COPY t TO under an RLS-restricted role and its consumer expects the generated column in a fixed column position. After a minor update the column disappears and every later field shifts by one. The consumer then loads wrong values or rejects the rows, and the error shows up downstream long after the upgrade.

The CommitFest entry is tagged Bugfix, and its target version is None. The proposal does not say whether the fix goes to back branches or only to master. Since generated columns arrived, the RLS path has emitted them, so some exports may depend on today's column layout. Matching the documentation is a good reason to change master, but changing output shape in a minor release is a separate decision that the thread has not discussed.

Thread: [#1 (Ayoub Kazar, 2026-11-04)](https://www.postgresql.org/message-id/1915ae77-392a-4bdf-b3e2-157d63af320c%40data-bene.io), [#3 (Ayoub Kazar, 2026-11-05)](https://www.postgresql.org/message-id/1936f1d8-e530-4d03-8af7-c65552153cb3%40data-bene.io)

**Question:** Is this fix meant for master only, and if it is meant for back branches, how would users who rely on the current RLS output learn about the change?

Counter-check: held. The entry is tagged Bugfix with target version None, and no message says which branches the fix is for. The export scenario is concrete, and the thread has not weighed a minor-release change of output columns.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Backpatch would change COPY TO output columns in a minor release |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch changes which columns the RLS rewrite of COPY TO reads, so the SELECT privilege check no longer covers generated columns. |
| backpatchability | **yes** | The entry is tagged Bugfix with target version None, and a back-patch would change COPY TO output columns under RLS. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the fix meant for master only, and if for back branches, how would users who rely on the current RLS output learn about the change?
2. Does an RLS policy whose expression references a generated column still apply correctly when the internal SELECT target list omits that column?
3. How does the RLS path treat virtual generated columns after the patch, and does a regression test cover them?
4. Does the new target list behave the same for partitioned tables and inheritance parents on the RLS path?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the COPY code and the RLS rewrite in DoCopy() would have to own it. The visible thread has the author and one reviewer.
- **Liability:** The owner signs for a change in COPY TO output columns under RLS and for the back-branch decision. The code change is small and removes a difference between the RLS and non-RLS paths.
- **Author follow-up:** The author posted v2 with the requested regression test one day after the review in message #2.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug affects only COPY TO without a column list on a table with RLS and a generated column. The patch is about 6 KiB in copy.c plus rowsecurity tests, and it reuses the CopyGetAttnums() list.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Back-patching would silently shift COPY TO output columns under RLS. Refuted (not specific): Duplicate of standards-compat-1, which states the same back-patch question and the same shifted-column scenario.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the retrieved precedents contain no earlier attempt at this idea, and the thread does not point to any earlier discussion. The patch fixes a mismatch with documented COPY behaviour, so there is no recorded rejection reason to carry over from the prior-attempts angle.
- standards-compat: 1 objection(s). The change makes the RLS path match the documented non-RLS behavior, and the stricter column-privilege handling follows the same rule. Only the backpatch question remains from this angle.
- replication-ondisk: 0 objection(s). The patch only changes the target list of the internal SELECT that DoCopy() builds for COPY TO under RLS. It adds no on-disk format, WAL record, catalog column, wire protocol message or extension-facing API, so this angle has nothing to object to. Back-branch output changes are a behaviour question for other angles.
- security: 0 objection(s). No security objection. The patch narrows what the RLS rewrite of COPY TO reads to the columns the non-RLS path already checks, so a column-privileged user gets only the columns they are granted, as in message #2. Two points depend on code not shown: that policy quals on generated columns still apply, and that virtual generated columns are excluded.
- maintenance-fit: 1 objection(s). From the maintenance angle the design is small and lowers long-run cost, because the RLS path now uses the same CopyGetAttnums() list as the non-RLS path, so the two paths cannot drift apart. It adds no hot-path cost, no API and no new dependency.
- already-exists: 0 objection(s). This is a bug fix that makes COPY TO under RLS follow the documented rule and the non-RLS path, so no existing feature or extension gives the same result. An explicit column list is a workaround for affected users, but it does not fix the documented default, and the fix adds no new mechanism.
- demand: 0 objection(s). This is a bug fix: COPY TO under RLS breaks the documented rule on generated columns and the COPY TO / COPY FROM round trip, and it raises a false permission error under column privileges. Any RLS user who runs COPY TO on a table with a generated column meets it, so demand needs no separate evidence. Whether the fix goes to back branches is a design question for another angle.

### Precedents retrieved

- P1: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P2: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant; reason summarised from its thread
- P3: [foreign key on virtual generated column](https://commitfest.postgresql.org/patch/5782/) (cf, Withdrawn, 2026-09-23) - judged not relevant; reason summarised from its thread
- P4: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P5: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - judged not relevant; reason summarised from its thread
- P6: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P7: [Infinite Autovacuum loop caused by failing virtual generated column expression](https://commitfest.postgresql.org/patch/7100/) (cf, Withdrawn, 2026-08-19) - judged not relevant; reason summarised from its thread
- P8: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant; reason summarised from its thread
- P9: [Document the limitations of row-level security](https://commitfest.postgresql.org/patch/2027/) (cf, Returned with feedback, 2019-04-04) - judged not relevant
- P10: [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (cf, Rejected, 2019-03-19) - judged not relevant
- P11: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant
- P12: [minor error message enhance: print RLS policy name when only one permissive policy exists](https://commitfest.postgresql.org/patch/6180/) (cf, Withdrawn, 2026-09-23) - judged not relevant
- P13: [Multi-tenancy with RLS](https://commitfest.postgresql.org/patch/722/) (cf, Rejected, 2017-01-31) - judged not relevant
- P14: [More isolated multi-tenancy with RLS](https://commitfest.postgresql.org/patch/357/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P15: [Add generate_series(date,date,\[step\])](https://commitfest.postgresql.org/patch/492/) (cf, Returned with feedback, 2016-03-17) - judged not relevant
- P16: [START/END line number for COPY FROM](https://commitfest.postgresql.org/patch/1940/) (cf, Rejected, 2019-01-30) - judged not relevant
- P17: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1148/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P18: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1568/) (cf, Returned with feedback, 2018-03-01) - judged not relevant
- P19: [Revert "Refactor CopyAttributeOut{CSV,Text}() to use a callback in COPY TO"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aa8324b81fa4979ffdc6ccf81d560eac9446948) (revert, Reverted, 2024-02-06) - judged not relevant
- P20: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P21: [Patch to clean Query after rewrite-and-analyze](https://commitfest.postgresql.org/patch/2229/) (cf, Withdrawn, 2019-08-05) - judged not relevant
- P22: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P23: [Bug in query rewriter - hasModifyingCTE not getting set](https://commitfest.postgresql.org/patch/3134/) (cf, Returned with feedback, 2021-09-08) - judged not relevant
- P24: [BUG #19703: information_schema.usage_privileges omits a sequence owner's implicit USAGE privilege](https://commitfest.postgresql.org/patch/7540/) (cf, Withdrawn, 2026-10-28) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 105 s; cost reported by the CLI: $1.10
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 53 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:03:06.267Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
