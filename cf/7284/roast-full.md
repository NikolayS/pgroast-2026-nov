# Design pre-review: {heap,minimal}_expand_tuple() are unused

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7284](https://commitfest.postgresql.org/patch/7284/) (Needs review); authors: Aleksander Alekseev (a.alekseev)
- Thread: [{heap,minimal}_expand_tuple() are unused](https://www.postgresql.org/message-id/flat/gf6hokynqjp6g7o4zh6yy7ekbdbig5cnnd6ilk6iw3aodilj43%405x7xrz52jhve), started 2026-08-07
- Messages read: 6 of 6
- Patch prose read: v1-0001-Remove-unused-tuple-expansion-and-copying-functio.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at any level. The patch deletes four functions with no callers in core, and two committers who know heaptuple.c proposed and backed the removal in messages #1 to #3. The only cost is that an out-of-tree caller fails to build against the new major version, which is how Postgres normally removes C API. Confidence is medium because only three of six messages are visible and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so this review did not confirm that the four functions have no remaining callers or that the reworded fast_default comments are accurate. Messages #4 to #6 were not shown.

## The proposal, restated

**Problem.** Four functions in heaptuple.c have no callers in core: heap_expand_tuple(), minimal_expand_tuple(), the static expand_tuple(), and heap_copytuple_with_tuple(). Andres Freund argues the code is nontrivial and should not stay around only because an outside caller might exist.

**Proposed design.** The patch deletes the four functions from heaptuple.c and their declarations from htup_details.h. It also rewords two comments in the fast_default regression test that name expand_tuple(), so they refer to the code that now handles missing attributes.

**User-visible surface.**
- C API: heap_expand_tuple() removed from htup_details.h.
- C API: minimal_expand_tuple() removed from htup_details.h.
- C API: heap_copytuple_with_tuple() removed from htup_details.h.

**What the author claims.**
- minimal_expand_tuple() has been unused since 4da597edf1b and heap_expand_tuple() since 20d3fe9009d.
- heap_copytuple_with_tuple() has had no callers since 141fd1b66ce.
- Andres Freund found no external callers.
- The code can be restored from git if a use case appears.

**Thread state.** Tom Lane gave +1 and pointed out heap_copytuple_with_tuple(); Kyle Snavely reviewed v1 and found no issues, with tests passing. v2 is a rebase, and nobody has objected so far.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-07; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch removes heap_expand_tuple(), minimal_expand_tuple() and heap_copytuple_with_tuple() from htup_details.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch lowers the burden by deleting nontrivial tuple-expansion code from heaptuple.c that nothing in core calls. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which external code was searched for callers of the three exported functions, for example which extension repositories or code search service?
2. Will the release notes for the new major version list the three removed functions for extension authors?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund proposed the removal in message #1 and Tom Lane backed it in messages #2 and #3; both work on the heap tuple code named in the thread.
- **Liability:** Small and short-lived: a possible report from an extension author whose build breaks on the new major version, answered by restoring the code from git.
- **Author follow-up:** Aleksander Alekseev posted v2 as a rebase after CFbot flagged v1, so the author is keeping the patch current.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch only deletes uncalled functions in two files and rewords two test comments, so review is cheap and the result is less code to maintain. Two committers agree, a reviewer found no issues, and nobody has objected.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent in the list matches this proposal or gives a reason that applies to it. [P22], [P12] and [P16] were dead-code removals, but none records why it was not accepted. [P6] stalled on unused parameters kept for API consistency and on backpatch conflicts; neither concern applies to deleting whole uncalled functions. In the thread, Tom Lane gave +1 (message #2) and Kyle Snavely found no issues (message #5). Not checked: which external code was searched for callers (message #1).
- standards-compat: 0 objection(s). Nothing in the user-facing contract changes: no SQL, GUC, catalog, pg_dump or psql surface. The only compatibility effect is removing three exported C functions in a major release, which Postgres does routinely; an extension that calls them fails loudly at build or load time, not silently. Tom Lane agreed in message #2. [P6] does not apply because its concern was churn in code that is still used.
- replication-ondisk: 0 objection(s). The patch creates no persistent or cross-node state, so this angle finds no objection. It removes C functions with no callers from master only, and changes no on-disk tuple format, WAL record, catalog, or wire protocol message. Fast-default tuples on disk are still read by the code that handles missing attributes today. Dropping exported functions in a new major release is normal practice, Andres Freund found no external callers, and Tom Lane agreed in message #2.
- security: 0 objection(s). The patch only deletes unused C functions from heaptuple.c and htup_details.h and rewords two test comments. It adds no privilege, SQL-callable surface, GUC, catalog entry or new data exposure, so there is no security or privilege aspect to object to.
- maintenance-fit: 0 objection(s). From the maintenance angle this patch lowers the long-run cost, since it deletes about 250 lines of nontrivial tuple-expansion code that nothing in core calls. Two committers who know heaptuple.c, Andres Freund and Tom Lane, proposed and backed it in messages #1 to #3. A third-party extension that still calls these functions would fail to build against the new major version, which is the usual way Postgres removes C API in a major release. Andres Freund searched for external callers and found none (message #1), though the thread does not say what code he searched. The objection recorded in [P6], backpatch conflicts for little gain, does not apply here. The functions are removed whole, so a fix to them on a back branch has nothing to merge into in master.
- already-exists: 0 objection(s). This angle has nothing to raise. The proposal adds no feature that an existing mechanism or extension could cover; it removes four functions with no core callers. Any concern about out-of-core callers belongs to a compatibility angle, not this one.
- demand: 0 objection(s). The demand angle does not fit a removal of dead code: the patch adds no feature that needs users, and it lowers maintenance cost for everyone. Two committers who know the area proposed or backed it (messages #1, #2, #3), and no external caller was reported. The one residual cost, out-of-tree breakage, is a compatibility question for another angle.

### Precedents retrieved

- P1: [Removing unused trailing linepointers from heap pages](https://commitfest.postgresql.org/patch/3035/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P2: [Remove unused #include's](https://commitfest.postgresql.org/patch/6112/) (cf, Withdrawn, 2025-10-15) - judged not relevant; reason summarised from its thread
- P3: [Simplify ExecWithoutOverlapsNotEmpty by removing unused parameter](https://commitfest.postgresql.org/patch/6556/) (cf, Withdrawn, 2026-06-06) - judged not relevant; reason summarised from its thread
- P4: [Remove unused function parameters, part 1: contrib](https://commitfest.postgresql.org/patch/6263/) (cf, Withdrawn, 2026-02-13) - used in the review; reason summarised from its thread
- P5: [Expand applicability of aggregate's sortop optimization](https://commitfest.postgresql.org/patch/4980/) (cf, Returned with feedback, 2025-07-08) - judged not relevant; reason summarised from its thread
- P6: [Remove unused function parameters, part 2: replication](https://commitfest.postgresql.org/patch/6264/) (cf, Withdrawn, 2026-02-13) - used in the review; reason summarised from its thread
- P7: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - judged not relevant; reason summarised from its thread
- P8: [Expand the use of check_canonical_path() for more GUCs](https://commitfest.postgresql.org/patch/2564/) (cf, Returned with feedback, 2020-07-10) - judged not relevant; reason summarised from its thread
- P9: [Revert "Remove pointless HeapTupleHeaderIndicatesMovedPartitions calls"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=93f84d59f80d11a3d8ade9ae71560162d6f3ecb2) (revert, Reverted, 2020-10-15) - judged not relevant
- P10: [Remove size limitations of vacuums dead_tuples array](https://commitfest.postgresql.org/patch/2302/) (cf, Returned with feedback, 2019-11-30) - judged not relevant
- P11: [Revert the logic for expanding AND/OR conditions in pred_test() to what it was in 7.4, and add some comments explaining why it has to be this way.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4e89bae704f79af8b2b6ac3d43ea963c22df1947) (revert, Reverted, 2005-03-01) - judged not relevant
- P12: [Remove dead code in ExecForPortionOfLeftovers()](https://commitfest.postgresql.org/patch/6697/) (cf, Withdrawn, 2026-08-02) - used in the review
- P13: [Revert "Enable fast default for domains with non-volatile constraints"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0354e29c41a9fb7491b3c7c23f079b1923c045a) (revert, Reverted, 2026-06-08) - judged not relevant
- P14: [Adding Support for Copy callback functionality on COPY TO api](https://commitfest.postgresql.org/patch/2660/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P15: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P16: [(DUPLICATE) Removing dead code in pgcrypto](https://commitfest.postgresql.org/patch/3860/) (cf, Withdrawn, 2022-08-30) - used in the review
- P17: [Make deparsing of column defaults faster](https://commitfest.postgresql.org/patch/1657/) (cf, Returned with feedback, 2018-09-24) - judged not relevant
- P18: [Forcing new tuples of updates off the page](https://commitfest.postgresql.org/patch/4440/) (cf, Withdrawn, 2024-03-15) - judged not relevant
- P19: [Minimizing calls to SubTransSetParent()](https://commitfest.postgresql.org/patch/3806/) (cf, Withdrawn, 2023-03-22) - judged not relevant
- P20: [Add option --drop-cascade for pg_dump/restore](https://commitfest.postgresql.org/patch/3065/) (cf, Returned with feedback, 2021-12-01) - judged not relevant
- P21: [Avoid unused value (src/fe_utils/print.c)](https://commitfest.postgresql.org/patch/4339/) (cf, Rejected, 2023-07-11) - judged not relevant
- P22: [Remove dead macro exec_subplan_get_plan](https://commitfest.postgresql.org/patch/3894/) (cf, Rejected, 2023-04-04) - used in the review
- P23: [AT add columns with default in O(1)](https://commitfest.postgresql.org/patch/843/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P24: [Add a function to get the version of installed extension](https://commitfest.postgresql.org/patch/5514/) (cf, Withdrawn, 2025-04-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 72 s; cost reported by the CLI: $0.77
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:37:19.003Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
