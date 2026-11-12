# Design pre-review: Proposal: QUALIFY clause

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6954](https://commitfest.postgresql.org/patch/6954/); [thread](https://www.postgresql.org/message-id/flat/CAFY6G8euMWRjTEdRV9M%3DRL2_EJNDm5UNeHat9U1FGGwzfjGSfw%40mail.gmail.com), 40 of 40 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread, on the condition that the standard reserves QUALIFY. The list has not chosen between the rewrite patch and the planner-based patch, and each has an open design cost: lost view deparse (standards-compat-2) or duplicated pushdown code (maintenance-fit-1). The claim of standard acceptance also has no source yet (prior-attempts-1). Code review makes sense after these are settled.

Not checked: Without the code, it cannot be checked that the rewrite keeps column-privilege checks for columns used only in QUALIFY, or that the rewritten query gives the same results in all cases with DISTINCT, aggregates and set operations.

## Prior attempts

None found before 2025-07-21.

## Objections a reviewer is likely to raise

### 1. Rewrite design stores views in subquery form, against the reviewer's request

A user creates a view with QUALIFY, then reads it back with \d+ or pg_dump and gets a nested subquery with generated column names for the projected window functions. If the project later moves to the planner design, the same view deparses differently across major versions. Tools that compare schema dumps then report false differences.

In message #30 Vik Fearing asked that ruleutils print QUALIFY as written, and that any transformation happen in the rewriter. The complete patch rewrites the query during parse analysis instead, and message #38 names the deparse change as its main downside. Choosing the rewrite design fixes the stored form of every QUALIFY view, so a later change of design also changes pg_dump output for views users already have. Neither patch says whether a dumped and restored view keeps its column names and its alias-versus-input-column resolution.

**Question:** Does a QUALIFY view dumped by pg_dump restore with the same column names and meaning, and which stored form will views keep if the project later switches to the planner design?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#30](https://www.postgresql.org/message-id/c5e9fe6e-0f93-4933-97f1-85a0acc774be%40postgresfriends.org), [#34](https://www.postgresql.org/message-id/b4ed9908-f000-4a5d-aae8-625607f80757%40postgresfriends.org), [#38](https://www.postgresql.org/message-id/DJLM1EHPGO3K.3PDCCHL8OZ08V%40gmail.com); `standards-compat-2`</sub>

### 2. Planner-based design copies window pushdown logic into a second place

A later patch improves run conditions or qual pushdown below WindowAgg in set_subquery_pathlist() and does not update the QUALIFY path. Hand-written subqueries then get the new plan and QUALIFY queries do not. Nothing fails, so users find out only when they compare EXPLAIN output.

The author leans toward the planner-based patch (message #38) and says it duplicates the run-condition and qual-pushdown code. David Rowley (message #36) said acceptance depends on how much code is reused, and he named a past optimization that would have needed a second copy. Richard Guo (message #37) asked that QUALIFY inherit every existing window optimization. The refactoring of check_and_push_window_quals() into shared helpers is proposed but has not been done.

**Question:** Can the subquery pushdown helpers be refactored first, as a separate patch, so that QUALIFY and subqueries share a single code path?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#36](https://www.postgresql.org/message-id/CAApHDvq5UTReLSdwEL9WnmDEW1VDg-EYTmszPVOGwf8qH-cj9Q%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/CAMbWs48vmkAHpdPoX9_gnbcVMv8c2w2Y6UAFzZjTynz9fVGsuA%40mail.gmail.com), [#38](https://www.postgresql.org/message-id/DJLM1EHPGO3K.3PDCCHL8OZ08V%40gmail.com); `maintenance-fit-1`</sub>

### 3. Standardization condition from 2025 is claimed but not yet shown

An application has a table or column named qualify. After an upgrade to a release with this patch, its unquoted queries fail with syntax errors. If the standard later changes the keyword or its scope, the project cannot cite the standard to explain the break.

In messages #4 and #6, Tom Lane accepted the reserved word only once QUALIFY is in the standard, so the breakage can be blamed on SQL:20xx. Vik Fearing agreed in messages #11 and #26 to hold back until then. Message #38 says the proposal is accepted as feature O001 but gives no reference, and the conformance doc hunk is truncated. No reviewer has confirmed the claim since the revival, so the condition from 2025 is still open.

**Question:** Which standard document or committee decision records QUALIFY as feature O001, with the keyword reserved and the semantics the patch implements?

<sub>fixable, level 1b, prior rejection; messages [#4](https://www.postgresql.org/message-id/440577.1753108882%40sss.pgh.pa.us), [#6](https://www.postgresql.org/message-id/463108.1753119052%40sss.pgh.pa.us), [#9](https://www.postgresql.org/message-id/88d8f6c9-d19d-4cee-bbc5-826987c14587%40postgresfriends.org), [#11](https://www.postgresql.org/message-id/0b190baa-460d-48a8-9f7f-ca8c8cfe13bd%40postgresfriends.org), [#26](https://www.postgresql.org/message-id/737a77a2-83eb-4b97-a109-e58d3ce1189a%40postgresfriends.org), [#38](https://www.postgresql.org/message-id/DJLM1EHPGO3K.3PDCCHL8OZ08V%40gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. Which standard document or committee decision records QUALIFY as feature O001, with the keyword reserved and the semantics the patch implements?
2. Does a view defined with QUALIFY dump and restore with the same column names and the same alias-versus-input-column resolution under the rewrite patch?
3. Can the subquery pushdown helpers be refactored first, as a separate patch, so that QUALIFY and subqueries share one code path?
4. How does the planner-based patch filter QUALIFY conditions that contain no window function?
5. Is the crash David Rowley reported with an alias in QUALIFY fixed in v4, and does a regression test cover it?

## Touch points

- **ABI / extension API:** Both patches change parsenodes.h, and the planner-based patch also changes optimizer/paths.h.
- **SQL-standard conflict or non-standard syntax:** The patch adds a QUALIFY clause and makes QUALIFY a fully reserved keyword.
- **planner behaviour change:** The planner-based patch adds QUALIFY pushdown and run conditions in allpaths.c and planner.c.
- **pg_dump / restore round-trip:** With the rewrite patch, a view defined with QUALIFY deparses and dumps in subquery form.
- **backpatchability:** The new reserved word breaks unquoted uses of qualify as an identifier on upgrade.
- **maintenance burden ("who maintains this in 10 years"):** The planner-based patch keeps a second copy of the window pushdown logic that later patches must update.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the grammar and keyword policy, and one who knows window function planning. Committers commented earlier in the thread, but none has posted since the June 2026 revival.
- Priority: fix design first; audience a sizeable minority; review cost medium. Several other systems ship QUALIFY and users migrating from them gain, but the choice between two patches of about 86 KiB each is open. Reviewing either in detail before that choice risks wasted work.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
