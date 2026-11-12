# Design pre-review: Docs: Create table description for constraints markup fix and label tweaks

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6648](https://commitfest.postgresql.org/patch/6648/) (Needs review); authors: David Johnston (polobo)
- Thread: [Docs: Create table description for constraints markup fix and label tweaks](https://www.postgresql.org/message-id/flat/CAKFQuwbsGWx5msYbRsWrRs%2Bss1k0utDHEhVq5unAKODQ9YOf9w%40mail.gmail.com), started 2026-04-03
- Messages read: 3 of 3
- Patch prose read: v2-0002-pgsql-doc-Make-constraint-type-labels-not-match-s.patch, v2-0001-pgsql-doc-Fix-malformed-sgml-element-nesting-for-.patch, v1-0002-pgsql-doc-Make-constraint-type-labels-not-match-s.patch, v1-0001-pgsql-doc-Fix-malformed-sgml-element-nesting-for-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. Both patches change only doc/src/sgml/ref/create_table.sgml: 0001 corrects element nesting for UNIQUE and PRIMARY KEY, and 0002 answers a reader report on -general cited in message #1. A reviewer confirmed in message #3 that make html builds. The remaining points are scope and wording choices for a committer.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The documentation hunks were read in full, but the rendered HTML was not seen. Whether "variant" is the best wording, and whether the same labels exist on other pages, could not be checked from the material.

## The proposal, restated

**Problem.** In the create table reference page, the UNIQUE and PRIMARY KEY table constraint entries put the INCLUDE clause outside the enclosing literal element. A report on -general also said the labels "(column constraint)" and "(table constraint)" look like the defined syntax names column_constraint and table_constraint, which confuses readers of the syntax area.

**Proposed design.** Patch 0001 moves the optional INCLUDE clause inside the literal element in the UNIQUE and PRIMARY KEY terms. Patch 0002 adds the word "variant" to the six table and column constraint labels for UNIQUE, PRIMARY KEY and REFERENCES/FOREIGN KEY. Both patches touch only doc/src/sgml/ref/create_table.sgml.

**User-visible surface.**
- Create table reference page: the INCLUDE clause renders inside the literal markup for UNIQUE and PRIMARY KEY table constraints.
- Create table reference page: labels change to "(column constraint variant)" and "(table constraint variant)" in six places.

**What the author claims.**
- The current labels match a defined syntax label, and this confused a user on -general.
- Adding "variant" to the labels removes that confusion.
- UNIQUE had the same markup error as PRIMARY KEY, so v2 fixes both.

**Thread state.** The author posted v1 and then v2, which extends the markup fix to UNIQUE. Dapeng Wang reviewed v2, confirmed that make html builds, and said it looks good; no committer has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-04-03; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | **yes** | The CommitFest entry is tagged Bugfix, and the thread does not say whether the 0001 markup fix goes to back branches. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should the 0001 markup fix be back-patched, and if so to which branches?
2. Do other reference pages, such as alter table, use the "(column constraint)" and "(table constraint)" labels and need the same change?
3. Is the inconsistent use of optional markup versus literal brackets, noted in message #2, in scope for this patch set or left for later?
4. Does the original reporter on -general find that "(column constraint variant)" removes the confusion with the column_constraint syntax name?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Any committer who handles documentation patches and knows the reference page markup conventions could take this; none has replied in the thread.
- **Liability:** The committer signs for the rendered create table page and for the new "variant" wording, which later readers may ask to see applied on other pages. There is no code, format or behaviour liability.
- **Author follow-up:** The author posted v2 on the same day as v1 with a wider fix, which suggests continued attention; there is no reply after the review in message #3.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** most users
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The two patches are a few lines in one widely read reference page and already have a positive review in message #3. The entry has moved through two CommitFests without a committer reply, so it needs a committer to pick it up.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P19] is about consistency of server application synopses, and [P10] is about making join syntax more prominent on the select page. Neither records a reason that applies to a markup nesting fix or a label change on the create table page. The thread does not mention an earlier attempt at this change.
- standards-compat: 0 objection(s). This is a docs-only change to create_table.sgml, with no change to SQL syntax, GUCs, pg_dump, psql or client behaviour. The markup fix aligns UNIQUE and PRIMARY KEY with how their syntax is written, and the "variant" wording conflicts with no existing term. Other pages such as alter_table.sgml may use the same labels, but that is a follow-up question, not a failure.
- replication-ondisk: 0 objection(s). Both patches change only doc/src/sgml/ref/create_table.sgml. They add no on-disk format, WAL record, catalog entry, protocol message or extension API, so nothing in this proposal is persistent or crosses a process or machine boundary.
- security: 0 objection(s). The patch changes only markup and label text on the create table reference page. It adds nothing that affects privileges, authentication, data exposure or resource use, so the security angle has no objection.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The patches change only doc/src/sgml/ref/create_table.sgml, add no mechanism, GUC, API or hot-path cost, and leave no future feature with a new place to update. The markup fix corrects a real nesting error, and a reviewer confirmed that make html builds. Open points such as back-patching and other reference pages are scope questions for a committer to decide.
- already-exists: 0 objection(s). This is a docs-only fix to markup and labels on the create table page. No existing feature, option, extension or tool already does this, so the already-exists angle has no objection to raise.
- demand: 0 objection(s). Demand is shown for a docs-only fix. Patch 0001 corrects malformed markup, which needs no demand case. Patch 0002 answers a concrete reader report on -general (message #1), costs nothing on any hot path, and touches one reference page. Whether "variant" is the best wording is a question for other angles.

### Precedents retrieved

- P1: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P2: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P3: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P4: [Parallel Inserts in CREATE TABLE AS](https://commitfest.postgresql.org/patch/2841/) (cf, Withdrawn, 2021-03-20) - judged not relevant; reason summarised from its thread
- P5: [create table like: ACCESS METHOD](https://commitfest.postgresql.org/patch/2865/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant; reason summarised from its thread
- P7: [Document creating an extension in replication setup](https://commitfest.postgresql.org/patch/3494/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P8: [Replace literal 0 values with the appropriate Invalid* constants](https://commitfest.postgresql.org/patch/6480/) (cf, Withdrawn, 2026-02-13) - judged not relevant; reason summarised from its thread
- P9: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant; reason summarised from its thread
- P10: [Select Reference Page - Make Join Syntax More Prominent](https://commitfest.postgresql.org/patch/3726/) (cf, Returned with feedback, 2022-09-10) - used in the review
- P11: [Build the docs if there are changes in docs and don't run other tasks if the changes are only in docs](https://commitfest.postgresql.org/patch/4555/) (cf, Withdrawn, 2024-05-17) - judged not relevant
- P12: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P13: [deferred primary key and logical replication](https://commitfest.postgresql.org/patch/2631/) (cf, Returned with feedback, 2020-11-29) - judged not relevant
- P14: [Dump-restore loosing 'attnotnull' bit for DEFERRABLE PRIMARY KEY column(s)](https://commitfest.postgresql.org/patch/4729/) (cf, Rejected, 2023-12-28) - judged not relevant
- P15: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant
- P16: [Migrate documentation from DocBook 4.5 to 5.2](https://commitfest.postgresql.org/patch/5171/) (cf, Withdrawn, 2024-09-18) - judged not relevant
- P17: [Autoprepare: implicitly replace literals with parameters and store generalized plan](https://commitfest.postgresql.org/patch/1747/) (cf, Returned with feedback, 2020-07-01) - judged not relevant
- P18: [Confusing docs about GetForeignUpperPaths in fdwhandler.sgml](https://commitfest.postgresql.org/patch/763/) (cf, Rejected, 2016-11-04) - judged not relevant
- P19: [DOCS: Make the Server Application docs synopses more consistent](https://commitfest.postgresql.org/patch/5647/) (cf, Withdrawn, 2026-03-23) - used in the review
- P20: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P21: [Add recovery to pg_control and remove backup_label](https://commitfest.postgresql.org/patch/4629/) (cf, Withdrawn, 2024-01-28) - judged not relevant
- P22: [Confusing list foreign table options in psql - Fixing slash usage text](https://commitfest.postgresql.org/patch/2545/) (cf, Returned with feedback, 2020-11-29) - judged not relevant
- P23: [Revert "Fix bool/int type confusion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a890432a872afc9ca2327573f3313fd994d17384) (revert, Reverted, 2017-09-22) - judged not relevant
- P24: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 65 s; cost reported by the CLI: $0.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:58:09.972Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
