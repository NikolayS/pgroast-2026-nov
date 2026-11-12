# Design pre-review: pg_trgm: don't lose rows in index scans with a zero similarity threshold

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7382](https://commitfest.postgresql.org/patch/7382/) (Needs review); authors: Manuel Reyes Bravo (manu15)
- Thread: [BUG #19701: GIN trigram index loses rows at similarity_threshold 0](https://www.postgresql.org/message-id/flat/19701-c861a62e79bf49ce%40postgresql.org), started 2026-09-19
- Messages read: 6 of 6
- Patch prose read: 0001-Don-t-lose-rows-in-pg_trgm-index-scans-with-a-zero-s.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The proposal fixes a wrong-results bug in existing contrib opclasses, and no surviving objection questions the goal. The one surviving objection, standards-compat-1, is fixable: pgtrgm.sgml says "greater than" while the patch makes the indexes follow >=, and nobody has confirmed which meaning is intended. A committer who knows pg_trgm has to settle that and the back-patch decision.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and the commit message and touched files of v3-0001 are not shown. Whether the consistent-function changes are correct for all strategies and for the GIN pending list could not be judged.

## The proposal, restated

**Problem.** With a pg_trgm similarity threshold of 0, which the GUC and set_limit() accept, every row matches %, <% and <<%. GIN index scans return only rows sharing a trigram with the query, and return nothing for a query with no trigrams. GiST also loses rows on internal pages for such queries. Any user who sets a threshold of 0 gets silently wrong results from plans the planner picks itself.

**Proposed design.** gin_extract_query_trgm() asks for GIN_SEARCH_MODE_ALL when the threshold is 0, as it already does for queries with no trigrams. The GIN consistent functions and the GiST consistent function on internal pages accept candidates when the query has no trigrams and the threshold is 0. v3 adds a separate 0001 that fixes an out-of-bounds read in calc_word_similarity() when neither string has a trigram.

**User-visible surface.**
- Changed results: %, <%, <<% and commutators through gin_trgm_ops return every row at threshold 0, including rows sharing no trigram with the query.
- Changed results: gin_trgm_ops and gist_trgm_ops return rows for a query with no trigrams (such as '' or '#') at threshold 0.
- Changed plan cost in practice: at threshold 0 a GIN scan becomes a full index scan.

**What the author claims.**
- The operators mean similarity >= threshold, so a threshold of 0 must match every row.
- With default thresholds, all tested queries return the same rows as before.
- Without the C changes the new regression tests fail; with them all pg_trgm tests pass, also under -fsanitize=undefined and cassert.
- The patch applies cleanly to REL_14_STABLE through REL_19_STABLE; tests were run on master, REL_18_STABLE and REL_14_STABLE.
- The calc_word_similarity() over-read also happens on master with word_similarity('', '') and never changes the result.

**Thread state.** Palak Chaturvedi reviewed v1 and v2, compared row sets for all six operator forms, asked for more tests and called v2 good. Manu then posted v3 to fix a cfbot sanitizer failure; no one has replied to v3 and no committer has commented.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-19; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Patch fixes indexes to >= while pgtrgm.sgml still documents "greater than"

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-1`

A user reads pgtrgm.sgml, which says % is true when similarity is "greater than" the threshold. The user sets the threshold to 0 to mean "any shared trigram". After the patch, every row comes back, including rows with similarity 0, and the documentation still describes the old meaning.

Message #1 states that the documented contract (>) and the implemented one (>=) differ, and that at threshold 0 this choice decides the whole outcome. The patch touches only trgm_gin.c, trgm_gist.c and tests, so it settles the semantics in code without changing the user-facing contract. The reporter's argument for >= is reasonable. But no reviewer or committer in the thread has confirmed it, and a committer would likely want the documentation changed in the same commit.

Thread: [#1 (PG Bug reporting form, 2026-09-19)](https://www.postgresql.org/message-id/19701-c861a62e79bf49ce%40postgresql.org), [#2 (Manu, 2026-09-23)](https://www.postgresql.org/message-id/179019160285.4009503.14265883830439530659%40gmail.com), [#5 (Palak Chaturvedi, 2026-09-27)](https://www.postgresql.org/message-id/CALfch1_qFgqr24XQY4kB93Gr62Jx%2BGUqLA0QYTpWsfrEZZ0wWg%40mail.gmail.com)

**Question:** Will the patch change pgtrgm.sgml to "greater than or equal to" for %, <% and <<%, and why is >= preferred over making 0 behave as documented?

Counter-check: held. The patch touches only trgm_gin.c, trgm_gist.c and the tests, while its commit message states the >= meaning that pgtrgm.sgml contradicts. Nobody in the thread has confirmed >= as the intended semantics, so the documentation change is still missing.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch fixes the meaning of %, <% and <<% as >=, while pgtrgm.sgml documents "greater than". |
| new GUC | no |  |
| planner behaviour change | **yes** | At threshold 0 a GIN scan becomes a full index scan, and the proposal does not discuss its cost. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #2 says the patch applies to REL_14_STABLE through REL_19_STABLE; it changes query results at threshold 0 in minor releases. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the patch change pgtrgm.sgml to say "greater than or equal to" for %, <% and <<%, in the same commit?
2. Why is fixing the indexes to >= preferred over changing the operators to > so that threshold 0 behaves as documented?
3. Is the fix meant for back-patching to REL_14_STABLE and later, given that GIN results at threshold 0 change in a minor release?
4. What does v3-0001 touch, and should the calc_word_similarity() over-read fix be committed and back-patched separately from the index fix?
5. Should the documentation tell users that a GIN scan at threshold 0 reads the whole index?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the pg_trgm opclasses and the GIN and GiST consistent-function contracts, including GIN search modes.
- **Liability:** The owner signs for the >= meaning of the three operators at threshold 0 in every supported branch, and for changed GIN and GiST results in minor releases if back-patched.
- **Author follow-up:** The author posted v2 after review and v3 after a cfbot sanitizer failure, which suggests the author stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Few users set a threshold of 0, but the planner picks plans that silently lose rows for those who do. The patch is small and has one review, but no committer has replied, and the semantics and back-patch decisions need one.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Back-patch changes rows and scan cost at threshold 0 in minor releases. Refuted (no failure scenario): The scenario describes the fix itself. A sequential scan and GiST already return every row at threshold 0, so today's GIN result depends on the plan and no site can rely on it. Any correct plan must read every row.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the retrieved list has no earlier rejected, returned or reverted proposal on this idea. Message #1 cites BUG #14202 (2016) as the fix that made gtrgm_consistent use >=, which supports the >= reading and is not a rejection. Other angles cover the semantics and back-patch questions.
- standards-compat: 2 objection(s). Both objections concern the user-facing contract and can be fixed in a new version. The correctness fix itself raises no syntax, GUC, pg_dump or psql concern.
- replication-ondisk: 0 objection(s). The patch only changes query-time logic in the GIN extract_query and consistent functions and the GiST consistent function. Index contents, WAL, catalogs, the wire protocol and extension-visible structs stay the same, so existing indexes need no rebuild after a minor update. A standby on an older minor release can return different index results during a rolling update, but that is the existing wrong-results bug, and a sequential scan on either node already returns the full set.
- security: 0 objection(s). From the security angle there is nothing to object to. The threshold is a per-session setting, so the full GIN index scan at 0 costs only the session that sets it, and is no worse than a sequential scan it could already force. RLS and column privileges are unchanged, the operators' leakproofness is unchanged, and v3-0001 removes an over-read that did not change results. Back-patch effects on query results belong to other angles.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The patch fixes wrong results in existing contrib opclasses, touches only trgm_gin.c, trgm_gist.c and tests, and adds a small threshold check that has no cost at non-zero thresholds and creates no new API or ongoing work. The full GIN scan at threshold 0 costs only users who choose that threshold. Separate from the verdict: no committer has engaged yet, and the docs ('greater than' versus >=) and back-patching are not yet settled, which other angles cover.
- already-exists: 0 objection(s). No existing feature, option or extension gives correct index results for %, <% and <<% at threshold 0, so the need is not already met. This is a wrong-results bug in core contrib, and no workaround returns the rows a sequential scan returns. Leaving out the predicate or using a tiny positive threshold changes the query, and keeping the index means editing pg_trgm itself. The patch extends the GIN_SEARCH_MODE_ALL path that already exists for queries with no trigrams, so it adds no second mechanism. Whether to forbid 0 or switch the operators to > is a design question for other angles.
- demand: 0 objection(s). This is a wrong-results bug fix, and the demand test does not apply in the usual way. Threshold 0 is a legal value of the GUC and of set_limit(), and message #1 shows the planner picking a GIN plan on its own that returns 0 rows instead of 1. Message #2 confirms the bug on master, and a second person compared row sets in message #3. The extra cost of a full GIN scan falls only on sessions that set threshold 0. No other user pays for it. Few people may use threshold 0, but an index that changes query results is still a defect. Whether to fix the indexes or forbid 0 is a design question for other angles.

### Precedents retrieved

- P1: [Fillfactor for GIN indexes](https://commitfest.postgresql.org/patch/13/) (cf, Rejected, 2015-01-16) - judged not relevant; reason summarised from its thread
- P2: [Making autovacuum logs indicate if insert-based threshold was the triggering condition](https://commitfest.postgresql.org/patch/3802/) (cf, Withdrawn, 2022-11-27) - judged not relevant
- P3: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P4: [Increasing the default values of from_collapse_limit, join_collapse_limit, and/or geqo_threshold](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P5: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P6: [PGDLLIMPORT-ify old_snapshot_threshold.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=14216649f3dc8bd9839702440dd593e958b0920b) (revert, Partially reverted, 2016-04-21) - judged not relevant
- P7: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant; reason summarised from its thread
- P8: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant; reason summarised from its thread
- P9: [Don't allow to EXECUTE empty queries (fixes Assert violation)](https://commitfest.postgresql.org/patch/5918/) (cf, Rejected, 2025-07-17) - judged not relevant; reason summarised from its thread
- P10: [Revert "Show index search count in EXPLAIN ANALYZE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d00107cd63e780753aa25563fa37603369997d0c) (revert, Reverted, 2025-03-05) - judged not relevant
- P11: [Rename trim_array() for consistency with the rest of array_* functions](https://commitfest.postgresql.org/patch/5335/) (cf, Withdrawn, 2024-11-04) - judged not relevant; reason summarised from its thread
- P12: [Extracting only the columns needed for a query](https://commitfest.postgresql.org/patch/2331/) (cf, Withdrawn, 2020-07-01) - judged not relevant; reason summarised from its thread
- P13: [Make selectivity example match wording](https://commitfest.postgresql.org/patch/3695/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P14: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P15: [Tab-completion refactoring and word-removal feature (was IF-NOT_EXISTS for tab-completion)](https://commitfest.postgresql.org/patch/855/) (cf, Returned with feedback, 2017-03-17) - judged not relevant
- P16: [Make the am_xxx function Asserts consistent](https://commitfest.postgresql.org/patch/4570/) (cf, Withdrawn, 2024-01-09) - judged not relevant
- P17: [allow to set log_statement_sample_limit - maximal duration for sampling](https://commitfest.postgresql.org/patch/2163/) (cf, Rejected, 2019-11-06) - judged not relevant
- P18: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant
- P19: [Optimize numeric comparison and aggregation via packed-datum extraction](https://commitfest.postgresql.org/patch/6651/) (cf, Rejected, 2026-05-10) - judged not relevant
- P20: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P21: [Pass sk_attno to GIST consistent function](https://commitfest.postgresql.org/patch/4902/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P22: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P23: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P24: [Fix out-of-bounds with function PQescapeInternal](https://commitfest.postgresql.org/patch/4910/) (cf, Rejected, 2024-04-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 103 s; cost reported by the CLI: $1.20
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 56 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:48:41.672Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
