# Design pre-review: Halve memory allocation for toast_flatten_tuple

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7362](https://commitfest.postgresql.org/patch/7362/) (Needs review); authors: Pavel Borisov (pborisov)
- Thread: [\[PATCH\] Halve peak memory allocation for toast_flatten_tuple](https://www.postgresql.org/message-id/flat/CALT9ZEG8W0-xVCYQmxebYQ55fE-SJmKOTrBqgMbcVekhGNMaAw%40mail.gmail.com), started 2026-09-21
- Messages read: 3 of 3
- Patch prose read: v2-0001-Halve-peak-memory-allocation-for-toast_flatten_tu.patch, v1-0001-Halve-peak-memory-allocation-for-toast_flatten_tu.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and no level fails. The problem is stated, and message #2 independently confirms the memory cut. Two fixable design points remain: the stub header must predict what detoast_external_attr() returns (maintenance-fit-1), and every heap_form_tuple() caller now takes the _attr path without CPU numbers (maintenance-fit-2). demand-1 is open but does not block review, because the thread shows no field report either way.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm that the stub header always matches the detoasted value, or that a mismatch raises an error before the write. The compiled cost of the extra arguments in the shared fill path is also unknown.

## The proposal, restated

**Problem.** toast_flatten_tuple() holds every detoasted external value and the result tuple at once, so peak memory is about 2x the tuple. The author says this mainly hurts logical replication with REPLICA IDENTITY FULL on tables with several large toasted columns, where every UPDATE/DELETE flattens the old tuple.

**Proposed design.** The patch computes the tuple layout first, passing a header-only stub per external attribute through a new heap_form_tuple_attr() with skip_copy=true, which records each attribute's destination. It then detoasts one attribute at a time straight into its slot and frees it. INDIRECT externals are still detoasted inline as before.

**User-visible surface.**
- New C function heap_form_tuple_attr() with skip_copy and attr_data_out arguments; heap_form_tuple() becomes a wrapper.
- New _attr variants of fill_val() and heap_fill_tuple() in heaptuple.c, declared through htup_details.h.

**What the author claims.**
- Peak memory drops from 2x the tuple to 1x the tuple plus one detoasted value.
- The result tuple is byte-identical to the old code path, including short-header varlena packing.
- The extra per-attribute cost is negligible next to the toast-fetch I/O.
- Other callers, catcache loading and PL/pgSQL record assignment, handle small tuples where the difference is negligible.

**Thread state.** The reviewer, Johannes Edmeier, measured a 25 to 44% cut inside the function and up to 28% in backend peak RSS, and suggested heap_form_tuple_attr(); v2 adopts it. CFbot reports that the patch needs a rebase, and nobody has commented on v2.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-21; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No field report shows this memory peak hurts anyone

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer reworks heap_form_tuple(), fill_val() and heap_fill_tuple() into _attr variants for a narrow workload: REPLICA IDENTITY FULL tables with several large toasted columns. Users of that workload saw no failure before. Everyone else carries the reshaped core tuple-forming code and its new arguments in later patches and back-patches.

The only numbers come from one synthetic single-row test in message #2, and none come from a production system. Message #1 calls the 2x peak a strain but cites no out-of-memory report, -bugs thread or user complaint. The gain needs two or more large external columns: with one column the cut is 0%, and backend RSS drops 6 to 19% at a 96 MB payload. The cost lands in heaptuple.c, which every heap_form_tuple() caller uses, so non-users carry the maintenance.

Thread: [#1 (Pavel Borisov, 2026-09-21)](https://www.postgresql.org/message-id/CALT9ZEG8W0-xVCYQmxebYQ55fE-SJmKOTrBqgMbcVekhGNMaAw%40mail.gmail.com), [#2 (Johannes Edmeier, 2026-10-15)](https://www.postgresql.org/message-id/CAHYEV2tW%2BogNGXC7tnVL17Q6J26S-2R7%2BvBCkFEsd_VTwUiWiA%40mail.gmail.com)

**Question:** Which production systems using REPLICA IDENTITY FULL hit memory limits or OOM during toast_flatten_tuple(), and how large were their old tuples?

Counter-check: held. The independent measurements in message #2 confirm the 2x peak and a 28% RSS cut, but the thread cites no production case or user report. The statement that users "saw no failure before" is not in the material and must not be used as support.

### 2. Stub headers must track detoast output with no loud failure

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a compression method or a new external varlena kind and changes what detoast_external_attr() returns. The stub logic in heaptoast.c is not updated, so the layout reserves a different size than the detoasted value needs. The write overruns the result tuple or leaves invalid data in it, and users find out from corrupted rows on a subscriber.

The design needs the stub header to predict the size and compression tag that detoast_external_attr() will return. That is a second copy of detoast format knowledge, kept in a different file. The commit message already treats INDIRECT as a case the stub cannot predict, which shows the prediction is fragile. The prose does not mention a runtime check that compares the detoasted length with the reserved space before the copy.

Thread: [#1 (Pavel Borisov, 2026-09-21)](https://www.postgresql.org/message-id/CALT9ZEG8W0-xVCYQmxebYQ55fE-SJmKOTrBqgMbcVekhGNMaAw%40mail.gmail.com), [#3 (Pavel Borisov, 2026-10-15)](https://www.postgresql.org/message-id/CALT9ZEEiDGAjG65B%2B15-RcHf1g3DN2uqf3K0SbTE5QgLstnwVw%40mail.gmail.com)

**Question:** Does v2 check, in production builds, that each detoasted value's length equals the reserved stub size and raise an error before it writes, not after?

Counter-check: held. The commit message says the stub header must match what detoast_external_attr() returns, and it excludes INDIRECT because the stub cannot predict that case. The prose mentions no runtime length check before the write, so the question stands.

### 3. Every heap_form_tuple() call now goes through the new _attr path

fixable · level 2, rubric item 8 · performance on the hot path, ABI / extension API · `maintenance-fit-2`

heap_form_tuple() and heap_fill_tuple() run for nearly every tuple that executor nodes, COPY and catalog code build. After v2, each call goes through the _attr variants, which take extra skip_copy and attr_data arguments. If those are not fully compiled away, a bulk INSERT ... SELECT or COPY into a wide table gets slower for users who never flatten a toasted tuple.

Measurements in message #2 cover only toast_flatten_tuple() memory, not the CPU cost of the shared fill path. The new arguments exist for one caller, but the cost of the shared path falls on all callers. A committer would likely ask for a micro-benchmark of heap_form_tuple() on many-column tuples before and after.

Thread: [#2 (Johannes Edmeier, 2026-10-15)](https://www.postgresql.org/message-id/CAHYEV2tW%2BogNGXC7tnVL17Q6J26S-2R7%2BvBCkFEsd_VTwUiWiA%40mail.gmail.com), [#3 (Pavel Borisov, 2026-10-15)](https://www.postgresql.org/message-id/CALT9ZEEiDGAjG65B%2B15-RcHf1g3DN2uqf3K0SbTE5QgLstnwVw%40mail.gmail.com)

**Question:** What is the measured cost of heap_form_tuple() and heap_fill_tuple() on a wide, many-attribute tuple before and after v2, for example in a COPY benchmark?

Counter-check: held. In v2 heap_form_tuple() and heap_fill_tuple() become wrappers around the _attr variants, so every caller takes the new path. The thread has memory numbers only, and the author's cost claim covers the flatten path, not the shared fill path.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The flattened old tuple for REPLICA IDENTITY FULL is written to WAL and decoded by logical replication. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | htup_details.h declares the new heap_form_tuple_attr() and the _attr variants of heap_fill_tuple() and fill_val(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The patch reshapes fill_val() and heap_fill_tuple() in heaptuple.c, so later fixes there apply differently to older branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The stub logic in heaptoast.c is a second copy of what detoast_external_attr() returns and must stay in step with it. |
| performance on the hot path | **yes** | heap_form_tuple() and heap_fill_tuple() become wrappers around the _attr variants, so every tuple-forming caller takes the new path. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No field report shows this memory peak hurts anyone |

## 5. Questions the author must answer before anyone reviews code

1. Does v2 check in production builds that each detoasted value's length equals the space reserved by its stub, and raise an error before it writes?
2. What does heap_form_tuple() cost on a wide, many-attribute tuple before and after v2, for example in a COPY benchmark?
3. Which production systems with REPLICA IDENTITY FULL hit memory limits during toast_flatten_tuple(), and how large were their old tuples?
4. What is the CPU cost of v2 when a tuple has many small external attributes, where the per-attribute stub work is not hidden by large toast fetches?
5. Is heap_form_tuple_attr() meant as an API for extensions, or as an internal helper for toast_flatten_tuple() only?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the heap tuple forming code in heaptuple.c and the TOAST code. The visible thread has the author and one reviewer, and no such committer has spoken.
- **Liability:** The owner signs for the correctness of every tuple formed through the shared fill path and for the old tuples that REPLICA IDENTITY FULL writes to WAL. The stub logic must be kept in step with detoast_external_attr() for as long as both exist.
- **Author follow-up:** The author posted v2 on the same day as the review in message #2. The CFbot rebase request of 2026-11-11 has no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The gain needs two or more large external columns and mainly helps REPLICA IDENTITY FULL tables, but message #2 measured up to 28% lower backend peak RSS. The patch is about 19 KiB, adds no user-visible surface, and changes shared code in heaptuple.c, so it needs a rebase and the two design answers.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The only listed precedent, [P2], changed how catalog code forms tuples for updates and converted nearly all call sites. Its recorded objection is the cost of a system-wide API change. This proposal adds heap_form_tuple_attr() and keeps heap_form_tuple() as a wrapper, so existing callers and extensions stay unchanged. No precedent found, and the thread points to no earlier discussion of the same idea.
- standards-compat: 0 objection(s). The patch changes no SQL syntax, GUC, catalog, dump output, psql behaviour or wire format, and heap_form_tuple() keeps its signature as a wrapper, so existing callers and extensions compile unchanged. The claimed byte-identical result tuple means nothing changes on disk or on the replication stream. [P2] does not fit because it rejected a system-wide API change, which this patch does not make. Whether heap_form_tuple_attr() should be exported to extensions is an internal API question for another angle.
- replication-ondisk: 0 objection(s). The patch changes only in-backend memory use during toast_flatten_tuple(), with no change to on-disk format, WAL record types, catalogs or protocol, and the author claims the result is byte-identical. The one persistent path is the REPLICA IDENTITY FULL old tuple written to WAL. A wrong stub length would corrupt that WAL data, but toast values are immutable and the commit message explains the compressed and INDIRECT cases, so I have no concrete failure scenario. heap_form_tuple() keeps its signature, so the new heap_form_tuple_attr() adds API without breaking extensions. Without the code I could not check one point: whether the per-attribute detoast length is checked against the stub size at runtime, not only by assertion.
- security: 0 objection(s). The proposal changes only how a backend allocates memory while flattening a tuple. It adds no privilege, no new exposure and no user-controlled input path, and toast pointers come from stored data, not from users. One point could not be checked without the code: whether the patch errors out cleanly when the fetched value's length differs from the stub size, for example with corrupted toast data. That is a safety question for the design and implementation reviewers, not a privilege issue.
- maintenance-fit: 2 objection(s). Two fixable design concerns only. The idea fits core, an independent reviewer measured the memory gain, and the API change adds functions without changing existing callers.
- already-exists: 0 objection(s). No existing feature, setting or extension gives this result. The patch lowers peak memory inside toast_flatten_tuple(), which only core code can change. REPLICA IDENTITY USING INDEX avoids flattening the old tuple, but only for tables with a suitable unique index, and FULL exists for the tables that lack one. heap_form_tuple_attr() extends the existing heap_form_tuple() path as a wrapper and does not add a second way to form tuples. [P2] concerns a catalog-wide API change and does not apply here.
- demand: 1 objection(s). One objection. The problem is stated and the reviewer measured the gain, but neither shows users who need it.

### Precedents retrieved

- P1: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - judged not relevant; reason summarised from its thread
- P2: [Refactor how we form HeapTuples for CatalogTuple(Insert|Update)](https://commitfest.postgresql.org/patch/6221/) (cf, Withdrawn, 2026-01-30) - used in the review; reason summarised from its thread
- P3: [Forcing new tuples of updates off the page](https://commitfest.postgresql.org/patch/4440/) (cf, Withdrawn, 2024-03-15) - judged not relevant; reason summarised from its thread
- P4: [Tidy fill hstv array (src/backend/access/heap/pruneheap.c)](https://commitfest.postgresql.org/patch/4734/) (cf, Returned with feedback, 2024-03-04) - judged not relevant; reason summarised from its thread
- P5: [Refuse setting toast.* reloptions when TOAST relation does not exist](https://commitfest.postgresql.org/patch/1486/) (cf, Returned with feedback, 2018-04-10) - judged not relevant; reason summarised from its thread
- P6: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant; reason summarised from its thread
- P7: [\[doc\] vacuum full, fillfactor, and "extra space"](https://commitfest.postgresql.org/patch/2412/) (cf, Returned with feedback, 2020-08-02) - judged not relevant; reason summarised from its thread
- P8: [Fillfactor for GIN indexes](https://commitfest.postgresql.org/patch/13/) (cf, Rejected, 2015-01-16) - judged not relevant; reason summarised from its thread
- P9: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - judged not relevant
- P10: [Prefetch the next tuple's memory during seqscans](https://commitfest.postgresql.org/patch/3978/) (cf, Withdrawn, 2024-01-22) - judged not relevant
- P11: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P12: [XLog size reductions: smaller XLogRecordBlockHeader](https://commitfest.postgresql.org/patch/4324/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P13: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P14: [Reducing Memory Consumption (aset and generation)](https://commitfest.postgresql.org/patch/3673/) (cf, Rejected, 2022-07-12) - judged not relevant
- P15: [Write Amplification Reduction Method (WARM)](https://commitfest.postgresql.org/patch/775/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P16: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P17: [explain to report bucket and memory stats for TupleHashTable nodes](https://commitfest.postgresql.org/patch/2428/) (cf, Returned with feedback, 2020-07-23) - judged not relevant
- P18: [propagating replica identity to partitions](https://commitfest.postgresql.org/patch/2044/) (cf, Returned with feedback, 2019-09-11) - judged not relevant
- P19: [Make transformAExprIn() return a flattened bool expression directly](https://commitfest.postgresql.org/patch/6723/) (cf, Withdrawn, 2026-07-11) - judged not relevant
- P20: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - judged not relevant
- P21: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P22: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant
- P23: [Fix TOCTOU race in ReplicationSlotsComputeRequiredLSN()](https://commitfest.postgresql.org/patch/6682/) (cf, Withdrawn, 2026-04-17) - judged not relevant
- P24: [Refactoring of heapam code](https://commitfest.postgresql.org/patch/700/) (cf, Returned with feedback, 2016-09-26) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 106 s; cost reported by the CLI: $1.04
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 59 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:45:03.655Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
