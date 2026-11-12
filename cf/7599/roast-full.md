# Design pre-review: doc: Mention that * does not include system columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7599](https://commitfest.postgresql.org/patch/7599/) (Needs review); authors: Arvind Kandpal (arvindk12)
- Thread: [4.1.4. Special Characters](https://www.postgresql.org/message-id/flat/178782026936.2606573.10270614257871556098%40wrigleys.postgresql.org), started 2026-08-27
- Messages read: 10 of 10
- Patch prose read: v1-0001-doc-Mention-that-does-not-include-system-columns.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The change rewrites one sentence on the SELECT reference page to state existing behaviour, and Peter Eisentraut's placement concern in message #3 was addressed by moving the text there. Confidence is medium because only the v1 hunk is shown, so the v4 wording cannot be checked.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only the v1 hunk for syntax.sgml is shown. The v4 text, its exact place on the SELECT reference page, and whether it covers table_name.* could not be checked.

## The proposal, restated

**Problem.** A doc comment on the lexical syntax page asked to tell readers that * does not return system columns such as ctid. It suggested the example `SELECT ctid, * FROM t;` and a link to the System Columns section. The audience is SQL users reading the docs.

**Proposed design.** v1 added a sentence, an example and a link to the asterisk item in section 4.1.4. After Peter Eisentraut's review, v2 moved the change to the SELECT List section of the SELECT reference page. v4 rewrites the first sentence there to say * is a shorthand for all the columns of the selected rows, not including system columns, with a link to that section.

**What the author claims.**
- The docs build cleanly with make check and make html for each version.
- "not including system columns" fits the existing docs better than "user-defined columns", which the docs do not use for this.
- An endterm with free text fails the doc check, so v4 uses a link element that renders the same text.

**Thread state.** Peter Eisentraut moved the change to the SELECT reference page and did not reply after that. Matemática A3K shaped the v3 and v4 wording and gave a +1 on v4; no committer has commented on v4.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-27; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the v4 sentence on the SELECT reference page also cover table_name.*, which v2 named and v3 rewrote?
2. Do other pages that describe * expansion, such as composite types or row constructors, need the same note or a link to this one?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Eisentraut reviewed v1 in message #3 and chose the target page, but has not commented on v4; a committer who handles documentation would sign it.
- **Liability:** One sentence and one link on the SELECT reference page; the owner signs for the wording being accurate and the link target staying valid.
- **Author follow-up:** The author revised the patch from v1 to v4 in response to review, which suggests continued follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** A reader asked for this through the docs comment form (message #1), and the change is one sentence plus a link on one page. A reviewer gave +1 on v4; it needs a committer to read the final wording.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The precedent search returned no attempt with the same or a similar idea. The thread points to no earlier discussion of this doc change, so no earlier rejection reason applies.
- standards-compat: 0 objection(s). The patch changes only one sentence of documentation and touches no syntax, GUC, catalog, dump format or client behaviour, so nothing in the standards and compatibility angle can break. It also states existing behaviour correctly: in Postgres, * leaves out system columns. One open point is wording: v4 is not shown, so it cannot be checked whether table_name.* is still covered. That is a docs-review question, not an objection.
- replication-ondisk: 0 objection(s). The patch only changes documentation text on the SELECT reference page. It has no persistent or cross-node state: no on-disk format, WAL, catalog, wire protocol or ABI change, so this angle has nothing to object to.
- security: 0 objection(s). The patch changes one sentence of documentation about * and system columns. It grants nothing, exposes nothing and changes no defaults, so there is no privilege or exposure aspect to object to from the security angle.
- maintenance-fit: 0 objection(s). No maintenance or fit objection. The v4 change rewrites one sentence on the SELECT reference page and adds a link. It adds nothing to code, the catalog, the hot path or the extension API. Peter Eisentraut steered the change to the page that is the authoritative source (message #3), so the fit concern was settled in the thread. Only the v1 hunk is shown, so the exact v4 text and whether it covers table_name.* could not be checked.
- already-exists: 0 objection(s). From this angle there is no concrete existing alternative to name. Peter Eisentraut said in message #3 that the SELECT reference page does not state this, and I cannot confirm that any other page says * excludes system columns, so there is no duplication to raise. A docs-only change also has no extension or hook alternative.
- demand: 0 objection(s). A user asked for this through the docs comment form (message #1), and a second reader called it useful (message #10). The change rewrites one sentence on the SELECT reference page, so maintenance cost is close to zero and it adds no user-visible surface beyond the doc text. That demand is enough for a doc clarification of this size, so the demand angle has no objection to raise.

### Precedents retrieved

- P1: [pg_stat_statements: add last_execution_start column](https://commitfest.postgresql.org/patch/6624/) (cf, Withdrawn, 2026-08-21) - judged not relevant; reason summarised from its thread
- P2: [document the hook system](https://commitfest.postgresql.org/patch/2915/) (cf, Returned with feedback, 2021-03-10) - judged not relevant; reason summarised from its thread
- P3: [Revert "Fix bug in checking of IDENTIFY_SYSTEM result."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c4762886539b06da682045e19ba73dcee3789638) (revert, Reverted, 2014-08-19) - judged not relevant
- P4: [ALTER TABLE - check for column existence before starting operations](https://commitfest.postgresql.org/patch/1621/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P5: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P6: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P7: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P8: [ALTER SYSTEM READ { ONLY | WRITE }](https://commitfest.postgresql.org/patch/2602/) (cf, Withdrawn, 2022-01-10) - judged not relevant; reason summarised from its thread
- P9: [WITH SYSTEM VERSIONING Temporal Tables](https://commitfest.postgresql.org/patch/2316/) (cf, Returned with feedback, 2021-11-16) - judged not relevant; reason summarised from its thread
- P10: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P11: [System username in pg_stat_activity](https://commitfest.postgresql.org/patch/4769/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P12: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P13: [Hidden/invisible columns](https://commitfest.postgresql.org/patch/3361/) (cf, Rejected, 2021-10-29) - judged not relevant
- P14: [Add table expansion hook](https://commitfest.postgresql.org/patch/2614/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P15: [Add "\gi file" to psql to allow COPY with variable expansion](https://commitfest.postgresql.org/patch/5674/) (cf, Returned with feedback, 2025-06-17) - judged not relevant
- P16: [Indexes with Included Columns (was Covering + unique indexes)](https://commitfest.postgresql.org/patch/961/) (cf, Returned with feedback, 2017-04-08) - judged not relevant
- P17: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant
- P18: [Revert "Add key management system" (978f869b99) & later commits](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3187ef7c46c5b884267a88f2d6119c9a05f1bbba) (revert, Reverted, 2020-12-28) - judged not relevant
- P19: [Quick Start Guide to PL/pgSQL and PL/Python Documentation](https://commitfest.postgresql.org/patch/4705/) (cf, Withdrawn, 2024-03-23) - judged not relevant
- P20: [List of Available Figures/Graphics in the Documentation](https://commitfest.postgresql.org/patch/2204/) (cf, Returned with feedback, 2019-07-08) - judged not relevant
- P21: [Multi-column list partitioning](https://commitfest.postgresql.org/patch/3383/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P22: [ctidscan as an example of custom-scan](https://commitfest.postgresql.org/patch/18/) (cf, Rejected, 2015-07-22) - judged not relevant
- P23: [psql: --echo-hidden improvements](https://commitfest.postgresql.org/patch/4903/) (cf, Withdrawn, 2024-03-21) - judged not relevant
- P24: [Revert CREATE INDEX ... INCLUDING ...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b99edefcab1e82c43139a2c7dc06d31fb27b3e4) (revert, Reverted, 2016-04-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 99 s; cost reported by the CLI: $1.11
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 4 verified, 1 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 38 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:04:23.127Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
