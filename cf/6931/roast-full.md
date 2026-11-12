# Design pre-review: Report unsupported COPY FROM targets before reading input

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6931](https://commitfest.postgresql.org/patch/6931/) (Needs review); authors: Chao Li (chaol)
- Thread: [Reject unsupported COPY FROM targets before analyzing WHERE clause](https://www.postgresql.org/message-id/flat/84873FFF-4C50-4C85-9A36-62A01B860FF1%40gmail.com), started 2026-06-24
- Messages read: 8 of 8
- Patch prose read: v3-0001-Report-unsupported-COPY-FROM-targets-before-readi.patch, v1-0001-Reject-unsupported-COPY-FROM-targets-before-WHERE.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic. The only one raised, maintenance-fit-1, was refuted because the moved relkind check accepts foreign tables and file_fdw passes with v3. The change moves an existing check to the place COPY TO already uses, and its cost falls only on commands that fail anyway. Confidence is medium because the code was not read and no committer has replied.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the exact position of the moved check among the other BeginCopyFrom() steps is unverified. Its effect on out-of-tree BeginCopyFrom() callers rests on the critic's reading and the reported file_fdw test result.

## The proposal, restated

**Problem.** COPY FROM a sequence, view or other unsupported target only fails with "cannot copy to sequence" after psql has entered COPY input mode and the user has typed data. On master the server also opens the input file or runs the PROGRAM command before it rejects the target. The author frames this as poor user experience, not a correctness bug.

**Proposed design.** v3 moves the relation-kind check from CopyFrom() to BeginCopyFrom(), so it runs before the server reads input, opens a file or starts a program. Parse analysis, permission, column-list and WHERE-clause errors still come first, matching COPY TO, which checks relkind in BeginCopyTo(). v1 also put the check before WHERE-clause analysis, but the author dropped that after review.

**User-visible surface.**
- COPY FROM an unsupported target (sequence, view, matview) fails before entering COPY input mode.
- COPY FROM PROGRAM on an unsupported target no longer runs the program before the error.
- COPY seq FROM '/nonexistent/file' now reports "cannot copy to sequence" instead of "could not open file".
- Invalid COPY options such as (format bogus) are now reported after the relkind error.
- file_fdw gets the earlier check through BeginCopyFrom(), per a reviewer.

**What the author claims.**
- Failing before waiting for input is a better user experience.
- The change keeps the existing order of parse, permission and WHERE-clause errors.
- The change makes COPY FROM consistent with COPY TO.
- The change avoids opening the input file or starting an external program for a command that will fail.

**Thread state.** Horiguchi pushed back on reordering against WHERE-clause errors, and the author narrowed the patch in v2. Liu, solai v and Poondla tested v2/v3 and gave +1; no committer has replied, and the author plans to revisit check ordering for v20.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-06-24; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | BeginCopyFrom() is exported and used by file_fdw; it now rejects unsupported relation kinds itself. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | psql no longer enters COPY input mode for COPY FROM STDIN on a sequence, view or matview. |
| security / privileges | **yes** | COPY FROM PROGRAM on an unsupported target no longer runs the program; permission checks still come first. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the patch add a regression test for the early failure, such as COPY of a sequence FROM a nonexistent file, as a reviewer suggested?
2. Should the commit message say that invalid COPY options such as (format bogus) are now reported after the relkind error?
3. Can an out-of-tree caller of BeginCopyFrom() pass a relation kind that the moved check rejects but that worked before because the caller never called CopyFrom()?
4. Which release does the patch target, given that the CommitFest entry says 19 and the thread tests on 20devel?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the COPY code in src/backend/commands and the BeginCopyFrom() callers (file_fdw, tablesync) would have to own it. None has replied in the visible thread.
- **Liability:** The owner signs for a changed error order in COPY FROM commands that fail and for BeginCopyFrom() now rejecting unsupported relation kinds for all callers. The exposure is small and limited to one file.
- **Author follow-up:** The author revised the patch after review (v1 to v3) and says in message #3 that he plans further work on check ordering for v20.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The v3 patch is 4.8 KB in copyfrom.c, adds no user-visible surface and helps only users who run COPY FROM on an unsupported target. Three reviewers gave +1 over three CommitFests, but no committer has replied.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Write-target check moves into BeginCopyFrom(), which read-only callers also use. Refuted (no failure scenario): The relkind check in CopyFrom() accepts foreign tables and rejects only views, matviews, sequences and other non-table relations. An FDW that passes its own foreign table to BeginCopyFrom() does not hit it, and message #8 reports that file_fdw passes with v3.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the search returned no earlier attempt at moving the COPY FROM relkind check. The thread points to no earlier discussion of this idea; message #2 notes only that COPY TO and COPY FROM have long checked relkind at different stages, with no known reason for the difference.
- standards-compat: 0 objection(s). Within the standards and compatibility angle I find no objection worth sending. The patch adds no syntax, keyword, GUC or catalog object, so nothing new needs pg_dump or psql support. The only user-visible change is which error a failing COPY FROM reports first. That command fails either way, and the new order matches COPY TO (message #8). psql already skips the in-line data when COPY FROM STDIN fails early, so scripts with in-line data end the same way. One point is outside this angle and should go to the API angle. file_fdw and other BeginCopyFrom() callers pass relations that are not valid COPY targets, such as foreign tables without insert support. Whether the moved check can affect those callers cannot be checked without the code.
- replication-ondisk: 0 objection(s). v3 moves an existing relkind check inside copyfrom.c and creates no persistent or cross-node state: no on-disk, WAL, catalog or protocol change, and the BeginCopyFrom() signature is unchanged. Tablesync and file_fdw call BeginCopyFrom() with tables or foreign tables, which pass the check, and a reviewer reports that file_fdw passes. Without the code, I could not check out-of-tree extensions that call BeginCopyFrom() on other relation kinds.
- security: 0 objection(s). The patch adds no privilege or exposure surface. Permission checks in DoCopy() still run before the moved relkind check, and the relkind it reports is already visible in pg_class. With the change, COPY FROM PROGRAM and COPY FROM a file on an unsupported target no longer run the program or open the file, which slightly reduces what the command does before it fails.
- maintenance-fit: 1 objection(s). From the maintenance angle the change is small, touches one file, adds nothing to the per-row path, and copies the placement COPY TO already uses. The only open point is the effect on BeginCopyFrom() as an API used by extensions.
- already-exists: 0 objection(s). Postgres has no option, setting or extension that makes COPY FROM reject a sequence or view before it reads input. The patch moves an existing check to an earlier point and adds no new mechanism, so this angle has nothing to raise. Code-level effects on other BeginCopyFrom() callers could not be checked.
- demand: 0 objection(s). No demand objection applies. The change is a small error-placement fix in one file, and it touches no catalog, syntax or GUC. Its benefit reaches any user who runs COPY FROM on a sequence, view or matview, and it stops a COPY FROM PROGRAM that will fail from running its command (message #6). Four reviewers reproduced the problem and agreed (messages #2, #4-#6). The cost to non-users is close to nothing, so the accumulation test (item 5) does not fail.

### Precedents retrieved

- P1: [Fix bug of COPY (on_error set_null)](https://commitfest.postgresql.org/patch/6774/) (cf, Withdrawn, 2026-05-19) - judged not relevant; reason summarised from its thread
- P2: [Handling of the input data errors in COPY FROM](https://commitfest.postgresql.org/patch/1179/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P3: [Handling of the input data errors in COPY FROM](https://commitfest.postgresql.org/patch/1391/) (cf, Returned with feedback, 2018-03-03) - judged not relevant; reason summarised from its thread
- P4: [Improve handling of OOM errors in libpq making process hangling for COPY and bind (take 2)](https://commitfest.postgresql.org/patch/298/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P5: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P6: [Revert my bad decision of about a year ago to make PortalDefineQuery responsible for copying the query string into the new Portal.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1591fcbec77f6544a0f665758bae912c68e6fa42) (revert, Reverted, 2008-04-02) - judged not relevant
- P7: [Sequence Access Method](https://commitfest.postgresql.org/patch/44/) (cf, Returned with feedback, 2015-07-28) - judged not relevant; reason summarised from its thread
- P8: [Sequence Access Methods](https://commitfest.postgresql.org/patch/466/) (cf, Returned with feedback, 2016-04-06) - judged not relevant; reason summarised from its thread
- P9: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P10: [Fallback table AM for relkinds without storage](https://commitfest.postgresql.org/patch/2986/) (cf, Returned with feedback, 2021-03-14) - judged not relevant; reason summarised from its thread
- P11: [Fix test for pg_upgrade from 10x and some earlier versions.](https://commitfest.postgresql.org/patch/3757/) (cf, Returned with feedback, 2022-07-29) - judged not relevant
- P12: [\[PATCH\] Add UPDATE WHERE OFFSET IN clause](https://commitfest.postgresql.org/patch/3531/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P13: [Move all am-related reloption code into src/backend/access/\[am-name\] and get rid of relopt_kind for custom AM](https://commitfest.postgresql.org/patch/992/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P14: [Revert "Accept relations of any kind in LOCK TABLE".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=eeda7f6338095701cfe1ba3da37070508efe019e) (revert, Reverted, 2020-11-06) - judged not relevant
- P15: [Revert error-throwing wrappers for the printf family of functions.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0c071936e94c6859afb2ec8d2c8dddf7bcdab7ee) (revert, Reverted, 2015-05-19) - judged not relevant
- P16: [Extended protocol COPY FROM STDIN fix](https://commitfest.postgresql.org/patch/3180/) (cf, Returned with feedback, 2021-12-01) - judged not relevant
- P17: [BUG: cannot read big bytea (~600MB)](https://commitfest.postgresql.org/patch/1521/) (cf, Rejected, 2018-03-01) - judged not relevant
- P18: [Reimplement targetlist SRFs using ROWS FROM ()](https://commitfest.postgresql.org/patch/759/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P19: [JIT: Optimize generated functions earlier to lower memory usage](https://commitfest.postgresql.org/patch/2298/) (cf, Withdrawn, 2019-11-12) - judged not relevant
- P20: [get rid of StdRdOptions, use individual binary reloptions representation for each relation kind instead](https://commitfest.postgresql.org/patch/1536/) (cf, Returned with feedback, 2019-09-27) - judged not relevant
- P21: [add the last checkpoint kind to the control file](https://commitfest.postgresql.org/patch/3493/) (cf, Withdrawn, 2022-02-03) - judged not relevant
- P22: [apply_scanjoin_target_to_paths and partitionwise join](https://commitfest.postgresql.org/patch/4929/) (cf, Returned with feedback, 2025-01-13) - judged not relevant
- P23: [Allow VACUUM to have multiple target tables](https://commitfest.postgresql.org/patch/1275/) (cf, Rejected, 2017-09-05) - judged not relevant
- P24: [New argument "latest" for recovery_target parameter](https://commitfest.postgresql.org/patch/2347/) (cf, Returned with feedback, 2020-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 107 s; cost reported by the CLI: $1.11
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 51 text field(s) checked, 4 finding(s), 4 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:09:31.430Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
