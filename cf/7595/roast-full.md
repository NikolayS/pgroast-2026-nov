# Design pre-review: MERGE fails to reject source references in NOT MATCHED BY SOURCE actions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7595](https://commitfest.postgresql.org/patch/7595/) (Needs review); authors: Tender Wang (tndrwang)
- Thread: [MERGE fails to reject source references in NOT MATCHED BY SOURCE actions](https://www.postgresql.org/message-id/flat/CAHewXNmFA5dWp7AUivBjqLdjA3GyG4uVS2XDYdf%3DrChU-KMXGQ%40mail.gmail.com), started 2026-11-05
- Messages read: 1 of 1
- Patch prose read: 0001-Fix-source-visibility-in-MERGE-NOT-MATCHED-BY-SOURCE.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

No surviving objection questions the goal or the design: the patch makes an unaliased join source follow the rule that a single-table source already follows. The one surviving objection, standards-compat-1, is a fixable back-patch question that a committer who knows the MERGE parser must decide. Confidence is low because only message #1 exists and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it cannot be checked that the per-action namespace is restored on every path, or that aliased joins, subqueries, nested joins and NOT MATCHED BY TARGET actions keep their current behaviour.

## The proposal, restated

**Problem.** MERGE accepts references to source relations in NOT MATCHED BY SOURCE actions when the USING source is an unaliased join. Postgres rejects the same reference when USING names a single table. The query from the report returns "MERGE 0" with no error, so a user can write an action that can never see source rows.

**Proposed design.** setNamespaceForMergeWhen() hides only the top-level source RTE, so the input relations of an unaliased join stay visible by name. The patch restricts the namespace for each action by its match kind and restores it afterwards, as transformJoinOnClause() does. It uses the complete source namespace so the visibility rules from transformFromClause() for join inputs and aliases stay the same.

**User-visible surface.**
- MERGE with an unaliased join source now raises "invalid reference to FROM-clause entry" for source references in NOT MATCHED BY SOURCE UPDATE expressions.
- The same error now applies to source references in WHEN conditions of NOT MATCHED BY SOURCE actions.

**What the author claims.**
- Source references in NOT MATCHED BY SOURCE actions should be rejected whether USING is a single table or an unaliased join.
- Source references in WHEN conditions are also accepted by mistake today.
- The fix keeps the source visibility flags set by transformFromClause().
- Regression tests cover the case.

**Thread state.** Only the first message is visible. No one has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-05; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-patch policy for newly rejected MERGE queries is not stated

fixable · level 2, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

An application runs a MERGE with an unaliased join source that references a source column in a NOT MATCHED BY SOURCE action. After a minor-release update, that MERGE fails with "invalid reference to FROM-clause entry". The user finds out when the batch job errors in production.

The patch turns queries that parse and run today into parse errors, and it is tagged as a bugfix. Message #1 does not say which branches have the bug or whether the fix goes back. Rejecting these queries on master is consistent with the single-table case. On a stable branch, a committer would likely weigh the error against the current silent behaviour, where the source column has no matching source row and so cannot hold a source value.

Thread: [#1 (Tender Wang, 2026-11-05)](https://www.postgresql.org/message-id/CAHewXNmFA5dWp7AUivBjqLdjA3GyG4uVS2XDYdf%3DrChU-KMXGQ%40mail.gmail.com)

**Question:** Which branches should get this fix, and how does it weigh breaking running queries on a minor update against the silent result those queries give today?

Counter-check: held. Message #1 and the commit message name no branches and do not discuss back-patching, yet the patch makes queries that run today fail at parse time. The scenario is concrete and tied to this change, and a statement from the author would answer it, so it stays fixable.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch changes which MERGE statements with a join source pass parse analysis; it adds no syntax. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, and a back-patch would make MERGE statements that run today fail at parse time. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which released branches accept these source references today, and do you propose to back-patch the fix or apply it to master only?
2. If back-patched, how do you weigh a new parse error on a minor update against the result these MERGE statements give today?
3. How do an aliased join, a subquery and nested joins in USING behave before and after the patch?
4. Do NOT MATCHED BY TARGET and MATCHED actions see the same namespace after the patch as before, including target references?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows MERGE parse analysis and the parser namespace rules in transformFromClause() would have to own it; no one has replied to message #1.
- **Liability:** The owner signs for the new namespace handling in parse_merge.c and, if back-patched, for any MERGE statement that starts to fail on a minor update.
- **Author follow-up:** No sign either way; the thread has only message #1.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is small, touches only parse_merge.c and the merge regression test, and adds no user-visible surface beyond an error. It affects only MERGE with an unaliased join source and a NOT MATCHED BY SOURCE action.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: none of the 24 retrieved rows covers the same or a similar idea, and message #1 points to no earlier discussion. The patch makes MERGE with an unaliased join source match the single-table behaviour, so no earlier rejection reason applies.
- standards-compat: 1 objection(s). From the standards and compatibility angle, the change aligns the join case with the existing single-table rule and adds no syntax, GUC or dumped object. The only open point is the back-patch policy.
- replication-ondisk: 0 objection(s). The patch changes only parse-analysis namespace visibility in parse_merge.c plus regression tests. It touches no on-disk format, WAL, catalog, wire protocol or extension ABI, so nothing on this angle needs review. Stored rules or views cannot contain MERGE today, so no persisted parse trees are affected.
- security: 0 objection(s). The patch only makes MERGE reject source column references in NOT MATCHED BY SOURCE actions when the source is a join. It does not change privileges, RLS, column privilege checks, or what data a user can see, and it adds no new resource use. So there is no security or privilege objection from this angle.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The patch is a parser bug fix in parse_merge.c that reuses the save-and-restore namespace pattern of transformJoinOnClause(), costs only parse time and adds no GUC, catalog or API. Without the code, two points cannot be checked: whether NOT MATCHED BY TARGET actions keep their namespace, and how aliased joins, subqueries and nested joins behave. No committer has replied yet, so no owner is known. Back-patching would make queries that run today fail; that question belongs to the compatibility angle.
- already-exists: 0 objection(s). This is a parser bug fix that makes an unaliased join source in MERGE behave like a single-table source, which already raises an error. No existing feature, extension or hook does this job, and the patch adds no new mechanism. Its namespace handling follows transformJoinOnClause(). Back-patch impact and coverage of aliased joins and subqueries belong to other angles.
- demand: 0 objection(s). This is a bug fix. It makes an unaliased join source behave like the single-table case, which Postgres already rejects, so the demand angle does not apply. Users and maintainers pay no new cost. The only open demand-related point is whether to back-patch, since queries that run today would start to fail. That is a compatibility question for other angles.

### Precedents retrieved

- P1: [Revert MERGE patch](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=08ea7a2291db21a618d19d612c8060cda68f1892) (revert, Reverted, 2018-04-12) - judged not relevant
- P2: [some namespace.c refactoring](https://commitfest.postgresql.org/patch/4212/) (cf, Returned with feedback, 2024-01-26) - judged not relevant; reason summarised from its thread
- P3: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P4: [XML XPath default namespace support](https://commitfest.postgresql.org/patch/1085/) (cf, Rejected, 2019-02-04) - judged not relevant
- P5: [Revert "Forbid DROP SCHEMA on temporary namespaces"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65192e02441cedd106b6abebe0036fb8cc124fb3) (revert, Reverted, 2020-01-08) - judged not relevant
- P6: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P7: [Revert support for ALTER TABLE ... MERGE/SPLIT PARTITION(S) commands](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3890d90c1508125729ed20038d90513694fc3a7b) (revert, Reverted, 2024-08-24) - judged not relevant
- P8: [Better estimate merging for duplicate vars in clausesel.c](https://commitfest.postgresql.org/patch/1036/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P9: [possibility to rename root namespace in plpgsql](https://commitfest.postgresql.org/patch/2839/) (cf, Rejected, 2022-03-07) - judged not relevant; reason summarised from its thread
- P10: [Permute underscore separated components of columns before fuzzy matching](https://commitfest.postgresql.org/patch/4282/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P11: [Range Merge Join](https://commitfest.postgresql.org/patch/3160/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P12: [Indeterminate behavior for the GROUP BY clause](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P13: [Fix tab completion MERGE](https://commitfest.postgresql.org/patch/3890/) (cf, Returned with feedback, 2023-07-04) - judged not relevant; reason summarised from its thread
- P14: [Make selectivity example match wording](https://commitfest.postgresql.org/patch/3695/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P15: [Add pg_upgrade check for invalid role references in pg_init_privs](https://commitfest.postgresql.org/patch/6856/) (cf, Withdrawn, 2026-06-17) - judged not relevant; reason summarised from its thread
- P16: [Fixes a trivial bug in dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5996/) (cf, Withdrawn, 2025-09-02) - judged not relevant
- P17: [\[PATCH v1\] psql: Do not complete HEADER MATCH for COPY TO](https://commitfest.postgresql.org/patch/7585/) (cf, Withdrawn, 2026-11-04) - judged not relevant
- P18: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant
- P19: [Support specify tablespace for each merged/split partition](https://commitfest.postgresql.org/patch/5157/) (cf, Rejected, 2024-10-08) - judged not relevant
- P20: [Range Merge Join](https://commitfest.postgresql.org/patch/1106/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P21: [Add Option to Check All Addresses For Matching target_session_attr](https://commitfest.postgresql.org/patch/5396/) (cf, Rejected, 2026-10-20) - judged not relevant
- P22: [Docs: make source file path references consistent](https://commitfest.postgresql.org/patch/6380/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P23: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [Range Merge Join](https://commitfest.postgresql.org/patch/1449/) (cf, Returned with feedback, 2018-03-28) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 105 s; cost reported by the CLI: $0.76
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:03:43.299Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
