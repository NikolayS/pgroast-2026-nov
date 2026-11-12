# Design pre-review: pgstat: Add helper to drop entries from a stats kind's hash

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7546](https://commitfest.postgresql.org/patch/7546/) (Needs review); authors: Sami Imseih (simseih)
- Thread: [\[PATCH\] pgstat: Add helper to drop entries from a stats kind's hash](https://www.postgresql.org/message-id/flat/CAN12%2BYL7SrAwzMpzfNLrVcy%3Dw5L03p7czyO73RfS06kmdqs_fw%40mail.gmail.com), started 2026-10-27
- Messages read: 18 of 18
- Patch prose read: v6-0001-pgstat-Add-kind-filter-to-pgstat_drop_matching_en.patch, v1-0001-pgstat-Add-helper-to-drop-entries-from-a-stats-ki.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The change adds one argument to an internal function, on master only, and old callers fail at compile time. A committer who knows the area chose the single-function design and reports committing v6 in message #18. Confidence is medium because only 3 of 18 messages and no code were read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that the kind filter runs before the match callback and before the NULL-callback drop-all rests on the commit message and message #3. Messages #4 to #18 were not shown in full.

## The proposal, restated

**Problem.** pgstat_drop_matching_entries() scans every variable-numbered stats hash, even when the caller already knows which stats kind it wants. Since 44539786b82 a kind may have its own dshash, so a kind-specific drop does more work than needed. The stated users are extension stats kinds, first pg_stat_statements in follow-up work.

**Proposed design.** pgstat_drop_matching_entries() gains a stats kind argument. PGSTAT_KIND_INVALID keeps the old all-kinds scan, and a specific kind scans only the hash mapped to that kind. When that hash is the shared variable stats hash, entries of other kinds are skipped before the match callback runs or the NULL-callback drop-all applies.

**User-visible surface.**
- C API: pgstat_drop_matching_entries() takes a new PgStat_Kind argument; existing extension callers must change their call.
- C API: PGSTAT_KIND_INVALID passed as kind means all kinds.
- Test module test_custom_stats gains SQL test functions for kind-filtered drops.

**What the author claims.**
- Callers that know their kind avoid scanning unrelated hash tables.
- Callers no longer need to repeat the kind check in their match callbacks.
- The ABI break is acceptable on master, and updating an extension call is simple.
- pg_stat_statements follow-up work will use the kind filter.

**Thread state.** Bertrand Drouvot and Michael Paquier reviewed it, and the design moved from a separate _of_kind function to one function with a kind argument. Paquier committed a related test fix as 8e225d2ac57d and wrote in message #18 that he committed v6 with two comment tweaks, though the CommitFest entry still says Needs review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-27; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | pgstat_drop_matching_entries() in pgstat_internal.h gains a PgStat_Kind argument, so extension callers must change their call. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message says the signature change breaks ABI, so it fits master only. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does any known extension call pgstat_drop_matching_entries() today, and should the release notes mention the new argument?
2. What match callback will the pg_stat_statements follow-up pass, and does it need anything beyond the kind filter?
3. Is there a measurement of the time saved when a kind with its own dshash skips the shared variable stats hash?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier reviewed the patch, chose the single-function design, and writes in message #18 that he committed v6.
- **Liability:** One internal function signature in pgstat_shmem.c and its kind filter on the shared variable stats hash; small, and limited to master.
- **Author follow-up:** The author answered review within a day and produced six versions; a pg_stat_statements follow-up is announced.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 10 KiB, touches one backend file plus a test module, and serves authors of custom stats kinds. Message #18 reports it committed, so the CommitFest entry, still marked Needs review, can be updated.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at this idea, and the thread points to no earlier proposal. The only earlier work the thread names is 44539786b82, which this patch builds on, and the lost injection_points test (message #9, #11), which was restored as 8e225d2ac57d.
- standards-compat: 0 objection(s). From the standards and compatibility angle, there is nothing to object to. The change has no SQL syntax, GUC, pg_dump, psql or client-visible effect. The only contract change is the signature of pgstat_drop_matching_entries() in pgstat_internal.h, on master only, and old callers fail at compile time rather than misbehaving at run time. The use of PGSTAT_KIND_INVALID to mean all kinds was raised in message #9 and settled in message #11. Message #18 reports the patch as committed.
- replication-ondisk: 0 objection(s). The proposal changes no persistent or cross-node state: no stats file format, WAL record, catalog, or wire protocol change, and it only narrows which shared-memory dshash an in-memory drop scans. The signature change to pgstat_drop_matching_entries() in pgstat_internal.h targets master only, and old callers fail at compile time, so the break is visible to extension authors.
- security: 0 objection(s). The proposal changes an internal C function in pgstat_internal.h that only server code and loaded extensions can call. It adds no SQL-callable surface outside the test module and no new privilege, view or GUC. It does not change what any role can see or do. The kind filter only narrows which entries a drop can touch, so it lowers the chance of dropping other kinds' entries. Exposure is unchanged.
- maintenance-fit: 0 objection(s). From the maintenance angle there is no objection a committer would send. The change only adds a kind argument to an existing internal function, so the hot path is untouched and no per-kind registration point is added. The area committer reviewed the change, chose the single-function design in message #4, and reports committing v6 in message #18. Two points cannot be checked from the material: no numbers show the saving from the skipped scans, and the pg_stat_statements caller is not shown. Neither one supports a concrete failure scenario.
- already-exists: 0 objection(s). The existing idiom is pgstat_drop_matching_entries() with a callback that checks entry->key.kind, as the old injection_points test did. That idiom still scans every variable-numbered hash, including the shared hash with all relation entries, and no extension or hook can avoid that scan outside pgstat_shmem.c. The patch extends the existing function rather than adding a second mechanism, so this angle has no objection. The thread shows no performance numbers, and the brief already notes that.
- demand: 0 objection(s). Demand is modest but real. The removed injection_points test filtered by kind in its callback (message #11), and pg_stat_statements is the named next caller. The cost is one internal signature change on master, borne by the few kind-aware extensions, and the area committer reviewed and committed it (message #18).

### Precedents retrieved

- P1: [WAL-logging facility for pgstats kinds](https://commitfest.postgresql.org/patch/5466/) (cf, Withdrawn, 2025-02-12) - judged not relevant; reason summarised from its thread
- P2: [The Prepare statement with the skewed parameter uses the custom plan](https://commitfest.postgresql.org/patch/4839/) (cf, Withdrawn, 2024-09-22) - judged not relevant; reason summarised from its thread
- P3: [Revert "Rename routines for write/read of pgstats file"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=381b3cfe2be02aa114ef7a28c4ec24feaf377c63) (revert, Reverted, 2026-07-14) - judged not relevant
- P4: [Move all am-related reloption code into src/backend/access/\[am-name\] and get rid of relopt_kind for custom AM](https://commitfest.postgresql.org/patch/992/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P5: [\[PG19\]pg_dump --statistics restores extended statistics by attribute number](https://commitfest.postgresql.org/patch/7446/) (cf, Withdrawn, 2026-10-19) - judged not relevant; reason summarised from its thread
- P6: [Allow index AMs to calculate custom statistics / BRIN](https://commitfest.postgresql.org/patch/3952/) (cf, Returned with feedback, 2022-12-17) - judged not relevant; reason summarised from its thread
- P7: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P8: [add the last checkpoint kind to the control file](https://commitfest.postgresql.org/patch/3493/) (cf, Withdrawn, 2022-02-03) - judged not relevant; reason summarised from its thread
- P9: [get rid of StdRdOptions, use individual binary reloptions representation for each relation kind instead](https://commitfest.postgresql.org/patch/1536/) (cf, Returned with feedback, 2019-09-27) - judged not relevant
- P10: [Revert "Improve accounting for memory used by shared hash tables"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aff1dc8dfa7584ddbc695e1aae0f767db66326f) (revert, Reverted, 2025-04-04) - judged not relevant
- P11: [Pluggable toaster](https://commitfest.postgresql.org/patch/3490/) (cf, Rejected, 2023-04-03) - judged not relevant; reason summarised from its thread
- P12: [Add option --drop-cascade for pg_dump/restore](https://commitfest.postgresql.org/patch/3065/) (cf, Returned with feedback, 2021-12-01) - judged not relevant; reason summarised from its thread
- P13: [Flush pgstats file during checkpoints](https://commitfest.postgresql.org/patch/5047/) (cf, Returned with feedback, 2024-10-02) - judged not relevant
- P14: [Some refactoring for fixed-numbered stats template in injection_points](https://commitfest.postgresql.org/patch/5187/) (cf, Withdrawn, 2024-10-02) - judged not relevant
- P15: [Revert "pg_stat_statements: Add coverage for entry_dealloc()"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1141e29b61e9bf82b045540d41b27581fc3207ec) (revert, Reverted, 2023-12-31) - judged not relevant
- P16: [pg_hba_lookup function to get all matching pg_hba.conf entries](https://commitfest.postgresql.org/patch/356/) (cf, Returned with feedback, 2016-04-08) - judged not relevant
- P17: [Use enum type replacing #define for exec state of hashjoin and mergejoin](https://commitfest.postgresql.org/patch/6027/) (cf, Withdrawn, 2026-03-30) - judged not relevant
- P18: [Prevent incorrect entries in pg_stat_activity.](https://commitfest.postgresql.org/patch/5105/) (cf, Returned with feedback, 2025-10-17) - judged not relevant
- P19: [pg_stat_statements with plans (v02)](https://commitfest.postgresql.org/patch/1470/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P20: [Sampling in pg_stat_statements](https://commitfest.postgresql.org/patch/5390/) (cf, Returned with feedback, 2025-04-09) - judged not relevant
- P21: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - judged not relevant
- P22: [DROP relation IF EXISTS Docs and Tests - Bug Fix](https://commitfest.postgresql.org/patch/2603/) (cf, Withdrawn, 2021-08-11) - judged not relevant
- P23: [Memory accounting towards memory-bounded HashAgg](https://commitfest.postgresql.org/patch/59/) (cf, Rejected, 2015-01-16) - judged not relevant
- P24: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 95 s; cost reported by the CLI: $1.08
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:08:14.954Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
