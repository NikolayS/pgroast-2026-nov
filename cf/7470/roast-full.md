# Design pre-review: Use a bulk-write buffer access strategy for TOAST inserts

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7470](https://commitfest.postgresql.org/patch/7470/) (Needs review); authors: Bertrand Drouvot (bertranddrouvot.pg)
- Thread: [Use a bulk-write buffer access strategy for TOAST inserts](https://www.postgresql.org/message-id/flat/atHWFmxAz1KuKzqm%40bdtpg), started 2026-10-16
- Messages read: 2 of 2
- Patch prose read: v1-0001-Move-buffer-strategy-ownership-out-of-BulkInsertS.patch, v1-0002-Bind-BulkInsertState-to-one-relation.patch, v1-0003-Use-bulk-write-strategy-for-TOAST-inserts.patch, v1-0004-Use-bulk-write-strategy-for-TOAST-during-heap-rew.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

demand-1 is decisive: the thread shows one earlier report and no numbers for the eviction today or for the effect of the patch. In message #2 a committer who knows the buffer code says TOAST inserts should not use strategies, because later reads of the evicted TOAST data are random IO. That claim has no reply yet. The list has to agree that the change is wanted before the design of the four patches is reviewed.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so how the one-relation rule is enforced and how the TOAST state is cleaned up on error could not be checked. The design level was not evaluated, because the review stopped at level 1b.

## The proposal, restated

**Problem.** Bulk inserts use a BAS_BULKWRITE ring for the main relation but not for its TOAST relation, so large TOAST writes can evict useful shared buffers. This affects COPY, CREATE TABLE AS, materialized view refresh, table rewrites and VACUUM FULL. Justin Pryzby reported it in an earlier thread.

**Proposed design.** The caller, instead of BulkInsertState, now owns the buffer access strategy, and each BulkInsertState is bound to one relation; partitioned COPY keeps one state per leaf. Each BulkInsertState gets a second state for its TOAST relation that shares the same ring. Heap rewrites create a BAS_BULKWRITE ring and BulkInsertState for TOAST when they first need it.

**User-visible surface.**
- C API: GetBulkInsertState() takes a caller-provided strategy and a relation; the caller allocates and frees the strategy.
- C API: a BulkInsertState may only be used with the relation it was created for.
- pg_stat_io: TOAST writes from bulk inserts and VACUUM FULL now count in the bulkwrite context.

**What the author claims.**
- TOAST data from bulk operations can evict useful shared buffers today.
- Binding BulkInsertState to one relation removes a source of bugs like those fixed in b1ecb9b3fcf and 22655aa2313.
- 0002 implements the medium-term fix Andres suggested in 22655aa2313.
- TOAST relations skip bulk extension ranges to avoid reserving unused space in small TOAST tables.
- The read path is out of scope because TOAST reads can happen outside the producing scan.

**Thread state.** Andres Freund objects to using strategies for TOAST inserts because evicting just-toasted data causes costly random IO later. He suggests that tableam create the bulk state and reuse it across all DML; no reply yet, and CFbot reports the patch needs a rebase.

## 2. Prior attempts

### INSERT SELECT: BulkInsertState and table_multi_insert (related)

- **Link:** [https://commitfest.postgresql.org/patch/2553/](https://commitfest.postgresql.org/patch/2553/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20200508072545.GA9701@telsasoft.com](https://www.postgresql.org/message-id/flat/20200508072545.GA9701@telsasoft.com)
- **Date:** decided 2021-11-24; started 2020-05-08
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The thread stalled. The multi-insert part became complex: it cannot handle volatile column defaults, and the author proposed leaving it to the separate "New Table Access Methods for Multi and Single Inserts" patch. The author split off the simple BulkInsertState patch and said to close the entry if no one was interested in it alone. No reviewer took that patch up, so the commitfest manager proposed closing it.
- **Reason (quote):** "Since the thread has stalled, maybe that's the best course of action here." ([message](https://www.postgresql.org/message-id/AF742DB8-2992-4826-B85B-1FC36BBEF3A9%40yesql.se))
- **What is different this time:** P2 extended BulkInsertState rings to INSERT and INSERT SELECT. This proposal adds a ring for TOAST inserts in paths that already use a bulk state.


## 3. Objections a reviewer is likely to raise

### 1. No measurement shows the TOAST eviction problem in practice

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer weighs a four-patch series that changes GetBulkInsertState() for every caller. The only evidence of harm is one 2022 report with no numbers. If the eviction is rare or small, the project takes on the API churn and the new strategy path for little gain.

Message #1 cites a single earlier report from Justin Pryzby and gives no workload, buffer-hit figures or pg_stat_io numbers. No other user in the thread asks for the change. Using BAS_BULKWRITE for the main relation is established practice, so the idea is plausible. But the gain for TOAST-heavy loads has not been shown against a workload where it matters.

Thread: [#1 (Bertrand Drouvot, 2026-10-16)](https://www.postgresql.org/message-id/atHWFmxAz1KuKzqm%40bdtpg)

**Question:** On a realistic COPY or VACUUM FULL of TOAST-heavy data, how much of shared_buffers is evicted today, and how much does the patch change it?

Counter-check: held. The material shows one earlier report and no numbers for the eviction or for the patch. The main relation already uses the ring, which makes the idea plausible, but message #2 disputes that this carries over to TOAST.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0001 and 0002 change the GetBulkInsertState() signature in heapam.h and bind each state to one relation. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry is tagged Bugfix, but the series changes exported functions and targets version 20. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each BulkInsertState gains a TOAST state, and heap rewrites create a ring and state for TOAST on first use. |
| performance on the hot path | **yes** | TOAST writes in COPY, CREATE TABLE AS, matview refresh, table rewrites and VACUUM FULL move to a small ring, with no switch. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No measurement shows the TOAST eviction problem in practice |

## 5. Questions the author must answer before anyone reviews code

1. On a COPY or VACUUM FULL of TOAST-heavy data, how much of shared_buffers is evicted today, and how much does the patch change that?
2. For a load-then-query workload, do the later random reads of TOAST data cost more than the cache the ring saves for other sessions?
3. Are 0001 and 0002 a step toward the tableam-owned bulk state proposed in message #2, or would that design undo this API change?
4. Is the series meant for master only, given that the CommitFest entry is tagged Bugfix?
5. How must out-of-core table AMs and extensions that call GetBulkInsertState() adapt to the caller-owned strategy and the one-relation rule?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund takes part in message #2 and disagrees with both the TOAST ring and the API direction. The owner must know buffer access strategies and the tableam insert API.
- **Liability:** The owner signs for the read performance of bulk-loaded TOAST data in every release from 20 on, and for a changed GetBulkInsertState() contract that extensions must follow.
- **Author follow-up:** The author plans follow-up work on the read path once this series goes in. There is no reply to message #2 yet, and the patch needs a rebase.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A benchmark of a TOAST-heavy bulk load that shows the eviction of other sessions' buffers today, the effect of the patch, and the cost of reading the loaded TOAST data afterwards. More than one field report of the eviction would also help.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Bulk loads of wide values are common, but the benefit is not measured and message #2 disputes it. The four patches, about 60 KiB, touch heapam, COPY, the executor and TOAST code, and message #2 prefers a tableam-owned bulk state.

## Appendix

### Held up in the counter-check, not listed above

- `prior-attempts-1` (fixable, level 2) API reshaping may conflict with a tableam-owned bulk state. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) TOAST ring changes every bulk load, with no measurement or opt-out. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-1` GetBulkInsertState() contract changes twice if tableam direction follows. Refuted (not specific): Duplicate of prior-attempts-1. The extra point about enforcement is a code-level question, and the 0002 commit message says the patch ensures the state is used only with its relation.
- `standards-compat-2` Bugfix tag conflicts with an API change that cannot be backpatched. Refuted (already handled): The CommitFest entry sets target version 20, and no message proposes a backpatch. The scenario depends on a backpatch nobody asked for, so the tag is a labelling matter.
- `maintenance-fit-1` API change conflicts with the tableam-owned bulk state direction. Refuted (not specific): Duplicate of prior-attempts-1. The added backpatch remark fails for the same reason as standards-compat-2, because the entry targets version 20.
- `maintenance-fit-3` Heap rewrites would mix two write mechanisms in one operation. Refuted (wrong facts): Heap rewrites already use two write paths today. The main relation goes through BulkWriteState, and TOAST chunks go through heap_insert and shared buffers because the TOAST index must be maintained. 0004 only adds a ring to the existing TOAST path.
- `already-exists-1` Rewrite TOAST path adds a buffer ring next to existing bulk_write. Refuted (wrong facts): The rewrite TOAST path already writes through shared buffers with its own WAL logging, separate from BulkWriteState. 0004 does not create the split in write paths that the scenario describes.
- `demand-2` The gain may be offset by costlier later reads of TOAST data. Refuted (not specific): Duplicate of maintenance-fit-2, which raises the same load-then-query cost from message #2 and asks for the same benchmark.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent was rejected for a reason that applies to TOAST ring use. Andres's random-IO objection in message #2 is new in this thread, not a prior rejection, so other angles cover it.
- standards-compat: 2 objection(s). There is no SQL syntax, GUC, catalog or pg_dump surface. The only change users see is TOAST bulk writes moving into the bulkwrite context of pg_stat_io, which is more accurate accounting.
- replication-ondisk: 0 objection(s). The series changes how shared buffers are reused and adds no on-disk format, WAL record, catalog or wire protocol change, so pg_upgrade, standbys and subscribers see nothing new. The only cross-boundary item is the C API change to GetBulkInsertState(); it targets version 20 and breaks callers at compile time. That is normal for master and leaves no persistent or cross-node state to object to. If the Bugfix tag means a backpatch, a committer would likely ask how this API change fits back branches.
- security: 0 objection(s). The proposal has no privilege or exposure aspect: no new privilege, role, GUC, SQL surface or monitoring data. It changes only which buffer ring TOAST writes from existing bulk operations use, and the pg_stat_io change moves existing counts to another context without showing anything new.
- maintenance-fit: 3 objection(s). Objections are listed. P2 is related only: it extended BulkInsertState to INSERT and stalled for lack of review, which does not apply here.
- already-exists: 1 objection(s). For COPY, CREATE TABLE AS and matview refresh, no existing option, GUC or extension keeps TOAST inserts out of shared buffers, so the main idea is not already implemented.
- demand: 2 objection(s). Demand rests on one 2022 report with no numbers; the objections ask for measurements, not rejection.

### Precedents retrieved

- P1: [Eagerly evict bulkwrite strategy ring](https://commitfest.postgresql.org/patch/5919/) (cf, Withdrawn, 2026-02-19) - judged not relevant; reason summarised from its thread
- P2: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - used in the review; reason summarised from its thread
- P3: [Refuse setting toast.* reloptions when TOAST relation does not exist](https://commitfest.postgresql.org/patch/1486/) (cf, Returned with feedback, 2018-04-10) - judged not relevant; reason summarised from its thread
- P4: [Vectored I/O in bulk_write.c](https://commitfest.postgresql.org/patch/4891/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P5: [Revert fix missing call to table_finish_bulk_insert during COPY](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5db56fc4d6e95c582b61c99328ea0702b869fa0) (revert, Reverted, 2019-07-01) - judged not relevant
- P6: [Support for 8-byte TOAST values (aka the TOAST infinite loop problem)](https://commitfest.postgresql.org/patch/5830/) (cf, Returned with feedback, 2026-02-10) - judged not relevant; reason summarised from its thread
- P7: [Add MAIN_RELATION_CLEANUP and TOAST_TABLE_CLEANUP options to VACUUM](https://commitfest.postgresql.org/patch/2426/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P8: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P9: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P10: [Implement TODOs for index operator strategy number validation](https://commitfest.postgresql.org/patch/7152/) (cf, Withdrawn, 2026-09-09) - judged not relevant; reason summarised from its thread
- P11: [New strategies for freezing, advancing relfrozenxid early](https://commitfest.postgresql.org/patch/3843/) (cf, Withdrawn, 2023-01-26) - judged not relevant
- P12: [Revert the default toast compression strategy to the former behavior where there's no limit on the size of datum we'll try to compress.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=229bffedbc52f12aadb95ca2ee24d5bef36b7c7d) (revert, Reverted, 2009-01-06) - judged not relevant
- P13: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P14: [Auto-fill the free space map by scanning the buffer cache or by checking pages written by the background writer](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Vacuum) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P15: [Reuse data from readRecordBuf in XLogDecodeNextRecord](https://commitfest.postgresql.org/patch/6343/) (cf, Withdrawn, 2026-02-09) - judged not relevant
- P16: [\[doc\] vacuum full, fillfactor, and "extra space"](https://commitfest.postgresql.org/patch/2412/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P17: [Auto-tune shared_buffers to use available huge pages](https://commitfest.postgresql.org/patch/6427/) (cf, Withdrawn, 2026-06-24) - judged not relevant
- P18: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P19: [Persist injection points across server restarts](https://commitfest.postgresql.org/patch/5731/) (cf, Returned with feedback, 2025-07-02) - judged not relevant
- P20: [Push aggregation down to base relations and joins](https://commitfest.postgresql.org/patch/1247/) (cf, Withdrawn, 2021-03-11) - judged not relevant
- P21: [Push aggregation down to base relations and joins](https://commitfest.postgresql.org/patch/3764/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P22: [Per-table storage parameters for TableAM/IndexAM extensions](https://commitfest.postgresql.org/patch/3495/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P23: [Small typo fix on tableam documentation](https://commitfest.postgresql.org/patch/2083/) (cf, Rejected, 2019-04-05) - judged not relevant
- P24: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 115 s; cost reported by the CLI: $1.22
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 95 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:58:46.847Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
