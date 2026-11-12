# Design pre-review: UPDATE with invalid domain constraint

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6031](https://commitfest.postgresql.org/patch/6031/) (Needs review); authors: Jian He (stayreal)
- Thread: [UPDATE with invalid domain constraint](https://www.postgresql.org/message-id/flat/CACJufxE2oFcLmrqDrqJrH5k03fv%2Bv9%3D%2B-PBs-mV5WsJ%3D31XMyw%40mail.gmail.com), started 2025-08-19
- Messages read: 3 of 3
- Patch prose read: v1-0001-UPDATE-with-invalid-domain-constraint.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal: the critic found that the objections based on a different reading of not valid do not hold. What remains is fixable. standards-compat-1 asks whether a stable branch can take a change that makes accepted updates fail. standards-compat-3 and maintenance-fit-3 ask which row-changing paths and cached plans the rewriter change covers. Message #3 is still unanswered, so a committer who knows this area has to decide whether this is a bug.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot tell which paths reach the new wrapping step, where the TypeCacheEntry field sits, or whether the flag is reset when the type cache is invalidated. The statement that the documentation supports the change comes from the critic, not from the thread.

## The proposal, restated

**Problem.** After `alter domain ... add constraint ... not valid`, an update that does not assign a new value to a domain column skips the check for that column, even when the old value violates the constraint. Insert does check not valid domain constraints, and so does an update that assigns a new literal. The author asks whether update should also check, citing an earlier remark by Álvaro Herrera.

**Proposed design.** The rewriter checks each Var in the update target list. If its type is a domain with not valid constraints, it wraps the Var in a CoerceToDomain node. A column the statement does not mention gets a new resjunk target entry that does the same. The type cache gains a flag that records whether a domain has not valid constraints.

**User-visible surface.**
- Update now fails when an unchanged domain column, or one set to itself, violates a not valid domain constraint.
- Changed behaviour also covers `set c = c` and `set c = c::d1`.

**What the author claims.**
- Insert already checks not valid domain constraints, so the gap exists only for update.
- Álvaro Herrera raised the same point in an earlier thread.

**Thread state.** Chao Li replied that the current behaviour is arguably correct, because not valid means existing bad data is tolerated while unchanged, and asked for other opinions. He also gave code comments in case it is agreed to be a bug; the author has not replied, and nobody has settled whether this is a bug.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-08-19; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Backporting turns working updates into errors after a minor upgrade

fixable · level 1b, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A site has a domain with a not valid constraint and old rows that violate it, and its application runs `update ... set status = ...` on those rows. After a minor upgrade the patch makes those updates fail, even though they never touch the domain column. The site learns this from production errors, with no setting to restore the old behaviour.

The CommitFest entry is tagged Backport and Bugfix with target version stable, but the thread has not agreed that the current behaviour is a bug. Message #3 argues it is the intended meaning of not valid, and nobody answered. Stable branches normally do not take changes that make previously accepted statements fail unless the old behaviour is wrong. The patch prose has no release-note or documentation hunk that explains the change.

Thread: [#1 (jian he, 2025-08-19)](https://www.postgresql.org/message-id/CACJufxE2oFcLmrqDrqJrH5k03fv%2Bv9%3D%2B-PBs-mV5WsJ%3D31XMyw%40mail.gmail.com), [#3 (Chao Li, 2025-08-20)](https://www.postgresql.org/message-id/7BE9E57C-04A0-41E2-B3D1-074251FA42DE%40gmail.com)

**Question:** Why is this a bug fix that is safe for stable branches, and not a behaviour change for master only, given the disagreement in message #3?

Counter-check: held. The entry targets stable, and the thread has not agreed this is a bug. The `alter domain` documentation says updated rows are always checked, which supports the bug reading, but a minor release would still make accepted updates fail.

### 2. Rule covers update in the rewriter but not other row-changing paths

fixable · level 2, rubric item 13 · design, SQL-standard conflict or non-standard syntax · `standards-compat-3`

After the patch, `update dt1 set i = i + 1` fails on a violating row. A `merge ... when matched then update set i = i + 1` or `insert ... on conflict do update set i = excluded.i` on the same row may still succeed, and so may a before trigger that changes the row. Users find that the same row change is accepted by one statement and rejected by another.

Message #2 describes the change only in rewriteTargetListIU for update target lists. The material does not say whether merge, on conflict do update, updatable views, or rows changed by before triggers get the same check. If the goal is consistency with insert, a partial rule replaces one inconsistency with several.

Thread: [#2 (jian he, 2025-08-20)](https://www.postgresql.org/message-id/CACJufxGYwfzK-WH2FKtyaKzUAht_KjimGPzkspc4g5RHoerpmQ%40mail.gmail.com)

**Question:** Do merge, on conflict do update, updatable views and before-trigger changes all apply the same not valid domain check, and is there a test for each?

Counter-check: held. Merge actions and `on conflict do update` target lists also pass through rewriteTargetListIU, so they may be covered. The material does not confirm this, and rows changed by before triggers do not pass through the rewriter.

### 3. Rewriter hook every update-producing path must remember

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), performance on the hot path · `maintenance-fit-3`

A later patch adds or changes a path that builds update target lists, such as merge or insert on conflict do update, and does not call the new wrapping step. Rows pass in one path and fail in another, and no test fails until a user reports the difference. A prepared update cached before `alter domain ... not valid` may also skip the check.

Message #2 places the check in rewriteTargetListIU by adding resjunk entries for unmentioned columns. The proposal does not say how merge, on conflict, views or trigger-changed rows are covered, or whether cached plans are invalidated when the domain flag changes. It also adds a type cache lookup per domain column per update and per-row checks, with no measurement.

Thread: [#2 (jian he, 2025-08-20)](https://www.postgresql.org/message-id/CACJufxGYwfzK-WH2FKtyaKzUAht_KjimGPzkspc4g5RHoerpmQ%40mail.gmail.com), [#3 (Chao Li, 2025-08-20)](https://www.postgresql.org/message-id/7BE9E57C-04A0-41E2-B3D1-074251FA42DE%40gmail.com)

**Question:** Which update paths does this cover, how are cached plans invalidated after the constraint is added, and what does it cost per row?

Counter-check: held. The cost falls only on domains that have not valid constraints, which weakens the performance part. The proposal does not say whether a cached update plan is rebuilt after `alter domain ... not valid`, since the old plan has no CoerceToDomain node.

### 4. New TypeCacheEntry field conflicts with a stable-branch target

fixable · level 2, rubric item 11 · ABI / extension API, backpatchability · `replication-ondisk-2`

An extension is compiled against a minor release and reads TypeCacheEntry fields from lookup_type_cache(). If a backported patch changes the struct layout, that extension reads the wrong fields after the server is updated, and the problem shows up only at run time.

The patch touches src/include/utils/typcache.h, and the brief says it adds a flag to the type cache. Extensions use TypeCacheEntry, and the entry targets stable branches. The proposal does not say how the new field keeps the ABI stable in back branches.

Thread: [#2 (jian he, 2025-08-20)](https://www.postgresql.org/message-id/CACJufxGYwfzK-WH2FKtyaKzUAht_KjimGPzkspc4g5RHoerpmQ%40mail.gmail.com), [#3 (Chao Li, 2025-08-20)](https://www.postgresql.org/message-id/7BE9E57C-04A0-41E2-B3D1-074251FA42DE%40gmail.com)

**Question:** If this is backpatched, how does the TypeCacheEntry change keep the layout that already-built extensions rely on?

Counter-check: held. The patch touches typcache.h and the entry targets stable branches. A field added at the end of TypeCacheEntry is the usual safe form, but the material does not show where the field goes.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds a flag to TypeCacheEntry in src/include/utils/typcache.h, which extensions read. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Backporting turns working updates into errors after a minor upgrade |
| new GUC | no |  |
| planner behaviour change | **yes** | The rewriter adds CoerceToDomain nodes and resjunk target entries, so update plans and cached plans change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry is tagged Backport and targets stable, and the change makes accepted updates fail. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The check lives in rewriteTargetListIU, so every path that changes rows must go through it or repeat it. |
| performance on the hot path | **yes** | Update rewrite looks up the type cache for domain columns, and affected rows get an extra domain check. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why is this a bug fix that is safe for stable branches, and not a change for master only, given the unanswered disagreement in message #3?
2. Which documented rule supports checking a not valid domain constraint on columns the update does not assign, in reply to message #3?
3. Do merge, insert on conflict do update, updatable views and rows changed by before triggers all get the same check, and is each tested?
4. Is a cached update plan rebuilt after `alter domain ... add constraint ... not valid`, so that it gains the CoerceToDomain node?
5. If this is backpatched, where does the new TypeCacheEntry field go so that already-built extensions keep a valid struct layout?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the rewriter target-list code and domain constraint handling in the type cache. No committer has posted in the thread; one only moved the CommitFest entry.
- **Liability:** The owner signs for a behaviour change in update, possibly in stable branches, where sites with violating rows see new errors after a minor upgrade. The owner also signs for coverage of every row-changing path, with no end date.
- **Author follow-up:** The author moved the entry forward in December 2025 but has not answered message #3, and the patch has needed a rebase since April 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Only sites with not valid domain constraints and violating rows are affected, and the patch is small but changes the rewriter and a type cache struct. The entry has sat through four CommitFests because nobody qualified has said whether this is a bug.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` New not valid contract is not defined or documented. Refuted (wrong facts): The `alter domain` documentation already says newly inserted or updated rows are always checked against all constraints, even not valid ones. Table check constraints follow the same rule, so the proposal has a written contract to follow.
- `replication-ondisk-1` Backport would make existing updates fail after a minor upgrade. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-1` Behaviour change proposed for backport to stable branches. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-2` Special case for domains, unlike not valid table check constraints. Refuted (wrong facts): Postgres evaluates every table check constraint, including not valid ones, on the new row of each update, whichever columns changed. An update of a violating row already fails for tables, so the proposal brings domains in line with them.
- `demand-1` The change breaks the main use of not valid domain constraints. Refuted (wrong facts): The argument rests on table check constraints not rechecking unchanged columns, but they do recheck the whole new row on every update. The documented rule for not valid already says updated rows are checked, so users of table constraints have this behaviour today.
- `demand-2` No user report shows the current behaviour causes a problem. Refuted (not specific): Its failure scenario duplicates standards-compat-1. User demand is a weak test for a change that makes update follow the documented rule that updated rows are always checked.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus search returned no earlier proposal on this or a similar idea. Message #1 points to an earlier remark by Álvaro Herrera on the same point, but the thread shows nothing of what it said or how it ended, so no prior-rejection objection can be raised.
- standards-compat: 3 objection(s). Objections raised; pg_dump round-trip is not affected because the proposal adds no object or syntax.
- replication-ondisk: 2 objection(s). The proposal adds no on-disk format, WAL record, catalog column or protocol change. The new type cache flag lives only in backend memory. The only concerns from this angle are the stable-branch target and the header change.
- security: 0 objection(s). The proposal has no privilege or exposure aspect beyond what domain check constraints already have: the same check expressions already run on insert and on update of assigned columns. More updates may fail or run checks, but no role gains a new ability, and no data becomes visible to anyone new.
- maintenance-fit: 3 objection(s). Objections listed above.
- already-exists: 0 objection(s). The proposal changes how update treats existing domain values under a not valid constraint, and no existing feature or extension makes update do that. `alter domain ... validate constraint` checks existing data, but it rejects the whole statement if old rows violate the constraint. Whether update should check unchanged columns is a question of semantics, raised in message #3, and other angles cover it.
- demand: 2 objection(s). Two objections apply; the precedent search found no related earlier proposal.

### Precedents retrieved

- P1: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P2: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant; reason summarised from its thread
- P3: [Add support for ON UPDATE/DELETE actions on ALTER CONSTRAINT](https://commitfest.postgresql.org/patch/1533/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P4: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P5: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P6: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant; reason summarised from its thread
- P7: [Allow values to be cast to domain types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [Revert psql bits to display NOT VALID for FKs](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=135a507991c8ea9eca201cc084294a689ca24ba9) (revert, Reverted, 2011-06-08) - judged not relevant
- P9: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P10: [Allow functions defined as casts to domains to be called during casting](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P11: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P12: [Revert "Secure Unix-domain sockets of "make check" temporary clusters."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8f5578d0f9681ef81bc71a3762a191d66a29c8b1) (revert, Reverted, 2014-03-29) - judged not relevant
- P13: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P14: [Revert "Rewrite some RI code to avoid using SPI"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a90641eac24dfc8889122d88eb7f482cd3db8b39) (revert, Reverted, 2022-04-07) - judged not relevant
- P15: [Reimplement targetlist SRFs using ROWS FROM ()](https://commitfest.postgresql.org/patch/759/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P16: [Allow AddForeignUpdateTargets to add extra target expressions other than columns](https://commitfest.postgresql.org/patch/1819/) (cf, Returned with feedback, 2019-11-27) - judged not relevant; reason summarised from its thread
- P17: [Rewrite the code in a different language](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant; reason summarised from its thread
- P18: [Bug in query rewriter - hasModifyingCTE not getting set](https://commitfest.postgresql.org/patch/3134/) (cf, Returned with feedback, 2021-09-08) - judged not relevant
- P19: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant
- P20: [Reduce lock level for ALTER TABLE ... ADD CHECK](https://commitfest.postgresql.org/patch/3088/) (cf, Withdrawn, 2021-11-21) - judged not relevant
- P21: [Patch to clean Query after rewrite-and-analyze](https://commitfest.postgresql.org/patch/2229/) (cf, Withdrawn, 2019-08-05) - judged not relevant
- P22: [Add ALWAYS DEFERRED option for CONSTRAINTs and CONSTRAINT TRIGGERs](https://commitfest.postgresql.org/patch/1319/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P23: [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P24: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 189 s; cost reported by the CLI: $1.66
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 92 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:38:09.942Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
