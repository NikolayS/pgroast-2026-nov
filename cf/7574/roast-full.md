# Design pre-review: psql: Don't offer USING INDEX after REPACK (CONCURRENTLY) or (ANALYZE)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7574](https://commitfest.postgresql.org/patch/7574/) (Needs review); authors: Ishjaap Singh (vennictus)
- Thread: [psql: Don't offer USING INDEX after REPACK (CONCURRENTLY) or (ANALYZE)](https://www.postgresql.org/message-id/flat/CA%2BY2BmeqAVC9Vhs%2BzyXzD%2Bu3DkNF8%3DzhRs%3DiTeK5LgFvfuHa8A%40mail.gmail.com), started 2026-10-31
- Messages read: 1 of 1
- Patch prose read: v1-0001-psql-Don-t-offer-USING-INDEX-after-REPACK-CONCURR.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is stated with the two failing commands and their error messages, and the cost stays inside psql tab completion. One fixable design objection survived, maintenance-fit-1: psql would carry its own copy of a server rule about REPACK options, and nothing ties the two together. That is a judgment for a committer who knows tab-complete.in.c, together with the back-patch question. Only one message is visible and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the reader's handling of unusual option lists and its fit with the existing completion rules are unchecked. The libedit behaviour is untested by the author.

## The proposal, restated

**Problem.** psql tab completion after a REPACK option list offers USING INDEX even when CONCURRENTLY or ANALYZE is enabled. Without a table name, REPACK USING INDEX processes every eligible table, which neither option allows, so the suggested command always fails. This affects psql users on master and REL_19_STABLE.

**Proposed design.** psql reads the completed REPACK option list and leaves out USING INDEX when CONCURRENTLY or ANALYZE (or ANALYSE) ends up enabled, with the last setting of an option winning as on the server. The reader only interprets plain unquoted option names and values. For quoted names or values, signed numbers or other forms, psql keeps today's completions.

**User-visible surface.**
- psql tab completion after REPACK (option list) no longer offers USING INDEX when CONCURRENTLY or ANALYZE is enabled.

**What the author claims.**
- REPACK (CONCURRENTLY) USING INDEX and REPACK (ANALYZE) USING INDEX without a table name always fail, so the suggestion is never useful.
- USING INDEX is still offered with no options, with VERBOSE, with both options off, and after an explicit table name.
- Handling quoted names, quoted values and signed numbers took about 80 more lines, which the author judged not worth it.
- New 010_tab_completion.pl cases fail without the fix, and all psql TAP tests pass on master and REL_19_STABLE with GNU readline.
- The patch applies to REL_19_STABLE; older branches lack REPACK and are not affected.

**Thread state.** Only the first message is visible. No one has replied, and back-patching to REL_19_STABLE and the choice of the simple or the 80-line parser are open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-31; 23 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Tab completion copies a server-side semantic rule that can drift

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), psql UX conventions · `maintenance-fit-1`

A later release lets REPACK (ANALYZE) or (CONCURRENTLY) run over all tables with a clustering index. The developer changes the server check and the docs but not tab-complete.in.c. psql keeps hiding USING INDEX for a valid command, no test fails, and users find out only when they notice the completion is missing.

Most psql completion follows the grammar, but this patch encodes a rule from the server's execution checks: which options forbid the no-table form. Message #1 adds its own reader for the option list, with last-setting-wins semantics, so psql now has a second copy of option evaluation that has to match the server. Nothing ties the two copies together. A newer psql connected to an older server, or the reverse, has no server-version check that would show the rule is per release.

Thread: [#1 (Ishjaap Singh, 2026-10-31)](https://www.postgresql.org/message-id/CA%2BY2BmeqAVC9Vhs%2BzyXzD%2Bu3DkNF8%3DzhRs%3DiTeK5LgFvfuHa8A%40mail.gmail.com)

**Question:** What would make a later change to the server's REPACK option rules also update or break this completion, and how should psql handle servers of a different major version?

Counter-check: held. The patch adds an option-list reader with last-setting-wins rules that copies a server execution check, and the new tests would not catch a server change. The cross-version part has no case today because REPACK is new in 19, and the drift only hides a suggestion.

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
| psql UX conventions | **yes** | psql stops offering USING INDEX after a REPACK option list that enables CONCURRENTLY or ANALYZE. |
| security / privileges | no |  |
| backpatchability | **yes** | Message #1 asks whether to back-patch the client-side change to REL_19_STABLE. |
| maintenance burden ("who maintains this in 10 years") | **yes** | tab-complete.in.c gains a REPACK option list reader that must keep matching the server's option rules. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. If a later release relaxes the server's REPACK option rules, what would make this completion get updated or make a test fail?
2. Does any other code in tab-complete.in.c already evaluate option values, or is this option list reader the first of its kind there?
3. What does the option list reader do when a value-less option is followed by an unknown option name?
4. Do the new 010_tab_completion.pl cases pass when psql is built with libedit?
5. Do other REPACK options or option combinations also lead to a completion that always fails?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows psql tab completion (tab-complete.in.c) and the REPACK option rules would have to own it. No one has replied to message #1.
- **Liability:** The owner signs for a small psql-side reader of the REPACK option list that must match the server's rules for as long as REPACK keeps them. If the two drift, psql hides a valid suggestion and the command itself still works.
- **Author follow-up:** Message #1 offers to post the 80-line version that handles quoted names and values if reviewers prefer it. No other sign.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The 7.1 KB patch touches only tab-complete.in.c and its TAP test and helps psql users who type REPACK with these options. The one surviving objection is fixable, and its worst case is a missing suggestion.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] changed how psql splits words inside single-quoted strings, which is a different idea, and its recorded reason does not apply here: this patch changes no word splitting and keeps today's completions when the option list contains quoted text. Message #1 cites only a related committed fix for ON/OFF completion in the REPACK option list, not an earlier rejected attempt.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch adds no syntax, keyword, GUC or catalog state, nothing pg_dump must round-trip, and no change to what clients see. The only visible change is that psql stops suggesting USING INDEX after REPACK (CONCURRENTLY) or (ANALYZE), where the server rejects it. One point other angles could weigh: psql completion now copies a server rule and should follow the server if that rule is relaxed. That is not a compatibility failure today.
- replication-ondisk: 0 objection(s). The patch changes only psql tab completion and its TAP test. It touches no on-disk format, WAL, catalog, wire protocol or server ABI, so this angle has nothing to object to. Back-patching to REL_19_STABLE carries no ABI risk because the change is client-side only.
- security: 0 objection(s). The patch changes only which words psql tab completion offers after a REPACK option list. It adds no privilege, exposure or denial-of-service path, since completion runs on the client and the server still checks the command.
- maintenance-fit: 1 objection(s). Beyond the drift risk, the cost falls only on psql tab completion users. The patch has no hot-path, catalog or API impact, and it is small. [P3] concerns word splitting inside quoted strings and does not apply here.
- already-exists: 0 objection(s). No existing Postgres feature, contrib module or extension filters psql tab completion of REPACK USING INDEX by option values, and this kind of fix can only live in tab-complete.in.c. The proposal adds no new mechanism: it narrows an existing completion rule. [P3] concerns word-break handling inside quoted strings and does not apply here.
- demand: 0 objection(s). No demand objection. This fixes a tab completion that always suggests a failing command, every psql user typing REPACK with these options can hit it, and the cost stays inside tab-complete.in.c. Whether a general option-list reader is worth adding for one suggestion is a maintainability question for other angles. [P3] concerns quoted-string word splitting and does not apply.

### Precedents retrieved

- P1: [psql: Add tab-complete for optional view parameters](https://commitfest.postgresql.org/patch/4053/) (cf, Returned with feedback, 2023-07-06) - judged not relevant; reason summarised from its thread
- P2: [Tab-completion refactoring and word-removal feature (was IF-NOT_EXISTS for tab-completion)](https://commitfest.postgresql.org/patch/855/) (cf, Returned with feedback, 2017-03-17) - judged not relevant; reason summarised from its thread
- P3: [improve tab-completion behavior when parse text in single quote](https://commitfest.postgresql.org/patch/3267/) (cf, Returned with feedback, 2021-12-03) - used in the review; reason summarised from its thread
- P4: [Tab completion for SELECT in psql](https://commitfest.postgresql.org/patch/1376/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P5: [tab-completion debug print](https://commitfest.postgresql.org/patch/1917/) (cf, Returned with feedback, 2019-02-02) - judged not relevant; reason summarised from its thread
- P6: [remove VersionedQuery support from psql tab completion](https://commitfest.postgresql.org/patch/6977/) (cf, Withdrawn, 2026-08-11) - judged not relevant; reason summarised from its thread
- P7: [psql: add tab completion for \df slash command suffixes](https://commitfest.postgresql.org/patch/2268/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P8: [Fix psql-completion for CREATE INDEX](https://commitfest.postgresql.org/patch/452/) (cf, Returned with feedback, 2016-02-03) - judged not relevant; reason summarised from its thread
- P9: [IF (NOT) EXISTS in psql-completion](https://commitfest.postgresql.org/patch/518/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P10: [Tab completion for ATTACH PARTITION](https://commitfest.postgresql.org/patch/4561/) (cf, Returned with feedback, 2024-01-29) - judged not relevant
- P11: [REPACK (CONCURRENTLY) doesn't handle invalid indexes](https://commitfest.postgresql.org/patch/7039/) (cf, Returned with feedback, 2026-09-13) - judged not relevant
- P12: [Track skipped vacuum and analyze in pg_stat_all_tables](https://commitfest.postgresql.org/patch/6640/) (cf, Withdrawn, 2026-04-03) - judged not relevant
- P13: [REPACK (CONCURRENTLY) doesn't handle invalid indexes](https://commitfest.postgresql.org/patch/7040/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P14: [Revert "vacuumdb: Add option for analyzing only relations missing stats."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19e57f4f78e4354d9a21c284868373d28bb0d368) (revert, Reverted, 2025-03-07) - judged not relevant
- P15: [Vertical Clustered Index (columnar storage extension)](https://commitfest.postgresql.org/patch/945/) (cf, Returned with feedback, 2017-03-06) - judged not relevant
- P16: [Revert analyze support for partitioned tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8127b7390119c21479f5ce495b7d2168930e82) (revert, Reverted, 2021-08-16) - judged not relevant
- P17: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P18: [Allow subfield references without parentheses](https://commitfest.postgresql.org/patch/5449/) (cf, Returned with feedback, 2025-02-14) - judged not relevant
- P19: [Autovacuum (analyze) on partitioned tables for ATTACH/DETACH/DROP commands](https://commitfest.postgresql.org/patch/3188/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [Confusing list foreign table options in psql - Fixing slash usage text](https://commitfest.postgresql.org/patch/2545/) (cf, Returned with feedback, 2020-11-29) - judged not relevant
- P21: [Fix tab completion MERGE](https://commitfest.postgresql.org/patch/3890/) (cf, Returned with feedback, 2023-07-04) - judged not relevant
- P22: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant
- P23: [Tab completion for CREATE SCHEMA AUTHORIZATION](https://commitfest.postgresql.org/patch/3252/) (cf, Returned with feedback, 2021-12-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 142 s; cost reported by the CLI: $1.13
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 8 verified, 0 dropped; relevance pass: done, 1 of 23 kept
- Tone check: 49 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T00:59:29.909Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
