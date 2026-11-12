# Design pre-review: [PATCH v1 0/1] Avoid carrying self lock-only xmax to updated tuple

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7253](https://commitfest.postgresql.org/patch/7253/) (Needs review); authors: Chee Wooson (chee.wooson)
- Thread: [\[PATCH v1 0/1\] Avoid carrying self lock-only xmax to updated tuple](https://www.postgresql.org/message-id/flat/20260903092738.2352692-1-chee.wooson%40gmail.com), started 2026-09-03
- Messages read: 6 of 6
- Patch prose read: v1-0001-Avoid-carrying-self-lock-only-xmax-to-updated-tup.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

Only standards-compat-1 survived, and it is a fixable level 2 point: after ON CONFLICT DO UPDATE the new row version shows xmax = 0 from SQL. No surviving objection questions the goal or the design. The open matter is whether the same-XID exception is correct under concurrency, and only a committer who knows heap tuple locking can sign that. Confidence is low because no code was read and no one in the visible thread reviewed correctness.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so this review cannot confirm that the exception excludes MultiXacts and other subtransactions as described, or that the old tuple's xmax handling is unchanged. No isolation or concurrency stress test covers the change.

## The proposal, restated

**Problem.** When a transaction locks a row and then updates it with the same XID, heap_update() copies its own lock-only xmax to the new tuple version. In patterns that repeat lock and update in a loop of subtransactions, later iterations then create, expand and reread MultiXacts. The author cites a 2019 thread that describes the same case for ON CONFLICT DO UPDATE and manual row locking.

**Proposed design.** heap_update() treats an old xmax that is a single XID, lock-only, and equal to the updating XID as having no locker to carry, so the new tuple gets an invalid xmax. Remote lockers, a different subtransaction's lock, and MultiXacts keep the current behavior. v1 adds a README.tuplock section on the invariant; v2 drops the pageinspect test and updates the ON CONFLICT DO UPDATE expected output in update.out.

**User-visible surface.**
- On-disk tuple state changes: in the same-XID case the new tuple version has xmax = 0 and HEAP_XMAX_INVALID, which pageinspect shows.

**What the author claims.**
- A carried self-owned KEYSHR_LOCK | LOCK_ONLY marker has no concurrency or referential-integrity role, because the new tuple's xmin is the same XID.
- In a repeated lock-and-update subtransaction loop with fsync off, 10000 iterations took about 126s before and about 27s after, about 4x faster.
- The speedup applies to this repeated lock/update pattern only and is not a general UPDATE performance claim.
- Remote key-share lockers, such as foreign key checks, stay protected.

**Thread state.** João Rito tested v2 with make check and manual pageinspect checks, saw the expected tuple states, an unchanged foreign key case, and a similar speedup; he listed many untested areas. No one has reviewed the correctness of the same-XID exception, which the author asked for.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-03; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Updated rows from ON CONFLICT DO UPDATE now show xmax = 0

fixable · level 2, rubric item 10 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

An application runs INSERT ... ON CONFLICT DO UPDATE ... RETURNING (xmax = 0) AS inserted, a widely copied idiom, to tell inserts from updates. After upgrade, every updated row returns inserted = true. The application miscounts or skips its update branch, and nobody notices until the data is wrong.

Message #5, test 3, shows that the patched server leaves xmax = 0 on the new version after ON CONFLICT DO UPDATE, where it was the updating XID before. v2 changes the existing ON CONFLICT expected output in update.out, so the change is visible from SQL as well as through pageinspect. The idiom is undocumented, but a committer would likely want it named in the release notes, with a pointer to a supported way to tell inserts from updates. The Bugfix tag also suggests a backpatch, and making this change in a minor release would be harder to defend.

Thread: [#4 (Chee Wooson, 2026-09-03)](https://www.postgresql.org/message-id/CAMfungPiUSxdTR%2BHNRhV0sVXTQgguFtuWgVrGBFKSY8dpgyC3w%40mail.gmail.com), [#5 (João Rito, 2026-11-12)](https://www.postgresql.org/message-id/CADMs5OPxgrppCUfx6JsZkOZhhQ9XcpMEpzp_UN6BP%2BKgyrb3nw%40mail.gmail.com)

**Question:** Will the patch stay master-only with a release-note entry for applications that read xmax after ON CONFLICT DO UPDATE, and what replacement will that entry point to?

Counter-check: held. ON CONFLICT DO UPDATE locks the existing row and then updates it with the same XID, so today the new version carries that XID as xmax. Message #5 test 3 and the update.out change in v2 show it becomes 0, and the thread does not discuss the SQL-visible effect.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | heap_update() writes xmax = 0 with HEAP_XMAX_INVALID on the new tuple version in the same-XID case; the state already exists on heap pages. |
| WAL / replication / standby | **yes** | The update WAL record carries the new tuple's xmax and infomask, so standbys replay the changed value. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Updated rows from ON CONFLICT DO UPDATE now show xmax = 0 |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix but targets version 20, and the thread does not say whether a backpatch is intended. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a same-XID special case to tuple-lock carrying in heapam.c, backed by an invariant in README.tuplock. |
| performance on the hot path | **yes** | The change is inside heap_update(), which every UPDATE runs; the measured gain covers one lock-and-update loop only. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the patch for master only, and will the release notes name the xmax = 0 change after ON CONFLICT DO UPDATE and point to a supported replacement?
2. Which isolation test shows that a concurrent FOR KEY SHARE locker and a lock from a different subtransaction still keep their xmax on the new tuple version?
3. Does the same-XID exception hold when the first lock is FOR SHARE, FOR KEY SHARE or FOR NO KEY UPDATE instead of FOR UPDATE?
4. Does v2 keep the README.tuplock section that states the invariant?
5. How did the 2019 discussion cited in message #1 end, and why was this change not made then?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows heap tuple locking, MultiXacts and the xmax infomask rules in heapam.c would have to own it. No such person has reviewed the same-XID exception in the visible thread.
- **Liability:** The owner signs for the correctness of the xmax that heap_update() writes after a same-XID row lock, with no time limit. A mistake here would be a concurrency or referential-integrity bug found in production.
- **Author follow-up:** The author posted v2 after v1 and asked for a correctness review, which suggests continued engagement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patch is small and adds no user-facing feature, and the measured gain covers lock-and-update loops in subtransactions. Review cost comes from the tuple-locking reasoning, and no one with that background has commented yet.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Same-XID exception may lose its README invariant and its focused test. Refuted (no failure scenario): The scenario is an unnamed future patch that makes an unnamed mistake, which fits any change to heap_update(). The material shows no code, so "comment-free branch" has no support, and test coverage and README text are outside levels 1-2.
- `maintenance-fit-2` Bugfix tag suggests a backpatch of a tuple-locking behavior change. Refuted (no failure scenario): The CommitFest entry targets version 20 and no message asks for a backpatch. A CommitFest tag does not put a patch into back branches, because the committer decides that, so the scenario has no path to happen.
- `demand-1` Benefit shown only for a synthetic subtransaction loop, no field report. Refuted (no failure scenario): The cost side is an unnamed "subtle visibility or freezing bug", with no concrete case. The patch adds no user-facing feature to maintain, and message #5 reproduced the speedup on the path the patch changes, in a loop that uses ordinary PL/pgSQL exception blocks.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent matches. [P8] lets one INSERT ... ON CONFLICT DO UPDATE update its own inserted row twice, and [P1] changes how VACUUM freezes MultiXacts. Neither recorded reason applies to dropping a same-XID lock-only xmax in heap_update(). Message #1 cites a 2019 discussion of this case, but the material does not say how it ended.
- standards-compat: 1 objection(s). There is no new syntax, GUC, catalog object or psql change, so there is nothing to check for pg_dump round-trip. The only user-facing effect is the xmax value seen from SQL.
- replication-ondisk: 0 objection(s). The change creates no new persistent or cross-node state. The new tuple version gets xmax = 0 with HEAP_XMAX_INVALID, a state heap pages already hold everywhere, so no on-disk format, pg_upgrade, catalog, wire protocol or ABI change follows, and tuples already carrying a self lock-only xmax stay valid. The update WAL record already carries the new tuple's xmax and infomask from the primary, so a standby replays the primary's choice. A rolling upgrade or mixed-version standby sees no new record type. Logical decoding does not read the new version's xmax. VACUUM and freezing see fewer MultiXacts, not new states. The open question is whether the same-XID exception is correct against concurrent lockers, and that belongs to the locking-semantics angle. Without the code or isolation tests, this angle cannot confirm that the old tuple's xmax handling is unchanged.
- security: 0 objection(s). The change has no privilege or exposure aspect. It only alters the xmax that heap_update() writes when the same XID locks and then updates a row; it adds no user, role or function, and exposes no new data. Whether it is correct under concurrency belongs to other angles.
- maintenance-fit: 2 objection(s). From the maintenance and fit angle the idea belongs in core. It adds one comparison to heap_update() with no visible cost to users who do not use the pattern. It also makes an updated tuple look like a tuple freshly inserted by the same XID. The objections above concern how the change is kept safe and where it lands, not whether it is wanted.
- already-exists: 0 objection(s). No existing feature, extension or hook does this, because the change is inside heap_update() and how it fills in the new tuple's xmax. Rewriting the application avoids part of the cost, for example one UPDATE ... RETURNING in place of SELECT ... FOR UPDATE then UPDATE, but ON CONFLICT DO UPDATE takes its lock internally and users cannot avoid that.
- demand: 1 objection(s). Demand is the only concern from this angle. The change has no UI and the pattern is plausible, so this is a request to show demand, not grounds to reject.

### Precedents retrieved

- P1: [Make VACUUM remove MultiXacts with updater XIDs more eagerly](https://commitfest.postgresql.org/patch/4095/) (cf, Withdrawn, 2023-01-26) - used in the review; reason summarised from its thread
- P2: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P3: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P4: [disallow HEAP_XMAX_COMMITTED and HEAP_XMAX_IS_LOCKED_ONLY](https://commitfest.postgresql.org/patch/3434/) (cf, Withdrawn, 2023-02-25) - judged not relevant; reason summarised from its thread
- P5: [Revert XactLockTableWait context setup in conditional multixact wait](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3adba736620c85112e0f9f6d043d3574e1c5830f) (revert, Reverted, 2014-08-25) - judged not relevant
- P6: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - judged not relevant
- P7: [Avoid computing ORDER BY junk columns unnecessarily](https://commitfest.postgresql.org/patch/4717/) (cf, Returned with feedback, 2024-02-10) - judged not relevant; reason summarised from its thread
- P8: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - used in the review; reason summarised from its thread
- P9: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P10: [CRC32C Parallel Computation Optimization on ARM](https://commitfest.postgresql.org/patch/4620/) (cf, Returned with feedback, 2025-04-02) - judged not relevant; reason summarised from its thread
- P11: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant
- P12: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P13: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P14: [Revert "Avoid creation of the free space map for small heap relations."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a23676503b746b7f1588cd2ab0c60411032d32da) (revert, Reverted, 2019-01-28) - judged not relevant
- P15: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant
- P16: [heap_insert() and heap_update() optimization](https://commitfest.postgresql.org/patch/1830/) (cf, Rejected, 2018-10-17) - judged not relevant
- P17: [Fix log_line_prefix to display the transaction id (%x) for statements not in a transaction block](https://commitfest.postgresql.org/patch/4606/) (cf, Withdrawn, 2024-05-18) - judged not relevant
- P18: [Revert "Use group updates when setting transaction status in clog."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=390811750d1647ed81a462da3af7f1fd66166e95) (revert, Reverted, 2017-03-10) - judged not relevant
- P19: [Refactor how we form HeapTuples for CatalogTuple(Insert|Update)](https://commitfest.postgresql.org/patch/6221/) (cf, Withdrawn, 2026-01-30) - judged not relevant
- P20: [Revert "Avoid the creation of the free space map for small heap relations".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7db0cde6b58eef2ba0c70437324cbc7622230320) (revert, Reverted, 2019-05-07) - judged not relevant
- P21: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P22: [Document efficient self-joins / UPDATE LIMIT techniques.](https://commitfest.postgresql.org/patch/4539/) (cf, Returned with feedback, 2024-02-13) - judged not relevant
- P23: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant
- P24: [Add a reset_computed_values function in pg_stat_statements](https://commitfest.postgresql.org/patch/2275/) (cf, Returned with feedback, 2019-12-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 139 s; cost reported by the CLI: $1.56
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 63 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:32:45.673Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
