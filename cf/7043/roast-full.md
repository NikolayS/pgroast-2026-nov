# Design pre-review: Make the logical replication conflict messages more like each other

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7043](https://commitfest.postgresql.org/patch/7043/) (Needs review); authors: Peter Smith (smithpb2250)
- Thread: [Make the logical replication conflict messages more like each other](https://www.postgresql.org/message-id/flat/CAHut%2BPuX_pPF6mU-UPRdQQToGgUPRpbOxOEFxPVC7B5Tevb-9A%40mail.gmail.com), started 2026-05-18
- Messages read: 4 of 4
- Patch prose read: none (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

Nothing that survived stops the proposal at levels 1a, 1b or 2. The one surviving objection, standards-compat-1, is a fixable wording point: two proposed reason lines are less exact than the detail lines they introduce. The proposal only rewords DETAIL text in errdetail_apply_conflict and adds no new interface. Confidence is low because no patch exists and the thread has one supporting reply.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: No patch exists, so the final strings, the translation impact and the test output changes could not be judged. The verdict rests on the strings suggested in message #1.

## The proposal, restated

**Problem.** The errdetail messages for logical replication apply conflicts, built in errdetail_apply_conflict in conflict.c, use mixed wording and structure. Only some conflict types give a general reason line before the detail line, and some lack a final period. The thread names log readers in general and does not cite a specific user complaint.

**Proposed design.** Every conflict type gets a first line with the general reason ("The row to update was not found") and then a detail line starting with "It was ..." for the origin and transaction. Every message ends with a period, and the wording follows the logical replication conflicts page of the docs. The unique-key detail lines stay the same.

**User-visible surface.**
- Changed DETAIL text of apply-worker conflict log messages for all conflict types, such as update_origin_differs, update_deleted and delete_missing.
- Unique-key conflicts get a new first line, "The row to insert violates a constraint", which replaces "Could not apply remote change".

**What the author claims.**
- Consistent wording makes these closely related messages easier to read.
- The new wording is closer to the conflict descriptions in the documentation.
- A supporter says consistency helps users match on wording in logs and helps future conflict types follow one style.

**Thread state.** The author got no replies for about two months and opened CommitFest entry 7043; Ajin Cherian later gave a +1 to the general idea. The author says he will not write a patch until there is agreement, and no patch is visible.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-05-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. New reason lines can contradict the detail lines they introduce

fixable · level 2, rubric item 12 · design · `standards-compat-1`

An apply worker hits insert_exists on a plain unique index that has no constraint and logs "The row to insert violates a constraint". The DBA looks in pg_constraint, finds no matching entry, and loses time before reading the next line about the unique index. In update_origin_differs, the log says "modified by another origin" and then "It was modified locally".

Message #1 adds summary lines that are less exact than the detail lines below them. The insert_exists and update_exists checks look only at unique indexes, and elsewhere Postgres names the object exactly, as in "violates unique constraint". Calling a local change "another origin" is defensible only from the remote side, and the message does not say so. The proposal is about consistency, so a committer would likely ask that the summary lines first agree with the detail lines they introduce.

Thread: [#1 (Peter Smith, 2026-05-18)](https://www.postgresql.org/message-id/CAHut%2BPuX_pPF6mU-UPRdQQToGgUPRpbOxOEFxPVC7B5Tevb-9A%40mail.gmail.com)

**Question:** Can the reason lines say exactly what was detected, such as a unique index and a local or other-origin change, without contradicting the detail line?

Counter-check: held. Message #1 pairs "modified by another origin" with "It was modified locally", and "violates a constraint" with detail lines that name only unique indexes. The pg_constraint half is weaker, because Postgres already reports plain unique index violations as "violates unique constraint", but the wording concern stands as fixable.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The reworded messages come from the logical replication apply worker; replication behaviour does not change. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Changed message text needs new translations, so the change fits a new major version only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | One-time translator and expected-output updates; later conflict types must follow the shared pattern. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can the reason line for insert_exists and update_exists name a unique index or unique key instead of "a constraint"?
2. How should the update_origin_differs and delete_origin_differs reason line read when the detail line says "It was modified locally"?
3. Does the logical replication conflicts page of the docs need changes to match the new text?
4. Which regression and TAP test expected outputs change, and by how much?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the logical replication apply worker and conflict detection code, and the error message style guide. No committer has spoken in the visible thread.
- **Liability:** The final wording of about a dozen DETAIL strings, their translations and the changed test output. The liability is small and ends once the strings settle.
- **Author follow-up:** The author pinged the thread twice and opened the CommitFest entry, and says in message #1 that he will write the patch once there is agreement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The change rewords strings in one function and benefits people who read apply-worker conflict logs. It is waiting for agreement from someone who knows the area, because the author will not write a patch without it and only one +1 has arrived.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no matching row. Message #1 says the author first raised this idea in another thread, where it was ruled out of scope. Being out of scope there is not a rejection on the merits, so no earlier rejection reason applies here.
- standards-compat: 1 objection(s). Breaking log parsers is not raised as an objection. Postgres does not promise stable message text across major versions, translations already vary it, and the stable conflict=<type> tag in the main message is unchanged.
- replication-ondisk: 0 objection(s). The proposal only rewords errdetail text in errdetail_apply_conflict, which the apply worker writes to the server log. It changes no on-disk format, WAL record, catalog, wire protocol message, replication slot or extension-facing struct, so this angle has nothing to object to. Mixed-version publisher and subscriber pairs see no change in behaviour.
- security: 0 objection(s). The proposal only rewords the DETAIL text of existing apply-worker conflict messages and adds no new data to them. The origin names, transaction ids, timestamps and key values stay in the same messages, which reach the same server log, so nobody can see or do anything new and there is no privilege or exposure aspect.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The change rewords existing DETAIL strings for one function in conflict.c. It adds no GUC, catalog, API, hot-path work or external dependency, so a single shared pattern should make later conflict types cheaper to add. The one-time cost is translator updates and expected-output changes in tests, which is normal for any message cleanup. Postgres does not treat message text as a stable interface, so breaking log parsers is not a fit objection. Without a patch I could not check the final strings or the test churn.
- already-exists: 0 objection(s). No existing feature, extension or tool rewrites these DETAIL strings, so this angle finds nothing already meeting the need. The proposal changes existing messages and adds no second mechanism or anything an extension could provide.
- demand: 0 objection(s). Demand evidence is thin: no user complaint, and one +1 in message #4. Even so, a demand objection would not hold up. Making message wording follow the project's error message style is routine upkeep and does not need field reports. The cost falls on translators and on tools that parse DETAIL text, and tools can match on the conflict type in the main message instead. These messages are recent, so changing them now costs less than changing them later. Without a patch, I could not check the final strings or how many translations they touch.

### Precedents retrieved

- P1: [Detection and resolution of logical replication conflict](https://commitfest.postgresql.org/patch/5021/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P2: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant; reason summarised from its thread
- P3: [Add errdetail() with PID and UID about source of termination signal](https://commitfest.postgresql.org/patch/6505/) (cf, Withdrawn, 2026-03-03) - judged not relevant; reason summarised from its thread
- P4: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/1745/) (cf, Returned with feedback, 2019-06-28) - judged not relevant; reason summarised from its thread
- P5: [Revert dubious message wording change.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1c8f03d4de5bca70437aaff5548a18a2d4e16a11) (revert, Reverted, 2009-06-22) - judged not relevant
- P6: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/2186/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P7: [DOCS: Make the Server Application docs synopses more consistent](https://commitfest.postgresql.org/patch/5647/) (cf, Withdrawn, 2026-03-23) - judged not relevant; reason summarised from its thread
- P8: [Allow using replication origins in SQL level parallel sessions](https://commitfest.postgresql.org/patch/6035/) (cf, Withdrawn, 2025-09-16) - judged not relevant; reason summarised from its thread
- P9: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P10: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P11: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant; reason summarised from its thread
- P12: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P13: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant
- P14: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P15: [Make selectivity example match wording](https://commitfest.postgresql.org/patch/3695/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P16: [NLS: use gettext() to translate system error messages](https://commitfest.postgresql.org/patch/6172/) (cf, Rejected, 2026-03-24) - judged not relevant
- P17: [Convert contrib SQL functions to new style](https://commitfest.postgresql.org/patch/3074/) (cf, Returned with feedback, 2021-09-01) - judged not relevant
- P18: [Revert refactoring of hex code to src/common/](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2576dcfb76aa71e4222bac5a3a43f71875bfa9e8) (revert, Reverted, 2021-08-19) - judged not relevant
- P19: [Tab-completion refactoring and word-removal feature (was IF-NOT_EXISTS for tab-completion)](https://commitfest.postgresql.org/patch/855/) (cf, Returned with feedback, 2017-03-17) - judged not relevant
- P20: [jsonb array-style subscription](https://commitfest.postgresql.org/patch/485/) (cf, Returned with feedback, 2016-03-21) - judged not relevant
- P21: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P22: [Mention column name in error messages](https://commitfest.postgresql.org/patch/324/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P23: [Mention column name in error messages](https://commitfest.postgresql.org/patch/818/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P24: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 73 s; cost reported by the CLI: $0.99
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:15:59.746Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
