# Design pre-review: Fix slot synchronization with two_phase decoding enabled

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5664](https://commitfest.postgresql.org/patch/5664/) (Needs review); authors: Zhijie Hou (houzj)
- Thread: [Fix slot synchronization with two_phase decoding enabled](https://www.postgresql.org/message-id/flat/TYAPR01MB5724CC7C288535BBCEEE65DA94A72%40TYAPR01MB5724.jpnprd01.prod.outlook.com), started 2025-03-25
- Messages read: 100 of 100
- Patch prose read: reproduce_without_slotsync_HEAD_twophase.sh.txt, 0001-table-sync-injection-points.patch, v1-0001-Tap-test-to-reproduce-the-issue.patch, v1-0003-Sync-the-two_phase_at-field-of-a-replication-slot.patch, v1-0002-Display-two_phase_at-in-the-pg_replication_slots-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The HEAD fix is committed and the bug on 17 is real, so the goal is not in question. The decisive objection is prior-attempts-1: v19 makes a minor release reject `create subscription` statements that dumps from older 17.x contain. Messages #94 and #99 leave this open, and message #100 proposes a different design. The list must choose a design for 17 before code review of v19 is useful.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The v19 patches for 17 were not read, so the exact checks, the binary-upgrade exception and the pg_dump change are judged from the thread only. The message #100 idea has no patch to judge.

## The proposal, restated

**Problem.** After failover, a transaction prepared before two-phase decoding was enabled on a slot is not sent when it is committed on the promoted standby. The subscriber then fails with "prepared transaction with identifier ... does not exist". The cause is that the slot's two_phase_at is not synced to the standby, which affects users of logical replication who set both failover and two_phase.

**Proposed design.** On HEAD the fix syncs two_phase_at and shows it in pg_replication_slots, but that catalog change cannot be backpatched to 17. For 17, the v19 patch rejects failover and two_phase together at slot and subscription creation, except in binary upgrade mode. It lets ALTER SUBSCRIPTION enable failover later, once two-phase state is enabled and the slot's restart_lsn has passed two_phase_at, and pg_dump writes such subscriptions with failover = false.

**User-visible surface.**
- New pg_replication_slots column two_phase_at (pg_lsn), committed on HEAD only.
- CREATE SUBSCRIPTION fails when both failover and two_phase are true.
- pg_create_logical_replication_slot() and CREATE_REPLICATION_SLOT fail when both failover and two_phase are set.
- ALTER SUBSCRIPTION ... SET (failover = true) and ALTER_REPLICATION_SLOT fail while two-phase is pending or restart_lsn is before two_phase_at.
- pg_dump emits failover = false for subscriptions that have both options enabled.

**What the author claims.**
- The bug exists in 17 too, though it needs rarer steps to trigger there than on 18.
- A catalog change is not acceptable in a minor release because this is not a security issue.
- pg_upgrade can safely create slots with both options because it checks that slot changes are consumed and no prepared transactions exist.
- Restoring an old 17.x dump into a fixed 17.x does not matter, since the docs recommend binary replacement for minor upgrades.

**Thread state.** Amit Kapila committed the HEAD fix and a related assertion fix, and v19 for 17 has review from Shveta Malik, Ajin Cherian and Peter Smith. Sawada objects that dumps from older 17.x releases would fail to restore after the fix, and the last message proposes instead reporting failover as false until restart_lsn passes two_phase_at.

## 2. Prior attempts

### Revert addition of PGDLLEXPORT in PG_FUNCTION_INFO_V1 macro. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f52fd3c6d28a4380a5afc51ae6f774c91837a38](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f52fd3c6d28a4380a5afc51ae6f774c91837a38)
- **Date:** decided 2016-10-12; started 2016-10-12
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This turns out not to be as harmless as I thought: MSVC will complain if it sees an "extern" declaration without PGDLLEXPORT and then one with. (Seems fairly silly, given that this can be changed after the fact by the linker, but there you have it.) Therefore, contrib modules that have extern's for V1 functions in header files are falling over in the buildfarm, since none of those externs are marked PGDLLEXPORT. We might or might not conclude that we're willing to plaster those declarations with PGDLLEXPORT in HEAD, but in any case there's no way we're going to ship this change in the back branches. Third-party authors would not thank us for breaking their code in a minor release. Hence, revert the addition of PGDLLEXPORT (but let's keep the extra info in the comment). If we do the other changes we can revert this commit in HEAD. Per buildfarm.
- **Reason (quote):** "Third-party authors would not thank us for breaking their code in a minor release."
- **What is different this time:** P23 reverted a macro change that broke extension builds in back branches. This proposal breaks restore of older 17.x dumps through new option checks, so P23 is only a loose analogy.


## 3. Objections a reviewer is likely to raise

### 1. Minor release makes dumps from older 17.x releases fail to restore

fixable · level 1b, rubric item 6 · backpatchability, pg_dump / restore round-trip, SQL-standard conflict or non-standard syntax · `prior-attempts-1`

A user on 17.5 runs pg_dump for backup and the dump contains create subscription ... with (two_phase = true, failover = true). After a routine upgrade to the fixed 17.x, the restore stops with an error at that statement. The user finds this only when they need the backup.

[P23] records a revert because a change would break third-party code in a minor release. The same reason applies to a minor release that rejects SQL that older 17.x dumps contain. Amit raised the case in message #81 and Sawada in messages #83, #94 and #99. Messages #89 and #96 answer with the docs on binary replacement and on using the newer pg_dump. Message #99 notes that pg_dump is also used for plain backups, and message #100 proposes a different design, so the thread has not settled the point.

**Precedents.**
- [Revert addition of PGDLLEXPORT in PG_FUNCTION_INFO_V1 macro.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f52fd3c6d28a4380a5afc51ae6f774c91837a38) (Reverted, 2016-10-12)

Thread: [#81 (Amit Kapila, 2025-06-02)](https://www.postgresql.org/message-id/CAA4eK1KiGU5Zr_S2g8w-8dUm-PRiZc72BgMnLdS83u7sgnctcg%40mail.gmail.com), [#83 (Masahiko Sawada, 2025-06-04)](https://www.postgresql.org/message-id/CAD21AoCvCw42m6UxdFA%3D3FgUaz_Eb%2BLvaYFNTbagMQJc1tHphQ%40mail.gmail.com), [#89 (Nisha Moond, 2025-06-05)](https://www.postgresql.org/message-id/CABdArM4pp6HdiENq%2BE1WrcD%3DckH62kT2c6S%2BV%3DW1Qoxhtf4_jg%40mail.gmail.com), [#94 (Masahiko Sawada, 2025-06-11)](https://www.postgresql.org/message-id/CAD21AoAOkfLCiSamfBYVtFiXXLr3LVCdpBjvm6ssYA2O_ZVF%3Dw%40mail.gmail.com), [#96 (shveta malik, 2025-06-12)](https://www.postgresql.org/message-id/CAJpy0uA88qiY1EXmtppchFNxGEqxaB1X_z6YLEmpM_f5j1i9UQ%40mail.gmail.com), [#99 (Masahiko Sawada, 2025-06-16)](https://www.postgresql.org/message-id/CAD21AoCnurDW-L-qoiVD3XDQW6iPGSa3WgW5mz4zU-7dDz5Dpg%40mail.gmail.com), [#100 (shveta malik, 2025-06-19)](https://www.postgresql.org/message-id/CAJpy0uDuGz0B%2BzACCiWt9GOGA2jH_%2Br_zM%3D9c2h2MKacd%2BqYkQ%40mail.gmail.com)

**Question:** How does a dump taken by 17.5 pg_dump restore on the fixed 17.x, or why is breaking that restore acceptable in a minor release?

Counter-check: held. Messages #94 and #99 leave this open: binary replacement covers the upgrade path but not pg_dump backups taken on older 17.x. [P23] concerns extension build breakage, so it is at most a loose analogy and should not carry the argument.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Binary upgrade mode keeps creating slots with both options, and existing slot state on disk passes through the minor update unchanged. |
| WAL / replication / standby | **yes** | The fix changes slot sync to the standby and which logical slots may have failover enabled. |
| wire protocol | no |  |
| catalog changes | **yes** | The HEAD fix adds the two_phase_at column to pg_replication_slots, which is why it cannot go to 17. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | CREATE SUBSCRIPTION and pg_create_logical_replication_slot() start to reject an option combination that 17.0 to 17.5 accept. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump writes failover = false for subscriptions with both options, and older 17.x dumps fail to restore. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry targets 17 only and changes the behaviour of existing commands in a minor release. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The v19 checks exist only on the 17 branch, because HEAD fixes the bug by syncing two_phase_at. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does a dump taken by 17.5 pg_dump with failover = true and two_phase = true restore on the fixed 17.x without manual editing?
2. Which design does the CommitFest entry now track for 17: the v19 restriction, or the idea in message #100 of reporting failover as false?
3. After the minor update, how does a 17 user find which existing failover slots with two_phase are still unsafe, given that 17 does not show two_phase_at?
4. Under the message #100 idea, what happens to a slot the standby has already synced when the primary starts to report failover as false?
5. How does a user who restores a dump from the new pg_dump learn that failover was turned off for the subscription?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Amit Kapila committed the HEAD fix and reviews the 17 patch. The 17 change needs a committer who knows slot sync and who accepts the minor-release behaviour change.
- **Liability:** The owner signs for a behaviour change in a minor release and for checks that exist only on the 17 branch until 17 reaches end of life. This includes any reports of dumps that no longer restore.
- **Author follow-up:** The author posted versions up to v19 and moved the entry to the next CommitFest in August 2025. The brief records no message after June 2025.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Only 17 users who set both failover and two_phase are affected, and the trigger on 17 is rare. v19 has review, but the dump restore objection is open and message #100 proposes another design, so the 17 design must be agreed first.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) pg_dump silently drops failover for subscriptions with both options. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Existing 17.x slots with failover and two_phase stay exposed after the update. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-3` (fixable, level 2) Reporting failover as false in the view can drop already synced slots. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-1` Minor release turns valid CREATE SUBSCRIPTION and slot calls into errors. Refuted (not specific): Duplicate of prior-attempts-1. The severity would also be fixable, since message #100 already proposes a design that avoids the new errors.
- `standards-compat-3` Existing subscriptions keep a state the fixed release forbids. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same gap and adds that two_phase_at is not visible on 17.
- `replication-ondisk-2` Older 17.x dumps of subscriptions fail to restore on fixed 17.x. Refuted (not specific): Duplicate of prior-attempts-1.
- `maintenance-fit-1` A rule set that exists only in 17 for the rest of its support life. Refuted (no failure scenario): The scenario is a hypothetical future back-patch conflict, and messages #35, #55, #76 and #78 show tests for each new check, including the pg_upgrade case. Branch divergence follows from the no-catalog-change rule, which messages #5 and #87 weighed.
- `maintenance-fit-2` A minor release rejects dumps and configurations that 17.x accepted. Refuted (not specific): Duplicate of prior-attempts-1. Its point about existing subscriptions is covered by replication-ondisk-1.
- `demand-1` Minor-release restriction for a bug no 17 user has reported. Refuted (already handled): Message #5 weighs the restriction against replication breaking after promotion, and message #11 shows a prepare during the default copy_data = true table sync triggers it. Message #30 narrowed the restriction; the dump part is a duplicate of prior-attempts-1.

### Review angles

- prior-attempts: 1 objection(s). No precedent row proposes the same fix. Inside the thread, message #4 first raised the idea of showing failover as false, message #5 dropped it as hard to explain, and message #100 brings it back. Message #5 also cites an earlier catalog backpatch that was done only for a CVE.
- standards-compat: 3 objection(s). Objections were raised. The SQL-API-only ALTER_REPLICATION_SLOT path was not raised because the thread accepted it as a restriction in message #47.
- replication-ondisk: 3 objection(s). Not applicable.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. The new two_phase_at column shows an LSN of the same kind as restart_lsn and confirmed_flush_lsn, which pg_replication_slots already shows. The v19 changes only add restrictions, and the binary-upgrade exception applies only inside pg_upgrade, so no role gains a new capability.
- maintenance-fit: 2 objection(s). Hot-path cost is not a concern here: the checks run only at slot or subscription creation and when the slot or subscription is altered.
- already-exists: 0 objection(s). This is a bug fix for slot sync on Postgres 17, and no existing feature, extension or external tool closes the gap. The only existing workaround is to set failover = false by hand, and that is exactly what the patch enforces, so nothing from this angle applies.
- demand: 1 objection(s). The HEAD fix answers a real, reproduced bug and is out of scope here. The demand concern applies only to the 17 backpatch.

### Precedents retrieved

- P1: [Logical decoding of two-phase transactions](https://commitfest.postgresql.org/patch/944/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P2: [pgbench - break out initialization timing info by init phase](https://commitfest.postgresql.org/patch/1493/) (cf, Returned with feedback, 2018-03-07) - judged not relevant; reason summarised from its thread
- P3: [hostorder and failover_timeout for libpq](https://commitfest.postgresql.org/patch/1631/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P4: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant; reason summarised from its thread
- P5: [pgbench - - break out timing data for initialization phases](https://commitfest.postgresql.org/patch/1308/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P6: [Revert "Include information on buffer usage during planning phase, in EXPLAIN output."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19db23bcbda99e93321cb0636677ec9c6e121a2a) (revert, Reverted, 2020-04-03) - judged not relevant
- P7: [Patch to warn about any potential orphaned prepared transactions](https://commitfest.postgresql.org/patch/2505/) (cf, Withdrawn, 2020-04-17) - judged not relevant; reason summarised from its thread
- P8: [Prepare Transaction support for ON COMMIT DROP temporary tables](https://commitfest.postgresql.org/patch/1954/) (cf, Returned with feedback, 2019-02-01) - judged not relevant; reason summarised from its thread
- P9: [Synchronizing slots from primary to standby](https://commitfest.postgresql.org/patch/1961/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P10: [Allow synchronous_standby_names to be disabled after communication failure with all synchronous standby servers exceeds some timeout](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P11: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant
- P12: [Revert "Catalog NOT NULL constraints" and fallout](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9ce04b50e120275afbc03ca0b80839dde3da8308) (revert, Reverted, 2023-04-12) - judged not relevant
- P13: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P14: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant
- P15: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P16: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P17: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant
- P18: [New GUC prepared_statement_limit to limit memory used by prepared statements](https://commitfest.postgresql.org/patch/2241/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P19: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P20: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - judged not relevant
- P21: [Fix off-by-one in logical slot resource retention](https://commitfest.postgresql.org/patch/1084/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P22: [Revert gmake/make release note changes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=993c3961a4166a766c9b0a67701e9c82432550cc) (revert, Reverted, 2014-02-12) - judged not relevant
- P23: [Revert addition of PGDLLEXPORT in PG_FUNCTION_INFO_V1 macro.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f52fd3c6d28a4380a5afc51ae6f774c91837a38) (revert, Reverted, 2016-10-12) - used in the review
- P24: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 209 s; cost reported by the CLI: $7.17
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 101 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:34:21.350Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
