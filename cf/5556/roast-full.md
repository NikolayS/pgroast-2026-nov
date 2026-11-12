# Design pre-review: Expanding HOT updates for expression and partial indexes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5556](https://commitfest.postgresql.org/patch/5556/) (Needs review); authors: Greg Burd (gregburd)
- Thread: [Expanding HOT updates for expression and partial indexes](https://www.postgresql.org/message-id/flat/78574B24-BE0A-42C5-8075-3FA9FA63B8FC%40amazon.com), started 2025-02-06
- Messages read: 88 of 88
- Patch prose read: v15-0001-Expand-HOT-update-path-to-include-expression-and.patch, v3-0001-Expand-HOT-update-path-to-include-expression-and-.patch, v4-0001-Expand-HOT-update-path-to-include-expression-and-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal, and demand is shown. Three fixable points remain. demand-1: updates that change an expensive index expression get slower, and the v46 messages name no opt-out. replication-ondisk-1: HOT safety now depends on extension answers that nothing checks. prior-attempts-1: the thread does not state the root cause of the 2018 revert. None of these stops review, but a committer who knows heapam and the index AMs must judge them.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: No code and no v46 patch prose were read. Whether the executor-side check keeps HOT, lock mode and replica identity decisions identical to master, and how toasted values are compared, could not be judged.

## The proposal, restated

**Problem.** Any update to a column used in an index expression or partial-index predicate blocks HOT today, even when the indexed value stays the same. The author names JSONB users with expression indexes on document fields, plus ORMs and logical replication apply that write back unchanged values. Extensions such as DocumentDB with RUM indexes over whole documents never get HOT updates.

**Proposed design.** The executor, instead of heap_update() under the buffer lock, finds which indexed attributes an UPDATE really changed and passes that set to table_tuple_update(). A column used only through expressions or predicates no longer blocks HOT when each index's datums are unchanged and the row stays on the same side of the predicate. An optional index AM callback, a function support request and image comparison of toasted values narrow the set further.

**User-visible surface.**
- New optional index AM callback amcomparedatums, implemented for GIN and hash; never asked about a column the index can return.
- New function support request SupportRequestSubattr, with support for jsonb, arrays, records and extension types.
- table_tuple_update() and heap_update() take the set of modified indexed attributes; ExecInsertIndexTuples() gets a new argument for the indexUnchanged hint.
- New relcache bitmap INDEX_ATTR_BITMAP_INDEXED, the union of HOT_BLOCKING and SUMMARIZED.
- Early versions had a GUC enable_expression_checks, later a reloption expression_checks; the material does not say whether either is still in v46.

**What the author claims.**
- A jsonb update with two expression indexes went from 3600 ms to 788 ms, and from 0% to 78% HOT.
- Write-back of an unchanged compressed value under btree is about 92% faster; controls and pgbench stay within 3%.
- When an expensive index expression's value does change, UPDATEs run 33-44% slower because the old row's expression is evaluated again.
- The executor-side attribute check (0002) leaves HOT, lock mode and replica identity decisions identical to master, checked across all regression tests.
- DocumentDB updates of unindexed fields went from 0% to 100% HOT with an amcomparedatums for its RUM fork.

**Thread state.** Matthias van de Meent, Jeff Davis and Nathan Bossart reviewed; the v19 attempt was deferred in March 2026 because only the refactoring was ready. v46 (nine patches) has had no visible review; Matthias's last review was of v39, and Peter Geoghegan's view on a per-row indexUnchanged hint is not shown.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-02-06; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Users of costly expression indexes pay with no opt-out shown

fixable · level 1b, rubric item 4 · performance on the hot path, new GUC · `demand-1`

A site keeps a to_tsvector() expression index on a text column that changes on most updates. After upgrade, each UPDATE evaluates the expression an extra time for the old row and gets no HOT update in return. The site sees the slowdown in production and finds no reloption to turn the check off.

Demand is real: JSONB expression indexes, ORM and logical replication write-back, a reverted earlier attempt, and reviewer support (messages #2, #45). The cost lands elsewhere. Message #81 reports +33-44% for changing expensive expressions, and that figure assumed 0010 removed one evaluation. Messages #83-84 dropped 0010, so v46 may be back to the 55% of message #79. Laurenz Albe asked for a per-table switch (message #2), the early versions had one, and the v46 messages do not mention it.

Thread: [#2 (Laurenz Albe, 2025-02-09)](https://www.postgresql.org/message-id/0a47ce53fa635787d36325b5ad7bac94cdca4246.camel%40cybertec.at), [#79 (Greg Burd, 2026-10-17)](https://www.postgresql.org/message-id/-ugooulGn7M5X4DFIzWHSgYRngUmXChD_a-JvcMaklnDNcHMGnQhcM82SpYY76Je5ss4TOrsQB5sv2USLRSlDcpwDX-6CgCCbGKlJuA0SOs%3D%40protonmail.com), [#81 (Greg Burd, 2026-10-19)](https://www.postgresql.org/message-id/j-s6ilS5lyXpV_6l6Q10JTSiumDI5TBUf2zsOfUZAsB3-TjQCJgxWWVIuoCvDZrhD5ljcouHg1s_S_FfjjBS-97Min2Wwyvm5vFSIcdsz_o%3D%40burd.me), [#83 (Greg Burd, 2026-10-21)](https://www.postgresql.org/message-id/XtuZX8xWYFYMwfJ0BFIwteRjtst6uj1MVjGxiXNb8PuUJwkAUhvdQqeE1wbZPiYvmZ9G92o6CORDsp1vb3S8ci-lublEhHrFTil4GIYQBvU%3D%40burd.me), [#84 (Greg Burd, 2026-10-22)](https://www.postgresql.org/message-id/atpXQaH1UBS3LY-6%40floki), [#88 (Greg Burd, 2026-11-11)](https://www.postgresql.org/message-id/179434258467.3460405.2730116922465377144%40burd.me)

**Question:** What does v46 cost for an UPDATE that changes an expensive index expression, and can a table with such indexes opt out of the extra evaluation?

Counter-check: held. Message #81 reports the regression for expensive expressions whose value changes, and v42 dropped the datum reuse that had lowered it from about 55%. No message from v39 onward says whether the expression_checks reloption remains, so these users have no stated opt-out.

### 2. HOT chain correctness now depends on extension answers, with no check

fixable · level 2, rubric item 7 · ABI / extension API, on-disk format / pg_upgrade, WAL / replication / standby · `replication-ondisk-1`

An extension's SupportRequestSubattr handler or amcomparedatums (for example the RUM fork in message #79) wrongly reports an index's entries as unchanged. The UPDATE goes HOT and no new index entry is written. The primary and every physical standby then hold an index that misses the new key, and queries silently skip the row until someone runs REINDEX.

Today HOT safety rests on a byte comparison inside core. With 0005 and 0006 it also rests on proofs and comparisons written in extension code, and a wrong answer is stored on disk and replicated, not just a bad plan for one query. Message #81 checks proofs only for in-tree types ("misc_functions also checks HOT counts"). The brief lists what happens when an extension answers wrongly as unclear. Wrong answers from planner support functions affect one query, while this one persists in the index.

**Precedents.**
- [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (Reverted, 2022-06-16); thread: [#1](https://www.postgresql.org/message-id/flat/05ebcb44-f383-86e3-4f31-0a97a55634cf@enterprisedb.com), [#2](https://www.postgresql.org/message-id/flat/CAFp7QwpMRGcDAQumN7onN9HjrJ3u4X3ZRXdGFT0K5G2JWvnbWg@mail.gmail.com)

Thread: [#79 (Greg Burd, 2026-10-17)](https://www.postgresql.org/message-id/-ugooulGn7M5X4DFIzWHSgYRngUmXChD_a-JvcMaklnDNcHMGnQhcM82SpYY76Je5ss4TOrsQB5sv2USLRSlDcpwDX-6CgCCbGKlJuA0SOs%3D%40protonmail.com), [#81 (Greg Burd, 2026-10-19)](https://www.postgresql.org/message-id/j-s6ilS5lyXpV_6l6Q10JTSiumDI5TBUf2zsOfUZAsB3-TjQCJgxWWVIuoCvDZrhD5ljcouHg1s_S_FfjjBS-97Min2Wwyvm5vFSIcdsz_o%3D%40burd.me), [#88 (Greg Burd, 2026-11-11)](https://www.postgresql.org/message-id/179434258467.3460405.2730116922465377144%40burd.me)

**Question:** What stops, or detects, an index left without entries when an extension's SupportRequestSubattr or amcomparedatums answer is wrong, for example a debug check or an amcheck test?

Counter-check: held. Extension opclass code can already corrupt an index, but a proof about which parts of a value a function reads is a new contract, and the thread shows checks only for in-tree types. P1 concerned a wrong assumption in core and must not be used as support.

### 3. Design does not say why the 2018 surjective-index revert cannot recur

fixable · level 2, rubric item 7 · prior rejection, performance on the hot path, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A committer pushes the series, and a user with an expression or GiST index hits a bug of the same family that forced the 2018 revert. Users see it as a crash or a wrong index result after release, and the only fix in reach is a revert of the series.

Message #1 says this grew out of surjective functional indexes, which were committed and then reverted for a bug and other quality issues. The only later trace of that bug is message #79 adding "Tom's 2018 gist point->box crash case" as a test. No message says which invariant that commit broke or why the executor-side check keeps it. Message #76 notes that a problem found after commit could only be met with a revert, which is how the earlier attempt ended.

Thread: [#1 ("Burd, Greg", 2025-02-06)](https://www.postgresql.org/message-id/78574B24-BE0A-42C5-8075-3FA9FA63B8FC%40amazon.com), [#76 (Nathan Bossart, 2026-03-25)](https://www.postgresql.org/message-id/acQYVfzHoL4lVTUE%40nathan), [#79 (Greg Burd, 2026-10-17)](https://www.postgresql.org/message-id/-ugooulGn7M5X4DFIzWHSgYRngUmXChD_a-JvcMaklnDNcHMGnQhcM82SpYY76Je5ss4TOrsQB5sv2USLRSlDcpwDX-6CgCCbGKlJuA0SOs%3D%40protonmail.com)

**Question:** What was the root cause of the 2018 revert, and which part of the v46 design rules it out, beyond one regression test?

Counter-check: held. The visible thread names the 2018 revert and adds Tom's gist point->box case as a test, but no message states the root cause or which part of the design rules it out. A test for one known case does not answer the invariant question.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-1: HOT chain correctness now depends on extension answers, with no check |
| WAL / replication / standby | **yes** | see objection replication-ondisk-1: HOT chain correctness now depends on extension answers, with no check |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The series adds the amcomparedatums index AM callback and SupportRequestSubattr, and changes the table_tuple_update(), heap_update() and ExecInsertIndexTuples() signatures. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Early versions added the GUC enable_expression_checks, then the reloption expression_checks; the v46 messages do not say whether either remains. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | v15 touches psql tab completion for the expression_checks reloption. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Nine patches change how the executor and heapam decide HOT, and add two contracts that extensions must answer correctly. |
| performance on the hot path | **yes** | Every UPDATE on a table with expression or partial indexes may evaluate the expression on the old row; message #81 reports 33-44% slower updates. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What does v46 cost for an UPDATE that changes an expensive index expression, now that v42 dropped the patch that removed one evaluation?
2. Does v46 keep a per-table way, such as the expression_checks reloption, to skip the extra expression evaluation?
3. What was the root cause of the 2018 revert of surjective functional indexes, and which part of the v46 design rules it out?
4. What detects a wrong SupportRequestSubattr or amcomparedatums answer from an extension before the index is left without an entry?
5. How does v46 handle exclusion constraints, which 0009 of v41 covered and v42 dropped?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows heapam HOT rules, the executor update path and the index AM interface. Committers reviewed earlier versions, but the CommitFest entry has no reviewer now and v46 has no visible review.
- **Liability:** The owner signs for index correctness under the new HOT rule, including answers from extension AMs and support functions, for as long as the two contracts exist. Message #76 notes that a problem found after commit would likely mean a revert.
- **Author follow-up:** Strong sign that the author stays: 46 versions since February 2025, replies to each review, and the author withdrew v29 over a 2-6% regression.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** JSONB expression-index users, ORM write-back and extensions such as DocumentDB gain, and the author's numbers show large wins. The nine-patch series touches heapam, the executor and the index AM interface, and no qualified reviewer has looked at v46.

## Appendix

### Refuted in the counter-check

- `prior-attempts-2` Expression cost question from the earlier attempt has no answer yet. Refuted (not specific): Duplicate of demand-1, which states the same cost and the same missing switch. demand-1 also notes that the 33-44% figure assumed v41's 0010, which v42 dropped.
- `standards-compat-1` No documented way to opt out of the extra index-expression evaluation. Refuted (not specific): Duplicate of demand-1.
- `replication-ondisk-2` Existing HOT chains outlive the amcanreturn condition that made them safe. Refuted (no failure scenario): The callback's contract is that old and new values give exactly the same index entries. Whatever an AM later returns from such an entry is the same as after a REINDEX, so the existing HOT chains add no wrong result.
- `maintenance-fit-1` Two new extension contracts can silently produce stale index entries. Refuted (not specific): Duplicate of replication-ondisk-1, which gives the more concrete scenario. The P1 revert was about a core assumption on BRIN summaries, so it does not support this point either.
- `maintenance-fit-2` Users of expensive expression indexes pay up to 44% with no switch. Refuted (not specific): Duplicate of demand-1.
- `maintenance-fit-3` Nine-patch series on the heap update path has no current reviewer. Refuted (not specific): The facts are accurate, but they describe review status and names no flaw in the design. Under the rubric this belongs in the committability field, and v46 was posted one day before this review, three weeks after Matthias reviewed v39.
- `already-exists-1` Stored generated columns already give HOT for the jsonb expression case. Refuted (already handled): The objection itself lists what a stored generated column cannot do: partial-index predicates, GIN or RUM over a whole document, and write-back of compressed values. The proposal targets all three, and the cost half of the objection is covered by demand-1.

### Review angles

- prior-attempts: 2 objection(s). No listed precedent matches this proposal's reason for failing. WARM [P16] and the BRIN revert [P1] are in the same area, but their recorded reasons do not apply to a HOT-only, all-or-none design. [P12] is the same: the rule that a row must not enter or leave the predicate addresses its HOT-chain concern. The earlier surjective attempt appears only in the thread (messages #1, #79), not in the precedent list.
- standards-compat: 1 objection(s). The standards and compatibility surface is mostly internal: a table AM signature change, an index AM callback and a support request. There is no SQL syntax, no new catalog object and nothing for pg_dump to handle. The safety of wrong SupportRequestSubattr answers belongs to another angle.
- replication-ondisk: 2 objection(s). The series adds no WAL record, no on-disk format change and no catalog column. HOT records replay as before, so pg_upgrade and standbys need nothing new. Replica identity and candidate-key columns are still compared as stored (message #84).
- security: 0 objection(s). The proposal has no privilege or exposure aspect that this angle can object to. Index expressions and partial-index predicates already run as the updating role, or as the apply worker's role, when the index entry is inserted. The patch adds one evaluation of the same immutable expression on the old row, under that same role, and adds no new privilege, view, log output or user-reachable setting. The material does not show how an extension's wrong SupportRequestSubattr or amcomparedatums answer is caught. That gap matters for index correctness and is not an escalation path, so it belongs to the safety angle. Exclusion constraint handling (0009) is out of v46, and v46 does not change uniqueness checks on candidate keys.
- maintenance-fit: 3 objection(s). Objections are given above. The proposal states the problem and shows demand (JSONB, ORM write-back, DocumentDB), so level 1 passes from this angle.
- already-exists: 1 objection(s). Only the expression-index part has an existing alternative. Partial-index predicates, whole-document GIN or RUM, and compressed write-back have no in-core workaround I can name.
- demand: 1 objection(s). Demand itself is shown: JSONB expression-index users, ORM and replication write-back, an extension with zero HOT today, and support from several reviewers. The only demand-side concern is who pays for it.

### Precedents retrieved

- P1: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - used in the review
- P2: [Revert "Prevent incorrect updates of pg_index while reindexing pg_index itself."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c096d19b74a637443109e528000342e896b150f3) (revert, Reverted, 2011-04-19) - judged not relevant
- P3: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P4: [Revert indexed and enlargable binary heap implementation.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=810f64a01567610af7b92b0de930f16f3e20064e) (revert, Reverted, 2024-04-11) - judged not relevant
- P5: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P6: [Improve use of expression indexes for ORDER BY](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P8: [Continue to allow VACUUM to mark last block of index dirty even when there is no work to do.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2d2841a56c8fa37a5dd5c6d33488ba6ca37116ff) (revert, Partially reverted, 2011-11-22) - judged not relevant
- P9: [Revert misguided change to postgres_fdw FOR UPDATE/SHARE code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8ec8760fc87ecde0516e511f1c55aec627b01ea7) (revert, Reverted, 2014-12-12) - judged not relevant
- P10: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P11: [On-disk bitmap indexes](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P12: [Support ALTER INDEX ... WHERE ... clause](https://commitfest.postgresql.org/patch/502/) (cf, Returned with feedback, 2016-04-08) - used in the review; reason summarised from its thread
- P13: [Revert bogus fixes of HOT-freezing bug](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c6764eb3aea63f3f95582bd660785e2b0d4439f9) (revert, Reverted, 2017-11-02) - judged not relevant
- P14: [create index failed. PARALLEL UNSAFE function does not prevent parallel index build](https://commitfest.postgresql.org/patch/4860/) (cf, Rejected, 2024-03-13) - judged not relevant; reason summarised from its thread
- P15: [heap_insert() and heap_update() optimization](https://commitfest.postgresql.org/patch/1830/) (cf, Rejected, 2018-10-17) - judged not relevant; reason summarised from its thread
- P16: [Write Amplification Reduction Method (WARM)](https://commitfest.postgresql.org/patch/775/) (cf, Returned with feedback, 2017-10-02) - used in the review; reason summarised from its thread
- P17: [format_datum debugging function](https://commitfest.postgresql.org/patch/5165/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P18: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P19: [Invalid permission check in pg_stats for functional indexes](https://commitfest.postgresql.org/patch/2274/) (cf, Returned with feedback, 2020-07-05) - judged not relevant
- P20: [Revert "Convert *GetDatum() and DatumGet*() macros to inline functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e8d78581bb864369035c736a82ecfb57482a1a09) (revert, Reverted, 2022-09-12) - judged not relevant
- P21: [Eliminating CREATE INDEX comparator TID tie-breaker overhead](https://commitfest.postgresql.org/patch/312/) (cf, Rejected, 2015-07-30) - judged not relevant
- P22: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant
- P23: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant

### Warnings

- 25 long messages in the middle of the thread were left out of the prompts
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 220 s; cost reported by the CLI: $7.27
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 98 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:32:36.447Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
