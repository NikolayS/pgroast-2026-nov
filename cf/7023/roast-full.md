# Design pre-review: A new C function `get_partition_root`.

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7023](https://commitfest.postgresql.org/patch/7023/) (Needs review); authors: Peter Smith (smithpb2250)
- Thread: [A new C function `get_partition_root`.](https://www.postgresql.org/message-id/flat/CAHut%2BPtc73UpWSn4teMyMxQoHwGJjRwhWiu%3DhGJqAj5k2AOXEw%40mail.gmail.com), started 2026-07-17
- Messages read: 20 of 20
- Patch prose read: v7-0001-Add-C-function-get_partition_root.patch, v1-0001-Add-C-function-get_partition_root.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal, and reviewers in messages #2 and #14 accept the helper. Two fixable points remain. maintenance-fit-1: with even_if_detached false the helper returns InvalidOid, while get_partition_parent() raises an error. demand-1: the callers that would settle that contract are in the unmerged EXCEPT patches. A committer who knows the partition detach code needs to decide the contract.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read and only 3 of 20 messages were shown. Two points are unchecked: which even_if_detached value pg_partition_root() passes, and whether its result changes for a partition under a pending detach.

## The proposal, restated

**Problem.** Backend code that needs only the root of a partition tree must fetch and free a full ancestors list and take llast_oid(ancestors). The author expects many more such call sites in the in-development FOR TABLES IN SCHEMA EXCEPT logical replication patches.

**Proposed design.** The patch adds get_partition_root() in partition.c, used by pg_partition_root() and pg_depend.c. It takes an even_if_detached flag, like get_partition_parent(), and returns InvalidOid when a concurrent detach is pending. In v7, check_rel_can_be_partition() becomes shared code, and both get_partition_root() and get_partition_ancestors() check their input with Assert.

**User-visible surface.**
- C API: new get_partition_root(relid, even_if_detached) in catalog/partition.h.
- C API: check_rel_can_be_partition() moves from partitionfuncs.c to common code.
- C API: get_partition_ancestors() gains Assert-based input checks in v7.

**What the author claims.**
- Removes the need to build and free an ancestors list just to get the root.
- Will simplify many call sites in the FOR TABLES IN SCHEMA EXCEPT patches.
- Passes make check-world.
- The CommitFest entry tags it "Refactoring Only".

**Thread state.** shveta malik, Chao Li and Álvaro Herrera reviewed it, and the v6 wrapper was dropped in v7 after Herrera's objection. Open: Assert versus if/elog validation, catalog lookups inside Assert, and whether a pending detach with even_if_detached false should raise an error or return InvalidOid.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-17; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Main demand rests on unmerged EXCEPT patches; two callers today

fixable · level 1b, rubric item 3 · demand, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer pushes get_partition_root() with its even_if_detached flag and InvalidOid-on-detach behaviour. The FOR TABLES IN SCHEMA EXCEPT patches then change shape or stall. Core is left with an exported helper whose detach semantics were never checked against the callers it was designed for.

In core, only pg_partition_root() and one site in pg_depend.c use the helper. Message #1 justifies it mainly by "many more code fragments" in the EXCEPT patches, which are "still in development". Message #2 also ties it to that patch. Messages #17 and #20 leave the detach behaviour open, and only real callers can settle whether an error or InvalidOid is the right answer.

Thread: [#1 (Peter Smith, 2026-07-17)](https://www.postgresql.org/message-id/CAHut%2BPtc73UpWSn4teMyMxQoHwGJjRwhWiu%3DhGJqAj5k2AOXEw%40mail.gmail.com), [#2 (shveta malik, 2026-07-28)](https://www.postgresql.org/message-id/CAJpy0uDrs7ag3QwU7QgDNXOo9trLxgq881Qu1TgzuJVwsNpnHA%40mail.gmail.com), [#17 (shveta malik, 2026-08-03)](https://www.postgresql.org/message-id/CAJpy0uDr_DzUvkSYwYrkuD7r3VOcu4VAU-BqtBPP5Xv-9iWQfA%40mail.gmail.com), [#20 (shveta malik, 2026-08-18)](https://www.postgresql.org/message-id/CAJpy0uBWQ2hfCU2w%2B_JZ-_a2dbdw0msJ%3Dt1_ebieTf5wbKXDJw%40mail.gmail.com)

**Question:** Which EXCEPT call sites would use get_partition_root(), and how does each one handle a pending detach, ideally shown as patch 0001 of that series?

Counter-check: held. Herrera's "looks okay" and shveta malik's support lower the doubt about desirability, but neither names the EXCEPT call sites. The detach contract is still open in message #20, and those callers are the evidence that would settle it.

### 2. Detach-pending case returns InvalidOid, unlike get_partition_parent

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

An EXCEPT-patch caller writes relid = get_partition_root(relid, false) and passes the result to a syscache lookup or table_open. During an ALTER TABLE ... DETACH PARTITION CONCURRENTLY, it gets InvalidOid, and the failure shows up as an odd 'cache lookup failed for relation 0' or a silent mismatch. Nobody sees it until a user hits the race in production.

Chao Li approved even_if_detached because it matches get_partition_parent() (message #12). With false, get_partition_parent() raises an error, but the new helper returns InvalidOid. Message #4 shows a caller pattern, relid = get_partition_root(relid), that assigns the result without checking it. The thread lists this as open (message #20), and no committer has answered. A helper meant for many future callers should not need each caller to remember a sentinel check when its sibling function raises an error instead.

Thread: [#4 (shveta malik, 2026-07-29)](https://www.postgresql.org/message-id/CAJpy0uD1Yxm54Y%3DDJsE2vL%2Bifws7V0HpMJBerJMLFU9B8YTxEQ%40mail.gmail.com), [#12 (Chao Li, 2026-08-03)](https://www.postgresql.org/message-id/107B6F21-277C-4A71-AAC2-0E79918AFD5C%40gmail.com), [#17 (shveta malik, 2026-08-03)](https://www.postgresql.org/message-id/CAJpy0uDr_DzUvkSYwYrkuD7r3VOcu4VAU-BqtBPP5Xv-9iWQfA%40mail.gmail.com), [#20 (shveta malik, 2026-08-18)](https://www.postgresql.org/message-id/CAJpy0uBWQ2hfCU2w%2B_JZ-_a2dbdw0msJ%3Dt1_ebieTf5wbKXDJw%40mail.gmail.com)

**Question:** Why should get_partition_root(relid, false) return InvalidOid instead of erroring like get_partition_parent(), and how will callers be kept from using the result unchecked?

Counter-check: held. Message #20 lists this as open, and no committer has answered. get_partition_parent() raises an error in the same case, so the departure needs a stated reason.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch exports get_partition_root() in catalog/partition.h and moves check_rel_can_be_partition() to shared code. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Core gets a second way to find the root, and its pending-detach behaviour differs from get_partition_parent(). |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why does get_partition_root(relid, false) return InvalidOid during a pending detach instead of raising an error like get_partition_parent()?
2. Which call sites in the FOR TABLES IN SCHEMA EXCEPT patches would use get_partition_root(), and what does each one do when a detach is pending?
3. Should input validation use if/elog instead of Assert, as Álvaro Herrera asked, given that the Assert checks do catalog lookups?
4. How does a caller tell a dropped relation from a relation that is not a partition?
5. Does the patch fix the crash risk that message #11 reports for existing get_partition_ancestors() callers during a detach, or is that out of scope?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera reviewed the patch and his objection led to v7, but the thread shows no reply from him on v7. The owner needs to know the DETACH PARTITION CONCURRENTLY code.
- **Liability:** The owner signs for the pending-detach contract of an exported helper that the EXCEPT patches and extensions may call. Changing that contract later means auditing every caller.
- **Author follow-up:** The author posted seven versions in response to reviews and moved the entry to the next CommitFest.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** bundle
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 9 KiB over four files and has two callers in core today. Its main users are the unmerged EXCEPT patches, so reviewing it as the first patch of that series would settle the detach contract against real callers.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Helper and its contract are justified by callers that are not yet committed. Refuted (not specific): Duplicate of demand-1. Its one extra point, the "Refactoring Only" tag, concerns Assert-only checks that do nothing in production builds and has no failure scenario.
- `already-exists-1` Helper duplicates the llast_oid idiom with only two callers today. Refuted (not specific): Duplicate of demand-1: same stall scenario and same request to commit with the EXCEPT series. Its point about differing detach behaviour is covered by maintenance-fit-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found for a C helper that returns a partition root. [P9] was a refactoring rejected for giving no readability gain, but this helper adds handling for a pending detach, and reviewers in messages #2 and #14 accept the idea. The open points in message #20 are design details, not a reason this idea died before.
- standards-compat: 0 objection(s). The patch adds an internal C helper and changes no SQL syntax, GUC, catalog, pg_dump output or psql behaviour. The diff is not shown, so one point stays unchecked: which even_if_detached value pg_partition_root() passes, and whether its result for a partition under a pending DETACH CONCURRENTLY changes.
- replication-ondisk: 0 objection(s). The patch adds a backend C helper and moves check_rel_can_be_partition() into shared code. Nothing changes on-disk format, WAL, catalogs, replication, the wire protocol or existing structs. The extension API change only adds functions, and detach-pending state is read, never written.
- security: 0 objection(s). The proposal adds an internal C helper and moves validation code. pg_partition_root() keeps its SQL surface and privileges, so nothing changes in who can call what or what they can see. The Assert reachable from pg_partition_root() during a concurrent detach (message #10) affects only assert-enabled builds and was addressed in v5, so I see no privilege, exposure or denial-of-service aspect.
- maintenance-fit: 2 objection(s). The helper adds no hot-path cost, because its checks are Assert-only, and it adds no external dependency. Álvaro Herrera has engaged, so the remaining concerns are API-level and fixable.
- already-exists: 1 objection(s). The objection is weak. Herrera called the approach okay in message #14, and the helper follows the even_if_detached convention of get_partition_parent().
- demand: 1 objection(s). Several reviewers support a root lookup helper, and Herrera called it okay in message #14. The only demand gap is that most of the expected callers are not yet in core.

### Precedents retrieved

- P1: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant; reason summarised from its thread
- P2: [Revert "Fix partition pruning setup during DETACH CONCURRENTLY"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b0ea16528cda1f3a993bbfeb2b93eed9762af4e6) (revert, Reverted, 2024-06-24) - judged not relevant
- P3: [Prevent pg_basebackup running as root](https://commitfest.postgresql.org/patch/2431/) (cf, Rejected, 2020-02-27) - judged not relevant; reason summarised from its thread
- P4: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P5: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant; reason summarised from its thread
- P6: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P7: [Revert support for ALTER TABLE ... MERGE/SPLIT PARTITION(S) commands](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3890d90c1508125729ed20038d90513694fc3a7b) (revert, Reverted, 2024-08-24) - judged not relevant
- P8: [Revert "Prevent running pg_basebackup as root"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dcddc3f813ff39d0e33c2b7c77092a6c0b0e9d0b) (revert, Reverted, 2020-02-07) - judged not relevant
- P9: [refactor ExecInitPartitionInfo](https://commitfest.postgresql.org/patch/6280/) (cf, Rejected, 2026-07-06) - used in the review; reason summarised from its thread
- P10: [Add OID descriptions to dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5987/) (cf, Withdrawn, 2025-08-25) - judged not relevant; reason summarised from its thread
- P11: [\[PATCH\] Equivalence Class Filters](https://commitfest.postgresql.org/patch/3524/) (cf, Returned with feedback, 2022-12-17) - judged not relevant; reason summarised from its thread
- P12: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P13: [Fixes a trivial bug in dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5996/) (cf, Withdrawn, 2025-09-02) - judged not relevant
- P14: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/2834/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P15: [dbsize.c functions and partitioned tables](https://commitfest.postgresql.org/patch/1412/) (cf, Returned with feedback, 2018-01-17) - judged not relevant
- P16: [Allow subfield references without parentheses](https://commitfest.postgresql.org/patch/5449/) (cf, Returned with feedback, 2025-02-14) - judged not relevant
- P17: [Add pg_get_owned_sequence function](https://commitfest.postgresql.org/patch/4535/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P18: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P19: [Revert "Add TAP test to automate the equivalent of check_guc"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=52377bb81a2c0427d357bba71e7149de02d016ce) (revert, Reverted, 2022-02-12) - judged not relevant
- P20: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/3364/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P21: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P22: [Add a function to get the version of installed extension](https://commitfest.postgresql.org/patch/5514/) (cf, Withdrawn, 2025-04-03) - judged not relevant
- P23: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant
- P24: [GET DIAGNOSTICS FUNCTION_NAME](https://commitfest.postgresql.org/patch/1457/) (cf, Rejected, 2018-03-07) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 100 s; cost reported by the CLI: $1.56
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 65 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:15:50.634Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
