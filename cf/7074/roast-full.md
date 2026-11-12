# Design pre-review: Introducing find_all_inheritors_ordered()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7074](https://commitfest.postgresql.org/patch/7074/) (Needs review); authors: Chao Li (chaol)
- Thread: [Introducing find_all_inheritors_ordered()](https://www.postgresql.org/message-id/flat/57100518-3FF1-48AD-B550-47B9A3E88688%40gmail.com), started 2026-07-30
- Messages read: 10 of 10
- Patch prose read: v3-0001-Add-find_all_inheritors_ordered.patch, v1-0001-Add-find_all_inheritors_ordered.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and no level fails. demand-1 asks for confirmation that the v3 refactor leaves find_all_inheritors() results and lock order unchanged, since many commands use that function and no one has replied to v3. maintenance-fit-1 asks for a second caller and for a guard against choosing the cycle-raising function on a path that must tolerate cycles. Both are fixable, and a committer who knows the inheritance locking code can settle them.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check whether v3 keeps find_all_inheritors() lock order and results, or whether the cycle check covers every cycle shape. Only 3 of 10 messages were shown in full.

## The proposal, restated

**Problem.** find_all_inheritors() can return a descendant before one of its ancestors when multiple inheritance gives both direct and indirect paths to a relation. Because of this, the fix in commit 0cd17fdd3c0 for altering CHECK constraint enforceability had to recurse upward through all ancestors. The audience is backend code, not end users.

**Proposed design.** A new backend function, find_all_inheritors_ordered(), returns the inheritance tree in topological order using Kahn's algorithm, without recursion, and raises an error when it finds a cycle. In v3 it shares data gathering and locking with find_all_inheritors(). ALTER CONSTRAINT ... NOT ENFORCED uses it so each child checks only its direct parents; ENFORCED keeps find_all_inheritors().

**User-visible surface.**
- C API: new function find_all_inheritors_ordered() declared in src/include/catalog/pg_inherits.h.
- ALTER TABLE ... ALTER CONSTRAINT ... NOT ENFORCED on an inheritance tree with a cycle now fails with an explicit cycle error instead of a stack depth error.

**What the author claims.**
- The new function significantly simplifies the enforceability fix from commit 0cd17fdd3c0.
- Other features that need parent-before-child traversal could use the function.
- v3 keeps master's success and failure outcomes in Zsolt Parragi's isolation test, replacing only the stack depth error with a cycle error.
- v3 shares data gathering and locking with find_all_inheritors(), so lock order stays the same.

**Thread state.** solai v and Ewan Young reviewed v1 favorably; Zsolt Parragi showed concurrent ALTER TABLE ... INHERIT can create cycles, and Tom Lane asked for shared data gathering, both addressed in v3. No one has replied to v3, and blocking cycles at creation is left for a separate thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. One caller does not yet justify a shared traversal refactor

fixable · level 1b, rubric item 4 · desirability, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `demand-1`

v3 changes the internals of find_all_inheritors(), which TRUNCATE, ANALYZE, LOCK and other DDL use on every inheritance tree. If the shared data-gathering step changes lock or result order in some case, users of those commands see new deadlocks or different behaviour. They find out through a bug report long after commit, and the only gain is a cleaner ALTER CONSTRAINT ... NOT ENFORCED path.

The only user problem shown is code shape. The fix in 0cd17fdd3c0 already works, and v3 keeps master's success and failure outcomes (message #8), apart from the error text in a cycle case that is reachable only through concurrent ALTER TABLE ... INHERIT (messages #4, #10). The claim that other features need parent-before-child order (message #1) names no feature. Message #7 shows the two functions are tied by lock order, so the risk of the shared refactor reaches every caller of find_all_inheritors(), while the benefit stays with one code path.

Thread: [#1 (Chao Li, 2026-07-30)](https://www.postgresql.org/message-id/57100518-3FF1-48AD-B550-47B9A3E88688%40gmail.com), [#7 (Tom Lane, 2026-09-07)](https://www.postgresql.org/message-id/460958.1788824290%40sss.pgh.pa.us), [#8 (Chao Li, 2026-09-08)](https://www.postgresql.org/message-id/478CC29D-1E62-4804-9FF0-C8E25E604FEE%40gmail.com), [#10 (Zsolt Parragi, 2026-09-08)](https://www.postgresql.org/message-id/CAN4CZFNBn%2BrQst%3D6_3HDLksbT_EHe_5SUc0X5ECu%3D3HV1gE0vA%40mail.gmail.com)

**Question:** Which existing or planned callers besides the NOT ENFORCED path need parent-before-child order, and has anyone confirmed find_all_inheritors() results and lock order are unchanged?

Counter-check: held. v3 rewrites the internals of find_all_inheritors(), which many commands use, and no one has replied to v3 to confirm that results and lock order are unchanged. Tom Lane asked for the sharing, but that does not confirm it is correct.

### 2. Second traversal API with different cycle semantics and one caller

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a DDL path and calls find_all_inheritors_ordered() because it wants parent-before-child order. Tests pass, since cycles appear only through concurrent ALTER TABLE ... INHERIT. Months later a user with such a cycle finds that the command now errors where find_all_inheritors() would have coped.

After v3, pg_inherits.h exports two functions that return the same set of OIDs but differ on cycles: one tolerates them, the other raises an error (message #5, message #8). The only rule for choosing between them is a header comment. The patch has one caller, and even ALTER CONSTRAINT splits by direction because the ordered function is unsafe for ENFORCED (message #6, message #8). The claim that other features could use it names no second caller.

Thread: [#5 (Chao Li, 2026-09-07)](https://www.postgresql.org/message-id/20EAC3F9-81A2-41DD-89FA-1539EA8ED11B%40gmail.com), [#6 (Zsolt Parragi, 2026-09-07)](https://www.postgresql.org/message-id/CAN4CZFNZD-1_XcEA%2BBGAi9_KgqzuONzecyVsMu9sB1Xp8%2BgHkg%40mail.gmail.com), [#8 (Chao Li, 2026-09-08)](https://www.postgresql.org/message-id/478CC29D-1E62-4804-9FF0-C8E25E604FEE%40gmail.com)

**Question:** Which other current or planned callers need ordered traversal, and how would a reviewer catch a caller that picks the cycle-raising function on a path where cycles must be tolerated?

Counter-check: held. Message #6 shows this hazard already occurred: v2 used the ordered function for ENFORCED and changed an outcome that master allowed. The thread names no second caller, and only a header comment guides the choice between the two functions.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch exports a new function, find_all_inheritors_ordered(), in src/include/catalog/pg_inherits.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Two traversal functions with different cycle behaviour share data gathering and locking, and the ordered one has one caller. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Has anyone confirmed that find_all_inheritors() returns the same results and takes locks in the same order after the v3 refactor, and how was that checked?
2. Which existing or planned callers besides the NOT ENFORCED path need parent-before-child order?
3. How would a reviewer catch a new caller that uses find_all_inheritors_ordered() on a path where an inheritance cycle must be tolerated?
4. What are the message text and SQLSTATE of the new cycle error?
5. Does v3 include the isolation test from message #6 and the comment cleanup and typedefs.list entry from message #3?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane asked for shared data gathering in message #7 but has not replied to v3. The owner needs to know inheritance traversal and DDL lock order.
- **Liability:** The owner signs for unchanged lock order and results in find_all_inheritors(), which TRUNCATE, ANALYZE, LOCK and other DDL use, and for a new exported function that extensions may call.
- **Author follow-up:** The author posted v2 and v3 in response to review and agreed in message #9 to look into blocking cycles at creation in a separate thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patch is a backend cleanup of one call site, about 19 KiB across pg_inherits.c and tablecmds.c, with no end-user surface beyond one error message. Review cost is medium because the v3 refactor changes the internals of find_all_inheritors() and lock order needs a careful read.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Cycle semantics fixed before the cycle-prevention work is decided. Refuted (no failure scenario): Blocking cycles at creation does not remove cycles already present in existing clusters, so the error branch stays reachable and v1's Assert would still be wrong. The ENFORCED/NOT ENFORCED split is internal to tablecmds.c and can be removed later whatever extensions call.
- `already-exists-1` Committed upward recursion already gives correct results; new API has one caller. Refuted (already handled): The scenario treats the shared helper as the risk, but Tom Lane asked for that sharing in message #7 so the two traversals cannot diverge in lock order. The one-caller point is a duplicate of maintenance-fit-1, and the unconfirmed refactor is covered by demand-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P8] concerns blocking DROP NOT NULL on child tables, and its recorded reason does not apply to an ordered traversal function. The thread mentions only the earlier bug fix from message #1 (commit 0cd17fdd3c0), which this patch builds on. Tom Lane's shared data-gathering request in message #7 is reported as addressed in v3.
- standards-compat: 0 objection(s). From the standards and compatibility angle the proposal has almost no user-facing surface: no syntax, keyword, GUC, catalog, pg_dump or psql change. The one visible change is a cycle error in place of a stack depth error, and only in an inheritance cycle that a race between concurrent ALTER TABLE ... INHERIT commands can create (message #4). Tom Lane wants such cycles blocked instead of supported (message #7). v3 keeps master's success and failure outcomes in the message #6 test (message #8), but that test covers two permutations and does not show that outcomes match for every cycle shape. The SQLSTATE and wording of the new error are not shown, which is a detail-level point.
- replication-ondisk: 0 objection(s). The proposal adds no persistent or cross-node state. It does not change the on-disk format, WAL records, catalogs, catversion or the wire protocol, and existing ABI is unaffected because it only adds one new backend function to pg_inherits.h. Lock order across backends is the one cross-process concern; message #7 raised it and v3 answers it with shared data gathering, which only a code read can confirm.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It adds an internal C traversal function and changes how ALTER TABLE ... ALTER CONSTRAINT ... NOT ENFORCED walks an inheritance tree that the caller must already own. Per message #9, v3 shares locking with find_all_inheritors(), so it takes no new locks and opens no new resource path. The only user-visible change replaces a stack depth error with a cycle error in a case that already fails on master (message #8). Building a cycle needs ALTER TABLE ... INHERIT on the tables involved, so an unprivileged user gains no new way to block another owner's DDL. One point could not be checked without the code: whether the cycle error names only relations the caller can already see.
- maintenance-fit: 2 objection(s). There is no hot-path cost, since this is DDL-only. Tom Lane's lock-order concern is addressed in v3 according to message #9, but no reviewer has confirmed it.
- already-exists: 1 objection(s). Postgres has no existing parent-before-child traversal, and no extension can provide one, so the only point from this angle is that the committed fix already meets the need.
- demand: 1 objection(s). This is backend cleanup with no end-user demand expected. The one objection asks for a second caller or proof that existing callers are unaffected. It does not claim the idea is unwanted.

### Precedents retrieved

- P1: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant; reason summarised from its thread
- P2: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant; reason summarised from its thread
- P3: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - judged not relevant
- P4: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P5: [Revert incorrect memory-conservation hack in inheritance_planner().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0f36045b2e3d037bb7647d84373404fa4ba9588) (revert, Reverted, 2011-01-13) - judged not relevant
- P6: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P7: [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://commitfest.postgresql.org/patch/5992/) (cf, Returned with feedback, 2026-04-02) - judged not relevant; reason summarised from its thread
- P8: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - used in the review; reason summarised from its thread
- P9: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P10: [Fix alter subscription concurrency errors](https://commitfest.postgresql.org/patch/3842/) (cf, Withdrawn, 2023-03-28) - judged not relevant; reason summarised from its thread
- P11: [Allow meson to find ICU in non-standard location](https://commitfest.postgresql.org/patch/4203/) (cf, Withdrawn, 2023-03-08) - judged not relevant; reason summarised from its thread
- P12: [Allow subfield references without parentheses](https://commitfest.postgresql.org/patch/5449/) (cf, Returned with feedback, 2025-02-14) - judged not relevant; reason summarised from its thread
- P13: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant; reason summarised from its thread
- P14: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant
- P15: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P16: [Fix for PL/Python slow input arrays traversal issue](https://commitfest.postgresql.org/patch/687/) (cf, Rejected, 2016-09-22) - judged not relevant
- P17: [Merging statistics from children instead of re-sampling everything](https://commitfest.postgresql.org/patch/3052/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P18: [Find additional connection service files in pg_service.conf.d directory](https://commitfest.postgresql.org/patch/1480/) (cf, Returned with feedback, 2018-09-30) - judged not relevant
- P19: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P20: [Allow ordered partition scans in more cases](https://commitfest.postgresql.org/patch/4198/) (cf, Withdrawn, 2023-06-24) - judged not relevant
- P21: [Fix GROUP BY ALL handling of ORDER BY operator semantics](https://commitfest.postgresql.org/patch/6952/) (cf, Withdrawn, 2026-07-16) - judged not relevant
- P22: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant
- P23: [SIMD optimization for list_sort and tuple-sort](https://commitfest.postgresql.org/patch/5607/) (cf, Withdrawn, 2025-08-06) - judged not relevant
- P24: [logical column ordering (WIP)](https://commitfest.postgresql.org/patch/21/) (cf, Returned with feedback, 2015-05-03) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 93 s; cost reported by the CLI: $1.35
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 64 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:17:32.681Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
