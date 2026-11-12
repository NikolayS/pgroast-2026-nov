# Design pre-review: Fix jsonpath printing of operations followed by an accessor

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7621](https://commitfest.postgresql.org/patch/7621/) (Needs review); authors: Ishjaap Singh (vennictus)
- Thread: [Fix jsonpath printing of operations followed by an accessor](https://www.postgresql.org/message-id/flat/CA%2BY2BmewUC0FwoZ_ktQFc%3D%3DG5OBXVqmQjG3T6x%2B%3DYVfCNO%3DHrA%40mail.gmail.com), started 2026-11-10
- Messages read: 1 of 1
- Patch prose read: v1-0001-Fix-jsonpath-printing-of-operations-followed-by-a.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The first message states the bug with reproducible examples: a printed jsonpath that returns a different result or does not parse, which reaches users through pg_dump and pg_get_viewdef(). The fix extends the rule printJsonPathItem() already uses for numeric literals. Confidence is medium because only one message is visible and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that output changes only for paths that print wrong today is unchecked. Whether the new rule covers every operation and predicate kind is also unchecked.

## The proposal, restated

**Problem.** The jsonpath output function drops parentheses around an operation followed by an accessor or method, so the printed path means something else or does not parse. Because pg_dump and pg_get_viewdef() use this output, a view can return different results after dump and restore, and a jsonpath column can fail to restore. The bug dates from v12.

**Proposed design.** printJsonPathItem() decides about parentheses only from operator priority. The patch makes it parenthesize any operation or predicate that has an accessor after it, the same way it already handles numeric literals. Regression tests print each kind and compare query results before and after a reparse.

**User-visible surface.**
- jsonpath output text changes for paths whose printed form is wrong today, which also changes pg_dump output and view, rule and index definitions.

**What the author claims.**
- '(-$.a).abs() + 1' on {"a": -2} returns 3, but its printed form returns -1.
- On v15 and later some printed forms, such as '($.a + 1).abs() == 2', fail with "trailing junk after numeric literal".
- "!", "is unknown" and "exists" followed by an accessor print in a form that fails to parse on every version tried.
- Output changes only for paths that print wrong today.
- A fuzzer found only this bug on master and found no mismatch in three runs of 20,000 expressions with the patch.

**Thread state.** Only the first message is visible. The author offers versions for REL_14 to REL_16 if a back-patch is wanted.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-10; 24 candidates retrieved, none is the same or a similar idea).

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
| pg_dump / restore round-trip | **yes** | pg_dump and pg_get_viewdef() emit jsonpath output text, which the patch changes for paths that print wrong today. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bug dates from v12, and the author offers versions for REL_14 to REL_16 if a back-patch is wanted. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does every printed form that the patch changes still parse, with the same meaning, on each supported older release that might load a newer dump?
2. What should users do with dumps already taken that hold the broken printed forms, and does that belong in the release notes?
3. If the fix is back-patched, can any index or stored value built on jsonpath text output differ after the minor upgrade?
4. Which parts of the jsonpath grammar does the fuzzer generate, and can you post it so reviewers can repeat the runs?
5. Does a jsonpath operator or predicate added later get parentheses under the new rule without a further change to printJsonPathItem()?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the jsonpath parser and output code in jsonpath.c, and who can judge a back-patch that changes output text.
- **Liability:** The owner signs for the changed jsonpath output text on every branch that gets the fix, including dump and restore across minor versions, for as long as those branches are supported.
- **Author follow-up:** The author offers to send versions for REL_14 to REL_16 if the fix is back-patched; no other sign yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug gives wrong results and failed restores, but only for paths with a parenthesized operation followed by an accessor. The patch is 8.8 KiB, touches jsonpath.c and its regression test, and adds no new user-visible surface.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. P24 is a different deparse bug in XMLNAMESPACES with no recorded reason. P13 reverts a JSON_TABLE behaviour change, and its reason (an unannounced semantic change) does not apply to a bug fix in jsonpath output, so neither carries a reason that applies here.
- standards-compat: 0 objection(s). This is a deparse bug fix. The new output is the only form that reads back with the same meaning, and the text changes only for paths that print wrongly today, so I found no compatibility objection worth sending. Not checkable without the code: whether every changed form still parses on older servers, which matters when a new dump is loaded into an older release. A committer would also likely weigh, for a back-patch, that dumps already taken with the broken forms stay wrong, and that indexes built on jsonpath text output would change.
- replication-ondisk: 0 objection(s). The patch changes only the jsonpath output text; the binary jsonpath format in heap columns and stored view, rule and index expressions stays the same, so pg_upgrade and existing data are unaffected. Dumps and logical replication text taken from unfixed servers keep the old broken forms, but that is the existing bug, which the patch does not make worse.
- security: 0 objection(s). The patch changes only how the jsonpath output function prints parentheses, and adds no privilege, role, function or exposure path. The existing bug could change the meaning of a jsonpath in an RLS policy or security barrier view after dump and restore, but that argues for the fix.
- maintenance-fit: 0 objection(s). From the maintenance angle this is a narrow bug fix in printJsonPathItem() that follows the existing numeric-literal rule. It adds no hot-path cost, no new API and no lasting commitment, and its tests compare results after a reparse. Without the code, I could not check whether future jsonpath operators fall under the new rule without any extra change.
- already-exists: 0 objection(s). This is a bug fix to the jsonpath output function. No existing feature, setting or extension gives a correct printed form, so users cannot avoid the broken pg_dump and view-definition output today. Whether the fix fits in core needs no answer, and the patch extends the existing parenthesization logic for numeric literals.
- demand: 0 objection(s). This is a correctness bug fix: jsonpath output that changes query results or fails to restore through pg_dump affects anyone who uses such paths, and no workaround exists for dump round-trips. Demand is not the test for a wrong-result and failed-restore bug, and the output change is limited to paths that print wrong today.

### Precedents retrieved

- P1: [Add jsonpath string method $.replace(from, to)](https://commitfest.postgresql.org/patch/5261/) (cf, Withdrawn, 2024-09-25) - judged not relevant; reason summarised from its thread
- P2: [Revert "Reorganise jsonpath operators and methods"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d) (revert, Reverted, 2024-01-03) - judged not relevant
- P3: [Revert "Add gitignore entries for jsonpath_gram.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7b084b38310cfe9c8b58cc615a81df625c771f5d) (revert, Reverted, 2019-03-22) - judged not relevant
- P4: [Allow subfield references without parentheses](https://commitfest.postgresql.org/patch/5449/) (cf, Returned with feedback, 2025-02-14) - judged not relevant; reason summarised from its thread
- P5: [tab-completion debug print](https://commitfest.postgresql.org/patch/1917/) (cf, Returned with feedback, 2019-02-02) - judged not relevant; reason summarised from its thread
- P6: [Dump-restore loosing 'attnotnull' bit for DEFERRABLE PRIMARY KEY column(s)](https://commitfest.postgresql.org/patch/4729/) (cf, Rejected, 2023-12-28) - judged not relevant; reason summarised from its thread
- P7: [Removing unused trailing linepointers from heap pages](https://commitfest.postgresql.org/patch/3035/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P8: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P9: [Avoid computing ORDER BY junk columns unnecessarily](https://commitfest.postgresql.org/patch/4717/) (cf, Returned with feedback, 2024-02-10) - judged not relevant; reason summarised from its thread
- P10: [doc: a restore executes code chosen by any dumped object's owner](https://commitfest.postgresql.org/patch/7185/) (cf, Withdrawn, 2026-10-01) - judged not relevant; reason summarised from its thread
- P11: [Use JOIN USING aliases in ruleutils.c](https://commitfest.postgresql.org/patch/3538/) (cf, Returned with feedback, 2022-03-25) - judged not relevant; reason summarised from its thread
- P12: [\[PG19\]pg_dump --statistics restores extended statistics by attribute number](https://commitfest.postgresql.org/patch/7446/) (cf, Withdrawn, 2026-10-19) - judged not relevant
- P13: [Revert cascading of JSON_TABLE's ON ERROR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=af6fad879fbf9b542cfde05c52695c9dffa6bb47) (revert, Reverted, 2026-07-14) - used in the review
- P14: [SQL:2023 JSON simplified accessor support](https://commitfest.postgresql.org/patch/5214/) (cf, Returned with feedback, 2026-03-31) - judged not relevant
- P15: [autovacuum: add possibility to change priority of vacuumed tables](https://commitfest.postgresql.org/patch/1519/) (cf, Returned with feedback, 2018-03-02) - judged not relevant
- P16: [Fix pg_restore --no-globals SQL output for pg_dumpall archives](https://commitfest.postgresql.org/patch/6899/) (cf, Withdrawn, 2026-06-20) - judged not relevant
- P17: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant
- P18: [Make deparsing of column defaults faster](https://commitfest.postgresql.org/patch/1657/) (cf, Returned with feedback, 2018-09-24) - judged not relevant
- P19: [Avoid unused value (src/fe_utils/print.c)](https://commitfest.postgresql.org/patch/4339/) (cf, Rejected, 2023-07-11) - judged not relevant
- P20: [Replace literal 0 values with the appropriate Invalid* constants](https://commitfest.postgresql.org/patch/6480/) (cf, Withdrawn, 2026-02-13) - judged not relevant
- P21: [Operation log for major operations](https://commitfest.postgresql.org/patch/4018/) (cf, Withdrawn, 2023-03-02) - judged not relevant
- P22: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P23: [psql JSON output format](https://commitfest.postgresql.org/patch/4707/) (cf, Returned with feedback, 2024-05-15) - judged not relevant
- P24: [A COLLATE clause in the URI of XMLNAMESPACES does not read back](https://commitfest.postgresql.org/patch/7492/) (cf, Withdrawn, 2026-10-28) - used in the review

### Warnings

- tone lint: 1 finding(s) remain after the repair call; see tone_lint in roast.json before forwarding this text

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 107 s; cost reported by the CLI: $0.68
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 44 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 1 left
  - `brief.author_claims[2]`: exclamation mark ("!")
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:08:40.997Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
