# Design pre-review: Extend skipping FK checks on replicas to include ADD FK and TRUNCATE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5774](https://commitfest.postgresql.org/patch/5774/); [thread](https://www.postgresql.org/message-id/flat/CAMT0RQROjBXX0hFaYyr7Yzwc_VRmsg4CRd8Nm4FivXaQGztVrw%40mail.gmail.com), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The proposal fails at level 1b, so design objections are not ranked. prior-attempts-1: [P9] proposed the same truncate change and was closed as not needed, and this thread names no workflow that needs it. maintenance-fit-2: the proposal states no rule for which DDL checks session_replication_role = replica may skip. security-2: the back-patch request in message #2 would widen existing grants on the setting. A new version can answer all three, so the list should settle them before code review.

Not checked: Without the code, this review could not check whether the constraint is marked convalidated, whether the apply worker reaches the truncate bypass, or how truncate cascade and partitioned tables behave. Level 2 objections on those points were not evaluated.

## Prior attempts

- [TRUNCATE behavior when session_replication_role = replica](https://www.postgresql.org/message-id/flat/ff835f71-3c6c-335e-4c7b-b9e1646cf3d7@2ndquadrant.it) (same idea; Rejected, 2018-04-07). "This part of the patch didn't end up being necessary, since the updated implementation of logical decoding of TRUNCATE could do without it." ([message](https://www.postgresql.org/message-id/151e60cf-22de-8b0d-7b80-ad7160d9c485%402ndquadrant.com)) Different now: The truncate change is the same. This proposal also skips validation of existing rows in add foreign key and asks for a back-patch as a bug fix.

## Objections a reviewer is likely to raise

### 1. Earlier objection to extending replica-role FK bypass to truncate is unanswered

An operator sets session_replication_role = replica in a migration script, runs truncate on a referenced table and then add foreign key. Both succeed, so orphan rows sit under a constraint the planner and users trust. Nobody finds out until a join drops rows or a later validate constraint or restore fails.

[P9] proposed the same truncate change. Per the AI summary of its thread, the objection was that the current DELETE behaviour is bad and should not be extended, and that no need was shown. Messages #1 and #8 use DELETE as the justification, which is the argument [P9]'s objector did not accept, and this proposal extends the bypass further to add foreign key. No reply in the thread addresses that earlier reasoning.

Precedent: [TRUNCATE behavior when session_replication_role = replica](https://www.postgresql.org/message-id/flat/ff835f71-3c6c-335e-4c7b-b9e1646cf3d7@2ndquadrant.it) (Rejected, 2018-04-07)

**Question:** Why should the replica-role bypass grow when the earlier objection was that the existing DELETE bypass is already harmful, and which workload needs truncate here?

<sub>fixable, level 1b, prior rejection; messages [#1](https://www.postgresql.org/message-id/CAMT0RQROjBXX0hFaYyr7Yzwc_VRmsg4CRd8Nm4FivXaQGztVrw%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CA%2BFpmFdNFehW5r8KpVEFO%2BswCeTKc6pEETfwKkE8242wKg-dHw%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAMT0RQR_PeqFzSsg_Sr7dXnHF43qTEn-LqBA-yZYGs-7y-p-5g%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. session_replication_role grows from trigger firing into DDL bypass

Once add foreign key and truncate skip checks under replica role, the next requests are add check, not null, unique and exclusion validation, because the same argument fits each one. Each one adds a special case in a different command. No test fails when a new constraint path does not follow the rule, so the behaviour drifts.

Today the setting only controls which triggers fire, and foreign keys are skipped because they are triggers, as the docs quoted in message #1 say. This patch adds checks of the setting inside tablecmds.c, so the setting becomes a general unsafe-load switch with no defined edges. [P9] proposed the same truncate change and was not accepted. Its summarized reason was that the delete behaviour should not be extended. That summary is model-written, so it is a pointer only.

Precedent: [TRUNCATE behavior when session_replication_role = replica](https://www.postgresql.org/message-id/flat/ff835f71-3c6c-335e-4c7b-b9e1646cf3d7@2ndquadrant.it) (Rejected, 2018-04-07)

**Question:** What rule decides which DDL checks session_replication_role = replica may skip, and how does it stop later special cases for other constraint types?

<sub>fixable, level 1b, prior rejection; messages [#1](https://www.postgresql.org/message-id/CAMT0RQROjBXX0hFaYyr7Yzwc_VRmsg4CRd8Nm4FivXaQGztVrw%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAMT0RQR_PeqFzSsg_Sr7dXnHF43qTEn-LqBA-yZYGs-7y-p-5g%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Back-patching widens what existing grants on session_replication_role allow

An admin ran grant set on parameter session_replication_role to a non-superuser load role for faster DML. After a minor upgrade, that role, with only truncate privilege on a referenced table, can truncate it while other owners' tables still reference it. Today that needs truncate privilege on the referencing tables through cascade.

Message #2 asks to back-patch to all supported branches, so a grant made under the old meaning would change meaning silently. Truncate is a separate privilege from delete, so the comparison with delete in message #8 does not cover a role that holds truncate but not delete. The patch has no documentation change telling admins that the setting now also bypasses these checks.

**Question:** Which role holding set on session_replication_role but not superuser gains new power under this patch, and why is that safe to give out in a minor release?

<sub>fixable, level 1b, backpatchability; messages [#2](https://www.postgresql.org/message-id/CAMT0RQQy3hH0ygvkq4DHE6khGgWZ0kVu_DNpWAG_pLV96_E9SA%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAMT0RQR_PeqFzSsg_Sr7dXnHF43qTEn-LqBA-yZYGs-7y-p-5g%40mail.gmail.com); `security-2`</sub>

## Questions for the author

1. Which replication or load workflow needs truncate of a referenced table under replica role, and how does it differ from the case closed in [P9]?
2. What does the replica setup need that `add foreign key ... not valid` cannot give, and is the new constraint marked convalidated in pg_constraint?
3. What rule decides which DDL checks session_replication_role = replica may skip, so that check, not null and unique validation do not follow by the same argument?
4. Does the truncate bypass apply inside the logical replication apply worker, where a replicated truncate could orphan subscriber-local referencing rows?
5. Which documented promise does the current behaviour break, so that removing two errors in minor releases is justified?

## Touch points

- **WAL / replication / standby:** Logical replication apply workers run with session_replication_role = replica, so the truncate change may reach replicated truncate on subscribers.
- **catalog changes:** The proposal does not say whether pg_constraint marks the unchecked foreign key as validated.
- **SQL-standard conflict or non-standard syntax:** Plain add foreign key would skip the check of existing rows, beside the existing `not valid` spelling.
- **new GUC:** No new GUC, but session_replication_role grows from trigger firing to skipping DDL checks.
- **planner behaviour change:** The planner uses validated foreign keys for join estimates, and the constraint may be recorded as validated without a check.
- **pg_dump / restore round-trip:** A dump of a table with orphan rows under such a constraint fails at add foreign key on restore.
- **security / privileges:** A role with set on session_replication_role and truncate on the referenced table gains a new ability.
- **backpatchability:** Message #2 asks to back-patch to all supported branches, which removes two errors in minor releases.
- **maintenance burden ("who maintains this in 10 years"):** Tests of the setting inside tablecmds.c invite the same special case for other constraint types.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** `add foreign key ... not valid` already skips the scan of existing rows.

## Who would commit it, and when

- Owner: silent. Someone who knows foreign key DDL in tablecmds.c and the logical replication apply path. The author asked Álvaro for review in message #9, and no committer has replied on the design.
- Priority: needs design rethink; audience niche; review cost low. The patch is about 6 KiB in tablecmds.c plus tests, and the stated need is foreign key creation on very large tables after an initial copy. The entry has been in 7 CommitFests without a design reply, because the relation to `not valid`, the catalog state and the scope of the setting are open.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 11 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
