# Design pre-review: Extend skipping FK checks on replicas to include ADD FK and TRUNCATE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5774](https://commitfest.postgresql.org/patch/5774/) (Needs review); authors: Hannu Krosing (hannukrosing)
- Thread: [\[PATCH\] Extending FK check skipping on replicas to ADD FK and TRUNCATE](https://www.postgresql.org/message-id/flat/CAMT0RQROjBXX0hFaYyr7Yzwc_VRmsg4CRd8Nm4FivXaQGztVrw%40mail.gmail.com), started 2025-05-24
- Messages read: 12 of 12
- Patch prose read: v5-0001-rebase-to-current-head.patch, v1-0001-Skip-FK-Validations-when-Role-is-Replica.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The proposal fails at level 1b, so design objections are not ranked. prior-attempts-1: [P9] proposed the same truncate change and was closed as not needed, and this thread names no workflow that needs it. maintenance-fit-2: the proposal states no rule for which DDL checks session_replication_role = replica may skip. security-2: the back-patch request in message #2 would widen existing grants on the setting. A new version can answer all three, so the list should settle them before code review.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check whether the constraint is marked convalidated, whether the apply worker reaches the truncate bypass, or how truncate cascade and partitioned tables behave. Level 2 objections on those points were not evaluated.

## The proposal, restated

**Problem.** With session_replication_role = replica, DML skips foreign key checks, but alter table ... add foreign key still validates existing rows and truncate of a referenced table is still blocked. The author says validation can take tens of days when the referenced table has hundreds of billions of rows. This hurts logical replication setups that load data first and add constraints later.

**Proposed design.** When session_replication_role is replica, add foreign key skips the check of existing rows and truncate skips the error for referencing foreign keys. The patch wraps two code sections in tablecmds.c in a test of the role and adds regression tests.

**User-visible surface.**
- Changed behavior of session_replication_role = replica: alter table ... add foreign key no longer validates existing rows.
- Changed behavior of session_replication_role = replica: truncate on a table referenced by foreign keys no longer fails.

**What the author claims.**
- Validation on huge referenced tables can take tens of days and is not needed when data comes from a known good source.
- Truncate adds no new risk, since delete under replica role can already leave orphan rows.
- The current behavior is a bug and should be back-patched to all supported branches.
- Replica setup gets faster when the primary key index and foreign keys are created after the initial copy.
- Existing documentation may already cover the change, so no docs update may be needed.

**Thread state.** Rafia Sabih showed that add foreign key now accepts a row with no matching key, and the author replied this is intended. Nobody else has reviewed the design; the author asked Álvaro for review, and the other replies are about email reply style.

## 2. Prior attempts

### TRUNCATE behavior when session_replication_role = replica (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/1447/](https://commitfest.postgresql.org/patch/1447/)
- **Thread:** [https://www.postgresql.org/message-id/flat/ff835f71-3c6c-335e-4c7b-b9e1646cf3d7@2ndquadrant.it](https://www.postgresql.org/message-id/flat/ff835f71-3c6c-335e-4c7b-b9e1646cf3d7@2ndquadrant.it)
- **Date:** decided 2018-04-07; started 2017-12-29
- **Outcome:** Rejected (cf)
- **Who objected:** Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Peter Eisentraut argued that the current DELETE behaviour, which lets replicas hold data that violates foreign keys, is bad and should not be extended to TRUNCATE. He also saw no case for the change being needed. He closed the entry because the updated implementation of logical decoding of TRUNCATE no longer needed it.
- **Reason (quote):** "This part of the patch didn't end up being necessary, since the updated implementation of logical decoding of TRUNCATE could do without it." ([message](https://www.postgresql.org/message-id/151e60cf-22de-8b0d-7b80-ad7160d9c485%402ndquadrant.com))
- **What is different this time:** The truncate change is the same. This proposal also skips validation of existing rows in add foreign key and asks for a back-patch as a bug fix.


## 3. Objections a reviewer is likely to raise

### 1. Earlier objection to extending replica-role FK bypass to truncate is unanswered

fixable · level 1b, rubric item 6 · prior rejection, WAL / replication / standby, catalog changes · `prior-attempts-1`

An operator sets session_replication_role = replica in a migration script, runs truncate on a referenced table and then add foreign key. Both succeed, so orphan rows sit under a constraint the planner and users trust. Nobody finds out until a join drops rows or a later validate constraint or restore fails.

[P9] proposed the same truncate change. Per the AI summary of its thread, the objection was that the current DELETE behaviour is bad and should not be extended, and that no need was shown. Messages #1 and #8 use DELETE as the justification, which is the argument [P9]'s objector did not accept, and this proposal extends the bypass further to add foreign key. No reply in the thread addresses that earlier reasoning.

**Precedents.**
- [TRUNCATE behavior when session_replication_role = replica](https://commitfest.postgresql.org/patch/1447/) (Rejected, 2018-04-07); thread: [link](https://www.postgresql.org/message-id/flat/ff835f71-3c6c-335e-4c7b-b9e1646cf3d7@2ndquadrant.it)

Thread: [#1 (Hannu Krosing, 2025-05-24)](https://www.postgresql.org/message-id/CAMT0RQROjBXX0hFaYyr7Yzwc_VRmsg4CRd8Nm4FivXaQGztVrw%40mail.gmail.com), [#7 (Rafia Sabih, 2026-07-20)](https://www.postgresql.org/message-id/CA%2BFpmFdNFehW5r8KpVEFO%2BswCeTKc6pEETfwKkE8242wKg-dHw%40mail.gmail.com), [#8 (Hannu Krosing, 2026-08-06)](https://www.postgresql.org/message-id/CAMT0RQR_PeqFzSsg_Sr7dXnHF43qTEn-LqBA-yZYGs-7y-p-5g%40mail.gmail.com)

**Question:** Why should the replica-role bypass grow when the earlier objection was that the existing DELETE bypass is already harmful, and which workload needs truncate here?

Counter-check: held. [P9] proposed the same truncate change, and its verbatim key sentence says the change was not needed. Messages #1 and #8 justify truncate only by comparison with delete and name no workflow that needs it.

### 2. session_replication_role grows from trigger firing into DDL bypass

fixable · level 1b, rubric item 5 · prior rejection, new GUC, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-2`

Once add foreign key and truncate skip checks under replica role, the next requests are add check, not null, unique and exclusion validation, because the same argument fits each one. Each one adds a special case in a different command. No test fails when a new constraint path does not follow the rule, so the behaviour drifts.

Today the setting only controls which triggers fire, and foreign keys are skipped because they are triggers, as the docs quoted in message #1 say. This patch adds checks of the setting inside tablecmds.c, so the setting becomes a general unsafe-load switch with no defined edges. [P9] proposed the same truncate change and was not accepted. Its summarized reason was that the delete behaviour should not be extended. That summary is model-written, so it is a pointer only.

**Precedents.**
- [TRUNCATE behavior when session_replication_role = replica](https://commitfest.postgresql.org/patch/1447/) (Rejected, 2018-04-07); thread: [link](https://www.postgresql.org/message-id/flat/ff835f71-3c6c-335e-4c7b-b9e1646cf3d7@2ndquadrant.it)

Thread: [#1 (Hannu Krosing, 2025-05-24)](https://www.postgresql.org/message-id/CAMT0RQROjBXX0hFaYyr7Yzwc_VRmsg4CRd8Nm4FivXaQGztVrw%40mail.gmail.com), [#8 (Hannu Krosing, 2026-08-06)](https://www.postgresql.org/message-id/CAMT0RQR_PeqFzSsg_Sr7dXnHF43qTEn-LqBA-yZYGs-7y-p-5g%40mail.gmail.com)

**Question:** What rule decides which DDL checks session_replication_role = replica may skip, and how does it stop later special cases for other constraint types?

Counter-check: held. The docs tie the foreign key effect to trigger firing, and the patch tests the setting inside tablecmds.c without stating a rule for which checks it may skip. [P9] is an AI summary about truncate only and must not be used as support here.

### 3. Back-patching widens what existing grants on session_replication_role allow

fixable · level 1b, rubric item 6 · backpatchability, security / privileges · `security-2`

An admin ran grant set on parameter session_replication_role to a non-superuser load role for faster DML. After a minor upgrade, that role, with only truncate privilege on a referenced table, can truncate it while other owners' tables still reference it. Today that needs truncate privilege on the referencing tables through cascade.

Message #2 asks to back-patch to all supported branches, so a grant made under the old meaning would change meaning silently. Truncate is a separate privilege from delete, so the comparison with delete in message #8 does not cover a role that holds truncate but not delete. The patch has no documentation change telling admins that the setting now also bypasses these checks.

Thread: [#2 (Hannu Krosing, 2025-05-24)](https://www.postgresql.org/message-id/CAMT0RQQy3hH0ygvkq4DHE6khGgWZ0kVu_DNpWAG_pLV96_E9SA%40mail.gmail.com), [#8 (Hannu Krosing, 2026-08-06)](https://www.postgresql.org/message-id/CAMT0RQR_PeqFzSsg_Sr7dXnHF43qTEn-LqBA-yZYGs-7y-p-5g%40mail.gmail.com)

**Question:** Which role holding set on session_replication_role but not superuser gains new power under this patch, and why is that safe to give out in a minor release?

Counter-check: held. A role with set on the parameter and truncate privilege only on the referenced table gains a new ability. The delete comparison in message #8 does not cover it, and the proposal does not discuss privileges.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | Logical replication apply workers run with session_replication_role = replica, so the truncate change may reach replicated truncate on subscribers. |
| wire protocol | no |  |
| catalog changes | **yes** | The proposal does not say whether pg_constraint marks the unchecked foreign key as validated. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | Plain add foreign key would skip the check of existing rows, beside the existing `not valid` spelling. |
| new GUC | **yes** | No new GUC, but session_replication_role grows from trigger firing to skipping DDL checks. |
| planner behaviour change | **yes** | The planner uses validated foreign keys for join estimates, and the constraint may be recorded as validated without a check. |
| pg_dump / restore round-trip | **yes** | A dump of a table with orphan rows under such a constraint fails at add foreign key on restore. |
| psql UX conventions | no |  |
| security / privileges | **yes** | A role with set on session_replication_role and truncate on the referenced table gains a new ability. |
| backpatchability | **yes** | Message #2 asks to back-patch to all supported branches, which removes two errors in minor releases. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Tests of the setting inside tablecmds.c invite the same special case for other constraint types. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | `add foreign key ... not valid` already skips the scan of existing rows. |

## 5. Questions the author must answer before anyone reviews code

1. Which replication or load workflow needs truncate of a referenced table under replica role, and how does it differ from the case closed in [P9]?
2. What does the replica setup need that `add foreign key ... not valid` cannot give, and is the new constraint marked convalidated in pg_constraint?
3. What rule decides which DDL checks session_replication_role = replica may skip, so that check, not null and unique validation do not follow by the same argument?
4. Does the truncate bypass apply inside the logical replication apply worker, where a replicated truncate could orphan subscriber-local referencing rows?
5. Which documented promise does the current behaviour break, so that removing two errors in minor releases is justified?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows foreign key DDL in tablecmds.c and the logical replication apply path. The author asked Álvaro for review in message #9, and no committer has replied on the design.
- **Liability:** The owner signs for foreign keys that may hold orphan rows while pg_dump, restore and the planner trust them, and for a wider meaning of session_replication_role, in every branch that gets the change.
- **Author follow-up:** The author has rebased the patch and moved the entry across CommitFests for over a year, and answered the reviewer in message #8.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is about 6 KiB in tablecmds.c plus tests, and the stated need is foreign key creation on very large tables after an initial copy. The entry has been in 7 CommitFests without a design reply, because the relation to `not valid`, the catalog state and the scope of the setting are open.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Unchecked foreign keys recorded as validated, beside the existing not valid path. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-1` (fixable, level 2) Logical replication apply would stop rejecting truncates that break subscriber FKs. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-3` (fixable, level 2) Back-patching changes truncate and add foreign key in minor releases. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-2` Silent change to what replica role means for existing tools and scripts. Refuted (not specific): Duplicate of maintenance-fit-3 for the back-patch part and of maintenance-fit-2 for the change in what the setting means.
- `standards-compat-3` Truncate exemption repeats a proposal that was rejected before. Refuted (not specific): Duplicate of prior-attempts-1.
- `replication-ondisk-1` Unchecked foreign key is stored as validated in pg_constraint. Refuted (not specific): Duplicate of standards-compat-1.
- `replication-ondisk-2` Subscriber apply of truncate would stop failing and leave orphans. Refuted (not specific): Duplicate of security-1, which states the same apply worker case without asserting where the patch places the test.
- `replication-ondisk-3` Back-patching changes replica-role semantics in minor releases. Refuted (not specific): Duplicate of maintenance-fit-3.
- `maintenance-fit-1` Unchecked constraint may be recorded as validated, duplicating NOT VALID. Refuted (not specific): Duplicate of standards-compat-1. The added scenario about FK-based join removal concerns a feature that does not exist.
- `already-exists-1` add foreign key ... not valid already skips the existing-row check. Refuted (not specific): Duplicate of standards-compat-1.
- `already-exists-2` truncate of a referenced table is already covered by delete. Refuted (not specific): Duplicate of prior-attempts-1. Its own claim that delete meets the need is weak, since delete on a very large table costs far more than truncate.
- `demand-1` Existing not valid option already covers the add foreign key case. Refuted (not specific): Duplicate of standards-compat-1, which asks the same question about `not valid`.
- `demand-2` No user need shown for truncate on referenced tables under replica. Refuted (not specific): Duplicate of prior-attempts-1.
- `demand-3` Existing replica-role users bear a behaviour change they did not ask for. Refuted (not specific): Duplicate of maintenance-fit-3, since its scenario depends on the back-patch.

### Review angles

- prior-attempts: 1 objection(s). One objection from the prior-attempts angle. The verbatim reason of [P9] is only that the change was not needed. Its substantive objection comes from an AI summary, so it carries less weight.
- standards-compat: 3 objection(s). Objections found; see list.
- replication-ondisk: 3 objection(s). Physical replication and WAL format are not affected, because the change is DDL behavior on the primary.
- security: 2 objection(s). Two objections raised. Setting session_replication_role needs superuser or an explicit grant, so ordinary users do not gain anything new.
- maintenance-fit: 3 objection(s). Committability: the author asked Álvaro for review in message #9, and no committer has replied on the design yet.
- already-exists: 2 objection(s). Two objections raised; both rest on existing ways to get the result.
- demand: 3 objection(s). Demand evidence is limited to the author's own case, so the objections are about showing demand, not about rejecting the idea outright.

### Precedents retrieved

- P1: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P2: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/1252/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P3: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/2966/) (cf, Returned with feedback, 2021-09-14) - judged not relevant
- P4: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant; reason summarised from its thread
- P5: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P6: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P7: [Revert "Fix an O(N^2) problem in foreign key references".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3d9e8db9e56beb0501d168779ae9644587924061) (revert, Reverted, 2015-09-15) - judged not relevant
- P8: [Revert changes to CREATE TRIGGER and ALTER TABLE ADD FOREIGN KEY locking, per request from Tom.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50ce8ab9fcd49dfcbff9e19c5aa53efe0fc6dc6e) (revert, Reverted, 2005-03-24) - judged not relevant
- P9: [TRUNCATE behavior when session_replication_role = replica](https://commitfest.postgresql.org/patch/1447/) (cf, Rejected, 2018-04-07) - used in the review; reason summarised from its thread
- P10: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2409/) (cf, Returned with feedback, 2020-01-28) - judged not relevant; reason summarised from its thread
- P11: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2499/) (cf, Returned with feedback, 2020-08-02) - judged not relevant; reason summarised from its thread
- P12: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant; reason summarised from its thread
- P13: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - judged not relevant; reason summarised from its thread
- P14: [Respect client-initiated CopyDone during logical streaming replication](https://commitfest.postgresql.org/patch/621/) (cf, Withdrawn, 2019-07-08) - judged not relevant; reason summarised from its thread
- P15: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P16: [Allow replication roles to use file access functions](https://commitfest.postgresql.org/patch/339/) (cf, Returned with feedback, 2015-09-04) - judged not relevant
- P17: [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (cf, Returned with feedback, 2021-10-06) - judged not relevant
- P18: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant
- P19: [Quorum commit for multiple synchronous replication](https://commitfest.postgresql.org/patch/696/) (cf, Returned with feedback, 2016-10-02) - judged not relevant
- P20: [Add mandatory access control for TRUNCATE](https://commitfest.postgresql.org/patch/2220/) (cf, Withdrawn, 2019-08-26) - judged not relevant
- P21: [Add a pg_truncate_freespace_map function](https://commitfest.postgresql.org/patch/4882/) (cf, Withdrawn, 2024-08-20) - judged not relevant
- P22: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P23: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P24: [Add `truncate` option to subscription commands](https://commitfest.postgresql.org/patch/2758/) (cf, Withdrawn, 2020-11-25) - judged not relevant

### Warnings

- review angle "replication-ondisk" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 201 s; cost reported by the CLI: $1.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 135 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:35:32.861Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
