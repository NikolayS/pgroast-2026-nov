# Design pre-review: UPDATE with invalid domain constraint

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6031](https://commitfest.postgresql.org/patch/6031/); [thread](https://www.postgresql.org/message-id/flat/CACJufxE2oFcLmrqDrqJrH5k03fv%2Bv9%3D%2B-PBs-mV5WsJ%3D31XMyw%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal: the critic found that the objections based on a different reading of not valid do not hold. What remains is fixable. standards-compat-1 asks whether a stable branch can take a change that makes accepted updates fail. standards-compat-3 and maintenance-fit-3 ask which row-changing paths and cached plans the rewriter change covers. Message #3 is still unanswered, so a committer who knows this area has to decide whether this is a bug.

Not checked: Without the code, this review cannot tell which paths reach the new wrapping step, where the TypeCacheEntry field sits, or whether the flag is reset when the type cache is invalidated. The statement that the documentation supports the change comes from the critic, not from the thread.

## Prior attempts

None found before 2025-08-19.

## Objections a reviewer is likely to raise

### 1. Backporting turns working updates into errors after a minor upgrade

A site has a domain with a not valid constraint and old rows that violate it, and its application runs `update ... set status = ...` on those rows. After a minor upgrade the patch makes those updates fail, even though they never touch the domain column. The site learns this from production errors, with no setting to restore the old behaviour.

The CommitFest entry is tagged Backport and Bugfix with target version stable, but the thread has not agreed that the current behaviour is a bug. Message #3 argues it is the intended meaning of not valid, and nobody answered. Stable branches normally do not take changes that make previously accepted statements fail unless the old behaviour is wrong. The patch prose has no release-note or documentation hunk that explains the change.

**Question:** Why is this a bug fix that is safe for stable branches, and not a behaviour change for master only, given the disagreement in message #3?

<sub>fixable, level 1b, backpatchability; messages [#1](https://www.postgresql.org/message-id/CACJufxE2oFcLmrqDrqJrH5k03fv%2Bv9%3D%2B-PBs-mV5WsJ%3D31XMyw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/7BE9E57C-04A0-41E2-B3D1-074251FA42DE%40gmail.com); `standards-compat-1`</sub>

### 2. Rule covers update in the rewriter but not other row-changing paths

After the patch, `update dt1 set i = i + 1` fails on a violating row. A `merge ... when matched then update set i = i + 1` or `insert ... on conflict do update set i = excluded.i` on the same row may still succeed, and so may a before trigger that changes the row. Users find that the same row change is accepted by one statement and rejected by another.

Message #2 describes the change only in rewriteTargetListIU for update target lists. The material does not say whether merge, on conflict do update, updatable views, or rows changed by before triggers get the same check. If the goal is consistency with insert, a partial rule replaces one inconsistency with several.

**Question:** Do merge, on conflict do update, updatable views and before-trigger changes all apply the same not valid domain check, and is there a test for each?

<sub>fixable, level 2, design; messages [#2](https://www.postgresql.org/message-id/CACJufxGYwfzK-WH2FKtyaKzUAht_KjimGPzkspc4g5RHoerpmQ%40mail.gmail.com); `standards-compat-3`</sub>

### 3. Rewriter hook every update-producing path must remember

A later patch adds or changes a path that builds update target lists, such as merge or insert on conflict do update, and does not call the new wrapping step. Rows pass in one path and fail in another, and no test fails until a user reports the difference. A prepared update cached before `alter domain ... not valid` may also skip the check.

Message #2 places the check in rewriteTargetListIU by adding resjunk entries for unmentioned columns. The proposal does not say how merge, on conflict, views or trigger-changed rows are covered, or whether cached plans are invalidated when the domain flag changes. It also adds a type cache lookup per domain column per update and per-row checks, with no measurement.

**Question:** Which update paths does this cover, how are cached plans invalidated after the constraint is added, and what does it cost per row?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/CACJufxGYwfzK-WH2FKtyaKzUAht_KjimGPzkspc4g5RHoerpmQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/7BE9E57C-04A0-41E2-B3D1-074251FA42DE%40gmail.com); `maintenance-fit-3`</sub>

### 4. New TypeCacheEntry field conflicts with a stable-branch target

An extension is compiled against a minor release and reads TypeCacheEntry fields from lookup_type_cache(). If a backported patch changes the struct layout, that extension reads the wrong fields after the server is updated, and the problem shows up only at run time.

The patch touches src/include/utils/typcache.h, and the brief says it adds a flag to the type cache. Extensions use TypeCacheEntry, and the entry targets stable branches. The proposal does not say how the new field keeps the ABI stable in back branches.

**Question:** If this is backpatched, how does the TypeCacheEntry change keep the layout that already-built extensions rely on?

<sub>fixable, level 2, ABI / extension API; messages [#2](https://www.postgresql.org/message-id/CACJufxGYwfzK-WH2FKtyaKzUAht_KjimGPzkspc4g5RHoerpmQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/7BE9E57C-04A0-41E2-B3D1-074251FA42DE%40gmail.com); `replication-ondisk-2`</sub>

## Questions for the author

1. Why is this a bug fix that is safe for stable branches, and not a change for master only, given the unanswered disagreement in message #3?
2. Which documented rule supports checking a not valid domain constraint on columns the update does not assign, in reply to message #3?
3. Do merge, insert on conflict do update, updatable views and rows changed by before triggers all get the same check, and is each tested?
4. Is a cached update plan rebuilt after `alter domain ... add constraint ... not valid`, so that it gains the CoerceToDomain node?
5. If this is backpatched, where does the new TypeCacheEntry field go so that already-built extensions keep a valid struct layout?

## Touch points

- **ABI / extension API:** The patch adds a flag to TypeCacheEntry in src/include/utils/typcache.h, which extensions read.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Backporting turns working updates into errors after a minor upgrade
- **planner behaviour change:** The rewriter adds CoerceToDomain nodes and resjunk target entries, so update plans and cached plans change.
- **backpatchability:** The CommitFest entry is tagged Backport and targets stable, and the change makes accepted updates fail.
- **maintenance burden ("who maintains this in 10 years"):** The check lives in rewriteTargetListIU, so every path that changes rows must go through it or repeat it.
- **performance on the hot path:** Update rewrite looks up the type cache for domain columns, and affected rows get an extra domain check.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the rewriter target-list code and domain constraint handling in the type cache. No committer has posted in the thread; one only moved the CommitFest entry.
- Priority: needs champion; audience niche; review cost medium. Only sites with not valid domain constraints and violating rows are affected, and the patch is small but changes the rewriter and a type cache struct. The entry has sat through four CommitFests because nobody qualified has said whether this is a bug.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
