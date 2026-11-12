# Design pre-review: Validate GIN posting lists before decoding them

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7352](https://commitfest.postgresql.org/patch/7352/) (Needs review); authors: Sehrope Sarkuni (sehrope)
- Thread: [Validate GIN posting lists before decoding them](https://www.postgresql.org/message-id/flat/CAH7T-aoyWXh1GDZRJg5AzG_sAmUnvPdWoV9ybW%2BEA5VP%3DhMwmw%40mail.gmail.com), started 2026-09-27
- Messages read: 1 of 1
- Patch prose read: v1-0005-gin-size-the-posting-list-decode-output-array-fro.patch, v1-0003-gin-reject-invalid-items-in-a-decoded-posting-lis.patch, v1-0002-gin-check-each-posting-list-segment-fits-before-d.patch, v1-0001-gin-bound-decode_varbyte-against-the-segment-end.patch, v1-0004-pageinspect-add-tests-for-corrupt-GIN-posting-lis.patch, v1-0007-gin-scope-decode-locals-to-the-segment-loop.patch, v1-0006-gin-replace-unreachable-repalloc_array-with-Asser.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal: the patches remove out-of-bounds reads on corrupt GIN pages. Two fixable design points remain at level 2, both tied to the backpatch. replication-ondisk-1 asks what redo does with a corrupt posting list in back branches and how a stopped node recovers. maintenance-fit-1 asks for a measurement against unpatched code. Only message #1 is visible, so the judgement rests on thin material.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. The error level in redo, the errcode, which callers reach the new checks during replay, and the real cost of the loop form of decode_varbyte() could not be checked.

## The proposal, restated

**Problem.** ginPostingListDecodeAllSegments() and decode_varbyte() do not check segment sizes, varbyte streams or decoded items against the end of the posting list. A corrupt page is read past its end and returns invalid TIDs on release builds or aborts assert builds. Index scans, VACUUM, WAL replay and pageinspect's gin_leafpage_items() reach this code.

**Proposed design.** Seven patches add bounds checks: decode_varbyte() takes a segment end pointer and becomes a loop, each segment header and its nbytes are checked against the remaining length, and out-of-range or out-of-order items raise an error instead of an Assert(). The output array is sized from len, an unreachable repalloc_array() becomes an Assert(), and pageinspect gets tests.

**User-visible surface.**
- New error when a corrupt GIN posting list is decoded; the thread does not give the message text or SQLSTATE.
- WAL replay now stops with an error on a corrupt GIN posting list instead of continuing.
- gin_leafpage_items() reports corruption instead of aborting or returning invalid TIDs.

**What the author claims.**
- Reproduced on release builds of 14 through 18 and on master.
- Not a security issue, since normal paths need an already-corrupt page and the crafted-input path is pageinspect.
- The loop form of decode_varbyte() shows no measurable performance difference from the nested-if form.
- Sizing the array from len does not over-allocate in practice, since a dense list uses about a byte per item.
- 0001 to 0005 apply back to 14; 0007 is cosmetic and for master only.

**Thread state.** Only the first message is visible. The author asks reviewers to weigh the WAL replay behaviour change before backpatching, and whether to combine patches.

## 2. Prior attempts

### Revert refactoring of hex code to src/common/ (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2576dcfb76aa71e4222bac5a3a43f71875bfa9e8](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2576dcfb76aa71e4222bac5a3a43f71875bfa9e8)
- **Thread:** [https://www.postgresql.org/message-id/flat/1629039545467.80333@nidsa.net](https://www.postgresql.org/message-id/flat/1629039545467.80333@nidsa.net), [https://www.postgresql.org/message-id/flat/20201231003557.GB22199@momjian.us](https://www.postgresql.org/message-id/flat/20201231003557.GB22199@momjian.us), [https://www.postgresql.org/message-id/flat/X%2F%2FggAqmTtt+3t7X@paquier.xyz](https://www.postgresql.org/message-id/flat/X%2F%2FggAqmTtt+3t7X@paquier.xyz)
- **Date:** decided 2021-08-19; started 2020-12-24
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This is a combined revert of the following commits: - c3826f8, a refactoring piece that moved the hex decoding code to src/common/. This code was cleaned up by aef8948, as it originally included no overflow checks in the same way as the base64 routines in src/common/ used by SCRAM, making it unsafe for its purpose. - aef8948, a more advanced refactoring of the hex encoding/decoding code to src/common/ that added sanity checks on the result buffer for hex decoding and encoding. As reported by Hans Buschmann, those overflow checks are expensive, and it is possible to see a performance drop in the decoding/encoding of bytea or LOs the longer they are. Simple SQLs working on large bytea values show a clear difference in perf profile. - ccf4e27, a cleanup made possible by aef8948. The reverts of all those commits bring back the performance of hex decoding and encoding back to what it was in ~13. Fow now and post-beta3, this is the simplest option.
- **Reason (quote):** "As reported by Hans Buschmann, those overflow checks are expensive, and it is possible to see a performance drop in the decoding/encoding of bytea or LOs the longer they are."
- **What is different this time:** P13 added overflow checks to hex encoding and decoding in src/common/. This proposal adds bounds checks to GIN posting list decoding, so only the recorded cost of checks in a decode loop carries over.


## 3. Objections a reviewer is likely to raise

### 1. Backpatched replay error can stop standbys and crash recovery after a minor update

fixable · level 2, rubric item 7 · WAL / replication / standby, backpatchability · `replication-ondisk-1`

A site updates its standby to the new minor release first, as the release notes advise, while the primary on the old minor keeps writing GIN insert records against a corrupt leaf page. The standby's startup process hits the new error during redo, exits, and loops on the same record at each restart. The site finds out from replication lag alerts and must rebuild the standby.

An error in the startup process ends recovery, so on a standby, or on a primary in crash recovery, it takes the whole node down. Before the patch the same record replayed and the node kept running with one bad index. Message #1 calls the change preferable but leaves the backpatch question open. It does not say what level the redo path raises, or how a user gets past the record without a new base backup or pg_resetwal. Today a REINDEX after promotion or startup fixes a bad GIN index, and that path is lost once the node cannot start.

Thread: [#1 (Sehrope Sarkuni, 2026-09-27)](https://www.postgresql.org/message-id/CAH7T-aoyWXh1GDZRJg5AzG_sAmUnvPdWoV9ybW%2BEA5VP%3DhMwmw%40mail.gmail.com)

**Question:** In back branches, what does redo do with a corrupt posting list, and how does an operator bring a standby or crashed primary back up without restoring from backup?

Counter-check: held. Message #1 names the replay change and asks reviewers to weigh it, but gives no error level and no recovery path for a stopped node. The old behaviour also decoded invalid data, so this is a trade-off to settle before backpatching.

### 2. Per-byte bounds checks on the GIN decode hot path are not measured against unpatched code

fixable · level 2, rubric item 8 · performance on the hot path, backpatchability · `maintenance-fit-1`

A user on a minor release runs a full-text or array GIN query that decodes many large posting lists, so decode_varbyte() runs once per item. The new end-pointer test and the loop form add work per byte. The user sees a slower query after the update with no error, and finds out only by comparing before and after.

Message #1 reports no measurable difference between the loop form and the nested-if-with-goto form, but both variants have the checks, so the cost of the checks against unpatched code is not shown. [P13] was reverted because overflow checks added to hex decoding slowed bytea decoding in a visible way, which is the same kind of change: per-byte validation in a decode routine. 0001 to 0005 are meant for 14 and later, where a slowdown found after release is hard to take back.

**Precedents.**
- [Revert refactoring of hex code to src/common/](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2576dcfb76aa71e4222bac5a3a43f71875bfa9e8) (Reverted, 2021-08-19); thread: [#1](https://www.postgresql.org/message-id/flat/1629039545467.80333@nidsa.net), [#2](https://www.postgresql.org/message-id/flat/20201231003557.GB22199@momjian.us), [#3](https://www.postgresql.org/message-id/flat/X%2F%2FggAqmTtt+3t7X@paquier.xyz)

Thread: [#1 (Sehrope Sarkuni, 2026-09-27)](https://www.postgresql.org/message-id/CAH7T-aoyWXh1GDZRJg5AzG_sAmUnvPdWoV9ybW%2BEA5VP%3DhMwmw%40mail.gmail.com)

**Question:** What is the decode cost of patched versus unpatched master on a scan over large, dense GIN posting lists, such as a common-word full-text search?

Counter-check: held. Message #1 compares only two forms that both carry the new checks, so it shows no unpatched baseline. P13 concerns per-byte checks in a decode routine, so it supports asking for a measurement before a backpatch.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | 0001 to 0003 make WAL replay raise an error and stop recovery on a corrupt GIN posting list. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0001 changes decode_varbyte() to take a segment end pointer; whether any exported signature changes cannot be seen without the code. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patches remove out-of-bounds reads reachable with crafted bytes through pageinspect's gin_leafpage_items(); the author states it is not a security issue. |
| backpatchability | **yes** | The author proposes 0001 to 0005 for branches back to 14, including the replay behaviour change. |
| maintenance burden ("who maintains this in 10 years") | **yes** | New error paths and a rewritten decode_varbyte() in ginpostinglist.c, plus new pageinspect tests with crafted page bytes. |
| performance on the hot path | **yes** | decode_varbyte() gains an end-pointer check per byte and item checks, and it runs in GIN index scans and VACUUM. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. In back branches, what error level does redo raise on a corrupt posting list: ERROR, PANIC, or a WARNING that keeps the old behaviour?
2. How does an operator bring back a standby or a crashed primary stopped by this error without restoring from a base backup?
3. What is the decode cost of patched versus unpatched master on a scan over large, dense GIN posting lists, such as a common-word full-text search?
4. What are the error message and errcode, and does the error use ERRCODE_INDEX_CORRUPTED?
5. Is 0006's Assert() meant for back branches, given that 0005 is listed for backpatching and 0006 is not?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the GIN access method and its redo routines, and who is willing to judge a behaviour change in stable branches.
- **Liability:** The owner signs for a redo behaviour change and a decode-path change in every supported branch back to 14. A standby outage or a slowdown found after a minor release is hard to take back.
- **Author follow-up:** No later message is visible. Message #1 raises the replay change for review and offers to combine patches, which suggests willingness to revise.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The fix helps only GIN users with a corrupt page, but it removes out-of-bounds reads and silent invalid TIDs, and adds no new user-visible surface. The patches are small and in one file, but the replay change and the backpatch need a committer who knows GIN, and none has spoken.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` No measurement of the added checks against unpatched decoding. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about the missing unpatched baseline and asks for a more concrete measurement.
- `standards-compat-1` Backpatched replay error turns a corrupt GIN index into standby outage. Refuted (not specific): Duplicate of replication-ondisk-1, which covers the same replay behaviour change and also includes crash recovery on a primary.

### Review angles

- prior-attempts: 1 objection(s). No earlier attempt at hardening GIN posting list decoding appears in the precedent list. The one row found is related only through its performance reason.
- standards-compat: 1 objection(s). The user-facing surface is small: no syntax, GUC, catalog or dump change. Only the backpatched WAL replay behaviour raises a compatibility concern. The error text and errcode are not shown, so I could not check them against existing index corruption reporting.
- replication-ondisk: 1 objection(s). The patch changes no on-disk format, WAL record, catalog, protocol or exported API. Only the replay behaviour change touches this angle.
- security: 0 objection(s). The proposal adds no privilege or exposure; it removes out-of-bounds reads on corrupt GIN pages. A non-superuser cannot write crafted page bytes, and gin_leafpage_items() is superuser-only, so the author's 'not a security issue' claim holds from this angle. The standby-stops-on-error change belongs to other angles; [P13] concerns hot-path cost, not security.
- maintenance-fit: 1 objection(s). The fit with core is sound. This is validation inside an existing core routine, adds no new API or dependency, and needs little ongoing maintenance. The only maintenance-angle concern is the unmeasured cost on the decode path.
- already-exists: 0 objection(s). No existing feature does this job. amcheck checks an index only when someone runs it, so it cannot stop an index scan, VACUUM or WAL replay from decoding a corrupt posting list. Bounds checks inside the decoder are the only place this protection can live, and no hook or extension could add them.
- demand: 0 objection(s). This is a bug fix: decoding a corrupt GIN posting list reads past the page end and returns invalid TIDs, reproduced on 14 through master. Any GIN user with a damaged page benefits, the cost falls mostly on the decode path, and a committer would not ask for demand evidence before fixing an out-of-bounds read. The thread shows no field reports, but that does not weaken the case for a fix. The WAL replay change and the cost on the decode path belong to other angles.

### Precedents retrieved

- P1: [use AV work items infrastructure for GIN pending list's cleanup](https://commitfest.postgresql.org/patch/3060/) (cf, Returned with feedback, 2021-09-08) - judged not relevant; reason summarised from its thread
- P2: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P3: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P4: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P5: [Allow Postgres to pick an unused port to listen](https://commitfest.postgresql.org/patch/4269/) (cf, Returned with feedback, 2023-08-02) - judged not relevant; reason summarised from its thread
- P6: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P7: [Allow pageinspect's bt_page_stats function to return a set of rows instead of a single row](https://commitfest.postgresql.org/patch/3713/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P8: [pageinspect: Hash index support](https://commitfest.postgresql.org/patch/758/) (cf, Returned with feedback, 2016-12-20) - judged not relevant
- P9: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant; reason summarised from its thread
- P10: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P11: [amcheck verification for GiST](https://commitfest.postgresql.org/patch/1800/) (cf, Returned with feedback, 2019-11-28) - judged not relevant; reason summarised from its thread
- P12: [Remove page-read callback from XLogReaderState](https://commitfest.postgresql.org/patch/2113/) (cf, Returned with feedback, 2021-10-04) - judged not relevant; reason summarised from its thread
- P13: [Revert refactoring of hex code to src/common/](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2576dcfb76aa71e4222bac5a3a43f71875bfa9e8) (revert, Reverted, 2021-08-19) - used in the review
- P14: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant
- P15: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P16: [Fix out-of-bounds with function PQescapeInternal](https://commitfest.postgresql.org/patch/4910/) (cf, Rejected, 2024-04-04) - judged not relevant
- P17: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P18: [Use fadvise in wal replay](https://commitfest.postgresql.org/patch/3707/) (cf, Returned with feedback, 2023-04-09) - judged not relevant
- P19: [Don't keep closed WAL segment in page cache after replay](https://commitfest.postgresql.org/patch/5894/) (cf, Withdrawn, 2026-08-20) - judged not relevant
- P20: [Improve overflow checks potentially optimized away by compiler](https://commitfest.postgresql.org/patch/400/) (cf, Returned with feedback, 2015-12-01) - judged not relevant
- P21: [Do not exit on postmaster death event inside CRIT sections.](https://commitfest.postgresql.org/patch/5964/) (cf, Returned with feedback, 2025-08-28) - judged not relevant
- P22: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant
- P23: [Memory-bounded HashAgg](https://commitfest.postgresql.org/patch/63/) (cf, Rejected, 2015-01-19) - judged not relevant
- P24: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 124 s; cost reported by the CLI: $0.98
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 67 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:43:54.253Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
