# Design pre-review: Include schema-qualified names in publication error messages (generalised to all of LR).

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7604](https://commitfest.postgresql.org/patch/7604/) (Needs review); authors: Peter Smith (smithpb2250)
- Thread: [Include schema-qualified names in publication error messages.](https://www.postgresql.org/message-id/flat/CAFiTN-scG7b11Jsp%2BVoDRT8ZFE84eSKLcDsSB18dZ8AaP%3DR-mw%40mail.gmail.com), started 2026-04-28
- Messages read: 39 of 39
- Patch prose read: v4-0001-Include-schema-qualified-names-in-publication-err.patch, v1-0001-Include-schema-qualified-names-in-publication-err.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal, and the demand objections were refuted. Three fixable design points remain at level 2: no written rule for which messages qualify names (prior-attempts-1), the unstated fate of the pushed RelationGetQualifiedRelationName() (maintenance-fit-1), and the open pg_temp versus pg_temp_N choice (maintenance-fit-3). None needs a different proposal, but a committer in this area has to settle them. The code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check whether patch 0001 leaves one helper or two for message use, or which call sites remain unqualified. The full list of converted messages was not shown.

## The proposal, restated

**Problem.** Many logical replication error and log messages name a relation without its schema, which is ambiguous when the same name exists in several schemas. Other messages build the qualified name by hand, or quote it twice inside a message that already supplies quotes. Users and administrators reading these messages are the ones affected.

**Proposed design.** Patch 0001 adds three helpers in relcache.c that return an unquoted "schema.relation" string for %s substitution, and converts every logical replication call site to use them, including index names. Patch 0002 removes get_qualified_objname() and inlines it into generate_qualified_relation_name() in ruleutils.c. The entry extends an earlier pushed EXCEPT-clause fix and Vignesh's v8 for check_publication_add_relation().

**User-visible surface.**
- Text of about 20 logical replication error, detail and log messages changes: relations, tables, sequences and indexes appear as schema.relation.
- Messages that used "%s.%s" change to a single "%s", so translations go fuzzy.
- Temporary tables appear with schema pg_temp_N in these messages.
- C API: new MsgQualifiedRelName(), MsgQualifiedRelNameByRel(), MsgQualifiedRelNameByRelId() in relcache.c.
- C API: get_qualified_objname() is removed.

**What the author claims.**
- The same name lookup is done ad hoc in three inconsistent ways across more than a dozen logical replication call sites.
- Unqualified names are ambiguous in routine per-schema publication setups.
- Commit 926627bf902 shows that double quoting already caused a real bug.
- Per the error style guide, the helpers should not quote, because the messages supply quotes.
- Translator cost is low early in the v20 cycle, since gettext marks changed strings fuzzy and keeps the old translation.

**Thread state.** The first part, for the EXCEPT clause, was pushed for v19; Vignesh's v8 for check_publication_add_relation() waits, and he wanted the wider change in a separate thread. Kuroda gave +1 to the idea but raised translator cost and preferred pg_temp over pg_temp_N; nobody else has reviewed the generalised patches.

## 2. Prior attempts

### pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled (related)

- **Link:** [https://commitfest.postgresql.org/patch/3811/](https://commitfest.postgresql.org/patch/3811/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com)
- **Date:** decided 2022-11-30; started 2022-08-10
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Alvaro Herrera (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane said pg_get_indexdef is the odd one out, since most generate_relation_name calls in ruleutils.c ignore the pretty flag, and Alvaro Herrera judged the patch too small for making PRETTYFLAG_SCHEMA work consistently. Tom asked for a defined rule on which call sites qualify names, with a documented helper function. The author did not reply to these reviews, and Michael Paquier returned the patch after six weeks without a reply.
- **Reason (quote):** "In other words, I think your patch is too small." ([message](https://www.postgresql.org/message-id/20220810085850.ghwvrkcqzimif5ok%40alvherre.pgsql))
- **What is different this time:** P5 changed deparsed SQL at one ruleutils.c call site. This patch changes error and log message text through shared helpers across many call sites, but it defines its scope by subsystem.

### Explicit relation name in VACUUM VERBOSE log (related)

- **Link:** [https://commitfest.postgresql.org/patch/1246/](https://commitfest.postgresql.org/patch/1246/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAD21AoCtiAHCmtNQV2o8FjhEEEKSjRYCgzBOo-uzq86h96Nvhw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAD21AoCtiAHCmtNQV2o8FjhEEEKSjRYCgzBOo-uzq86h96Nvhw@mail.gmail.com)
- **Date:** decided 2017-10-02; started 2017-08-15
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane objected that VACUUM VERBOSE output is meant for people reading it in the client, and repeating the full table name on every line would be annoying for that use. He also said scraping INFO messages is a poor workflow, since translation changes them. Robert Haas doubted that qualifying names everywhere was useful and noted that the scope had grown. Daniel Gustafsson returned the patch because these concerns stayed open and the patch had no update during the commitfest.
- **Reason (quote):** "Repeating the full table name in every line will be really annoying for that primary use-case." ([message](https://www.postgresql.org/message-id/14199.1503585323%40sss.pgh.pa.us))
- **What is different this time:** P1 qualified names on every VACUUM VERBOSE line read in a client. This patch changes single-line error and log messages, where the name is not repeated.


## 3. Objections a reviewer is likely to raise

### 1. Two relation-name helpers with opposite quoting rules

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch author needs a qualified name for an errmsg and finds RelationGetQualifiedRelationName() next to MsgQualifiedRelNameByRel() in relcache.c. They pick the quoting one inside "%s", and the double-quote bug fixed by 926627bf902 comes back. No test or build check catches it.

Message #28 placed a quoting helper in relcache.c, and message #31 pushed it. Patch 0001 adds three non-quoting helpers in the same file, and 0002 removes only get_qualified_objname(). The proposal does not say whether RelationGetQualifiedRelationName() is kept, renamed or replaced. Two near-identical helpers that differ only in quoting are the API shape most likely to be misused.

Thread: [#28 (Amit Kapila, 2026-05-07)](https://www.postgresql.org/message-id/CAA4eK1KL809TGPcK1%3DbFOs3i44Htxv9Cv88MBiO5taoSchPXGw%40mail.gmail.com), [#31 (Amit Kapila, 2026-05-08)](https://www.postgresql.org/message-id/CAA4eK1KsY8qppUKTZ9gRALCojJrj%2BCM9gLoDxj6CViS99S9JCA%40mail.gmail.com), [#36 (vignesh C, 2026-09-24)](https://www.postgresql.org/message-id/CALDaNm2dPWz8_GcRzvhX5pk%2BgSVnU_uDTQ9wkzZsUYRn3WeYrg%40mail.gmail.com), [#37 (Peter Smith, 2026-10-07)](https://www.postgresql.org/message-id/CAHut%2BPsQ-77Mw5Ujj293kgo1GGZwfUhx90K1XgqePAHQet%2BtMw%40mail.gmail.com)

**Question:** Is RelationGetQualifiedRelationName() removed or redefined by this series, and how would a new caller know which helper fits an errmsg?

Counter-check: held. Message #37 removes get_qualified_objname() but does not say what happens to the pushed RelationGetQualifiedRelationName(). Without the code, the material cannot show that only one helper remains for message use.

### 2. Logical replication boundary gives no rule for which messages qualify names

fixable · level 2, rubric item 12 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A user runs `alter table` on a published table and gets a tablecmds.c error naming "public.t". A later error from the same command, outside logical replication, names it as "t". A future patch adds a new logical replication message with RelationGetRelationName(), and no test or check catches it.

[P5] was returned after reviewers asked for a defined rule on which call sites qualify names, backed by a documented helper; its reason is an AI summary, so it is only a pointer. Here the patch converts messages in shared files such as tablecmds.c, objectaddress.c and execReplication.c because they mention publications, not because of a general rule. The brief notes there is no statement for messages outside logical replication, and message #33 shows matview errors with the same double-quoting left as is. Message #38 lists four kinds of call sites kept unqualified by judgement.

**Precedents.**
- [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (Returned with feedback, 2022-11-30); thread: [link](https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com)

Thread: [#33 (vignesh C, 2026-05-11)](https://www.postgresql.org/message-id/CALDaNm17Vh8OccJDD97%3D88R5fSyo-BtJ4aKkfMtkGSDGsYDWEA%40mail.gmail.com), [#37 (Peter Smith, 2026-10-07)](https://www.postgresql.org/message-id/CAHut%2BPsQ-77Mw5Ujj293kgo1GGZwfUhx90K1XgqePAHQet%2BtMw%40mail.gmail.com), [#38 ("Hayato Kuroda (Fujitsu)", 2026-10-16)](https://www.postgresql.org/message-id/OS7PR01MB183177A958E31DC4239D23BA2F59B2%40OS7PR01MB18317.jpnprd01.prod.outlook.com)

**Question:** What written rule tells a future patch author whether a new message must use MsgQualifiedRelName(), and why does it stop at logical replication?

Counter-check: held. The proposal states no rule beyond "all LR call sites", and message #38 shows the kept exceptions rest on judgement. [P5] concerns deparsed SQL in ruleutils.c, so it is only a weak pointer and must not be used as support.

### 3. pg_temp_N makes temp-table messages untestable by text

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

A regression test hits one of the converted messages with a temporary table. The output contains pg_temp_N, which differs by backend, so the test fails at random. The test then falls back to VERBOSITY sqlstate and stops checking the message text.

Message #36 already had to switch a publication test to sqlstate-only output for this reason. Patch 0001 extends the same behaviour to about 20 messages and keeps pg_temp_N. Message #38 asked for pg_temp, and the thread has no decision. Each future test on these paths carries the same workaround.

Thread: [#36 (vignesh C, 2026-09-24)](https://www.postgresql.org/message-id/CALDaNm2dPWz8_GcRzvhX5pk%2BgSVnU_uDTQ9wkzZsUYRn3WeYrg%40mail.gmail.com), [#37 (Peter Smith, 2026-10-07)](https://www.postgresql.org/message-id/CAHut%2BPsQ-77Mw5Ujj293kgo1GGZwfUhx90K1XgqePAHQet%2BtMw%40mail.gmail.com), [#38 ("Hayato Kuroda (Fujitsu)", 2026-10-16)](https://www.postgresql.org/message-id/OS7PR01MB183177A958E31DC4239D23BA2F59B2%40OS7PR01MB18317.jpnprd01.prod.outlook.com)

**Question:** Which display of temp schemas is chosen, and how do regression tests for the converted messages keep checking message text?

Counter-check: held. Message #36 shows a core test already dropped its text check because of pg_temp_N, and message #37 leaves the choice open. The author calls it a one-line change, but the thread has no decision yet.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Patch 0001 adds three MsgQualifiedRelName*() helpers in relcache.c, and patch 0002 removes get_qualified_objname(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The text of about 20 existing messages changes, and message #19 objects to backpatching; the target is v20. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Future logical replication messages must use the new helpers, and nothing fails when a new message prints a bare name. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the series keep, rename or remove RelationGetQualifiedRelationName(), and how does a new caller know which helper fits a message that already supplies quotes?
2. What written rule tells a future patch author whether a new message must use MsgQualifiedRelName(), and where is that rule documented?
3. Do the converted messages print pg_temp or pg_temp_N for temporary tables, and how do regression tests keep checking the message text?
4. Which logical replication messages still print unqualified names after patch 0001, and what is the reason for each exception listed in message #38?
5. Should messages outside logical replication, such as the matview errors in message #33, follow the same rule in a later patch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Amit Kapila pushed the first part (message #31) but has not reviewed the generalised patches. The owner needs to know logical replication messages and the relcache.c helper conventions.
- **Liability:** The owner signs for the text of about 20 messages, the translation update, and a helper API in relcache.c that later patches will copy. The risk is low and limited to error and log paths.
- **Author follow-up:** Peter Smith created the CommitFest entry on 2026-11-05 and answered the translator concern in message #39, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The change helps logical replication users who have the same relation name in several schemas, and it touches only error and log paths. The three open design points are small and can be settled in the existing thread.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Logical replication gets its own naming rule with no project-wide rule. Refuted (not specific): Duplicate of prior-attempts-1 for the missing rule and of maintenance-fit-1 for the two helpers. The matview nested quoting in its scenario exists today and is not caused by this patch.
- `standards-compat-2` pg_temp_N makes temporary-table error text change between runs. Refuted (not specific): Duplicate of maintenance-fit-3, which rests on the core regression test evidence in message #36.
- `maintenance-fit-2` General helper, but the qualification rule covers only logical replication. Refuted (not specific): Duplicate of prior-attempts-1, which makes the same point with the message #38 exceptions as evidence.
- `already-exists-1` New message helpers sit beside the pushed RelationGetQualifiedRelationName(). Refuted (not specific): Duplicate of maintenance-fit-1. [P5] concerns ruleutils.c deparse output and does not support a point about message helpers.
- `demand-1` No user report shows unqualified names confuse anyone in these messages. Refuted (already handled): In message #15 a committer calls the current behaviour not correct, and a worker log line that names only "t" gives the reader no other way to find the schema. [P1] concerns repeated VACUUM VERBOSE lines read in a client, which is a different case.
- `demand-2` Message churn falls on log tools and translators, not on feature users. Refuted (already handled): Message #19 opens with "-1 for backpatching" and says "the target is v20", so the log tool concern was about back branches. Message #39 answers the translator cost, and message text changes in a major release are normal practice.

### Review angles

- prior-attempts: 1 objection(s). The repetition concern in P1 does not apply to one-line errors. Only the scope-rule point from P5 still applies.
- standards-compat: 2 objection(s). Euler's log-tool concern in message #19 was mainly about backpatching. For HEAD, changes to message text are routine, so I did not raise it as a separate objection.
- replication-ondisk: 0 objection(s). The proposal changes only the text of server error and log messages and some backend helper functions; it adds no on-disk, WAL, catalog or protocol state. Kuroda (message #38) confirms that replication protocol messages still send namespace and relation separately, so publishers, subscribers and mixed-version setups see no change. Removing get_qualified_objname(), which shipped in v19, is a normal major-release API change.
- security: 0 objection(s). The proposal changes only how existing messages print a relation name, so it has no privilege or exposure aspect. The added schema names belong to relations the session already named or owns, or they appear in server logs that only administrators read.
- maintenance-fit: 3 objection(s). No hot-path cost. These are error and log paths only. Amit Kapila, a committer in this area, engaged in the first part but has not yet reviewed the generalised patches.
- already-exists: 1 objection(s). No existing feature or extension gives schema-qualified names in these messages today. The only overlap is the in-core helper family, raised above.
- demand: 2 objection(s). The generalised patch has two demand gaps: no user report, and costs that fall on everyone. Neither is fatal; evidence of user confusion would settle both.

### Precedents retrieved

- P1: [Explicit relation name in VACUUM VERBOSE log](https://commitfest.postgresql.org/patch/1246/) (cf, Returned with feedback, 2017-10-02) - used in the review; reason summarised from its thread
- P2: [Mention column name in error messages](https://commitfest.postgresql.org/patch/324/) (cf, Returned with feedback, 2015-12-22) - judged not relevant; reason summarised from its thread
- P3: [Fix pg_stat_statements query jumbling to use relation name instead of relation oid](https://commitfest.postgresql.org/patch/365/) (cf, Rejected, 2015-11-15) - judged not relevant; reason summarised from its thread
- P4: [Mention column name in error messages](https://commitfest.postgresql.org/patch/818/) (cf, Returned with feedback, 2016-11-29) - judged not relevant; reason summarised from its thread
- P5: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - used in the review; reason summarised from its thread
- P6: [simple decomposition of qualified identifier](https://commitfest.postgresql.org/patch/337/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P7: [Move all am-related reloption code into src/backend/access/\[am-name\] and get rid of relopt_kind for custom AM](https://commitfest.postgresql.org/patch/992/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P8: [Revert "Improve error message on snapshot import in snapmgr.c"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=be022908cfe2959dbb17c10475deeb492b8e6944) (revert, Reverted, 2023-09-14) - judged not relevant
- P9: [HINT on qualified LHS references in UPDATE .. SET](https://commitfest.postgresql.org/patch/60/) (cf, Rejected, 2015-01-16) - judged not relevant
- P10: [NLS: use gettext() to translate system error messages](https://commitfest.postgresql.org/patch/6172/) (cf, Rejected, 2026-03-24) - judged not relevant; reason summarised from its thread
- P11: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P12: [Revert "pg_createsubscriber: Add log message when no publications exist to drop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=45a7faf130176ae7c152413967c7c90068476a94) (revert, Reverted, 2025-10-15) - judged not relevant
- P13: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P14: [Decoupling our alignment assumptions about int64 and double](https://commitfest.postgresql.org/patch/6441/) (cf, Withdrawn, 2026-02-03) - judged not relevant
- P15: [Logical Replication Helper Functions](https://commitfest.postgresql.org/patch/45/) (cf, Rejected, 2015-02-16) - judged not relevant
- P16: [Optimize walsender handling invalid messages of 'drop publication'](https://commitfest.postgresql.org/patch/4393/) (cf, Withdrawn, 2023-10-20) - judged not relevant
- P17: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P18: [Revert "docs: change "default role" wording to "predefined role""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c185a57753e64fe3ffc996ea18da8672b302a5a4) (revert, Reverted, 2020-02-09) - judged not relevant
- P19: [Revert "Forbid DROP SCHEMA on temporary namespaces"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65192e02441cedd106b6abebe0036fb8cc124fb3) (revert, Reverted, 2020-01-08) - judged not relevant
- P20: [Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet](https://commitfest.postgresql.org/patch/4077/) (cf, Withdrawn, 2022-12-27) - judged not relevant
- P21: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1710/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P22: [Add Hook Functions for Disk Quota Extension](https://commitfest.postgresql.org/patch/1883/) (cf, Returned with feedback, 2020-07-31) - judged not relevant
- P23: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant
- P24: [Revert dubious message wording change.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1c8f03d4de5bca70437aaff5548a18a2d4e16a11) (revert, Reverted, 2009-06-22) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 21; wall time: 198 s; cost reported by the CLI: $2.86
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 8 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 97 text field(s) checked, 3 finding(s), 3 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:05:31.902Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
