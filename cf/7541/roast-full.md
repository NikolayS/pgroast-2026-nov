# Design pre-review: BUG #19703: information_schema.usage_privileges omits a sequence owner's implicit USAGE privilege

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7541](https://commitfest.postgresql.org/patch/7541/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [BUG #19703: information_schema.usage_privileges omits a sequence owner's implicit USAGE privilege](https://www.postgresql.org/message-id/flat/19703-49d6f1796001a6d7%40postgresql.org), started 2026-09-19
- Messages read: 5 of 5
- Patch prose read: v1-0002-Add-tests-for-sequence-USAGE-privileges-in-inform.patch, v1-0001-Fix-information_schema.usage_privileges-for-seque.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at any level. The problem is stated with a reproducible case in message #1, the cause is identified in message #2, and the fix changes one acldefault type code in the sequences arm of the view. The thread agreed to fix master only, so no installed view changes on a minor upgrade. Confidence is medium because only 3 of 5 messages and no code were read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code and messages #4 and #5 were not read. Whether the test covers the right cases and whether the fix touches only the sequences arm could not be checked.

## The proposal, restated

**Problem.** For a sequence with no explicit grants, has_sequence_privilege reports USAGE for the owner, but information_schema.usage_privileges shows no row. The row appears only after a redundant explicit grant. This affects anyone who reads sequence privileges through information_schema.

**Proposed design.** The sequences arm of usage_privileges calls acldefault('r', relowner), but sequences use the 's' default. A relation default ACL has no USAGE bit, so the prtype filter drops every row. The patch changes the call to acldefault('s', relowner) and adds regression tests.

**User-visible surface.**
- information_schema.usage_privileges now lists the owner's implicit USAGE row for sequences without explicit grants.

**What the author claims.**
- Impact is low: only the view result is wrong, and access to the sequence works.
- The new test returns zero rows with the fix reverted.
- The bug goes back to commit 82e83f46a, per Tom Lane.

**Thread state.** Tom Lane found the cause and voted against back-patching because the bug went unnoticed for 14 years, and the author agreed to fix master only. A catversion bump is usual for information_schema changes but not strictly needed, and nobody has decided it.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-19; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch edits a view definition in src/backend/catalog/information_schema.sql, and the catversion bump is undecided. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch changes the rows returned by the SQL-standard view information_schema.usage_privileges. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The view now reports the sequence owner's implicit USAGE; no access right changes. |
| backpatchability | **yes** | Message #3 raised a back-patch to 14; the thread chose master only because older branches need a manual view replacement. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do other arms of usage_privileges, or other information_schema views, pass an acldefault type code that does not match the object kind?
2. Should the commit bump catversion, as is usual for information_schema.sql changes?
3. Should the release notes tell users of existing installations how to replace the view definition?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane diagnosed the cause in message #2 and took part in the back-patch decision. The owner needs to know information_schema.sql and the ACL default codes.
- **Liability:** One changed type code in a view definition plus a regression test in privileges.sql. The lasting risk is small: tools that read the view see one more owner row per sequence without explicit grants.
- **Author follow-up:** The author reproduced the bug, sent the fix and test one day after the report, and agreed to the master-only plan.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix is a small change to one file with a test, and a committer already confirmed the cause. Few users read sequence privileges through information_schema, but the view returns a wrong result today.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier proposal with the same or a similar idea, and the thread points to no earlier discussion of this fix. Message #2 traces the bug to commit 82e83f46a, but that is where the bug came from, not an earlier rejected fix.
- standards-compat: 0 objection(s). The fix makes information_schema.usage_privileges agree with has_sequence_privilege and with how the view already lists owner rows for other object kinds. The only behaviour change is the missing owner USAGE row, which is the bug, and keeping it master-only means no installed view changes on a minor upgrade. Whether other information_schema views pass the wrong acldefault type is an open question without a failure scenario, so it is not an objection.
- replication-ondisk: 0 objection(s). The only persistent state touched is the information_schema view definition, which initdb creates and pg_upgrade takes from the new cluster, so upgraded clusters get the fix. There is no WAL, replication, wire protocol or ABI change, and message #4 and message #5 settled back-patching. Whether to bump catversion is a committer convention with no compatibility effect, per message #4.
- security: 0 objection(s). The patch makes no privilege or exposure change that needs an objection. It changes only what information_schema.usage_privileges reports, and the new row states the owner's USAGE, which has_sequence_privilege already reveals. The view's existing filter on enabled roles still limits who sees the row, and nothing grants new access.
- maintenance-fit: 0 objection(s). This is a one-token correction in information_schema.sql, from acldefault('r') to acldefault('s'), with a regression test. It adds no hot-path cost, no new mechanism, no API and no lasting liability. Tom Lane diagnosed the cause and agreed on a master-only fix in message #2 and message #4. Auditing other views for a similar wrong acldefault type is a fair follow-up but does not count against this patch.
- already-exists: 0 objection(s). This is a bug fix to an existing view. It adds no feature, so no existing feature, extension or tool could replace it. has_sequence_privilege and aclexplode(acldefault('s', ...)) give the right answer, but they do not fix the standard information_schema view that users and tools query.
- demand: 0 objection(s). This is a one-token correctness fix for a standard view that returns wrong results. A demand objection does not apply, because users reading information_schema expect it to agree with has_sequence_privilege, and a field report (message #1) shows the mismatch on three versions. The fix is master only, as message #4 and message #5 agreed, so there is no hot path, UI or maintenance cost to weigh against the size of the user group.

### Precedents retrieved

- P1: [psql: add schema size information to \dn+](https://commitfest.postgresql.org/patch/2010/) (cf, Withdrawn, 2019-02-25) - judged not relevant; reason summarised from its thread
- P2: [Tab completion for CREATE SCHEMA AUTHORIZATION](https://commitfest.postgresql.org/patch/3252/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P3: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P4: [Add CREATE SCHEMA ... LIKE support](https://commitfest.postgresql.org/patch/6481/) (cf, Returned with feedback, 2026-04-06) - judged not relevant; reason summarised from its thread
- P5: [Prevent extension creation in temporary schemas](https://commitfest.postgresql.org/patch/1969/) (cf, Returned with feedback, 2019-03-07) - judged not relevant; reason summarised from its thread
- P6: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P7: [Add pg_ownerships and pg_privileges system views](https://commitfest.postgresql.org/patch/5033/) (cf, Withdrawn, 2025-01-04) - judged not relevant; reason summarised from its thread
- P8: [Add pg_privileges, pg_ownerships, and pg_get_acl() \[DUP\]](https://commitfest.postgresql.org/patch/5042/) (cf, Withdrawn, 2024-06-16) - judged not relevant
- P9: [Various small unrelated doc patches: plpgsql, schemas, permissions, oidvector](https://commitfest.postgresql.org/patch/4581/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P10: [USAGE privilege on PUBLICATION](https://commitfest.postgresql.org/patch/3641/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P11: [Revert "Forbid DROP SCHEMA on temporary namespaces"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65192e02441cedd106b6abebe0036fb8cc124fb3) (revert, Reverted, 2020-01-08) - judged not relevant
- P12: [Revert MAINTAIN privilege and pg_maintain predefined role.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=151c22deee66a3390ca9a1c3675e29de54ae73fc) (revert, Reverted, 2023-07-07) - judged not relevant
- P13: [Sequence Access Method](https://commitfest.postgresql.org/patch/44/) (cf, Returned with feedback, 2015-07-28) - judged not relevant; reason summarised from its thread
- P14: [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (cf, Returned with feedback, 2021-11-26) - judged not relevant; reason summarised from its thread
- P15: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P16: [Revert "Include information on buffer usage during planning phase, in EXPLAIN output."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19db23bcbda99e93321cb0636677ec9c6e121a2a) (revert, Reverted, 2020-04-03) - judged not relevant
- P17: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant
- P18: [fix and document CLUSTER privileges](https://commitfest.postgresql.org/patch/4054/) (cf, Returned with feedback, 2023-01-14) - judged not relevant
- P19: [Autoprepare: implicitly replace literals with parameters and store generalized plan](https://commitfest.postgresql.org/patch/1747/) (cf, Returned with feedback, 2020-07-01) - judged not relevant
- P20: [pg_catversion builtin function](https://commitfest.postgresql.org/patch/906/) (cf, Rejected, 2016-12-14) - judged not relevant
- P21: [Revert commit 0725065b37b8b0e9074a624a8d3e3ac1844fc820.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d4e6133c681f0038861e5cf8707302bd80917fa0) (revert, Reverted, 2013-11-05) - judged not relevant
- P22: [logical decoding and replication of sequences, take 2](https://commitfest.postgresql.org/patch/3823/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P23: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant
- P24: [Sequence Access Methods](https://commitfest.postgresql.org/patch/466/) (cf, Returned with feedback, 2016-04-06) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 76 s; cost reported by the CLI: $0.52
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 40 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:07:10.094Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
