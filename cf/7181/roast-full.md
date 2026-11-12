# Design pre-review: SQL/JSON boolean DEFAULT expression silently replaced with 'false'

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7181](https://commitfest.postgresql.org/patch/7181/) (Needs review); authors: Man Zeng (zengman)
- Thread: [BUG #19625: SQL/JSON boolean DEFAULT expression silently replaced with 'false'](https://www.postgresql.org/message-id/flat/19625-683b498c92087bc8%40postgresql.org), started 2026-08-17
- Messages read: 5 of 5
- Patch prose read: json-bool-default-fix.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

This is a wrong-result fix in one parser function, and two bug reports show users hit it. The only surviving objection, replication-ondisk-1, is a fixable follow-up about views and rules that keep the stored 'false'::jsonb. It asks for release-note guidance and does not question the fix. A committer who knows the area plans to commit and back-patch it.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check how ruleutils deparses the new to_jsonb() wrapper in a dumped view. Messages #4 and #5 were not shown, so the v2 details rest on the brief.

## The proposal, restated

**Problem.** When a DEFAULT expression in JSON_VALUE or JSON_QUERY is boolean-valued and the RETURNING type is neither boolean nor integer, the parser replaces it with jsonb 'false' without evaluating it. So DEFAULT true ON ERROR returns false. Any user of SQL/JSON functions with a boolean DEFAULT gets wrong results with no error.

**Proposed design.** transformJsonBehavior() in parse_expr.c wraps a boolean-valued DEFAULT expression in a to_jsonb() call, so json_populate_type() gets the value the expression actually evaluates to. In v2, Amit Langote keeps the existing jsonb Const for the canned TRUE and FALSE behaviors, so their node tree in back branches does not change.

**User-visible surface.**
- JSON_VALUE and JSON_QUERY with a boolean DEFAULT ... ON ERROR or ON EMPTY return the evaluated value instead of false.
- Stored views and rules created before the fix still contain 'false'::jsonb, per the v2 commit message.

**What the author claims.**
- Non-boolean DEFAULT expressions, and boolean ones with RETURNING boolean or integer, already work correctly.
- to_jsonb() handles both canned TRUE/FALSE constants and user-supplied boolean expressions, since there is no bool to jsonb cast.
- A reviewer's 40-case test matrix went from 13 to 40 passing, and all nine JSON regression tests passed.
- v2 also fixes the cases in bug #19822.

**Thread state.** Amit Langote posted v2 on 2026-11-12 and says he intends to commit it and back-patch to v17. An AI-assisted review on 2026-10-12 found no blocking issue and asked for more tests, which v2 adds.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-17; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Stored views keep the wrong 'false'::jsonb after the fix, with no detection path

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, on-disk format / pg_upgrade, catalog changes, backpatchability · `replication-ondisk-1`

A user has a view with JSON_VALUE(... DEFAULT true ON ERROR) created on 17.x. They install the minor release with the fix, and later use pg_upgrade or dump and restore to move to v20. The view still returns false on error, and the user has no query that finds such views.

Message #5 says views and rules created before the fix keep the wrong 'false'::jsonb in their stored parse tree. The deparsed definition then shows DEFAULT 'false'::jsonb, so pg_dump, pg_upgrade and a physical standby all carry the wrong value forward as if the user wrote it. The fix is back-patched to v17, so many clusters may already hold affected rules. The thread does not say whether the release notes will tell users to recreate these views or how to find them.

Thread: [#5 (Amit Langote, 2026-11-12)](https://www.postgresql.org/message-id/CA%2BHiwqGO9i5MUhOnnnXXrvKpou0A99wCfOuXrJcXRvTecmzTTg%40mail.gmail.com)

**Question:** Will the release notes give a catalog query that finds affected views and rules, and tell users to recreate them before or after upgrading?

Counter-check: held. Message #5 confirms that views and rules created before the fix keep 'false'::jsonb, and the v2 commit message only records that fact. The thread gives no way to find such views and no advice to recreate them, so this stands as a release-note follow-up.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Stored rules created before the fix keep 'false'::jsonb, and pg_upgrade carries them forward unchanged. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | No catalog change, but stored parse trees in pg_rewrite differ before and after the fix. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The fix changes the result of SQL/JSON DEFAULT ... ON ERROR and ON EMPTY to the evaluated value. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Views created before the fix dump with the stored 'false'::jsonb default, so restore keeps the wrong value. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The committer plans to back-patch to v17; v2 keeps the Const for canned TRUE and FALSE behaviors. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the release notes tell users to recreate views and rules that were created before the fix and hold the wrong 'false'::jsonb default?
2. Can the release notes give a catalog query that finds views and rules with a boolean DEFAULT in JSON_VALUE, JSON_QUERY or JSON_TABLE?
3. Does v2 add regression tests for JSON_TABLE column DEFAULT clauses with a boolean expression?
4. How does a new view with a boolean DEFAULT deparse after the fix, and does that text restore to the same behavior?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Amit Langote posted v2 in the thread and says he intends to commit it and back-patch to v17.
- **Liability:** A changed node tree for user-supplied boolean DEFAULT expressions in v17 and later, and the release-note wording for views that keep the old value.
- **Author follow-up:** The author posted a rebased patch with regression tests one week after the report (message #3).

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix removes a silent wrong result for users of SQL/JSON functions with a boolean DEFAULT. It touches one function in parse_expr.c, and a committer who knows the area plans to commit it.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent matches this fix. [P6] reverted unrelated SQL/JSON commits after buildfarm failures, and [P1] reverted a JSON_TABLE ON ERROR cascade that changed behaviour without documentation, so neither recorded reason applies to a wrong-result fix. A committer who knows the area (message #5) plans to commit it. No earlier rejected attempt at this fix appears in the thread or the precedent list.
- standards-compat: 0 objection(s). This fixes a wrong-result bug, a committer owns it, and standard semantics require the evaluated DEFAULT. Stored views keeping 'false'::jsonb after dump or pg_upgrade is a release-note matter, already in the commit message. Not checkable without code: whether new views deparse as an explicit to_jsonb(...) call. Neither precedent's reason applies.
- replication-ondisk: 1 objection(s). Otherwise the change is parse-time only and uses existing node types. It needs no catversion bump and adds no WAL or protocol change. Older minor versions and standbys can still read the new FuncExpr in stored rules.
- security: 0 objection(s). The fix has no privilege or exposure aspect. It evaluates a DEFAULT expression the user already wrote and wraps it in the built-in to_jsonb(), bound by OID, so search_path cannot redirect it, and other DEFAULT types already run the user's expression. The fix adds no new privilege, catalog object, default or resource use.
- maintenance-fit: 0 objection(s). This is a parse-time bug fix in one function, with no hot-path cost, catalog, GUC or API change, and a committer who knows the area plans to commit and back-patch it. Without the code I could not check how ruleutils deparses the new to_jsonb() wrapper in a dumped view. [P1] and [P6] are SQL/JSON reverts whose recorded reasons do not apply to this fix.
- already-exists: 0 objection(s). This is a wrong-result bug fix inside an existing core SQL/JSON feature, so no existing feature or extension meets the need. The workaround of writing DEFAULT 'true' or using RETURNING boolean only avoids the bug and does not justify keeping wrong results.
- demand: 0 objection(s). This is a wrong-result bug fix, not a feature, so the demand question does not apply. Two independent bug reports (#19625 and #19822) show users hit it, and a committer who knows the area plans to commit and back-patch it.

### Precedents retrieved

- P1: [Revert cascading of JSON_TABLE's ON ERROR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=af6fad879fbf9b542cfde05c52695c9dffa6bb47) (revert, Reverted, 2026-07-14) - used in the review
- P2: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - judged not relevant; reason summarised from its thread
- P3: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P4: [Boolean partition syntax](https://commitfest.postgresql.org/patch/1410/) (cf, Returned with feedback, 2018-03-21) - judged not relevant; reason summarised from its thread
- P5: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant; reason summarised from its thread
- P6: [Revert recent SQL/JSON related commits](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4d7e24e0f4d05b546228488ccdc2848a80245ffb) (revert, Reverted, 2024-09-06) - used in the review
- P7: [Revert "PL/Python: Remove workaround for returning booleans in Python <2.3"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7e938e3c56590899bcfa587ad0220869cb9be951) (revert, Reverted, 2013-01-05) - judged not relevant
- P8: [Revert "psql: Show all query results by default"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fae65629cec824738ee11bf60f757239906d64fa) (revert, Reverted, 2021-04-15) - judged not relevant
- P9: [SQL/JSON: JSON_TABLE](https://commitfest.postgresql.org/patch/1473/) (cf, Returned with feedback, 2020-08-03) - judged not relevant; reason summarised from its thread
- P10: [Allow to warn or error on ... = NULL expressions](https://commitfest.postgresql.org/patch/1728/) (cf, Returned with feedback, 2018-12-01) - judged not relevant; reason summarised from its thread
- P11: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant; reason summarised from its thread
- P12: [Fix behavior of geo_ops when NaN is involved](https://commitfest.postgresql.org/patch/2710/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P13: [psql: default to ON_ERROR_STOP](https://commitfest.postgresql.org/patch/3446/) (cf, Returned with feedback, 2022-01-17) - judged not relevant
- P14: [Don't allow to EXECUTE empty queries (fixes Assert violation)](https://commitfest.postgresql.org/patch/5918/) (cf, Rejected, 2025-07-17) - judged not relevant
- P15: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P16: [Disable parallel query by default](https://commitfest.postgresql.org/patch/5751/) (cf, Rejected, 2026-02-11) - judged not relevant
- P17: [protocol default result formats setting](https://commitfest.postgresql.org/patch/2812/) (cf, Returned with feedback, 2021-04-07) - judged not relevant
- P18: [Revert "Reorganise jsonpath operators and methods"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d) (revert, Reverted, 2024-01-03) - judged not relevant
- P19: [json_table function PLAN clause](https://commitfest.postgresql.org/patch/6332/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P20: [Apply a better fix to mdunlinkfork().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=85d8b30724c0fd117a683cc72706f71b28463a05) (revert, Partially reverted, 2022-11-09) - judged not relevant
- P21: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P22: [SQL/JSON json_table() function PLAN clause](https://commitfest.postgresql.org/patch/6590/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P23: [Revert b2a459edf "Fix GRANTED BY support in REVOKE ROLE statements"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e68570e388f08c2e36ce7d2a9564941b89db6549) (revert, Reverted, 2021-12-30) - judged not relevant
- P24: [Revert "Enable fast default for domains with non-volatile constraints"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0354e29c41a9fb7491b3c7c23f079b1923c045a) (revert, Reverted, 2026-06-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 63 s; cost reported by the CLI: $0.79
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:25:52.664Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
