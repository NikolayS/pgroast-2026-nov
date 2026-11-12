# Design pre-review: Reject incompatible options specified with VACUUM ONLY_DATABASE_STATS

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7503](https://commitfest.postgresql.org/patch/7503/) (Needs review); authors: Chao Li (chaol)
- Thread: [Reject incompatible options specified with VACUUM ONLY_DATABASE_STATS](https://www.postgresql.org/message-id/flat/466138D9-580D-4C7B-B614-3382AB093533%40gmail.com), started 2026-10-22
- Messages read: 4 of 4
- Patch prose read: v3-0001-Reject-incompatible-options-specified-with-VACUUM.patch, v1-0001-Reject-incompatible-options-specified-with-VACUUM.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is sound: `vacuum (ONLY_DATABASE_STATS, PARALLEL 1)` succeeds against the documented rule. Two fixable points remain. maintenance-fit-2 asks why v3 also rejects explicitly disabled options, which the current docs wording ("enabled") allows. maintenance-fit-1 notes that per-option flags accept a future option unless its author remembers to set the flag. Neither needs a different proposal.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so how the parser sets the conflict flag and whether a shared struct such as VacuumParams changes are not known. Message #4 was not shown; it may already address the choice between per-option flags and a loop over option names.

## The proposal, restated

**Problem.** The docs say ONLY_DATABASE_STATS allows no other option except VERBOSE, but the current check looks only at option bits. So `vacuum (ONLY_DATABASE_STATS, PARALLEL 1)` succeeds while `SKIP_LOCKED` fails, which can confuse users.

**Proposed design.** The patch records during parsing which incompatible options were given, following the "_given" pattern in publicationcmds.c and foreigncmds.c. It rejects any explicitly specified option other than VERBOSE, whatever its value, including explicitly disabled options and explicit PROCESS_MAIN and PROCESS_TOAST. Implicit defaults stay allowed.

**User-visible surface.**
- VACUUM ONLY_DATABASE_STATS now errors when given PARALLEL, INDEX_CLEANUP, TRUNCATE or BUFFER_USAGE_LIMIT.
- VACUUM ONLY_DATABASE_STATS now errors on explicitly disabled options and on explicit PROCESS_MAIN or PROCESS_TOAST, which were accepted before.
- The vacuum.sgml text changes from "no other option may be enabled" to "no other option may be specified".

**What the author claims.**
- The current behaviour does not match the documentation.
- PARALLEL, INDEX_CLEANUP, TRUNCATE and BUFFER_USAGE_LIMIT have no effect with ONLY_DATABASE_STATS.
- A presence-based check fits the generic error message, like the FULL with BUFFER_USAGE_LIMIT check.
- The "_given" pattern scales better to future options than a loop over option names.

**Thread state.** Jim Jones reviewed it, posted an alternative v2 that loops over option names in ExecVacuum, then agreed with keeping the v1 approach. v3 adds a test for `VERBOSE false`; no committer has replied.

## 2. Prior attempts

### COPY: validate option presence rather than option values (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/6744/](https://commitfest.postgresql.org/patch/6744/)
- **Thread:** [https://www.postgresql.org/message-id/flat/C1D2509E-E5D1-46B0-932C-B57AA7B963A1@gmail.com](https://www.postgresql.org/message-id/flat/C1D2509E-E5D1-46B0-932C-B57AA7B963A1@gmail.com)
- **Date:** decided 2026-06-29; started 2026-05-06
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** The same author proposed presence-based option validation for COPY and withdrew it; no reason is recorded. This proposal covers one VACUUM option and points to a documentation mismatch.


## 3. Objections a reviewer is likely to raise

### 1. Rejecting explicit PROCESS_TOAST/PROCESS_MAIN reverses a deliberate tolerance

fixable · level 1b, rubric item 6 · desirability, maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-2`

A maintenance script runs `vacuum (ONLY_DATABASE_STATS, PROCESS_TOAST false)` and works today. After the upgrade to 20 it fails with an error, and the operator finds out when a cron job or tool reports failure after the upgrade.

The current code has the comment "don't require people to turn off PROCESS_TOAST/MAIN explicitly" (quoted in message #2), so accepting these options looks intentional. The patch rejects them anyway, and also rejects explicitly disabled options that are harmless. No message gives a user report that asks for the stricter rule, and none weighs the compatibility cost or says whether vacuumdb sends such combinations. Fixing PARALLEL, INDEX_CLEANUP, TRUNCATE and BUFFER_USAGE_LIMIT does not require breaking these no-op forms.

Thread: [#1 (Chao Li, 2026-10-22)](https://www.postgresql.org/message-id/466138D9-580D-4C7B-B614-3382AB093533%40gmail.com), [#2 (Jim Jones, 2026-10-24)](https://www.postgresql.org/message-id/e88d7d21-7694-4913-bd4c-22266b737076%40uni-muenster.de)

**Question:** Why must explicitly disabled options and explicit PROCESS_TOAST/PROCESS_MAIN be rejected, when rejecting only the options that have effects would fix the reported bug?

Counter-check: held. The docs today say no other option may be "enabled", so explicitly disabled options are accepted as documented, and v3 changes that. The thread does not weigh this cost, and a check on enabled options would fix the reported cases.

### 2. Denylist of given options fails open for future VACUUM options

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later release adds a new VACUUM option and its author does not set the new conflict flag in the parser. Then `vacuum (ONLY_DATABASE_STATS, NEWOPT)` is accepted and silently ignored, the same bug this patch fixes. Nothing fails at build time or in the existing tests, so a user finds out when the documented rule does not hold.

The brief and commit message say the patch tracks the incompatible options that were given, which is a denylist. The allowlist loop in message #2 rejects any new option by default and fails closed. Message #4 says both approaches need the same care, but only the denylist needs every future option author to remember ONLY_DATABASE_STATS. Message #3 argues the "_given" pattern scales better, but the publicationcmds.c and foreigncmds.c cases mainly detect duplicate options and do not enforce an allowlist.

Thread: [#2 (Jim Jones, 2026-10-24)](https://www.postgresql.org/message-id/e88d7d21-7694-4913-bd4c-22266b737076%40uni-muenster.de), [#3 (Chao Li, 2026-10-26)](https://www.postgresql.org/message-id/897FDEBB-8721-43DE-83D0-82E068F93185%40gmail.com), [#4 (Jim Jones, 2026-10-27)](https://www.postgresql.org/message-id/404203a7-977b-49b9-b158-5006323f8e4d%40uni-muenster.de)

**Question:** When someone adds a VACUUM option next year without touching this check, what makes ONLY_DATABASE_STATS reject it, or which test fails?

Counter-check: held. Message #2 names a conflict flag that the parser sets per option, so a new option that does not set it is accepted. Message #4 says both approaches need the same care, but the loop over option names rejects unknown options by default.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | VACUUM option lists that are accepted today, such as `(ONLY_DATABASE_STATS, PROCESS_TOAST false)`, become errors; no new syntax is added. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix with target version 20, and the patch makes previously accepted commands fail. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each future VACUUM option must set the conflict flag in the parser, or ONLY_DATABASE_STATS accepts it silently. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why must explicitly disabled options and explicit PROCESS_TOAST or PROCESS_MAIN be rejected, when rejecting only enabled options would fix the reported cases?
2. Do vacuumdb or other known clients send any option other than VERBOSE together with ONLY_DATABASE_STATS?
3. When someone adds a VACUUM option without touching this check, what makes ONLY_DATABASE_STATS reject it, or which test fails?
4. Is the change meant for version 20 only, given that the entry is tagged Bugfix?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the VACUUM command code and its option parsing would have to own it; none has replied in the visible thread.
- **Liability:** The owner signs for a user-visible behaviour change in version 20: some VACUUM commands that worked start to fail. The liability is small and limited to option validation in vacuum.c.
- **Author follow-up:** The author answered the review within two days and posted v3 with the requested test.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 10 KiB, touches only vacuum.c, the VACUUM reference page and regression tests, and already has one reviewer. Few users combine other options with ONLY_DATABASE_STATS, and the two open points can be settled in the thread.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Rejecting explicitly disabled options breaks commands accepted since the feature began. Refuted (not specific): Duplicate of maintenance-fit-2. The quoted code comment covers implicit defaults, which v3 still allows, and [P1] concerned a behaviour that its code comment and commit message recorded as intentional.
- `standards-compat-1` Rejects commands that were accepted, with no evidence they caused harm. Refuted (not specific): Duplicate of maintenance-fit-2. The CommitFest entry targets version 20, so no back-patch is proposed, and the code comment covers implicit defaults, not an explicit `false`.
- `demand-1` Rejecting explicit 'off' options breaks working commands with no reported need. Refuted (not specific): Duplicate of maintenance-fit-2 for the compatibility point. The request for user reports is weak here, because `vacuum (ONLY_DATABASE_STATS, PARALLEL 1)` succeeds against the documented rule, and [P1] concerned a behaviour recorded as intentional.

### Review angles

- prior-attempts: 1 objection(s). [P19] records no reason, so it supports no objection. The core fix of rejecting enabled PARALLEL, INDEX_CLEANUP, TRUNCATE and BUFFER_USAGE_LIMIT draws no objection from the precedents.
- standards-compat: 1 objection(s). The SQL surface itself adds no keyword or syntax. The only concern from this angle is the stricter acceptance of option lists that were valid before.
- replication-ondisk: 0 objection(s). The proposal changes only how VACUUM validates its options when it parses them, plus one doc sentence and regression tests. It adds no on-disk format, WAL record, catalog column, replication or wire protocol change. The code is not shown, so it is not known whether a shared struct such as VacuumParams changes, which would matter for extension ABI only if the fix were back-patched; the target is version 20.
- security: 0 objection(s). The patch only tightens option validation for VACUUM ONLY_DATABASE_STATS. It changes which commands raise an error and adds no new capability, privilege path, exposed data or resource use. A user who could run the command before cannot do or see more after it, so there is no security or privilege aspect to object to.
- maintenance-fit: 2 objection(s). There is no hot-path cost, no new dependency, and no new API. The fix fits in core. The concerns are limited to how future options are covered and whether old commands should break.
- already-exists: 0 objection(s). No existing Postgres feature, contrib module or extension covers this. The patch tightens the error check that ExecVacuum already has for ONLY_DATABASE_STATS, so it changes an existing mechanism instead of adding a second way to do the job. [P19] looks at presence-based option checks for COPY, but its reason for withdrawal is not recorded, and the compatibility of rejecting explicit PROCESS_TOAST false belongs to another angle.
- demand: 1 objection(s). Fixing the mismatch between the docs and the code is reasonable. The objection is about rejecting commands that are harmless and work today, and nobody has asked for that.

### Precedents retrieved

- P1: [pg_stat_database update stats_reset only by pg_stat_reset](https://commitfest.postgresql.org/patch/2116/) (cf, Withdrawn, 2019-07-19) - used in the review; reason summarised from its thread
- P2: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P3: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P4: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P5: [Revert "vacuumdb: Add option for analyzing only relations missing stats."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19e57f4f78e4354d9a21c284868373d28bb0d368) (revert, Reverted, 2025-03-07) - judged not relevant
- P6: [VACUUM Progress Checker](https://commitfest.postgresql.org/patch/349/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P7: [Revert "Validate subscription conninfo on owner change"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ac230cd3ce93c4752489564cfe6b37b584a465ac) (revert, Reverted, 2026-08-05) - judged not relevant
- P8: [allow specifying action when standby encounters incompatible parameter settings](https://commitfest.postgresql.org/patch/3619/) (cf, Withdrawn, 2022-07-25) - judged not relevant; reason summarised from its thread
- P9: [history of vacuum / checkpoint runs](https://commitfest.postgresql.org/patch/5462/) (cf, Withdrawn, 2025-11-14) - judged not relevant; reason summarised from its thread
- P10: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant; reason summarised from its thread
- P11: [Block level parallel vacuum](https://commitfest.postgresql.org/patch/1223/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P12: [Add MAIN_RELATION_CLEANUP and TOAST_TABLE_CLEANUP options to VACUUM](https://commitfest.postgresql.org/patch/2426/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P13: [Revert "Reject degenerate SPLIT PARTITION with DEFAULT partition"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0392fb900eb89f52988cccd33046443c39c70d1c) (revert, Reverted, 2026-05-20) - judged not relevant
- P14: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant
- P15: [pg_stat_toast](https://commitfest.postgresql.org/patch/3457/) (cf, Withdrawn, 2022-07-01) - judged not relevant
- P16: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P17: [Revert MAINTAIN privilege and pg_maintain predefined role.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=151c22deee66a3390ca9a1c3675e29de54ae73fc) (revert, Reverted, 2023-07-07) - judged not relevant
- P18: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant
- P19: [COPY: validate option presence rather than option values](https://commitfest.postgresql.org/patch/6744/) (cf, Withdrawn, 2026-06-29) - used in the review
- P20: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant
- P21: [Block level parallel vacuum](https://commitfest.postgresql.org/patch/734/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P22: [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P23: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - judged not relevant
- P24: [Support to COMMENT ON DATABASE CURRENT_DATABASE](https://commitfest.postgresql.org/patch/1320/) (cf, Rejected, 2018-03-06) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (maintenance-fit-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 116 s; cost reported by the CLI: $1.25
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 70 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:02:32.901Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
