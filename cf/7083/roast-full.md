# Design pre-review: Fix write skew under SERIALIZABLE for Tid Scan & Tid Range Scan

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7083](https://commitfest.postgresql.org/patch/7083/) (Needs review); authors: Aleksander Alekseev (a.alekseev), Jacob Brazeal (polkerty)
- Thread: [Write skew observed under serializable isolation](https://www.postgresql.org/message-id/flat/CA%2BCOZaBo%2BZpKgMvxcdACUjNtdYipe9Em06iX5KHLTVaTmFibiw%40mail.gmail.com), started 2026-07-27
- Messages read: 14 of 14
- Patch prose read: serializable-silently-accepts-write-skew-tid-range-scan-take-v1.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The TID range scan fix (v5-0001) follows the existing seqscan rule and no surviving objection is against it. The TID scan fix (v5-0002) brings back the relation-level locks that cb5b28613d5 removed, and message #14 does not say why a narrower lock is not enough (prior-attempts-1). The entry targets stable branches without saying whether that includes v5-0002 (maintenance-fit-2). Confidence is medium because the v5 patches and their commit messages were not shown.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The v5 patches, their commit messages and the README-SSI changes were not shown, so the exact lock condition in v5-0002 is taken from messages #13 and #14. Whether a lock taken only for probes that find no tuple closes the gap in message #11 cannot be judged without the code.

## The proposal, restated

**Problem.** Under SERIALIZABLE, a TID range scan takes no predicate lock, so two transactions that read the same ctid range and then write can both commit (write skew). A TID scan that probes an empty TID also takes no lock, so a concurrent insert at that TID creates no rw-conflict. Users who rely on SERIALIZABLE get results that match no serial order.

**Proposed design.** heap_beginscan() acquires a relation-level predicate lock for TID range scans, as it already does for seqscans and sample scans. A second patch does the same for TID scans. The patches add a note to the SO_TYPE definitions and update README-SSI.

**User-visible surface.**
- More serialization failures for SERIALIZABLE transactions that use TID scans or TID range scans, including UPDATE ... WHERE CURRENT OF according to a reviewer.
- New isolation test predicate-tidrangescan, plus tests for inserts into an empty table, bitmap scans and TID scans.

**What the author claims.**
- The bug exists in all branches since TID range scans arrived in Postgres 14; target is stable branches.
- Page-level locks are not enough: heap_insert() checks only relation-level locks, and a range can extend past the current end of the relation.
- A relation lock is the correct fix with minimum overhead; narrower locking can come as a separate patch.
- SERIALIZABLE allows false positives by design, so extra aborts are acceptable.

**Thread state.** Andrey Borodin finds the TID range scan change correct, and Andres Freund asked for the SO_TYPE note and bitmap scan tests, now added. Matthias van de Meent and Zsolt Parragi object to relation-level locks, and Zsolt cites cb5b28613d5, which added SO_TYPE_TIDSCAN to avoid exactly such locks.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-27; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. TID scan relation lock reverses cb5b28613d5 without addressing its reason

fixable · level 2, rubric item 12 · prior rejection, backpatchability, performance on the hot path · `prior-attempts-1`

An application runs a SERIALIZABLE cursor loop with UPDATE ... WHERE CURRENT OF on a table that other sessions write to. After a minor upgrade with v5-0002, each loop takes a relation-level SIREAD lock, so any concurrent write to that table can abort it. The application then sees new serialization failures with no change on its side.

Message #13 reports that cb5b28613d5 added SO_TYPE_TIDSCAN in 2020 to avoid relation-level locks for TID scans, and v5-0002 brings those locks back. Message #14 cites false positives being allowed by design, but does not say why the reasoning of that commit was wrong. The bug in message #11 needs a lock only when a probe finds no tuple, while v5-0002 locks the relation for every TID scan. The CF entry targets stable branches, so the extra aborts would arrive in minor releases.

Thread: [#11 (Zsolt Parragi, 2026-09-23)](https://www.postgresql.org/message-id/CAN4CZFOfQL_nOs2ukYf-CkcdP6EOQHu0dx2A%2Ba90F%2BKDqajLKg%40mail.gmail.com), [#12 (Aleksander Alekseev, 2026-09-23)](https://www.postgresql.org/message-id/CAJ7c6TMEJ7sZhASiK5_VjnO6yfr-yawZ_FFzFYBVRFQWNPm7ig%40mail.gmail.com), [#13 (Zsolt Parragi, 2026-09-23)](https://www.postgresql.org/message-id/CAN4CZFMs1W11eBpvo5bJsGaXoEBKkdcgZiBW9VzWTmGkD3zW5w%40mail.gmail.com), [#14 (Aleksander Alekseev, 2026-09-23)](https://www.postgresql.org/message-id/CAJ7c6TMCQmm-7akUPPE0GoAwYdeN5E%3D%3DKU87%3DE8g9yjArVHXMQ%40mail.gmail.com)

**Question:** Why does v5-0002 lock the relation for TID probes that find a visible tuple, and for WHERE CURRENT OF, when cb5b28613d5 deliberately avoided that?

Counter-check: held. Message #13 confirms the relation lock for WHERE CURRENT OF and the conflict with cb5b28613d5, and message #14 answers neither point specifically. The material confirms only the WHERE CURRENT OF case, so "every TID scan" should be stated as likely.

### 2. Back-patching a contested TID scan change adds aborts in minor releases

fixable · level 1b, rubric item 6 · backpatchability, performance on the hot path · `maintenance-fit-2`

A site on Postgres 14 to 19 applies a minor update. Its SERIALIZABLE workload, which uses ctid lookups or cursor updates, starts to fail with serialization errors at a higher rate. Nothing in the release notes points the site to the cause, so it finds out from application retries and errors.

The CommitFest entry targets stable branches, and both v5 patches sit in one entry. v5-0001 for TID range scans has support in message #7, but v5-0002 is disputed in message #13 and partly undoes an earlier deliberate design choice. Changing locking behaviour that affects ordinary cursor loops is a larger change than the range-scan gap, and the proposal does not separate the two for back-patching.

Thread: [#7 (Andrey Borodin, 2026-09-21)](https://www.postgresql.org/message-id/3670DAB7-E3BE-48AC-8DF9-884509775F78%40yandex-team.ru), [#12 (Aleksander Alekseev, 2026-09-23)](https://www.postgresql.org/message-id/CAJ7c6TMEJ7sZhASiK5_VjnO6yfr-yawZ_FFzFYBVRFQWNPm7ig%40mail.gmail.com), [#13 (Zsolt Parragi, 2026-09-23)](https://www.postgresql.org/message-id/CAN4CZFMs1W11eBpvo5bJsGaXoEBKkdcgZiBW9VzWTmGkD3zW5w%40mail.gmail.com)

**Question:** Is v5-0002 meant for stable branches, and if so, what evidence shows that the added serialization failures are acceptable in a minor release?

Counter-check: held. The CommitFest entry targets stable branches and holds both patches, and no message says whether the disputed v5-0002 is meant for back-patching. The release notes detail is speculative, but the request to state the back-patch scope of each patch stands.

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
| backpatchability | **yes** | The entry targets stable branches, so new serialization failures for TID scans and WHERE CURRENT OF would arrive in minor releases. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | v5-0002 takes a relation-level SIREAD lock for TID scans under SERIALIZABLE, which raises the abort rate for cursor loops and ctid updates. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does v5-0002 take the relation lock for every TID scan, or only when a TID probe finds no tuple?
2. Why is a lock taken only for probes that find no tuple not enough, given that cb5b28613d5 added SO_TYPE_TIDSCAN to avoid relation-level locks?
3. Is v5-0002 meant for stable branches, or only v5-0001?
4. How much do serialization failures rise for a SERIALIZABLE cursor loop with UPDATE ... WHERE CURRENT OF on a concurrently written table with v5-0002 applied?
5. Can v5-0001 be committed and back-patched alone while the TID scan lock scope is still under discussion?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund asked for the SO_TYPE note and the bitmap scan tests, and both were added. The owner needs to know SSI predicate locking and the heap scan flags, and nobody has yet settled the lock scope dispute on v5-0002.
- **Liability:** The owner signs for changed abort behaviour under SERIALIZABLE in every branch since Postgres 14, and for the partial reversal of cb5b28613d5 if v5-0002 goes in as is.
- **Author follow-up:** The authors posted five versions, added the requested note and tests, and replied to the objection in message #14. A second author joined in September.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug is a reproducible SERIALIZABLE anomaly, and the change is small and limited to heapam.c plus isolation tests. v5-0001 looks ready for a committer, but the lock scope of v5-0002 and its back-patch status need agreement first.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Back-patched TID scan relation lock adds aborts to WHERE CURRENT OF loops. Refuted (not specific): Duplicate of prior-attempts-1 for the lock scope and of maintenance-fit-2 for the back-patch question. It uses the same scenario and the same requested change.
- `maintenance-fit-1` Relation lock on every TID scan reverses cb5b28613d5 without measurement. Refuted (not specific): Duplicate of prior-attempts-1. It uses the same messages and the same cursor loop scenario, and asks for the same narrower lock condition.
- `demand-1` TID scan fix makes common TID scan users pay for a rare case. Refuted (not specific): Duplicate of prior-attempts-1 on the cost to TID scans that find a tuple. The claim that few applications probe empty TIDs has no support in the material, and a SERIALIZABLE anomaly is a correctness bug that needs no demand evidence.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent matches this idea. [P8] is about narrowing locks for index-only scans and [P4] about SP-GiST predicate locking, and neither recorded reason applies against this proposal. The relation lock for TID range scans (v5-0001) is answered in message #5, so the only objection concerns v5-0002.
- standards-compat: 1 objection(s). The TID range scan fix (v5-0001) follows the existing rule for seqscans and sample scans and adds no syntax, GUC, catalog or dump change. I have no standards or compatibility objection to it.
- replication-ondisk: 0 objection(s). The proposal changes only which in-memory SIREAD locks heap_beginscan() takes, and it reuses the existing relation-level lock that seqscans and sample scans already take. It adds no WAL record, catalog, on-disk format, wire protocol message or changed struct, and prepared transactions keep their locks through the existing 2PC path. A standby does not run SERIALIZABLE transactions, so replay is unaffected. Back-patching changes only abort behaviour, and other angles cover that.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes which SIREAD locks a serializable transaction takes during TID scans and TID range scans, and adds no new object, privilege, setting or visible information. The extra false-positive aborts are a desirability and design question for other angles. A user with only SELECT can already force relation-level SIREAD locks and similar aborts through a seqscan, so the change gives no new way to disturb other sessions. A relation lock replaces many finer locks, so it does not grow predicate lock memory.
- maintenance-fit: 2 objection(s). The TID range scan fix (v5-0001) follows the existing seqscan rule in heap_beginscan(), and I have no maintenance objection to it. The page-lock alternative was answered in message #5.
- already-exists: 0 objection(s). This is a correctness bug fix inside core SSI predicate locking. No existing feature, setting, extension or external tool gives TID scans and TID range scans the missing conflict-in edge. The patch extends the existing heap_beginscan() relation-lock mechanism and adds no second one.
- demand: 1 objection(s). The TID range scan fix (v5-0001) has no demand problem. It fixes a reproducible SERIALIZABLE anomaly in a released feature, and its cost falls only on transactions that use TID range scans under SERIALIZABLE.

### Precedents retrieved

- P1: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P2: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant; reason summarised from its thread
- P3: [support tid range scan in parallel](https://commitfest.postgresql.org/patch/4982/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P4: [Predicate locking in SP-Gist index](https://commitfest.postgresql.org/patch/1215/) (cf, Returned with feedback, 2017-10-02) - used in the review; reason summarised from its thread
- P5: [Tid range scan](https://commitfest.postgresql.org/patch/1755/) (cf, Returned with feedback, 2019-12-01) - judged not relevant; reason summarised from its thread
- P6: [SERIALIZABLE on standby servers](https://commitfest.postgresql.org/patch/1799/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P7: [WIP: Pgbench Errors and serialization/deadlock retries](https://commitfest.postgresql.org/patch/1419/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P8: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - used in the review; reason summarised from its thread
- P9: [WIP: Pgbench Errors and serialization/deadlock retries](https://commitfest.postgresql.org/patch/1645/) (cf, Returned with feedback, 2018-10-01) - judged not relevant
- P10: [Fix pg_rewind false positives caused by shutdown-only WAL](https://commitfest.postgresql.org/patch/6041/) (cf, Withdrawn, 2026-05-01) - judged not relevant
- P11: [Revert workarounds for -Wmissing-braces false positives on old GCC](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=618c64ffd3967cb5313b4b11e1e1043a074f2139) (revert, Reverted, 2025-03-20) - judged not relevant
- P12: [heap_insert() and heap_update() optimization](https://commitfest.postgresql.org/patch/1830/) (cf, Rejected, 2018-10-17) - judged not relevant
- P13: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant
- P14: [Fix race condition in SSI when reading PredXact->SxactGlobalXmin](https://commitfest.postgresql.org/patch/6145/) (cf, Withdrawn, 2025-10-22) - judged not relevant
- P15: [Adjust pg_stat_get_lock() prorows to match lock types](https://commitfest.postgresql.org/patch/6740/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P16: [ctidscan as an example of custom-scan](https://commitfest.postgresql.org/patch/18/) (cf, Rejected, 2015-07-22) - judged not relevant
- P17: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P18: [Sort leaf pages by ctid for gist indexes built using sorting method](https://commitfest.postgresql.org/patch/3455/) (cf, Withdrawn, 2021-12-16) - judged not relevant
- P19: [Defend against encoding ID version skew](https://commitfest.postgresql.org/patch/4601/) (cf, Withdrawn, 2023-10-05) - judged not relevant
- P20: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P21: [Mark function arguments of type "T *" as "const T *" where possible](https://commitfest.postgresql.org/patch/6302/) (cf, Withdrawn, 2026-01-13) - judged not relevant
- P22: [Allow reads to proceed during FK/trigger drops by reducing relation-level lock from AccessExclusive to ShareRowExclusive](https://commitfest.postgresql.org/patch/6124/) (cf, Withdrawn, 2025-11-25) - judged not relevant
- P23: [sort leaf pages by ctid for gist indexes built using sorted method](https://commitfest.postgresql.org/patch/3454/) (cf, Returned with feedback, 2022-03-22) - judged not relevant
- P24: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 110 s; cost reported by the CLI: $1.93
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 69 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:18:30.629Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
