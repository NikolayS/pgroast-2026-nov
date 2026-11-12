# Design pre-review: psql: avoid CREATE command completion after GRANT/REVOKE CREATE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7374](https://commitfest.postgresql.org/patch/7374/) (Needs review); authors: Haruna Miwa (miwa-h)
- Thread: [\[PATCH\] psql: avoid CREATE command completion after GRANT/REVOKE CREATE](https://www.postgresql.org/message-id/flat/OSOP286MB78539E88C47403451BFFF140F18A2%40OSOP286MB7853.JPNP286.PROD.OUTLOOK.COM), started 2026-10-01
- Messages read: 7 of 7
- Patch prose read: v3-0001-psql-avoid-CREATE-command-completion-after-GRANT-REV.patch, 0001-psql-avoid-CREATE-command-completion-after-GRANT-REV.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The patch is a narrow psql tab-completion bug fix in one file, and the brief says a committer reviewed it and pushed v3 (message #7). The problem and the change are stated in message #1, and a reviewer confirmed the behaviour in message #2.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the exact match conditions and their effect on other inputs that end in CREATE could not be checked. Messages #4 to #7 were not shown; the push is taken from the brief and reviewer notes.

## The proposal, restated

**Problem.** In psql, the generic tab-completion rule for CREATE fires after GRANT CREATE and REVOKE CREATE, so psql offers CREATE object types where a privilege name was typed. This affects interactive psql users writing GRANT or REVOKE.

**Proposed design.** The patch excludes GRANT CREATE, REVOKE CREATE and REVOKE GRANT OPTION FOR CREATE from the generic CREATE completion rule in tab-complete.in.c. These inputs then reach the existing privilege-completion rule, which offers ON.

**User-visible surface.**
- psql tab completion after GRANT CREATE, REVOKE CREATE and REVOKE GRANT OPTION FOR CREATE now offers ON instead of CREATE object types.

**What the author claims.**
- The generic CREATE rule offers irrelevant object types after GRANT CREATE and REVOKE CREATE.
- The fix touches only src/bin/psql/tab-complete.in.c.

**Thread state.** Sho Ito tested v2, asked for the REVOKE GRANT OPTION FOR CREATE case, and approved it. Fujii Masao asked for a comment change and pushed v3 on 2026-10-07, but the CommitFest entry still says Needs review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-01; 18 candidates retrieved, none is the same or a similar idea).

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
| psql UX conventions | **yes** | psql now offers ON after GRANT CREATE, REVOKE CREATE and REVOKE GRANT OPTION FOR CREATE. |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, and the thread does not say whether the fix goes to back branches. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do other privilege names that are also command keywords, such as TRUNCATE, reach a generic command completion rule after GRANT or REVOKE?
2. Does the patch add tab-completion tests for the three new cases?
3. Should the commit message mention the REVOKE GRANT OPTION FOR CREATE case that v2 added?
4. Is the fix meant for back branches or for master only?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Fujii Masao reviewed the patch, asked for a comment change and pushed v3, per the brief and message #7.
- **Liability:** One more exclusion in the generic CREATE completion rule in tab-complete.in.c; a mistake would only give wrong psql suggestions.
- **Author follow-up:** The author posted v2 two days after the review in message #2 and a v3 after the committer's request.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 1 KiB in one psql file and fixes wrong suggestions for interactive users who type GRANT or REVOKE. The brief says v3 was pushed on 2026-10-07 while the CommitFest entry still says Needs review, so the entry status needs an update.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus search returned no earlier attempt at this fix, and no thread message points to one. The patch is a narrow psql tab-completion bugfix, a committer reviewed and pushed v3 (message #7), so no earlier rejection reason applies.
- standards-compat: 0 objection(s). No objection from the standards and compatibility angle. The patch changes only psql tab completion after GRANT CREATE, REVOKE CREATE and REVOKE GRANT OPTION FOR CREATE, so that psql offers ON as the existing privilege rule does. It adds no syntax, keyword, GUC or catalog object, and it does not change pg_dump output or server behaviour; a committer reviewed it and pushed v3 (message #7).
- replication-ondisk: 0 objection(s). The patch changes only psql tab completion in src/bin/psql/tab-complete.in.c. It has no on-disk, WAL, replication, catalog, wire protocol or server ABI effect, so this angle has nothing to object to.
- security: 0 objection(s). The patch changes only which keywords psql tab completion offers after GRANT CREATE, REVOKE CREATE and REVOKE GRANT OPTION FOR CREATE. It grants no privilege and exposes no data, so there is no security aspect to object to.
- maintenance-fit: 0 objection(s). No maintenance or fit objection holds. The patch is a small psql tab-completion bug fix in one file, follows the existing exclusion pattern in tab-complete.in.c, adds no hot-path cost, GUC, catalog change or external dependency, and a committer (Fujii Masao) reviewed and pushed it in message #7. The one open point is whether other privilege keywords, such as TRUNCATE, have the same problem; that is a follow-up question, not a reason to reject.
- already-exists: 0 objection(s). No existing feature, extension or tool gives correct psql tab completion after GRANT CREATE. The patch fixes wrong suggestions inside the existing completion rules in tab-complete.in.c. It extends the current mechanism and does not add a second one, so this angle has nothing to object to.
- demand: 0 objection(s). No demand objection applies. This is a small bug fix to psql tab completion: GRANT CREATE now offers ON, which every interactive psql user typing these commands needs. The patch touches only tab-complete.in.c, adds no UI or hot-path cost for non-users, and a committer reviewed and pushed it in message #7.

### Precedents retrieved

- P1: [psql: Add tab-complete for optional view parameters](https://commitfest.postgresql.org/patch/4053/) (cf, Returned with feedback, 2023-07-06) - judged not relevant; reason summarised from its thread
- P2: [Tab completion for SELECT in psql](https://commitfest.postgresql.org/patch/1376/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P3: [Tab-completion refactoring and word-removal feature (was IF-NOT_EXISTS for tab-completion)](https://commitfest.postgresql.org/patch/855/) (cf, Returned with feedback, 2017-03-17) - judged not relevant; reason summarised from its thread
- P4: [remove VersionedQuery support from psql tab completion](https://commitfest.postgresql.org/patch/6977/) (cf, Withdrawn, 2026-08-11) - judged not relevant; reason summarised from its thread
- P5: [psql: add tab completion for \df slash command suffixes](https://commitfest.postgresql.org/patch/2268/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P6: [Fix psql-completion for CREATE INDEX](https://commitfest.postgresql.org/patch/452/) (cf, Returned with feedback, 2016-02-03) - judged not relevant; reason summarised from its thread
- P7: [tab-completion debug print](https://commitfest.postgresql.org/patch/1917/) (cf, Returned with feedback, 2019-02-02) - judged not relevant; reason summarised from its thread
- P8: [IF (NOT) EXISTS in psql-completion](https://commitfest.postgresql.org/patch/518/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P9: [Tab completion for CREATE SCHEMA AUTHORIZATION](https://commitfest.postgresql.org/patch/3252/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P10: [improve tab-completion behavior when parse text in single quote](https://commitfest.postgresql.org/patch/3267/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P11: [Revert MAINTAIN privilege and pg_maintain predefined role.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=151c22deee66a3390ca9a1c3675e29de54ae73fc) (revert, Reverted, 2023-07-07) - judged not relevant
- P12: [Fix tab completion MERGE](https://commitfest.postgresql.org/patch/3890/) (cf, Returned with feedback, 2023-07-04) - judged not relevant
- P13: [Revert "psql \dX: list extended statistics objects"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1db0d173a2201119f297ea35edfb41579893dd8c) (revert, Reverted, 2021-01-17) - judged not relevant
- P14: [Tab completion for ATTACH PARTITION](https://commitfest.postgresql.org/patch/4561/) (cf, Returned with feedback, 2024-01-29) - judged not relevant
- P15: [Implement the SQL-standard mechanism whereby REVOKE ROLE revokes only the privilege granted by the invoking role, and not those granted by other roles](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Administration) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P16: [Identifying user-created objects](https://commitfest.postgresql.org/patch/2444/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P17: [fix and document CLUSTER privileges](https://commitfest.postgresql.org/patch/4054/) (cf, Returned with feedback, 2023-01-14) - judged not relevant
- P18: [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://commitfest.postgresql.org/patch/2664/) (cf, Returned with feedback, 2020-11-24) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 88 s; cost reported by the CLI: $0.63
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 18 kept
- Tone check: 40 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:47:15.857Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
