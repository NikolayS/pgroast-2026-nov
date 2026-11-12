# Design pre-review: teach vacuumlo to handle domains

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6765](https://commitfest.postgresql.org/patch/6765/) (Needs review); authors: Sami Imseih (simseih)
- Thread: [Vacuumlo improvements](https://www.postgresql.org/message-id/flat/CALsgZNAM%3DAYK-9ZLR7Z0YLx6Lyx5aSrjjse3T%2BFmsJ%3DjTvfhDQ%40mail.gmail.com), started 2026-05-12
- Messages read: 7 of 7
- Patch prose read: vacuumlo_repro.txt (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal: message #1 reports field data loss with a repro, and the change makes vacuumlo delete less. The two surviving objections are fixable design points. maintenance-fit-1 asks to split the domain fix from composite and array support, and maintenance-fit-2 asks for a warning on type kinds vacuumlo still cannot search. Neither stops review at level 2.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check that the generated expressions quote identifiers and cover nested cases correctly, or whether v2 adds tests. Messages #4 to #7 were not shown, so the scope of v2 rests on the brief.

## The proposal, restated

**Problem.** vacuumlo only scans columns whose type is named oid or lo, so it treats large objects referenced from a domain over oid or lo as orphaned and deletes them. The docs mention this, but users still lose data during routine maintenance. vacuumlo gives no warning, and -n lists these objects as removable without saying why.

**Proposed design.** Patch 0001 makes vacuumlo search the catalog recursively for oid or lo inside domains, including domains over domains, composite types and arrays. The catalog query returns a matching search expression for each column, which vacuumlo then uses in its delete statement. Patch 0002 documents the data loss risk for column types vacuumlo does not recognize and points users to the lo extension.

**User-visible surface.**
- vacuumlo now keeps large objects referenced from domain, composite and array columns that it used to remove.
- vacuumlo documentation gains a data loss warning for column types it does not recognize, such as text.

**What the author claims.**
- Domains over base types are common, and users have hit this data loss in the field.
- Everything vacuumlo needs is in the catalog, so it can cover all ways a column can hold an oid or lo.
- The change may help only somewhat, because some users store large object references in text or other types.
- Most users should use lo_manage() or custom triggers from the lo extension, which would make vacuumlo a cleanup tool for legacy schemas.

**Thread state.** Nathan Bossart sent v1 for domains, and Sami Imseih sent v2, which also covers composite and array types and adds docs. No one has reviewed v2 on the visible thread, and the entry has moved through 4 CommitFests in Needs review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-05-12; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Domain fix and composite/array expansion are bundled in one patch

fixable · level 2, rubric item 16 · maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-1`

A user on a released branch keeps a domain over oid. vacuumlo deletes those large objects, and v2 can only reach that user if a committer agrees to back-patch per-column delete expressions for composites and arrays. If no committer will take that on, the domain fix waits as well. Meanwhile the data loss continues on every supported branch.

v1 (message #4) was a small recursive query over typbasetype, which is a plausible back-patch candidate for a data loss bug. v2 (message #7) merges that fix with generated search expressions for composite and array types in one patch, 0001. Nobody has reviewed v2 on the visible thread across 4 CommitFests. The larger change is harder to review, test and back-patch, and few committers work on vacuumlo (message #2).

Thread: [#2 (Nathan Bossart, 2026-05-13)](https://www.postgresql.org/message-id/agPpPyLs-61qFw88%40nathan), [#4 (Nathan Bossart, 2026-05-14)](https://www.postgresql.org/message-id/agU9JPim3XNwRJ4b%40nathan), [#7 (Sami Imseih, 2026-06-02)](https://www.postgresql.org/message-id/CAA5RZ0ufaaZmJP-JLHBz07QXhyfAGPY%2BMOsbk0-QQ3Mn8kH7Qw%40mail.gmail.com)

**Question:** Can the domain-over-oid fix be split out as a standalone first patch suitable for back-patching, with composite and array support in a separate patch?

Counter-check: held. Message #7 puts domain, composite and array handling in one patch, 0001, and the visible thread has no review of v2. The back-patch part is speculative because the behavior is documented, but the request to split the patch stands without it.

### 2. Type kinds that are not covered are still skipped silently

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A user stores large object OIDs in a range or multirange over oid, or in a container kind added in a later release. vacuumlo removes those objects without a message. The user finds out when lo_get() fails.

Message #5 says the approach can "cover all potential ways an OID or LO can be used". The brief lists only domains, composites and arrays, so ranges and multiranges appear to be left out. A future container kind would also be missed, and nothing would fail loudly when that happens. Message #1 asked for a warning about skipped columns, and the brief does not say v2 adds one. Without that warning, coverage depends on someone remembering to update vacuumlo.

Thread: [#1 (Shawn McCoy, 2026-05-12)](https://www.postgresql.org/message-id/CALsgZNAM%3DAYK-9ZLR7Z0YLx6Lyx5aSrjjse3T%2BFmsJ%3DjTvfhDQ%40mail.gmail.com), [#5 (Sami Imseih, 2026-05-15)](https://www.postgresql.org/message-id/CAA5RZ0sF_p-ZE6LdVgzPr%3D9C32i2p1Pu7uxAPCV22-qh4tFh_Q%40mail.gmail.com)

**Question:** Does vacuumlo detect columns where oid or lo is nested in a type kind it cannot search, and warn about them, so that a missing case does not cause silent deletion?

Counter-check: held. Postgres allows a range over oid, and the material lists only domains, composites and arrays. The silent skip exists today and 0002 documents the risk, but the warning that message #1 suggested is not shown in v2.

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
| security / privileges | **yes** | vacuumlo, often run as superuser, builds delete statements from type and attribute names read from the catalog. |
| backpatchability | **yes** | The patch fixes data loss on released branches, and the thread does not settle whether it is back-patched. |
| maintenance burden ("who maintains this in 10 years") | **yes** | vacuumlo gains a recursive catalog query and generated per-column search expressions that must follow new type kinds. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can the domain fix be a standalone first patch, with composite and array support in a second patch?
2. Does vacuumlo warn when a column nests oid or lo in a type kind it cannot search, such as a range or multirange?
3. Is the domain fix proposed for back-patching, given that the current behavior is documented?
4. Does the recursive catalog query run when a new vacuumlo connects to an older supported server?
5. How much longer does vacuumlo run on a database with many tables once it uses the deeper scan and the generated delete expressions?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows contrib client tools and the type catalog. Nathan Bossart sent v1 and is listed as reviewer, but the material does not show his role, and no one has reviewed v2.
- **Liability:** The owner signs for vacuumlo's delete decisions: a wrong generated search expression deletes referenced large objects, and that loss is irreversible. The type coverage must be kept current as new type kinds appear.
- **Author follow-up:** Sami Imseih sent v2 and moved the entry to the next CommitFest twice, most recently on 2026-11-10.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patch fixes irreversible data loss for vacuumlo users who keep large object references in domain columns, and the cost falls only on vacuumlo. Review cost is medium because no code was read and v2 generates SQL; splitting out the domain fix would lower it.

## Appendix

### Refuted in the counter-check

- `demand-1` Composite and array coverage goes beyond the demand that was reported. Refuted (no failure scenario): The scenario describes review effort, which is not a failure, and the cost falls only on vacuumlo. Composite and array columns have the same silent deletion, Nathan Bossart called the extension "worth exploring" in message #6, and maintenance-fit-1 covers the split.

### Review angles

- prior-attempts: 0 objection(s). No precedent found in the list. Message #2 reports that commit 64c604898e and its thread only documented the domain limitation and gave no reason against resolving domains, so no earlier rejection reason applies here.
- standards-compat: 0 objection(s). From the standards and compatibility angle I found no objection that would hold. The patch adds no syntax, no GUC, no catalog object and no dump state. The behaviour change goes in the safe direction: vacuumlo keeps large objects it used to delete, so no script loses data after the upgrade. I could not check four things without the code. First, whether the recursive catalog query still runs when a new vacuumlo connects to an older server. Second, whether the docs drop the sentence 'domains over them are not considered'. Third, whether range types over oid or lo are covered or listed as not handled. Fourth, whether the change is back-patched.
- replication-ondisk: 0 objection(s). vacuumlo is a client program that reads pg_type and pg_attribute and calls lo_unlink through ordinary SQL. The patches change no on-disk format, WAL record, catalog, wire protocol message or server ABI, so this angle finds no persistent or cross-node state to object to.
- security: 0 objection(s). The proposal does not change who may run vacuumlo or what it may delete; it only makes vacuumlo keep more large objects, which lowers data loss risk. Reading domain, composite and array columns runs no user-defined code, so a superuser-run vacuumlo gains no new escalation path. Without the code I could not check that generated expressions quote type and attribute names and schema-qualify any functions they call.
- maintenance-fit: 2 objection(s). The idea fits contrib and fixes data loss. The added cost falls only on vacuumlo runs over schemas that use these types, so the objections concern patch shape and coverage.
- already-exists: 0 objection(s). No existing feature does this job. The lo extension's lo_manage() prevents new orphans but cannot stop vacuumlo from deleting objects referenced from domain, composite or array columns, and vacuumlo has no option to name extra columns. The proposal fixes an existing contrib tool and adds no second mechanism.
- demand: 1 objection(s). Demand for the domain case is real: message #1 reports field data loss and includes a repro, and two committers-level participants agree it is worth fixing. The only objection is about the extra scope beyond domains.

### Precedents retrieved

- P1: [vacuumlo: report the number of large objects going to be removed](https://commitfest.postgresql.org/patch/2148/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P2: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P3: [improve large object handling in pg_upgrade](https://commitfest.postgresql.org/patch/3841/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P4: [Revert: looks like Binary Large OBject\[sic\] wasn't a misspelling](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7b243aa666e49431f45bec71565d8ca49b7c9ef6) (revert, Reverted, 2010-08-17) - judged not relevant
- P5: [Identifying user-created objects](https://commitfest.postgresql.org/patch/2444/) (cf, Returned with feedback, 2020-07-30) - judged not relevant; reason summarised from its thread
- P6: [Allow values to be cast to domain types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Revert: Avoid looping over all type cache entries in TypeCacheRelCallback()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8daa62a10c911c851f7e9ec5ef7b90cfd4b73212) (revert, Reverted, 2024-08-25) - judged not relevant
- P8: [Incorrect handling of OOM in WAL replay leading to data loss](https://commitfest.postgresql.org/patch/4590/) (cf, Withdrawn, 2023-10-24) - judged not relevant; reason summarised from its thread
- P9: [Allow functions defined as casts to domains to be called during casting](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P10: [Make domains work better with polymorphic functions](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P11: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1133/) (cf, Returned with feedback, 2018-03-30) - judged not relevant; reason summarised from its thread
- P12: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant; reason summarised from its thread
- P13: [SimpleLruTruncate() mutual exclusion (data loss from lack thereof)](https://commitfest.postgresql.org/patch/2025/) (cf, Withdrawn, 2020-01-25) - judged not relevant; reason summarised from its thread
- P14: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1666/) (cf, Returned with feedback, 2018-12-01) - judged not relevant
- P15: [quick and dirty fix: Quadratic planning time for ordered paths over partitioned tables](https://commitfest.postgresql.org/patch/5516/) (cf, Rejected, 2025-02-03) - judged not relevant
- P16: [loop over unbound cursor](https://commitfest.postgresql.org/patch/2558/) (cf, Withdrawn, 2020-08-10) - judged not relevant
- P17: [Extensible storage manager API - smgr hooks](https://commitfest.postgresql.org/patch/3216/) (cf, Returned with feedback, 2022-03-22) - judged not relevant
- P18: [iteration over record in plpgsql](https://commitfest.postgresql.org/patch/124/) (cf, Returned with feedback, 2015-07-07) - judged not relevant
- P19: [Convert contrib SQL functions to new style](https://commitfest.postgresql.org/patch/3074/) (cf, Returned with feedback, 2021-09-01) - judged not relevant
- P20: [Enable fine-grained control over what gets logged on connection attempt (reduces log size)](https://commitfest.postgresql.org/patch/4005/) (cf, Returned with feedback, 2023-08-03) - judged not relevant
- P21: [refactor AlterDomainAddConstraint (alter domain add constraint)](https://commitfest.postgresql.org/patch/5455/) (cf, Returned with feedback, 2026-04-02) - judged not relevant
- P22: [eXtensible Transaction Manager API (v2)](https://commitfest.postgresql.org/patch/512/) (cf, Returned with feedback, 2016-03-18) - judged not relevant
- P23: [orphaned function](https://commitfest.postgresql.org/patch/2850/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P24: [Anyarray contrib module](https://commitfest.postgresql.org/patch/145/) (cf, Rejected, 2015-04-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 134 s; cost reported by the CLI: $1.33
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 59 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T00:59:17.786Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
