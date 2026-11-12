# Design pre-review: BUG #19705: One NaN box makes BRIN and GiST box indexes omit unrelated rows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7370](https://commitfest.postgresql.org/patch/7370/) (Needs review); authors: Kirill Reshke (reshke), shihao zhong (zhong950419)
- Thread: [BUG #19705: One NaN box makes a BRIN box_inclusion_ops index omit unrelated rows](https://www.postgresql.org/message-id/flat/19705-548fda77321e062d%40postgresql.org), started 2026-09-19
- Messages read: 16 of 16
- Patch prose read: v1-0002-Add-tests-for-NaN-handling-in-bound_box-and-BRIN-.patch, v1-0001-Fix-BRIN-box_inclusion_ops-losing-rows-when-a-box.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is a reported wrong-results case with reproductions, and no objection survived at levels 1a or 1b. The one surviving objection, maintenance-fit-2, is a fixable design choice: master would carry box_mergeable while back branches carry the self-containment check. The author already offers to use one check everywhere, so a committer who knows BRIN can settle it.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the claims that old NaN summaries and internal keys are handled without REINDEX, and that the self-containment check catches every poisoned range, could not be checked. Only the first 3 of 16 messages were read directly.

## The proposal, restated

**Problem.** One box value with a NaN coordinate puts NaN into a BRIN range summary or a GiST union key, and no box operator is true against a NaN bound. Index scans then skip the whole page range or subtree, so queries silently return fewer rows than a seq scan. GiST KNN searches with a NaN in a key or query point also fail with "inconsistent point values".

**Proposed design.** On master, BRIN box_inclusion_ops gets a box_mergeable support function so a range holding a NaN is marked unmergeable and always scanned. Back branches instead scan any range whose union does not contain itself, with no catalog change. GiST unions replace NaN with infinity, old NaN internal keys match every search with KNN distance 0, and leaf ~=, polygon, circle and KNN checks handle NaN.

**User-visible surface.**
- New pg_amproc row for box_inclusion_ops (support function 12, box_mergeable) on master, with a catversion bump.
- New GiST box index keys store Infinity where NaN used to appear, visible through gist_page_items.
- v1 only: bound_box() returns an infinite bound for a NaN coordinate; later versions leave bound_box() unchanged.
- Index scans return rows they omitted before; a count in create_index.out changes from 4 to 5.

**What the author claims.**
- Old BRIN and GiST indexes return correct results after the fix without REINDEX, checked after pg_upgrade and a minor upgrade.
- A differential check of 6769 seq scan versus index comparisons per build drops to zero mismatches, except point <@ polygon with a NaN vertex.
- GiST patches 0002 to 0004 and the catalog-free BRIN fix back-patch to 14 with small rebases.
- The self-containment check could replace box_mergeable on master to avoid a catversion bump.
- The remaining polygon case is a heap bug in point_inside() and belongs in a separate master-only patch.

**Thread state.** Kirill Reshke, Andrey Borodin and Manu reviewed and tested through v6, and the author asked Tomas and Heikki to commit. John Naylor pointed to a Tom Lane message suggesting a different approach; the author replied that banning NaN cannot be back-patched, and no committer has answered.

## 2. Prior attempts

### Fix behavior of geo_ops when NaN is involved (related)

- **Link:** [https://commitfest.postgresql.org/patch/2710/](https://commitfest.postgresql.org/patch/2710/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAGf+fX70rWFOk5cd00uMfa__0yP+vtQg5ck7c2Onb-Yczp0URA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAGf+fX70rWFOk5cd00uMfa__0yP+vtQg5ck7c2Onb-Yczp0URA@mail.gmail.com)
- **Date:** decided 2022-08-01; started 2020-06-24
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane committed parts of the work in November 2020. The rest was rebased, and a reviewer was asked for comments in March 2021, but no reviewer answered that request or the expanded patch set posted in December 2021. Jacob Champion returned the entry in August 2022 for lack of reviewer interest, and said this was not a rejection.
- **Reason (quote):** "Rather than code changes, what this patch needs is more community interest." ([message](https://www.postgresql.org/message-id/f84318e2-2918-8c21-436f-a79823c9e097%40timescale.com))
- **What is different this time:** P1 changed what geo_ops operators return for NaN. This proposal leaves the operators alone and fixes BRIN and GiST so index scans match the heap. It also has three reviewers who tested it, which P1 lacked.


## 3. Objections a reviewer is likely to raise

### 1. Two different BRIN fixes for master and back branches

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), catalog changes, backpatchability · `maintenance-fit-2`

A committer later back-patches a fix in brin_inclusion_consistent() or brin_inclusion_add_value(). Master has the box_mergeable path and the back branches have the self-containment check, so the patch conflicts. A hand-merged version on 14 to 18 then gets no testing against the master code.

Message #7 places box_mergeable plus a consistent-side guard on master, and a separate self-containment check on 14 to 18. Messages #7, #14 and #16 say the self-containment check also works on master and needs no catversion bump. Master would then keep a pg_amproc row it does not need for the old-summary case. If master later bans NaN as message #15 suggests, the extra support function stays forever.

Thread: [#7 (shihao zhong, 2026-09-24)](https://www.postgresql.org/message-id/CAGRkXqQ7q0U3jmitwOjPY2BNBkxWsnyahgoYyMs4H%3DhwVtPFgw%40mail.gmail.com), [#14 (shihao zhong, 2026-09-30)](https://www.postgresql.org/message-id/CAGRkXqSQHEe0R5EuuLYthMf1C%3DL_tvPD74aPbgtuOT0Rg0Y99w%40mail.gmail.com), [#15 (John Naylor, 2026-09-30)](https://www.postgresql.org/message-id/CANWCAZZmuP9xsDv%2Bk6WrKAfQK-_6pzRDF4XrPQHuQxEj-qSSyA%40mail.gmail.com), [#16 (shihao zhong, 2026-10-01)](https://www.postgresql.org/message-id/CAGRkXqRbimqBSg9GYMGDSp7bpaDWed91Tyb6WHcr4Yw50bU8Rg%40mail.gmail.com)

**Question:** What does box_mergeable give on master that the catalog-free self-containment check does not, enough to justify carrying two BRIN fixes?

Counter-check: held. Messages #7, #14 and #16 say the self-containment check works on master with no catalog change, and no message says what box_mergeable adds beyond matching the inet practice. The author offers to switch, so this is a choice for the committer; the NaN ban part is unsupported.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | New GiST box keys store Infinity where NaN appeared, and old NaN summaries and keys must still be read correctly after pg_upgrade. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | Master adds a pg_amproc row for box_inclusion_ops (support function 12, box_mergeable) with a catversion bump. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The GiST patches and a catalog-free BRIN fix are planned for back-patch to 14, with a different BRIN fix on master. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Two BRIN fixes by branch, plus NaN checks in several GiST consistent, union and KNN code paths. |
| performance on the hot path | **yes** | Extra NaN checks run in GiST search and KNN distance paths, and the thread gives no measurement. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What does box_mergeable give on master that the catalog-free self-containment check does not, enough to justify two different BRIN fixes?
2. What is the measured cost of the added NaN checks in GiST search and KNN paths for box indexes that hold no NaN?
3. Do the rebased patches for 14 to 16 exist, and do they pass the same differential check of seq scan against index scan?
4. What is the plan for the SP-GiST box_ops and poly_ops mismatches that this set leaves unfixed?
5. What does the Tom Lane message that John Naylor linked propose for master, and how does this set fit with it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows the BRIN inclusion framework and the GiST geometric opclasses. The author asked Tomas and Heikki to commit, and neither has answered in the visible thread.
- **Liability:** A back-patch to 14 that changes index scan results and GiST key contents, plus on master a permanent pg_amproc row. The owner also signs that old NaN summaries and keys stay correct without REINDEX.
- **Author follow-up:** Good sign: the authors posted six versions in response to review and offer to switch the master BRIN fix to the self-containment check.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Few tables hold NaN boxes, but affected users get silently wrong results, and three reviewers have tested the set. It waits on a committer to choose the master BRIN fix; review cost is estimated because no code was read.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Master BRIN fix adds a catalog row a planned NaN ban may make unneeded. Refuted (not specific): Duplicate of maintenance-fit-2. This version also rests on a NaN ban that the material shows only through the author's one-line summary in message #16.
- `standards-compat-1` Master adds a catalog support function where a catalog-free check exists. Refuted (not specific): Duplicate of maintenance-fit-2, which states the same choice between box_mergeable and the self-containment check without relying on the unseen linked message.
- `standards-compat-2` Back-patched 0003 makes index scans match a heap answer called wrong. Refuted (no failure scenario): Back branches today return 4 or 5 rows for the same query depending on the plan, and 0003 removes that difference. The point_inside() change is planned for master only, so back-branch answers change once, and a behavior change in a new major version is normal.
- `maintenance-fit-1` NaN handling spread across GiST call sites with no single rule. Refuted (no failure scenario): A future GiST strategy without a NaN check is a risk that exists today without the patch, and the set adds none. The set does state its rule in messages #6, #7 and #11, and P1 was returned for lack of reviewers, so it does not support the objection.

### Review angles

- prior-attempts: 1 objection(s). No precedent records a rejection reason that applies to this bug fix. P4 died over removing the epsilon tolerance, and this proposal keeps the tolerance. P5 records no reason.
- standards-compat: 2 objection(s). The bug fix itself fits existing practice. It adds no new syntax and no GUC, and dump behaviour does not change. Old indexes are reported correct after pg_upgrade and minor upgrade.
- replication-ondisk: 0 objection(s). No objections from the on-disk and replication angle. The back-branch fixes change no on-disk format, WAL record or catalog, and the patched scan code accepts summaries and NaN internal keys written by old binaries, so pg_upgrade, minor upgrade, a minor downgrade and mixed-minor standbys keep returning correct results. Infinity keys are already handled by old binaries, the master pg_amproc row reaches pg_upgraded indexes through the new catalog, and no ABI-visible struct or hook changes.
- security: 0 objection(s). The proposal adds no privilege, role, function exposure or new information path; it fixes index scans that return fewer rows than a seq scan. A user with INSERT can force wider scans with a NaN box, but an infinite box already does the same, so no new denial-of-service path follows.
- maintenance-fit: 2 objection(s). This is a data-correctness bug fix, and its core goal is sound. Both objections ask for a smaller, more uniform shape. Neither argues against fixing the bug.
- already-exists: 0 objection(s). No existing feature or extension stops BRIN and GiST box indexes from returning fewer rows than a seq scan, so this angle finds nothing to object to. A CHECK constraint can keep new NaN boxes out, but it does not repair existing indexes or GiST KNN errors. The master BRIN fix extends the existing INCLUSION_UNMERGEABLE support-function mechanism, which inet already uses, instead of adding a second one.
- demand: 0 objection(s). This is a wrong-results bug with a field report (message #1, Postgres 18.6) and reproductions by four people. Index scans silently drop rows that hold no NaN, so the problem is real and needs no further demand evidence. Whether box_mergeable is needed on master, or the self-containment check is enough, is a design question for other angles.

### Precedents retrieved

- P1: [Fix behavior of geo_ops when NaN is involved](https://commitfest.postgresql.org/patch/2710/) (cf, Returned with feedback, 2022-08-01) - used in the review; reason summarised from its thread
- P2: [Revert "Test conversion of NaN between float4 and float8."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5c4c771dafefa7d0ee80a6bd6955b7a3940452a1) (revert, Reverted, 2018-05-05) - judged not relevant
- P3: [support for box type in SP-GiST index](https://commitfest.postgresql.org/patch/419/) (cf, Returned with feedback, 2016-01-28) - judged not relevant; reason summarised from its thread
- P4: [Floating point comparison inconsistencies of the geometric types](https://commitfest.postgresql.org/patch/679/) (cf, Returned with feedback, 2017-03-16) - used in the review; reason summarised from its thread
- P5: [Fix data types where equality comparison is not intuitive, e.g. box](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Data_Types) (do_not_want, Not worth doing, date unknown) - used in the review
- P6: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P7: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5974/) (cf, Withdrawn, 2025-08-19) - judged not relevant; reason summarised from its thread
- P8: [Fix inconsistency in reporting checkpointer stats](https://commitfest.postgresql.org/patch/4078/) (cf, Returned with feedback, 2024-10-10) - judged not relevant; reason summarised from its thread
- P9: [PGDOCS - inconsistent linkends to "monitoring" views](https://commitfest.postgresql.org/patch/4602/) (cf, Rejected, 2023-11-30) - judged not relevant; reason summarised from its thread
- P10: [Inconsistency in owner assignment between INDEX and STATISTICS](https://commitfest.postgresql.org/patch/6571/) (cf, Withdrawn, 2026-03-11) - judged not relevant; reason summarised from its thread
- P11: [elog/ereport VS misleading backtrace_function function address](https://commitfest.postgresql.org/patch/4977/) (cf, Rejected, 2024-08-22) - judged not relevant; reason summarised from its thread
- P12: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P13: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant
- P14: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5975/) (cf, Withdrawn, 2025-12-31) - judged not relevant
- P15: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/4792/) (cf, Returned with feedback, 2024-10-02) - judged not relevant
- P16: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P17: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P18: [Coverage (lcov) failing with inconsistent error in versions 2.x](https://commitfest.postgresql.org/patch/6966/) (cf, Withdrawn, 2026-07-27) - judged not relevant
- P19: [Optimize numeric comparison and aggregation via packed-datum extraction](https://commitfest.postgresql.org/patch/6651/) (cf, Rejected, 2026-05-10) - judged not relevant
- P20: [Make psql \di show index with type info](https://commitfest.postgresql.org/patch/1083/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P21: [Forbid use of LF and CR characters in database and role names](https://commitfest.postgresql.org/patch/711/) (cf, Rejected, 2017-03-29) - judged not relevant
- P22: [Forbid accessing other session's temporary tables](https://commitfest.postgresql.org/patch/5392/) (cf, Withdrawn, 2024-11-30) - judged not relevant
- P23: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P24: [Fix out-of-bounds with function PQescapeInternal](https://commitfest.postgresql.org/patch/4910/) (cf, Rejected, 2024-04-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 132 s; cost reported by the CLI: $2.20
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 74 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:46:44.648Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
