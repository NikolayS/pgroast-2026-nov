# Design pre-review: Doc: clarify pg_locks descriptions of classid/objid/objsubid

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7309](https://commitfest.postgresql.org/patch/7309/) (Needs review); authors: not given
- Thread: [Doc: clarify pg_locks descriptions of classid/objid/objsubid](https://www.postgresql.org/message-id/flat/CA%2Br%3DywWUHOPgHp-tegoETpfjiEkE5Ytzi4g9orcKav8bWym%2B_Q%40mail.gmail.com), started 2026-03-30
- Messages read: 2 of 2
- Patch prose read: 0001-clarify-pg_locks-descriptions-of-classid-objid-objsu.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The patch corrects column descriptions in system-views.sgml that contradict the per-lock-type prose under the same table, and it changes no behavior. A committer agreed with the direction in message #2 and registered as committer. Confidence is medium because the v2 wording is not shown.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only the v1 hunk was shown, so the exact v2 wording could not be checked. The claim that pg_lock_status() fills these columns for the three lock types rests on message #1 and was not verified against the code.

## The proposal, restated

**Problem.** The pg_locks docs say classid, objid and objsubid are null when the lock target is not a general database object. pg_lock_status() also fills these columns for advisory, speculative insertion and apply transaction locks, so the column table contradicts the prose below it. Users reading pg_locks get a wrong description.

**Proposed design.** Each column description is scoped to the general database object case and points to the per-lock-type prose below, without listing the other lock types. The prose for apply transaction locks now says objid shows the subscription OID. In v2, Fujii Masao also scoped the "references" notes, updated the introductory paragraph and linked the subscription OID to pg_subscription.oid.

**User-visible surface.**
- Changed documentation of pg_locks columns classid, objid and objsubid in system-views.sgml.
- New documented meaning of pg_locks.objid for apply transaction locks: the subscription OID.

**What the author claims.**
- The current column descriptions are false for advisory, speculative insertion and apply transaction locks.
- Not listing the non-object lock types in the column descriptions keeps them from drifting as new lock types are added.
- The change is documentation only; no behavior changes.

**Thread state.** Fujii Masao agreed with the direction, posted v2 with extra edits and registered himself as committer. The author has not replied to v2.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-03-30; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | **yes** | The doc error exists in released branches, and the thread does not say whether the fix is back-patched. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should the fix be back-patched, and to which branches, given that apply transaction locks do not exist in all supported branches?
2. Which lock types still show null in classid, objid and objsubid, and does the v2 text still tell the reader that?
3. Does the per-lock-type prose in v2 describe the use of classid, objid and objsubid for every lock type that fills them, including speculative insertion locks?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Fujii Masao posted v2 in message #2 and is registered as committer on the CommitFest entry.
- **Liability:** The wording of three column descriptions and one paragraph in system-views.sgml; the cost is a later doc correction if a lock type is described wrongly.
- **Author follow-up:** No sign either way; the author has not replied to v2.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches one documentation file, corrects a description that contradicts the prose below it, and a committer has taken it on. Readers of pg_locks who use advisory locks or logical replication gain.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at this idea, and the thread does not mention one. This is a docs-only fix that a committer has taken on, so a prior-attempts objection has nothing to rest on.
- standards-compat: 0 objection(s). This is a documentation-only change to the pg_locks column descriptions. It changes no syntax, GUC, catalog, dump output, psql behaviour or client-visible result, so nothing on the standards and compatibility side breaks for existing users. The v1 text drops the remark that these columns are null for relation, page, tuple and transaction ID locks; reviewers of the v2 wording may want to check that the per-lock-type prose still covers those cases.
- replication-ondisk: 0 objection(s). The patch changes only doc/src/sgml/system-views.sgml. It adds no on-disk, WAL, catalog, protocol or ABI change, so it creates no persistent or cross-node state. The new note that objid shows the subscription OID for apply transaction locks describes what pg_lock_status() already reports, as message #1 states.
- security: 0 objection(s). The patch only changes the pg_locks column descriptions in system-views.sgml. It does not change who can read pg_locks or what pg_lock_status() returns. Documenting that objid shows the subscription OID for apply transaction locks describes data the view already exposes to anyone who can query it, so there is no privilege or exposure aspect.
- maintenance-fit: 0 objection(s). This patch changes documentation only. It fixes column descriptions that contradict the prose under the same table, and it adds no hot-path cost, API, GUC or dependency. Not listing the non-object lock types lowers future drift, and a committer who knows the area has taken the patch on. Maintenance and fit give no ground to reject it. Back-patching was not discussed, and the v2 wording is not shown.
- already-exists: 0 objection(s). This is a docs-only fix that corrects column descriptions contradicting the prose below them. No existing feature, extension or tool makes the docs accurate, so the already-exists angle has nothing to raise.
- demand: 0 objection(s). This is a docs-only fix for column descriptions that contradict the prose below them for advisory, speculative insertion and apply transaction locks. Any reader of pg_locks gains, the cost is near zero, and a committer has taken it on, so a demand objection does not apply.

### Precedents retrieved

- P1: [Revert "Display the time when the process started waiting for the lock, in pg_locks."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=890d2182a2c425aaa80f9bf9f7116d31e0c6538e) (revert, Reverted, 2021-02-09) - judged not relevant
- P2: [Document AccessExclusive lock behaviour on standbys](https://commitfest.postgresql.org/patch/5920/) (cf, Withdrawn, 2026-02-09) - judged not relevant; reason summarised from its thread
- P3: [Lockable views](https://commitfest.postgresql.org/patch/1334/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P4: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1133/) (cf, Returned with feedback, 2018-03-30) - judged not relevant; reason summarised from its thread
- P5: [Speed up LockReleaseAll](https://commitfest.postgresql.org/patch/3220/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P6: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1666/) (cf, Returned with feedback, 2018-12-01) - judged not relevant; reason summarised from its thread
- P7: [Revert: Honor OID status of CREATE LIKE'd tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=764ce22af30df3d62524d41815c41b4d696f3c10) (revert, Reverted, 2015-04-26) - judged not relevant
- P8: [Surround CheckRelation\[Oid\]LockedByMe() with USE_ASSERT_CHECKING](https://commitfest.postgresql.org/patch/5096/) (cf, Withdrawn, 2024-07-01) - judged not relevant; reason summarised from its thread
- P9: [group locking](https://commitfest.postgresql.org/patch/23/) (cf, Rejected, 2015-01-19) - judged not relevant; reason summarised from its thread
- P10: [Add pg_ownerships and pg_privileges system views](https://commitfest.postgresql.org/patch/5033/) (cf, Withdrawn, 2025-01-04) - judged not relevant; reason summarised from its thread
- P11: [Restore lock level to update statusFlags](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dcfff74fb16622898a9d1df33e530f477caab095) (revert, Partially reverted, 2020-11-26) - judged not relevant
- P12: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant; reason summarised from its thread
- P13: [pg_stat_walwrites system view](https://commitfest.postgresql.org/patch/1003/) (cf, Returned with feedback, 2017-11-27) - judged not relevant
- P14: [Revert "pg_dump: Lock all relations, not just plain tables".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d3adaabaf7d555ec8bb1d83c43f72e79f1bf0b7d) (revert, Reverted, 2020-11-06) - judged not relevant
- P15: [Add system views pg_permissions and pg_ownerships](https://commitfest.postgresql.org/patch/3033/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P16: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant
- P17: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - judged not relevant
- P18: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P19: [Revert: Let table AM insertion methods control index insertion](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da841aa4dc279bb0053de56121c927ec943edff3) (revert, Reverted, 2024-04-11) - judged not relevant
- P20: [Refactoring speculative insertion with unique indexes a little](https://commitfest.postgresql.org/patch/403/) (cf, Rejected, 2016-09-20) - judged not relevant
- P21: [pg_get_databasebyid(oid)](https://commitfest.postgresql.org/patch/2261/) (cf, Returned with feedback, 2019-09-02) - judged not relevant
- P22: [Check for tuplestorestate nullness before dereferencing](https://commitfest.postgresql.org/patch/5705/) (cf, Withdrawn, 2025-12-08) - judged not relevant
- P23: [Add OID descriptions to dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5987/) (cf, Withdrawn, 2025-08-25) - judged not relevant
- P24: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 59 s; cost reported by the CLI: $0.55
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 41 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:39:11.330Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
