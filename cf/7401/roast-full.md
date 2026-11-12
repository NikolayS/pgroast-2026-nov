# Design pre-review: gist_trgm_ops '=' operator: planner picks it over btree, ~300x slower

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7401](https://commitfest.postgresql.org/patch/7401/) (Needs review); authors: Vaibhav Dalvi (vaibhavdalvi)
- Thread: [gist_trgm_ops '=' operator: planner picks it over btree, ~300x slower](https://www.postgresql.org/message-id/flat/CA%2BvB%3DAHon6EB%3D90pbPDik_u8%2B%2B%2BTy8TA6Z4-RXo_X5QCr71kcQ%40mail.gmail.com), started 2026-09-02
- Messages read: 10 of 10
- Patch prose read: v2-Add-gist_trgm_ops_noeq.patch, v1-0001-Add-gist_trgm_ops_noeq.patch, v1-0001-Drop-equality-operator-from-gist_trgm_ops.patch, v1-0001-gist-trgm-real-signature-stats.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem is real and reproducible, and the patch costs non-users nothing. The thread has not settled whether a second opclass is the right fix. maintenance-fit-1 shows that Korotkov's point about the generic GiST estimator (message #4) is unanswered. standards-compat-1 shows that the opclass cannot be removed once indexes and dumps name it. Confidence is medium because only 3 of 10 messages were visible.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code I cannot confirm that gist_trgm_ops_noeq has its own opfamily, which keeping = away from the planner requires. Messages #4 to #10 were not visible, so the claims about them rest on the objections and the brief.

## The proposal, restated

**Problem.** When a text column has both a btree index and a gist_trgm_ops index, the planner can pick the GiST index for a plain = query. The two plans have almost equal estimated cost, but the GiST plan ran about 355 times slower in the author's test. Users who keep the trigram index only for LIKE or similarity searches are hit.

**Proposed design.** pg_trgm 1.7 adds a second opclass, gist_trgm_ops_noeq, identical to gist_trgm_ops except that it does not register operator 11 (=). Users create new indexes with it or rebuild existing ones onto it, so the planner can never use that index for =. The patch is pure SQL in the extension upgrade script, with no core, C or planner changes.

**User-visible surface.**
- New GiST operator class gist_trgm_ops_noeq in pg_trgm.
- pg_trgm extension version bump to 1.7, with upgrade script pg_trgm--1.6--1.7.sql.
- New paragraph in the pg_trgm docs describing the new opclass.

**What the author claims.**
- The GiST plan for = can be hundreds of times slower than btree; the test case shows 15.2 ms against 0.043 ms.
- Existing gist_trgm_ops indexes and behavior do not change.
- Correctness can be checked by reading the catalog entries alone.
- Tests show = cannot use the new opclass under any planner settings, and other operators still return correct results.
- David Rowley reported the same problem in 2024 and got no replies.

**Thread state.** Korotkov prefers fixing the cost estimate (option 2, or opclass statistics) and argues GiST cost for = should already be higher than btree. Dunstan calls an ANALYZE-time AM hook the real fix but a large project, and suggested a CF entry for patch 3; v2 fixes the Meson build and has no other review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-02; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Opclass twin works around a generic GiST costing gap

fixable · level 2, rubric item 5 · maintenance burden ("who maintains this in 10 years"), planner behaviour change · `maintenance-fit-1`

A user with btree_gist, or another opclass whose = or other operators are costed like btree, hits the same plan flip. Once gist_trgm_ops_noeq exists, the next fix proposed for that opclass is another "_noeq" twin. Each one adds names that users must know about, while gistcostestimate stays unchanged.

In message #4 Korotkov says GiST cost for = should already be above btree under the generic estimator, because GiST calls the consistent function for every item on a page. That points to a gistcostestimate change that would help every opclass. The thread never answers this point, and message #6 compares patch 3 only with the ANALYZE hook. A committer would likely ask whether ten such per-opclass twins per release would be acceptable.

Thread: [#4 (Alexander Korotkov, 2026-09-03)](https://www.postgresql.org/message-id/CAPpHfdtGzomdU%2B0wiwrZS52NRQYwMrzDraE0eCfwAM2RvQAcag%40mail.gmail.com), [#6 (Vaibhav Dalvi, 2026-09-18)](https://www.postgresql.org/message-id/CA%2BvB%3DAG1P9FMm30ov_w4%2B%2ByNZ9Cdy40hYdC0tLxxPT2GEQaQxQ%40mail.gmail.com)

**Question:** Why does a per-item consistent-call charge in gistcostestimate not fix this case, and which other opclasses would then not need a similar twin?

Counter-check: held. No message answers Korotkov's point in message #4 that the generic GiST estimator should charge more for =. The btree_gist scenario has no support in the material, so only the unanswered gistcostestimate question should be used.

### 2. Workaround opclass name becomes a permanent part of every dump

fixable · level 2, rubric item 6 · pg_dump / restore round-trip, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A user builds indexes with gist_trgm_ops_noeq in Postgres 20. A later release fixes GiST cost estimation for =, and pg_trgm wants to drop the extra opclass. Every dump and pg_upgrade from older clusters names it in CREATE INDEX, so removing it breaks restores, and the opclass stays for good.

Message #3 calls an ANALYZE-time hook the real fix, and message #4 argues that even the generic GiST estimator should charge more for =. Either fix would make this opclass redundant. The opclass name encodes a planner workaround in user DDL, and pg_dump writes it out for every index that uses it. The proposal gives no deprecation or retirement plan and does not say what happens to it once the cost model is fixed.

Thread: [#3 (Andrew Dunstan, 2026-09-02)](https://www.postgresql.org/message-id/69c88bb3-d9a8-4054-9024-1f324d31e5aa%40dunslane.net), [#4 (Alexander Korotkov, 2026-09-03)](https://www.postgresql.org/message-id/CAPpHfdtGzomdU%2B0wiwrZS52NRQYwMrzDraE0eCfwAM2RvQAcag%40mail.gmail.com), [#6 (Vaibhav Dalvi, 2026-09-18)](https://www.postgresql.org/message-id/CA%2BvB%3DAG1P9FMm30ov_w4%2B%2ByNZ9Cdy40hYdC0tLxxPT2GEQaQxQ%40mail.gmail.com)

**Question:** If GiST cost estimation for = is fixed later, how would gist_trgm_ops_noeq be retired without breaking dumps and pg_upgrade of existing indexes?

Counter-check: held. Index definitions name the opclass, so pg_trgm cannot drop it once users build on it. Message #7 says it costs almost nothing, but no message says what happens to it after a cost-model fix.

### 3. Users can already define this opclass themselves with plain SQL

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A user hit by the slow = plan can run `create operator class` today, reuse pg_trgm's support functions, leave out operator 11, and reindex. If pg_trgm ships gist_trgm_ops_noeq instead, the project keeps a second opclass in every later pg_trgm version, even after a cost-model fix makes it unneeded.

Message #3 says users can create the opfamily and opclass for themselves, as in patch 3, and rebuild their indexes. So shipping the opclass adds convenience and a documented name, not something that is impossible today. Message #6 argues that users should not have to hand-roll it, but neither message gives field reports or a count of affected users beyond the 2024 report that drew no replies. Users who need only LIKE or regex matching, and not the <-> ordering, can also use gin_trgm_ops, whose cost estimate message #1 says is already fixed.

Thread: [#1 (Vaibhav Dalvi, 2026-09-02)](https://www.postgresql.org/message-id/CA%2BvB%3DAHon6EB%3D90pbPDik_u8%2B%2B%2BTy8TA6Z4-RXo_X5QCr71kcQ%40mail.gmail.com), [#3 (Andrew Dunstan, 2026-09-02)](https://www.postgresql.org/message-id/69c88bb3-d9a8-4054-9024-1f324d31e5aa%40dunslane.net), [#4 (Alexander Korotkov, 2026-09-03)](https://www.postgresql.org/message-id/CAPpHfdtGzomdU%2B0wiwrZS52NRQYwMrzDraE0eCfwAM2RvQAcag%40mail.gmail.com), [#6 (Vaibhav Dalvi, 2026-09-18)](https://www.postgresql.org/message-id/CA%2BvB%3DAG1P9FMm30ov_w4%2B%2ByNZ9Cdy40hYdC0tLxxPT2GEQaQxQ%40mail.gmail.com), [#7 (Andrew Dunstan, 2026-10-03)](https://www.postgresql.org/message-id/ab8db543-bf0c-4a80-8d04-fe6eac817f3c%40dunslane.net)

**Question:** How many users hit this and cannot use GIN, and why is a documented user-defined opclass recipe not enough for them?

Counter-check: held. Message #3 confirms users can build the same opclass today, and the thread shows only two reports. One limit applies: `create operator class` needs superuser, so the recipe does not reach users without that privilege.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | pg_trgm--1.6--1.7.sql adds opclass and operator entries for gist_trgm_ops_noeq. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Workaround opclass name becomes a permanent part of every dump |
| new GUC | no |  |
| planner behaviour change | **yes** | The planner can no longer consider an index built with the new opclass for = conditions. |
| pg_dump / restore round-trip | **yes** | pg_dump writes gist_trgm_ops_noeq into CREATE INDEX, so restores need pg_trgm 1.7 or later. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The fix is an extension version bump to 1.7, so released branches with the same plan problem do not get it. |
| maintenance burden ("who maintains this in 10 years") | **yes** | pg_trgm must ship and document two nearly identical GiST opclasses in every later version. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #3 says users can create the same opclass themselves with plain SQL today. |

## 5. Questions the author must answer before anyone reviews code

1. Why does charging for a consistent call per index item in gistcostestimate, as message #4 suggests, not make btree win for = in the test case?
2. If GiST cost estimation for = is fixed later, how would gist_trgm_ops_noeq be retired without breaking dumps and pg_upgrade of existing indexes?
3. Which users hit this plan and cannot use gin_trgm_ops or a self-made opclass, for example because they need <-> ordering or lack superuser?
4. Why is a documented recipe for a user-defined opclass in the pg_trgm docs not enough?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andrew Dunstan reviewed the patch and supports patch 3 as a stopgap in message #3; Alexander Korotkov, who takes part in the thread, prefers a fix to the cost estimate.
- **Liability:** The owner signs for a second pg_trgm GiST opclass that stays for as long as any index or dump names it, including after a later cost-model fix.
- **Author follow-up:** The author posted v2 with a Meson build fix and opened the CommitFest entry, which suggests continued work.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 9 KiB of SQL, docs and tests in contrib/pg_trgm, and it affects only users with both a btree and a gist_trgm_ops index on one column. The thread must first choose between this opclass and a fix to the GiST cost estimate.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` New opclass is permanent with no retirement path. Refuted (not specific): Duplicate of standards-compat-1, which states the same retirement problem with a more concrete dump and pg_upgrade scenario.
- `demand-1` Built-in opclass only helps users who already diagnosed the problem. Refuted (not specific): Duplicate of already-exists-1, which makes the same demand and self-made opclass argument. The added point, that only users who diagnosed the plan benefit, holds for any opt-in opclass.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposes a second trigram opclass without = or rejects that idea. [P16] (AM-specific statistics, returned with feedback) is closer to Dunstan's ANALYZE-hook idea in message #3, and it records no reason. The thread cites an unanswered 2024 report of the same problem (message #1) and an uncommented patchset (message #5), but neither gives a rejection reason.
- standards-compat: 1 objection(s). Existing gist_trgm_ops behaviour is unchanged, and the change adds no grammar, keywords or GUCs. A dump that uses the new opclass fails on a pg_trgm 1.6 server, but that holds for any new extension object.
- replication-ondisk: 0 objection(s). The patch adds catalog rows (opclass and operator entries) through an ordinary pg_trgm upgrade script; the index pages, WAL records, core catalogs and wire protocol do not change. pg_upgrade and pg_dump treat the new opclass like any other extension member, and a standby receives the rows through normal WAL replay. Without the code I cannot confirm that the opclass has its own opfamily, which keeping = out of the planner requires.
- security: 0 objection(s). The patch only adds an operator class that reuses existing pg_trgm operators and support functions, with no new C code. I see no new privilege, data exposure, escalation path or DoS surface; the upgrade script's handling of a trusted extension's search_path needs the code to check.
- maintenance-fit: 2 objection(s). No hot-path cost: the patch is SQL only, and only users who choose the new opclass are affected. Dunstan, a committer, supports adding it as a stopgap (messages #3 and #7), which lowers confidence in both objections.
- already-exists: 1 objection(s). One objection only. The GIN alternative does not cover users of GiST <-> ordering, so it supports the objection but does not stand alone.
- demand: 1 objection(s). Demand is thin but real, and the cost to non-users is small, so this is a not-now, show-demand point. Dunstan, a committer, supports the patch in message #7 as cheap.

### Precedents retrieved

- P1: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P2: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P3: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant; reason summarised from its thread
- P4: [Operation log for major operations](https://commitfest.postgresql.org/patch/4018/) (cf, Withdrawn, 2023-03-02) - judged not relevant; reason summarised from its thread
- P5: [Better estimate for filter cost](https://commitfest.postgresql.org/patch/1296/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P6: [Add estimated hit ratio to Memoize in EXPLAIN to explain cost adjustment](https://commitfest.postgresql.org/patch/4236/) (cf, Returned with feedback, 2023-09-01) - judged not relevant; reason summarised from its thread
- P7: [Named Operators](https://commitfest.postgresql.org/patch/4123/) (cf, Returned with feedback, 2023-07-04) - judged not relevant; reason summarised from its thread
- P8: [Add OR REPLACE option for CREATE OPERATOR](https://commitfest.postgresql.org/patch/3758/) (cf, Rejected, 2023-03-22) - judged not relevant; reason summarised from its thread
- P9: [pgbench - more operators and functions](https://commitfest.postgresql.org/patch/596/) (cf, Returned with feedback, 2017-01-25) - judged not relevant
- P10: [Range phrase operator in tsquery](https://commitfest.postgresql.org/patch/1624/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P11: [ANALYZE: hash-accelerate MCV tracking for equality-only types](https://commitfest.postgresql.org/patch/6422/) (cf, Withdrawn, 2026-05-03) - judged not relevant; reason summarised from its thread
- P12: [Applying PMDK to WAL operations for persistent memory](https://commitfest.postgresql.org/patch/1485/) (cf, Returned with feedback, 2019-01-23) - judged not relevant
- P13: [Add GiST support for mixed-width integer operators](https://commitfest.postgresql.org/patch/5113/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P14: [Improving the heapgetpage function improves performance in common scenarios](https://commitfest.postgresql.org/patch/4524/) (cf, Withdrawn, 2023-09-06) - judged not relevant
- P15: [\[PATCH\] Equivalence Class Filters](https://commitfest.postgresql.org/patch/3524/) (cf, Returned with feedback, 2022-12-17) - judged not relevant
- P16: [Allow index AMs to calculate custom statistics / BRIN](https://commitfest.postgresql.org/patch/3952/) (cf, Returned with feedback, 2022-12-17) - used in the review
- P17: [Improvements in Copy From](https://commitfest.postgresql.org/patch/2642/) (cf, Withdrawn, 2020-09-27) - judged not relevant
- P18: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P19: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant
- P20: [Opclass parameters](https://commitfest.postgresql.org/patch/1559/) (cf, Returned with feedback, 2018-03-01) - judged not relevant
- P21: [Add sortsupport for range types and btree_gist](https://commitfest.postgresql.org/patch/3686/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P22: [RADIUS tests and improvements](https://commitfest.postgresql.org/patch/4103/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P23: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P24: [Opclass parameters](https://commitfest.postgresql.org/patch/2183/) (cf, Returned with feedback, 2019-12-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 150 s; cost reported by the CLI: $1.81
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 70 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:50:19.139Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
