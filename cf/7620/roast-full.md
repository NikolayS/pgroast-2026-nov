# Design pre-review: Fix interval input at minimum signed field values

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7620](https://commitfest.postgresql.org/patch/7620/) (Needs review); authors: Tinh Tran (tranvantinhnbk.gmail.com)
- Thread: [\[PATCH\] Fix interval input at minimum signed field values (BUG #19746)](https://www.postgresql.org/message-id/flat/CAEPgAbE%2BV7Wgx-1BYap276v_Ohc7Zk2HV3-U8Wp520Jz84NRmg%40mail.gmail.com), started 2026-10-20
- Messages read: 2 of 2
- Patch prose read: v2-0001-fix-interval-input-at-minimum-signed-values.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is a reported bug fix (BUG #19746) and nothing at levels 1a or 1b is in question. The one surviving objection, standards-compat-1, is fixable: message #2 lists four inputs that master accepts and v2 rejects when the minimum field comes last. A fix that may go to back branches must not reject input accepted before, so a revised version is needed before commit.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the cause of the field-order regression rests on message #2. Whether other field combinations or input styles regress cannot be judged without the code.

## The proposal, restated

**Problem.** Interval output can contain INT64_MIN microseconds, or minimum day and month values in postgres_verbose style with AGO, but interval input rejects that text. So a value that prints cannot be read back, and the bug report's pg_dump example restores no rows. Anyone who dumps and restores or round-trips such intervals as text is affected.

**Proposed design.** DecodeInterval in datetime.c applies the effective sign of time fields before checked accumulation, so INT64_MIN microseconds parse. It applies AGO while accumulating numeric fields instead of negating at the end, to handle the minimum day and month in verbose output. The commit message says cancellation between fields near the limits stays possible and out-of-range totals are rejected.

**User-visible surface.**
- Interval text input accepts values at the minimum signed microsecond, day and month boundaries that it rejected before.
- Some inputs that were already invalid now get a different error, such as "invalid input syntax" instead of "field value out of range" for a repeated unit.

**What the author claims.**
- The bug lets an interval with INT64_MIN microseconds print but fail when read back.
- The patch keeps cancellation between fields near the limits.
- New regression tests cover text round trips, boundary values, overflow and AGO output.

**Thread state.** Ishjaap Singh tested v2 and confirmed the reported cases and the pg_dump example are fixed, but found four inputs that work on master and fail with v2 when the minimum field comes last in the string. He asked for tests of both field orders, and no v3 is visible.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-20; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. v2 breaks interval inputs that master accepts when the minimum field comes last

fixable · level 2, rubric item 7 · backpatchability, pg_dump / restore round-trip · `standards-compat-1`

An application or a stored view default uses an interval literal such as '1 week -2147483648 days ago'. Master accepts it. After a minor release with v2 it fails with "interval field value out of range". The user finds out when a query, a restore or a CHECK constraint errors after upgrade.

Message #2 shows four inputs that work on unpatched master and fail with v2. In each, DecodeInterval negates the minimum field first because it reads right to left. A bugfix meant for back branches must not reject input that minor releases accepted before, because back-branch users do not expect behaviour changes. The commit message promises that cancellation is preserved, but v2 keeps it for only one field order, and no v3 is visible.

Thread: [#2 (Ishjaap Singh, 2026-11-04)](https://www.postgresql.org/message-id/CA%2BY2Bmd00%2BGkrJ6-4BR_1wrKUtQo-BoEDzObTZKdhscA5%3DXkbA%40mail.gmail.com)

**Question:** Does a revised version accept all four inputs from message #2, with regression tests for both field orders?

Counter-check: held. Message #2 lists four inputs that return true on unpatched master and fail with v2, and no v3 or author reply is visible. The commit message claims cancellation is preserved, so the material confirms the regression and does not answer it.

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
| pg_dump / restore round-trip | **yes** | The fix lets text dumps of minimum-value intervals restore, but v2 rejects some interval text that master accepts. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | It is a bug fix in DecodeInterval; v2 applies to master and REL_17-19 but not REL_15/16, and changes accepted input. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will a revised version accept the four inputs from message #2 that work on master and fail with v2?
2. How does the revised accumulation make the result independent of the order of fields in the input string?
3. Will the regression tests cover cancellation near the limits with the minimum field both first and last in the string?
4. Which branches is the fix meant for, and is a separate version planned for REL_15 and REL_16?
5. Is the change from "field value out of range" to "invalid input syntax" for a repeated unit intended?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the datetime and interval input code in datetime.c and the back-patch rules for input behaviour.
- **Liability:** The owner signs for interval text input behaviour on every branch that gets the fix, including any input that was accepted before and is rejected after a minor release.
- **Author follow-up:** The author registered the CommitFest entry on 2026-11-10, after message #2 suggested it, but no reply or v3 is visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is small and touches only datetime.c and the interval tests, and it fixes a reported dump and restore failure at boundary values. Message #2 found a field-order regression against master, so v2 needs a revision before further review.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Moving negation per field repeats the one-symptom fix pattern. Refuted (not specific): Duplicate of standards-compat-1, which states the field-order regression without a precedent. [P2] concerned a fix to one EncodeInterval output path and must not be used as support here.
- `standards-compat-2` Branch scope of the round-trip fix is not stated. Refuted (no failure scenario): The scenario describes the existing bug on REL_15/16, which the patch does not cause or worsen. The committer usually decides the back-patch range and adapts the patch per branch at commit time, so this is not a design flaw.
- `maintenance-fit-1` Sign handling during accumulation makes overflow depend on field order. Refuted (not specific): Duplicate of standards-compat-1. The failure scenario and the evidence from message #2 are the same, and the maintenance point depends only on the missing tests for the second field order.

### Review angles

- prior-attempts: 1 objection(s). One objection, fixable. [P5] concerns abs() for interval and was skipped because only keywords overlap.
- standards-compat: 2 objection(s). The changed error message for repeated units was judged acceptable in message #2, so it is not raised here. No precedent's recorded reason applies to an input-side fix.
- replication-ondisk: 0 objection(s). The patch changes only the text input path in DecodeInterval. The on-disk and binary interval format, WAL, catalogs, the wire protocol and the ABI stay the same. Dump and restore of boundary values gets better. The field-order regression in message #2 affects parsing semantics and belongs to other angles.
- security: 0 objection(s). The proposal changes how interval text input parses minimum signed field values and has no privilege or exposure aspect. It adds no new privilege, function, view or setting, and it does not change who may call interval input. The changed error message reveals nothing about other users' data, and parsing a short string adds no unbounded memory, disk, lock or worker cost.
- maintenance-fit: 1 objection(s). The fix stays inside datetime.c and adds no cost to hot paths. Making text output read back as input is a core invariant and belongs in core.
- already-exists: 0 objection(s). This is a bug fix for interval text input that rejects values interval output produces, and no existing feature, option or extension lets such a value be read back. Binary send/recv avoids the problem but does not help text dumps or the pg_dump example, so nothing from this angle applies.
- demand: 0 objection(s). No demand objection applies. The proposal fixes a reported bug (BUG #19746): interval output that cannot be read back, so a pg_dump example restores no rows (message #2). Dump and restore correctness needs no demand case, and the cost falls only on the interval input path. The field-order regression in message #2 is a design matter for other angles.

### Precedents retrieved

- P1: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant; reason summarised from its thread
- P2: [Fix formatting of Interval output](https://commitfest.postgresql.org/patch/3548/) (cf, Withdrawn, 2022-02-19) - used in the review; reason summarised from its thread
- P3: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant; reason summarised from its thread
- P4: [Improve overflow checks potentially optimized away by compiler](https://commitfest.postgresql.org/patch/400/) (cf, Returned with feedback, 2015-12-01) - judged not relevant; reason summarised from its thread
- P5: [abs function for interval](https://commitfest.postgresql.org/patch/2345/) (cf, Returned with feedback, 2019-11-28) - used in the review; reason summarised from its thread
- P6: [Multiple passwords, interval expirations](https://commitfest.postgresql.org/patch/4432/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P7: [Doc, more or less: uncomment tutorial example that was fixed long ago.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dce62490818170b6479dfe08a28aae4bcdf7cc2d) (revert, Partially reverted, 2021-01-13) - judged not relevant
- P8: [Revert ill-considered change in pg_resetwal output.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=918de37652138da7f7e54c79351178367d60f501) (revert, Reverted, 2022-09-15) - judged not relevant
- P9: [Revert my bad decision of about a year ago to make PortalDefineQuery responsible for copying the query string into the new Portal.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1591fcbec77f6544a0f665758bae912c68e6fa42) (revert, Reverted, 2008-04-02) - judged not relevant
- P10: [Revert refactoring of hex code to src/common/](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2576dcfb76aa71e4222bac5a3a43f71875bfa9e8) (revert, Reverted, 2021-08-19) - judged not relevant
- P11: [Add missing function abs (interval)](https://commitfest.postgresql.org/patch/3054/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P12: [Sequence Access Methods, round two](https://commitfest.postgresql.org/patch/4677/) (cf, Returned with feedback, 2026-02-27) - judged not relevant; reason summarised from its thread
- P13: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1148/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P14: [Add accumulated statistics for wait event](https://commitfest.postgresql.org/patch/2332/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P15: [Revert "Get rid of the "new" and "old" entries in a view's rangetable."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0e6d6d3c909e14eedd370dd77d7111c2ae24c02) (revert, Reverted, 2023-01-12) - judged not relevant
- P16: [BUG: cannot read big bytea (~600MB)](https://commitfest.postgresql.org/patch/1521/) (cf, Rejected, 2018-03-01) - judged not relevant
- P17: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1568/) (cf, Returned with feedback, 2018-03-01) - judged not relevant
- P18: [Support Int64 GUCs](https://commitfest.postgresql.org/patch/5253/) (cf, Rejected, 2024-09-29) - judged not relevant
- P19: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P20: [Revert "Cannot use WL_SOCKET_WRITEABLE without WL_SOCKET_READABLE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a58db3aa10e62e4228aa409ba006014fa07a8ca2) (revert, Reverted, 2020-12-15) - judged not relevant
- P21: [Protect syscache from bloating with negative cache entries](https://commitfest.postgresql.org/patch/931/) (cf, Returned with feedback, 2021-10-05) - judged not relevant
- P22: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - judged not relevant
- P23: [Revert (again) GUC patch to return commented fields to their default values, due to concern about the patch.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f058451871879392ed9295daed3f5310e85f3986) (revert, Reverted, 2006-08-14) - judged not relevant
- P24: [Protect against timestamp underflow in uuidv7(interval)](https://commitfest.postgresql.org/patch/6927/) (cf, Withdrawn, 2026-06-24) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 20; wall time: 132 s; cost reported by the CLI: $1.35
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 63 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:08:39.812Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
