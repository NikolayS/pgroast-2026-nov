# Design pre-review: Validate JSON object keys before omitting null values (BUG #19736)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7605](https://commitfest.postgresql.org/patch/7605/) (Needs review); authors: Yeongjae Min (yeongjaemin)
- Thread: [BUG #19736: `JSON_OBJECT ... ABSENT ON NULL` accepts invalid keys when the value is NULL](https://www.postgresql.org/message-id/flat/19736-3a44617ea5534c77%40postgresql.org), started 2026-10-02
- Messages read: 8 of 8
- Patch prose read: v1-0001-Validate-JSON-object-keys-before-omitting-null-values.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the goal of consistent key validation is clear. standards-compat-1 and standards-compat-2 are fixable points on back-patch scope, a release note and the SQL standard's rule for omitted pairs. A committer who knows SQL/JSON has to decide those, and none has spoken in the visible thread.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that omitted keys never reach output or cast functions is unchecked. Only the v1 commit message and 3 of 8 messages were available, so the v2 helper and documentation text could not be judged.

## The proposal, restated

**Problem.** JSON_OBJECT with ABSENT ON NULL skips key validation when the value is NULL, so it returns {} for a NULL key or an array key. The same keys are rejected when the value is not NULL or when uniqueness is checked. The reporter reads the documented key restrictions as applying to omitted pairs too.

**Proposed design.** The patch moves the json NULL-key check before the ABSENT ON NULL skip. Before an entry is skipped without a uniqueness check, it classifies the key type and rejects the same categories the conversion routines reject, without calling output or cast functions. v2 adds a shared helper json_validate_key_category(), applies the same fix to json_object_agg_strict() and jsonb_object_agg_strict(), and updates the documentation.

**User-visible surface.**
- JSON_OBJECT ... ABSENT ON NULL now errors on NULL, array, composite, json and jsonb keys when the value is NULL.
- JSON_OBJECTAGG ... ABSENT ON NULL gets the same key validation.
- json_object_agg_strict() and jsonb_object_agg_strict() reject invalid keys paired with NULL values.
- The documentation states that key restrictions apply even when the pair is omitted.

**What the author claims.**
- The new regression tests fail without the fix, and all 239 core regression tests pass with it on macOS/arm64 with assertions.
- Omitted keys still do not invoke output conversion without uniqueness checking, and WITH UNIQUE KEYS behaves as before.
- jsonb keys were already covered in v1 because json_categorize_type() maps jsonb to JSONTYPE_JSON.
- v3 is a rebase with no behavior changes from v2.
- A second poster's 260-case test matrix found 19 invalid-key cases now rejected and no other result changes.

**Thread state.** Dmitry Dolgov asked for a documentation change, a check of similar aggregate cases, a shared helper, and an explanation for JSONTYPE_JSONB; v2 and v3 address these. No one has replied to v2 or v3, and no committer has said whether the change should be back-patched.

## 2. Prior attempts

### COPY: validate option presence rather than option values (related)

- **Link:** [https://commitfest.postgresql.org/patch/6744/](https://commitfest.postgresql.org/patch/6744/)
- **Thread:** [https://www.postgresql.org/message-id/flat/C1D2509E-E5D1-46B0-932C-B57AA7B963A1@gmail.com](https://www.postgresql.org/message-id/flat/C1D2509E-E5D1-46B0-932C-B57AA7B963A1@gmail.com)
- **Date:** decided 2026-06-29; started 2026-05-06
- **Outcome:** Withdrawn (cf)
- **Who objected:** Masahiko Sawada (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Masahiko Sawada argued that "allowed only when" can be read as "allowed to be enabled only when", and that the patch makes COPY less flexible for tools that generate COPY commands with explicit true/false values. He added that VACUUM treats its boolean options the same way. The author withdrew the patch after Fujii-san raised the same objection on a related patch.
- **Reason (quote):** "My concern is that the proposed patch could make the command unnecessarily less flexible." ([message](https://www.postgresql.org/message-id/CAD21AoCsbjvgk5tOarepYF28dW7P6NG8SLVDQ72A7AJ3yEjriw%40mail.gmail.com))
- **What is different this time:** P11 rejected COPY options by their presence instead of their value. This patch rejects JSON keys whose pair is dropped anyway. Both add checks on input that does not affect the result.

### Revert cascading of JSON_TABLE's ON ERROR (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=af6fad879fbf9b542cfde05c52695c9dffa6bb47](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=af6fad879fbf9b542cfde05c52695c9dffa6bb47)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAA-aLv7aZGSExnbjJRw8eKkoXbu34TdoKLLA2gPye3aHjO5OSA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAA-aLv7aZGSExnbjJRw8eKkoXbu34TdoKLLA2gPye3aHjO5OSA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/cd0bb935-0158-78a7-08b5-904886deac4b@postgrespro.ru](https://www.postgresql.org/message-id/flat/cd0bb935-0158-78a7-08b5-904886deac4b@postgrespro.ru), [https://www.postgresql.org/message-id/flat/20220616233130.rparivafipt6doj3@alap3.anarazel.de](https://www.postgresql.org/message-id/flat/20220616233130.rparivafipt6doj3@alap3.anarazel.de), [https://www.postgresql.org/message-id/flat/abd9b83b-aa66-f230-3d6d-734817f0995d@postgresql.org](https://www.postgresql.org/message-id/flat/abd9b83b-aa66-f230-3d6d-734817f0995d@postgresql.org), [https://www.postgresql.org/message-id/flat/CA+HiwqE4XTdfb1nW=Ojoy_tQSRhYt-q_kb6i5d4xcKyrLC1Nbg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+HiwqE4XTdfb1nW=Ojoy_tQSRhYt-q_kb6i5d4xcKyrLC1Nbg@mail.gmail.com), [https://www.postgresql.org/message-id/flat/CAN-LCVP7HXmGu-WcinsHvdKqMGEdv=1Y67H4U58F6Y=Q0M5GyQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAN-LCVP7HXmGu-WcinsHvdKqMGEdv=1Y67H4U58F6Y=Q0M5GyQ@mail.gmail.com)
- **Date:** decided 2026-07-14; started 2026-07-09
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** 86ab7f4c721d commit made the table-level ON ERROR clause serve as the default ON ERROR for columns lacking their own, so that a top-level ERROR ON ERROR turned per-column evaluation errors into hard errors. The SQL standard does mandate this cascade, but introducing it should be a deliberate, separately-documented change, so restore the previous behavior for now. This also reverts the paired ruleutils.c logic that deparsed a column's behavior against an ERROR default: that dropped an explicit ERROR ON EMPTY from a dumped view, and otherwise emitted a redundant NULL ON EMPTY.
- **Reason (quote):** quote unavailable
- **What is different this time:** P7 changed how ON ERROR cascades in JSON_TABLE, and this patch changes key validation in the SQL/JSON constructors. Both change SQL/JSON behavior that existing queries may rely on.


## 3. Objections a reviewer is likely to raise

### 1. Queries that return {} today start failing, with no back-patch or release-note plan

fixable · level 1b, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

An application runs JSON_OBJECTAGG(k VALUE v ABSENT ON NULL RETURNING json) over a left join, so unmatched rows give a NULL key with a NULL value. Today the server skips such a row. After a minor or major upgrade the whole aggregate raises an error, and the user finds out in production.

The patch turns accepted inputs into errors for both constructors and aggregates (message #7), so users see a behavior change. Message #4 counts 19 previously accepted cases that are now rejected. No message says whether this goes to stable branches or gets a release-note entry. [P7] records that a change to SQL/JSON semantics should be deliberate and documented on its own.

**Precedents.**
- [Revert cascading of JSON_TABLE's ON ERROR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=af6fad879fbf9b542cfde05c52695c9dffa6bb47) (Reverted, 2026-07-14); thread: [#1](https://www.postgresql.org/message-id/flat/CAA-aLv7aZGSExnbjJRw8eKkoXbu34TdoKLLA2gPye3aHjO5OSA@mail.gmail.com), [#2](https://www.postgresql.org/message-id/flat/cd0bb935-0158-78a7-08b5-904886deac4b@postgrespro.ru), [#3](https://www.postgresql.org/message-id/flat/20220616233130.rparivafipt6doj3@alap3.anarazel.de), [#4](https://www.postgresql.org/message-id/flat/abd9b83b-aa66-f230-3d6d-734817f0995d@postgresql.org), [#5](https://www.postgresql.org/message-id/flat/CA+HiwqE4XTdfb1nW=Ojoy_tQSRhYt-q_kb6i5d4xcKyrLC1Nbg@mail.gmail.com), [#6](https://www.postgresql.org/message-id/flat/CAN-LCVP7HXmGu-WcinsHvdKqMGEdv=1Y67H4U58F6Y=Q0M5GyQ@mail.gmail.com)

Thread: [#4 (QQ Mail, 2026-10-12)](https://www.postgresql.org/message-id/tencent_BCD8AC02BF4A6B19A64CB09D718CBDE57E0A%40qq.com), [#7 (민영재, 2026-10-17)](https://www.postgresql.org/message-id/CAP%3DtSTAZS%2B04tDng43JdnOAvgQ-12Z52p5YaACUo%3DtvAbJN05w%40mail.gmail.com), [#8 (민영재, 2026-11-09)](https://www.postgresql.org/message-id/CAP%3DtSTAKkF6%3DNfGHRmPB7XyWescAxxqBAxa4YwFRk_tQg_0q3Q%40mail.gmail.com)

**Question:** Is the fix meant for master only, and which release-note text tells users that ABSENT ON NULL now rejects NULL or non-scalar keys with NULL values?

Counter-check: held. Message #4 counts 19 accepted cases that now fail, and no message settles back-patching or a release note. The scenario needs JSON_OBJECT or a non-scalar key, because the aggregates already reject NULL keys.

### 2. The chosen semantics for omitted pairs are not checked against the SQL standard

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax · `standards-compat-2`

The patch documents that key restrictions apply even to omitted pairs, which makes the behavior a promise. If the standard does not check a pair that ABSENT ON NULL drops, Postgres rejects queries that other conforming systems accept. Users who port SQL/JSON code then hit errors that only Postgres raises.

In message #6 Dmitry Dolgov says one can argue that skipped pairs need no key check, and that only the current doc wording insists on it. No reply addresses this design question. The patch then changes the documentation to match the code (message #7), so the wording settles the question instead of the standard. The array, composite and json key rules are Postgres-specific, but the NULL-key case may be covered by the standard's constructor rules.

Thread: [#6 (Dmitry Dolgov, 2026-10-16)](https://www.postgresql.org/message-id/atKHv08UIzPGHv91%40ddolgov-thinkpadt14sgen1.rmtde.csb), [#7 (민영재, 2026-10-17)](https://www.postgresql.org/message-id/CAP%3DtSTAZS%2B04tDng43JdnOAvgQ-12Z52p5YaACUo%3DtvAbJN05w%40mail.gmail.com)

**Question:** What does the SQL standard say about a NULL key whose pair ABSENT ON NULL drops, and does the patch follow it?

Counter-check: held. No message cites the SQL standard for a NULL key whose pair ABSENT ON NULL drops. Dmitry Dolgov's proposed wording in message #6 supports the stricter reading, but it does not answer the standard question.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v2 adds a shared helper json_validate_key_category() used by json.c and jsonb.c. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch changes which keys JSON_OBJECT and JSON_OBJECTAGG accept under ABSENT ON NULL, with no check against the SQL standard. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | A dumped view, check constraint or generated column whose expression hits a newly rejected key can fail when restored or refreshed. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | It is tagged as a bug fix, and queries that return {} today would raise an error if it reaches stable branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The documentation now promises key validation for omitted pairs, which future constructor changes must keep. |
| performance on the hot path | **yes** | The patch adds a key type classification for each pair that ABSENT ON NULL omits. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the fix meant for master only, or also for stable branches where queries that return {} today would start to fail?
2. What does the SQL standard say about a NULL key whose pair ABSENT ON NULL drops, and does the patch follow it?
3. Which release-note text tells users that ABSENT ON NULL now rejects NULL or non-scalar keys paired with NULL values?
4. What is the reason to validate keys of omitted pairs, given the point in message #6 that skipping the check is a defensible reading?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the SQL/JSON constructors and the json and jsonb aggregate code would have to own it. The visible thread shows review comments but no statement from a committer on the behavior change.
- **Liability:** The owner signs for a user-visible behavior change in JSON_OBJECT, JSON_OBJECTAGG and the two strict aggregates, and for the back-patch decision. The new documentation sentence makes the stricter rule a lasting promise.
- **Author follow-up:** The author posted v2 in response to review and v3 as a rebase, which is a sign of staying with the patch.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is small and touches only json.c, jsonb.c, the sqljson tests and the documentation. It affects only queries that pair an invalid key with a NULL value under ABSENT ON NULL, and it needs a committer's decision on back-patching and the standard.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Stricter key checks turn working queries into errors, back-patch scope unsettled. Refuted (wrong facts): Message #6 shows jsonb_object_agg_strict(NULL, NULL) already fails with "field name must not be null", so the scenario is today's behavior. The back-patch concern is a duplicate of standards-compat-1, and [P11] concerns COPY options.
- `maintenance-fit-1` Rejecting NULL keys paired with NULL values can break existing aggregate queries. Refuted (not specific): Duplicate of standards-compat-1. Its left join scenario also describes current behavior, because the aggregate transition functions check for a NULL key before the ABSENT ON NULL skip.
- `demand-1` Stricter key checks may break working LEFT JOIN aggregates with no user gain. Refuted (wrong facts): The aggregates already reject a NULL key paired with a NULL value (message #6 shows the jsonb case), so the left join scenario is not a new failure. The remaining cost and design points duplicate standards-compat-1 and standards-compat-2.

### Review angles

- prior-attempts: 1 objection(s). No precedent proposes the same idea. The one objection rests on the behavior change and its unsettled back-patch scope.
- standards-compat: 2 objection(s). Two objections, both about the user-facing contract. Design inside json.c and jsonb.c is outside this angle.
- replication-ondisk: 0 objection(s). The proposal adds no WAL, catalog, protocol or ABI change. It changes only which key inputs JSON_OBJECT, JSON_OBJECTAGG and the *_agg_strict functions reject. An index, generated column or check constraint built on such an expression with an invalid key paired with NULL would now fail on REINDEX or new writes, but the material shows no user who has this, and the back-patch question belongs to another angle.
- security: 0 objection(s). The patch only adds earlier type-category checks on keys in JSON constructors and the strict aggregates, so it has no privilege or exposure aspect. Per messages #2, #4 and #7 it calls no output or cast functions for omitted keys, so no user-defined code runs that did not run before.
- maintenance-fit: 1 objection(s). From the maintenance angle the fix is small and contained. The shared helper Dolgov asked for reduces duplication, and the extra key classification runs only for omitted pairs.
- already-exists: 0 objection(s). This is a bug fix that makes key validation in JSON_OBJECT, JSON_OBJECTAGG and the *_object_agg_strict() functions consistent. No existing feature, extension or external tool can make core constructors reject these keys, so nothing from this angle applies. The patch reuses the existing category checks through one shared helper and adds no second mechanism. Back-patching and whether the change is desirable at all belong to other angles.
- demand: 1 objection(s). This is a consistency bug fix, and its demand is a single report. The one objection asks whether the stricter checks are worth breaking queries that work today.

### Precedents retrieved

- P1: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P2: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - judged not relevant; reason summarised from its thread
- P3: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant; reason summarised from its thread
- P4: [make psql backslash commands honor "\pset null"](https://commitfest.postgresql.org/patch/4603/) (cf, Withdrawn, 2023-10-06) - judged not relevant; reason summarised from its thread
- P5: [Revert 95d737ff to add 'ignore_nulls'](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8a026e4f15e2063cd7fe6cb9ffe44f2a81398bd) (revert, Reverted, 2014-09-29) - judged not relevant
- P6: [SQL/JSON: JSON_TABLE](https://commitfest.postgresql.org/patch/1473/) (cf, Returned with feedback, 2020-08-03) - judged not relevant; reason summarised from its thread
- P7: [Revert cascading of JSON_TABLE's ON ERROR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=af6fad879fbf9b542cfde05c52695c9dffa6bb47) (revert, Reverted, 2026-07-14) - used in the review
- P8: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P9: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant; reason summarised from its thread
- P10: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P11: [COPY: validate option presence rather than option values](https://commitfest.postgresql.org/patch/6744/) (cf, Withdrawn, 2026-06-29) - used in the review; reason summarised from its thread
- P12: [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P13: [Implement TODOs for index operator strategy number validation](https://commitfest.postgresql.org/patch/7152/) (cf, Withdrawn, 2026-09-09) - judged not relevant
- P14: [Validate user-supplied c_args in meson builds](https://commitfest.postgresql.org/patch/6596/) (cf, Withdrawn, 2026-06-10) - judged not relevant
- P15: [Revert "Validate subscription conninfo on owner change"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ac230cd3ce93c4752489564cfe6b37b584a465ac) (revert, Reverted, 2026-08-05) - judged not relevant
- P16: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant
- P17: [new plpgsql.extra_errors check - strict_expr_check](https://commitfest.postgresql.org/patch/5044/) (cf, Withdrawn, 2026-03-24) - judged not relevant
- P18: [Revert "Reorganise jsonpath operators and methods"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d) (revert, Reverted, 2024-01-03) - judged not relevant
- P19: [pgbench - implement strict TPC-B benchmark](https://commitfest.postgresql.org/patch/2090/) (cf, Rejected, 2019-11-06) - judged not relevant
- P20: [Be strict when request to flush past end of WAL in WaitXLogInsertionsToFinish](https://commitfest.postgresql.org/patch/7294/) (cf, Withdrawn, 2026-09-12) - judged not relevant
- P21: [CREATE DATABASE ... STRICT_UNICODE](https://commitfest.postgresql.org/patch/4876/) (cf, Withdrawn, 2024-12-11) - judged not relevant
- P22: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant
- P23: [pg_basebackup has stricter tablespace rules than the server](https://commitfest.postgresql.org/patch/1125/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P24: [Avoid obscure DES key buffer loop bounds in pgcrypto](https://commitfest.postgresql.org/patch/6953/) (cf, Withdrawn, 2026-09-25) - judged not relevant

### Warnings

- precedent search failed for "SQL/JSON constructor null handling": Spread syntax requires ...iterable[Symbol.iterator] to be a function
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 165 s; cost reported by the CLI: $1.58
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 74 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:05:19.588Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
