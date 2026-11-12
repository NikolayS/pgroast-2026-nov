# Design pre-review: Support "ALTER FOREIGN TABLE ... RENAME CONSTRAINT ..."

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7558](https://commitfest.postgresql.org/patch/7558/) (Needs review); authors: Ian Barwick (barwick)
- Thread: [Support "ALTER FOREIGN TABLE ... RENAME CONSTRAINT ..."](https://www.postgresql.org/message-id/flat/CAB8KJ%3DgBwpo%2BiXR5n_NQ1xude3Ms2P3n%3DOmL6F%3DB%3DVSNgCXgfw%40mail.gmail.com), started 2026-10-30
- Messages read: 1 of 1
- Patch prose read: v1-0001-Support-ALTER-FOREIGN-TABLE-.-RENAME-CONSTRAINT.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

One objection survived, already-exists-1, and it is a fixable question of value against cost. Message #1 says `alter table ... rename constraint` already works on foreign tables, so the new form is a second spelling. It matches the existing rename column and add/drop constraint forms and reuses the existing rename path. The three command tag objections were refuted, so nothing that survived stops the proposal at levels 1-2.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the scope of the AlterObjectTypeCommandTag() refactor cannot be checked, so the claim that no existing command changes its tag is unverified. The behaviour of `only` and `*` for inherited constraints also could not be judged.

## The proposal, restated

**Problem.** alter foreign table can add and drop constraints but cannot rename them, while alter table can. The author calls this an inconsistency that users of foreign tables hit. alter table ... rename constraint already works on foreign tables, but users do not easily find this.

**Proposed design.** The patch adds a RENAME CONSTRAINT form to the alter foreign table grammar and reuses the existing rename path. It refactors AlterObjectTypeCommandTag() so that column and constraint operations return the command tag ALTER FOREIGN TABLE or ALTER TABLE, matching the command used. It also extends psql tab completion.

**User-visible surface.**
- New syntax: alter foreign table [ if exists ] [ only ] name [ * ] rename constraint old to new.
- Command tag for column or constraint operations now follows the command used (ALTER TABLE or ALTER FOREIGN TABLE).
- psql tab completion for alter foreign table rename constraint, and for add/drop column/constraint.

**What the author claims.**
- The feature is niche, but the author hit it while working intensively with foreign tables.
- The code change is fairly small.
- No reason exists not to support it, and alter foreign table already has non-standard extensions.
- It makes alter foreign table consistent with alter table, as with owner to.

**Thread state.** Only the first message is visible. No one has replied, and the CommitFest entry is in Needs review for version 20.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Rename constraint on foreign tables already works through alter table

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), SQL-standard conflict or non-standard syntax · `already-exists-1`

A user today runs `alter table ft_foo rename constraint alice to bob` and gets exactly the result the new syntax gives. The patch adds a second spelling, plus a command tag refactor in utility.c. If that refactor changes the tag of any existing column or constraint command, event triggers or tools that filter on the tag can stop matching after an upgrade.

Message #1 states that alter table ... rename constraint works on all table-like objects, so the gap is that users cannot find it, not that the job cannot be done. A documentation cross-reference in alter_foreign_table.sgml would cover the same need without grammar or command tag changes. The author's consistency case is real, since alter foreign table already has rename column and add/drop constraint. Still, [P1] shows a small ease-of-use gain being weighed against even modest added code.

**Precedents.**
- [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (Rejected, 2025-03-26); thread: [link](https://www.postgresql.org/message-id/flat/CADrsxdYj2SMquRgf=DS9d_qLMW_gnF7PxzRp0VKTM7OfBsPpVg@mail.gmail.com)

Thread: [#1 (Ian Lawrence Barwick, 2026-10-30)](https://www.postgresql.org/message-id/CAB8KJ%3DgBwpo%2BiXR5n_NQ1xude3Ms2P3n%3DOmL6F%3DB%3DVSNgCXgfw%40mail.gmail.com)

**Question:** Why is a cross-reference to alter table rename constraint in the alter foreign table docs not enough, and does the tag refactor change the tag of any existing command?

Counter-check: held. Message #1 confirms that `alter table ... rename constraint` already does the job and calls the need niche, so the documentation-only alternative needs an answer. [P1] concerns an option validator and gives weak support, and the tag-change part of the scenario has no basis in the material.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds a non-standard `alter foreign table ... rename constraint` form in gram.y. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | The patch extends tab completion for alter foreign table rename constraint and add/drop column/constraint. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch refactors AlterObjectTypeCommandTag() in tcop/utility.c and adds a second spelling to keep documented and tested. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #1 says `alter table ... rename constraint` already does this on foreign tables. |

## 5. Questions the author must answer before anyone reviews code

1. Why is a cross-reference to `alter table ... rename constraint` in the alter foreign table documentation not enough?
2. Does any existing statement return a different command tag after the AlterObjectTypeCommandTag() refactor, and if so, which?
3. Which constraint types on a foreign table can the new form rename?
4. How do `only` and `*` behave when the renamed constraint is inherited by or from another table?
5. Do the regression tests check the command tag returned for the new form and for the existing column and constraint forms?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the utility command grammar and the command tag code in tcop/utility.c would have to own it.
- **Liability:** One more grammar form to keep in step with alter table, and the refactored command tag logic, which event triggers and clients depend on, for as long as the syntax exists.
- **Author follow-up:** No sign either way; the thread has only the first message.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The author calls the need niche in message #1, and a workaround exists. The patch is about 11 KiB, reuses the existing rename path, and touches only the grammar, utility.c, psql completion, docs and tests.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Command tag change for existing alter foreign table rename column. Refuted (wrong facts): Postgres today takes the tag for a column rename from the relation type named in the command, so `alter foreign table ... rename column` already reports ALTER FOREIGN TABLE. The trigger in the scenario does not fire for that command before the patch either.
- `replication-ondisk-1` Command tag change for existing commands is not stated or bounded. Refuted (not specific): Duplicate of standards-compat-1, and it rests on the same scenario. `alter foreign table ... rename column` already reports ALTER FOREIGN TABLE, so the material shows no existing command whose tag changes.
- `maintenance-fit-1` Command tag change for existing commands is bundled with new syntax. Refuted (not specific): Duplicate of standards-compat-1. The one existing command it names, `alter foreign table ... rename column`, already reports ALTER FOREIGN TABLE today.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] is about postgres_fdw rejecting empty option values, which is a different idea. Its cost-versus-ease-of-use reason does not carry over to a small grammar addition that reuses an existing rename path. Message #1 points to no earlier discussion of this idea, so the prior-attempts angle gives no basis for an objection.
- standards-compat: 1 objection(s). The new syntax copies alter table rename constraint, adds no keyword, and needs no pg_dump change, because constraint names are already dumped. The only compatibility concern I found is the command tag side effect.
- replication-ondisk: 1 objection(s). Apart from the command tag, the patch adds no persistent or cross-node state. It adds a grammar form and reuses the existing rename path, so there is no catalog, WAL, on-disk or ABI change.
- security: 0 objection(s). The proposal adds a grammar alias for an operation that alter table ... rename constraint already performs on foreign tables, so it lets no one do anything new. Ownership checks and locking come from the existing rename path. The command tag change could matter to event triggers that filter on tags, but that is a compatibility question for another angle.
- maintenance-fit: 1 objection(s). The new syntax itself carries little long-run cost: it reuses the existing rename path, adds no catalog, GUC or hot-path work, and matches the existing rename column form. [P1] was rejected for low value per line of code, but here the code is small and the form fits existing grammar, so it is not cited.
- already-exists: 1 objection(s). One fixable objection only. The consistency argument for the new syntax is reasonable, so this is a question of value against cost, not a reason to reject.
- demand: 0 objection(s). Demand is thin: one user, self-described as niche, and alter table ... rename constraint already works on foreign tables. Still, the cost is one grammar form, one doc line and tab completion, and ten such consistency fixes per release would be acceptable. The command tag change is for the design angle to judge.

### Precedents retrieved

- P1: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - used in the review; reason summarised from its thread
- P2: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P3: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P4: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2409/) (cf, Returned with feedback, 2020-01-28) - judged not relevant; reason summarised from its thread
- P5: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P6: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2499/) (cf, Returned with feedback, 2020-08-02) - judged not relevant; reason summarised from its thread
- P7: [Revert changes to CREATE TRIGGER and ALTER TABLE ADD FOREIGN KEY locking, per request from Tom.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50ce8ab9fcd49dfcbff9e19c5aa53efe0fc6dc6e) (revert, Reverted, 2005-03-24) - judged not relevant
- P8: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/167/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P9: [Revert support for ALTER TABLE ... MERGE/SPLIT PARTITION(S) commands](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3890d90c1508125729ed20038d90513694fc3a7b) (revert, Reverted, 2024-08-24) - judged not relevant
- P10: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/20/) (cf, Rejected, 2015-01-16) - judged not relevant
- P11: [Revert "Rename unicode_combining_table to unicode_width_table"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1563ecbc1be8b8e5c57651cf5c87f90dea9aea8f) (revert, Reverted, 2021-08-26) - judged not relevant
- P12: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant; reason summarised from its thread
- P13: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant; reason summarised from its thread
- P14: [MERGE SQL command](https://commitfest.postgresql.org/patch/2920/) (cf, Returned with feedback, 2021-09-01) - judged not relevant; reason summarised from its thread
- P15: [psql command \graw](https://commitfest.postgresql.org/patch/1346/) (cf, Returned with feedback, 2018-01-23) - judged not relevant
- P16: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P17: [Confusing list foreign table options in psql - Fixing slash usage text](https://commitfest.postgresql.org/patch/2545/) (cf, Returned with feedback, 2020-11-29) - judged not relevant
- P18: [Adding CommandID to heap xlog records](https://commitfest.postgresql.org/patch/3882/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P19: [psql: Add tab-complete for optional view parameters](https://commitfest.postgresql.org/patch/4053/) (cf, Returned with feedback, 2023-07-06) - judged not relevant
- P20: [refactor AlterDomainAddConstraint (alter domain add constraint)](https://commitfest.postgresql.org/patch/5455/) (cf, Returned with feedback, 2026-04-02) - judged not relevant
- P21: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P22: [remove VersionedQuery support from psql tab completion](https://commitfest.postgresql.org/patch/6977/) (cf, Withdrawn, 2026-08-11) - judged not relevant
- P23: [Revert fix missing call to table_finish_bulk_insert during COPY](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5db56fc4d6e95c582b61c99328ea0702b869fa0) (revert, Reverted, 2019-07-01) - judged not relevant
- P24: [Tab completion for ALTER COLUMN SET STATISTICS](https://commitfest.postgresql.org/patch/377/) (cf, Rejected, 2015-12-21) - judged not relevant

### Warnings

- synthesis said NO-FUNDAMENTAL-OBJECTION but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 89 s; cost reported by the CLI: $0.86
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 65 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:09:45.508Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
